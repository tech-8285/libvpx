# VP9 透明视频：彩色画面与 Alpha 的配对显示分析

本文分析本仓库（libvpx）中透明度（alpha）信息从输入图像到解码输出的承载路径，
并评估播放端"仅凭两路解码返回顺序配对彩色帧与 alpha 帧"这一做法的可靠性。
所有结论均标注可追溯的代码位置。

---

## 1. 透明度信息在哪些接口有真实的数据承载

### 1.1 `vpx_image_t`：有 alpha 字段，但公开 API 造不出带 alpha 的图像

- `vpx/vpx_image.h:34` 定义了 `VPX_IMG_FMT_HAS_ALPHA (0x400)`，
  `vpx/vpx_image.h:103` 定义了 `VPX_PLANE_ALPHA (3)`，`planes[4]`/`stride[4]` 都预留了位置。
- 但 `vpx_img_fmt_t` 枚举（`vpx/vpx_image.h:38-51`）中**没有任何一个格式带 `HAS_ALPHA` 位**；
  `is_valid_img_fmt()`（`vpx/src/vpx_image.c:21-35`）只接受枚举内的格式，
  因此 `vpx_img_alloc()`/`vpx_img_wrap()` 无法创建带 alpha 平面的图像。
- `vpx_img_set_rect()`（`vpx/src/vpx_image.c:221-225`）和 `vpx_img_flip()`
  （`vpx/src/vpx_image.c:276-280`）中的 alpha 分支对公开 API 用户是**不可达代码**。

### 1.2 编码入口：alpha 平面在进入编码器时被丢弃

执行路径：`vpx_codec_encode()` → `encoder_encode()`（`vp9/vp9_cx_iface.c`，
实际转换在 `vp9_cx_iface.c:1567`）→ `image2yuvconfig()`。

- `image2yuvconfig()`（`vp9/vp9_iface_common.c:79-132`）只把
  `planes[VPX_PLANE_Y/U/V]` 拷进 `YV12_BUFFER_CONFIG`，**从不读取 `planes[VPX_PLANE_ALPHA]`**。
  即使调用者手工构造了一个带 alpha 平面的 `vpx_image_t`，alpha 数据也在此被静默丢弃。
- 内部帧缓冲 `YV12_BUFFER_CONFIG` 虽然声明了 `alpha_buffer`/`alpha_width`/`alpha_stride`
  （`vpx_scale/yv12config.h:42-49`），但分配时恒置 NULL/0
  （`vpx_scale/generic/yv12config.c:104-116`），全库再无任何读写点——是遗留字段。
- VP9 码流语法本身（profile 0–3，见 `vp9/decoder/vp9_decodeframe.c` 的帧头解析）
  没有 alpha 通道语法元素，编码器无处可写。

### 1.3 解码出口：返回图像的 alpha 平面恒为 NULL

执行路径：`vpx_codec_get_frame()` → `decoder_get_frame()`（`vp9/vp9_dx_iface.c:407-431`）
→ `vp9_get_raw_frame()`（`vp9/decoder/vp9_decoder.c:502`）→ `yuvconfig2image()`。

- `yuvconfig2image()`（`vp9/vp9_iface_common.c:51` 和 `:65`）显式执行
  `img->planes[VPX_PLANE_ALPHA] = NULL`（8bit 和 highbitdepth 两个分支都是）。
- VP8 解码接口同样置 NULL（`vp8/vp8_dx_iface.c:226`）。
- 仓库自带的工具链也不承载 alpha：`y4minput.c:1180` 置 NULL；
  `vpxenc.c`/`vpxdec.c`/`webmenc.cc`/`webmdec.cc` 中没有任何 alpha/BlockAdditional 处理。

### 1.4 真正承载 alpha 的层：容器（WebM），责任在封装/播放层

WebM 的透明视频方案是：把 alpha 平面当作一条**独立的 VP9 码流**（alpha 当亮度编码的
单色视频），复用时以 `BlockAdditional`（BlockAddID=1）附加在彩色 Block 上，
轨道级用 `AlphaMode=1` 声明。本仓库中只有 third_party 的 libwebm 具备该能力：

- 元素 ID：`third_party/libwebm/common/webmids.h:57-85`
  （`kMkvBlockAdditions`、`kMkvBlockAdditional`、`kMkvAlphaMode` 等）。
- 复用器接口：`third_party/libwebm/mkvmuxer/mkvmuxer.h:753`（`enum AlphaMode`）、
  `:770`（`SetAlphaMode`）、`:99`（`Frame::AddAdditionalData`）。
- **本仓库的 `webmenc.cc`/`vpxenc.c` 并未调用这些接口**，即自带工具链不产生也不消费
  带 alpha 的 WebM。

结论：**把 alpha 与彩色画面关联起来的责任完全在应用/封装层（muxer、demuxer、播放器），
不在 libvpx 编解码器。** libvpx 只提供两条"暗道"可供应用层自己建立关联（见 1.5）。

### 1.5 时间戳与帧级关联令牌的实际承载

- 编码侧：`vpx_codec_encode(ctx, img, pts, ...)` 的 `pts` 只用于码控
  （`vp9/vp9_cx_iface.c:1544` 转换为内部 tick），**不写入码流**；但编码器在输出包
  `VPX_CODEC_CX_FRAME_PKT` 中回显 `pkt.data.frame.pts`（`vp9/vp9_cx_iface.c:1684`）
  并通过 `get_frame_pkt_flags()`（`vp9/vp9_cx_iface.c:1367-1380`）设置
  `VPX_FRAME_IS_INVISIBLE`（`vpx/vpx_encoder.h:128`）。muxer 正是靠这两个字段给
  Block 打时间戳、决定哪些包不进 Block 时间轴。
- 解码侧：`vpx_codec_decode()` **没有 pts 参数**（`vpx/vpx_decoder.h:220-222`）。
  唯一可透传的帧级令牌是 `user_priv`：decode 时传入，经
  `ctx->user_priv`（`vp9/vp9_dx_iface.c:321`）在 `yuvconfig2image()` 时写入
  `img->user_priv`（`vp9/vp9_dx_iface.c:424`，字段定义 `vpx/vpx_image.h:117`），
  应用可在 `get_frame` 时读回。**这是本库提供的唯一帧级配对令牌。**

---

## 2. 仅凭两路解码返回顺序配对的可靠性评估

### 2.1 顺序配对成立的依据

- API 契约：`vpx/vpx_decoder.h:192-194` 与 `:237-238` 承诺
  "Frames produced will always be in PTS (presentation time stamp) order"。
  VP9 没有 B 帧式的显示重排，解码序即显示序。
- 每次 `vpx_codec_decode` 至多产出一帧：`decoder_get_frame()` 注释
  "we always return only 1 frame per decode call"（`vp9/vp9_dx_iface.c:411-413`）。
- 因此：**若**两路码流由同一帧序列、相同关键参数编码，且传输/解码无丢帧无损坏，
  两路 `vpx_codec_get_frame` 的第 N 次非空返回一一对应，顺序配对可靠。

### 2.2 破坏配对的具体路径（均可追溯到代码）

1. **编码器丢帧（最隐蔽）**：`vp9_rc_drop_frame()`（`vp9/encoder/vp9_ratectrl.c:627`，
   由 `VP8E_SET_DROPFRAME_THRESH` 使能，水位判断在 `vp9_ratectrl.c:530-555`）
   按各路自身码控状态独立丢帧。彩色路和 alpha 路内容不同、丢帧位置不同 →
   两路**显示帧总数不同**，顺序配对从此永久错位，且解码端无任何信号可察觉。
2. **解码错误/损坏后的 resync 抑制**：解码失败置 `need_resync`
   （`vp9/vp9_dx_iface.c:328-332`），此后 `decoder_get_frame()` 在
   `ctx->need_resync` 时直接返回 NULL（`vp9/vp9_dx_iface.c:423`），
   直到关键帧或 intra-only 帧才清除（`check_resync()`，`vp9/vp9_dx_iface.c:297-303`；
   关键帧清参考缓冲在 `vp9/decoder/vp9_decodeframe.c:2700-2704`）。
   两路各自独立进入/退出抑制 → 输出序列错位。
3. **不可见帧（show_frame=0，如 ARF）**：不出图
   （`vp9/decoder/vp9_decoder.c:515` `if (!cm->show_frame) return ret;`）。
   它本身不破坏"已显示帧序列"的 1:1（每个输入帧最终恰好显示一次），
   但若播放端按"喂进去的压缩包数"而不是"拿到的图像数"来配对，会立即错位。
4. **show_existing_frame**：不解码新帧、直接重显某个参考帧
   （`vp9/decoder/vp9_decodeframe.c:2664-2679`，强制 `show_frame = 1`）。
   输出图像数 ≠ 输入压缩帧数，同样惩罚"按压缩包计数"的配对方式。
5. **superframe**：一次 `vpx_codec_decode` 可解出多帧
   （`vp9/vp9_dx_iface.c:362-401` 的循环），但只有最后被 show 的帧能经
   `get_frame` 取出；且 `img->user_priv` 是**本次 decode 调用**的值，
   不是被显示帧最初进入解码器时的值——用 user_priv 做令牌时要注意这一偏移。

### 2.3 掉帧与画幅切换后何时可以恢复合成

**掉帧（压缩数据丢失）后**：VP9 帧间预测依赖参考帧（LAST/GOLDEN/ALTREF，
`ref_frame_map`）。丢帧后参考链断裂，解码器进入 need_resync，输出被抑制。
播放端恢复合成的最早时机是：**两路解码器各自重新出图之后（取较晚者）**——
即各自收到下一个关键帧或 intra-only 帧并成功输出。由于两路是独立编码的码流，
关键帧位置一般不对齐，恢复点取决于较晚恢复的一路；且恢复后必须**重新建立计数对齐**
（例如各自等关键帧输出后从该点重新配对），不能沿用丢帧前的序号。

**画幅切换后**：VP9 允许非关键帧改变分辨率
（`setup_frame_size_with_refs()`，`vp9/decoder/vp9_decodeframe.c:1555`，
调用点 `:2753`），参考帧会被缩放，解码器**不报错也不通知**，仅体现为
`vpx_image_t.d_w/d_h` 逐帧变化。两路独立编码时切换点不必同帧：
- 按输出序号的配对关系仍然成立（前提同 2.1）；
- 但合成前必须**逐帧核对两边的 d_w/d_h**，不一致时需缩放或等待另一路切换。

### 2.4 仅靠本库无法判定的条件（必须由容器或应用层补充）

1. **输出图像的显示时间戳**：解码 API 无 pts 概念。只能来自容器（WebM Block
   时间戳）或应用自己经 `user_priv` 透传。
2. **`get_frame` 返回 NULL 的原因**：不可见帧、superframe 中间帧、need_resync
   抑制、还是码流结束，接口不作区分（`VP9D_GET_FRAME_CORRUPTED`
   控制码和 decode 返回值只能部分推断损坏，无法回答"这一帧该不该显示"）。
3. **两路码流的丢帧/不可见帧模式是否一致**：编码器的丢帧与 show_frame 决策
   不写入任何解码侧可查询的元数据。
4. **WebM 中 BlockAdditional 与彩色 Block 的归属、AlphaMode 标记**：
   纯容器层信息，libvpx 完全看不到（本仓库的 `webmdec.cc` 也不解析这些元素）。
5. **两路的画幅切换是否发生在同一显示帧**：库内无任何跨流对齐信息。
6. **show_existing_frame 重显的是哪一帧的第几次显示**：解码侧不暴露原始帧序号，
   应用若需去重/对齐只能自行用 `user_priv` 编号。

### 2.5 实践建议

- 配对依据**不要**用"两路返回顺序"，而用帧级令牌：每路 decode 时把
  容器时间戳（或帧序号）作为 `user_priv` 传入，`get_frame` 时从
  `img->user_priv` 读回，按令牌配对。这是本库唯一支持的关联机制。
- 合成前逐帧检查两路 `d_w/d_h` 一致。
- 任一路 `vpx_codec_decode` 报错或长时间只返回 NULL 时，应视为对齐失效，
  等两路各自下一个关键帧输出后重新配对。
- 若可控编码端：两路关闭丢帧（`VP8E_SET_DROPFRAME_THRESH=0`）、对齐关键帧，
  可显著降低错位概率，但这属于库外约束，库本身不保证。
