# FLAC ReplayGain 全链路源码分析：从 PCM 响度分析到播放输出

> 分析对象：本仓库（Xiph FLAC 参考实现，分支 `task/57`，基线提交 `59fa340`）。
> 本文所有结论均来自源码静态阅读，未运行实测的判断统一在 [第 12 节](#12-尚未取得实测结果的判断) 标注。

---

## 1. 全景：三个阶段、三个共享库、一条责任链

ReplayGain 在 FLAC 工程中不是单一模块，而是分属三个职责清晰的阶段，代码全部位于 `src/share/`，命令行工具 `flac` 与 `metaflac` 只是编排者：

| 阶段 | 实现位置 | 职责 | 何时运行 |
|---|---|---|---|
| ① 响度/峰值分析 | `src/share/replaygain_analysis/replaygain_analysis.c`（LGPL，原始 ReplayGain 算法）+ `src/share/grabbag/replaygain.c` 包装层 | 从 PCM 统计响度增益（dB）与样本峰值（归一化线性值） | 编码时 `flac --replay-gain`；或事后 `metaflac --add/--scan-replay-gain` |
| ② 标签读写 | `src/share/grabbag/replaygain.c` + `src/libFLAC/metadata_object.c`（Vorbis Comment 块） | 把分析结果格式化为 5 个标签写入/读出 FLAC 元数据，不触碰音频帧 | 与①同批；解码时仅读取 |
| ③ 播放增益应用（synthesis） | `src/share/replaygain_synthesis/replaygain_synthesis.c` + `src/flac/decode.c` 编排 | 读标签 → 算线性缩放因子 → 浮点缩放 → 可选限幅 → 抖动/噪声整形 → 量化到目标位深并打包 | 仅 `flac -d --apply-replaygain-which-is-not-lossless` 解码时 |

关键架构事实：**libFLAC 本身（`src/libFLAC`）从不应用 ReplayGain**。增益应用只存在于 `flac` 命令行的解码输出路径，通过共享库完成。库用户拿到的解码 PCM 始终是原始样本。

数据流：

```
PCM (FLAC__int32, 源 bps)
  │  grabbag__replaygain_analyze()
  ├─► 响度路径: 归一化到 16-bit 等效 → 两级 IIR → 50ms RMS → 直方图 → 95 百分位 → gain(dB)
  └─► 峰值路径: 整数样本取绝对值最大 → /2^(bps-1) 归一化 → peak(线性, 0..1+)
        │
        ▼  写入 Vorbis Comment（5 个 REPLAYGAIN_* 标签）
════════ 元数据边界（标签可被第三方工具改写、删除、伪造）════════
        │  grabbag__replaygain_load_from_vorbiscomment()
        ▼
  gain(dB) + preamp + peak + 限幅模式
        │  grabbag__replaygain_compute_scale_factor() → scale(线性)
        ▼
  FLAC__replaygain_synthesis__apply_gain()
  浮点缩放 → 可选 tanh 硬限幅 → 抖动/噪声整形 → 舍入 → 移位截断 → 整数硬削波 → 字节打包
        ▼
  WAVE/AIFF/RAW 输出文件（target_bps = 源 bps 向上取整到字节）
```

---

## 2. PCM 响度统计（阶段①，响度部分）

### 2.1 入口与输入归一化

包装层入口 `grabbag__replaygain_analyze()`（`src/share/grabbag/replaygain.c:134`）接收 libFLAC 解码出的 `FLAC__int32` 交错声道缓冲，做两件互相独立的事：**喂给响度分析器**和**累加样本峰值**。

响度分析器内部按 16-bit 满量程刻度工作，因此先按源 bps 归一化（`replaygain.c:182-187`）：

- `bps == 16`：直接转 `flac_float_t`（即 `float`，单精度，见 `include/share/replaygain_analysis.h:45`）；
- `bps > 16`：乘以 `1 / 2^(bps-16)`（缩小）；
- `bps < 16`：乘以 `2^(16-bps)`（放大）。

即响度测量在"16-bit 等效整数刻度"上进行，与源位深解耦。每次最多 2048 样本一批送入 `AnalyzeSamples()`（`replaygain.c:137`），改善数据局部性。

### 2.2 采样率适配：滤波器表与 2 的幂降采样

`InitGainAnalysis()` → `CreateGainFilter()`（`replaygain_analysis.c:303`）从内置的 13 组系数表 `ReplayGainFilters[]`（`:156-263`）中选择与采样率匹配的一组，覆盖 8000 至 48000 Hz 的 13 个标准速率。

高于 48 kHz 的速率（如 88.2/96/192 kHz…，man 页 `man/flac.md:490` 列出至 604.8 kHz 的合法值）通过反复除以 2 降采样，直到落入表中速率（`:329-332`），`downsample` 记录 2 的幂倍数；`AnalyzeSamples()` 开头 `num_samples /= downsample`（`:428`）并在取输入样本时按该步长跳过。若降采样后仍无法匹配表项，`CreateGainFilter()` 返回 NULL，`ValidGainFrequency()` 为假，该速率被拒绝（见第 7 节兼容行为）。

### 2.3 两级 IIR 滤波（等响度曲线模拟）

每个声道串行通过两个 IIR（`AnalyzeSamples()`，`:466-470`）：

1. **10 阶 Yule 滤波**（`YULE_ORDER=10`，系数 `BYule/AYule`）——模拟人耳等响度响应；
2. **2 阶 Butterworth 滤波**（`BUTTER_ORDER=2`，`BButter/AButter`）——低频补偿/带限。

滤波器依赖历史样本，代码维护 `linpre/lstep/lout`（右声道对应 `r*`）三组带 MAX_ORDER 预缓冲的滑动缓冲，块间通过 `memmove` 保留尾部历史（`:487-490`、`:497-513`），保证分块调用与连续流等价。滤波运算中间量 `y` 用 `double`，存回时截断为 `float`（`filter()`，`:274-298`）。

### 2.4 50 ms RMS 窗口与 0.01 dB 直方图

- 窗口长度 `sampleWindow = rate * 50/1000`（`RMS_WINDOW_TIME=50`，`:109`、`:358`）；
- 窗口内累加两声道滤波后样本的平方和 `lsum/rsum`；窗口满时计算（`:481`）：

  ```
  E = (lsum + rsum) / totsamp * 0.5     /* 0.5 = 两声道平均 */
  val = 100 * 10 * log10(E + 1e-37)     /* 单位：0.01 dB 一格；1e-37 防 log10(0) */
  ```

  即直方图 `A[]` 的 bin 宽 0.01 dB（`STEPS_per_dB=100`），共 12000 bin 覆盖 120 dB（`MAX_dB=120`），越界钳到 `[0, 11999]`（`:483-484`）。每窗口占一个 bin 计数。

### 2.5 95 百分位与 PINK_REF 校准定标

`analyzeResult()`（`:519`）实现 ReplayGain 规范的"统计响度"：

- `upper = ceil(elems / 20)`（`:536`，即总窗口数的 5%，对应 `RMS_PERCENTILE=0.95`）；
- 从**最响的 bin 向低**扫描，累计计数直到覆盖 5% 窗口（`:538-541`），落点即第 95 百分位响度；
- 返回增益（`:543`）：

  ```
  gain(dB) = PINK_REF - i / 100        /* PINK_REF = 64.82，粉噪校准常数，:114 */
  ```

含义：比校准粉噪安静的素材得到正增益（需提升），更响的素材得到负增益（需衰减）。若一个完整 RMS 窗口都凑不齐（`elems==0`），返回哨兵值 `GAIN_NOT_ENOUGH_SAMPLES = -24601`（`replaygain_analysis.h:34`、`:529-530`）。

### 2.6 标题增益与专辑增益的累加语义

分析器维护两张直方图（`:137-138`）：

- `A[]`：当前标题（单曲）累积；`GetTitleGain()` 读取后把 `A` 合并进 `B` 并清零 `A`、重置滤波器历史（`:547-566`）——因此标题增益只反映"上次 `GetTitleGain()` 或 init 以来"的样本；
- `B[]`：整张专辑累积；`GetAlbumGain()` 只读不重置（`:569-573`）。

`grabbag` 层在 `flac` 编码多文件时，每文件结束调一次 `grabbag__replaygain_get_title()`，全部文件结束调一次 `grabbag__replaygain_get_album()`（`src/flac/main.c:1778` 与 `:577`）；metaflac 路径同理（`src/metaflac/operations.c:625-632`）。

参考响度常量 `ReplayGainReferenceLoudness = 89.0` dB SPL（`:104`）只用于写标签，见第 3 节。

---

## 3. PCM 峰值统计（阶段①，峰值部分）

峰值与响度在同一个函数内并行统计，但算法完全不同——**它是整数域的样本峰值，不做滤波、不加窗**。

`grabbag__replaygain_analyze()` 每个块维护 `block_peak`（`replaygain.c:139`）：

- 16-bit 快路径用 `abs()`（`:153` 等）；非 16-bit 路径用 `abs32()` 宏，对 `INT32_MIN` 做了保护（`:54`、`:196` 等），避免有符号整数绝对值溢出；
- 块结束后归一化（`:226-233`）：

  ```
  peak = block_peak / 2^(bps-1)        /* 满刻度 = 1.0；理论上峰值可等于 1.0 */
  ```

  该峰值同时与 `title_peak_`、`album_peak_` 取最大值。专辑峰值是跨文件最大值；`get_title/get_album` 取出后各自把对应累加器清零（`:238-250`）。

失真边界上的两个重要性质（源码可证，听觉影响属未实测，见第 12 节）：

1. **这是样本峰值（sample peak），不是真峰值（true peak）**。限带信号在 D/A 重建时可能产生超过样本峰值的 inter-sample 峰值，因此即使按该峰值防削波，模拟域仍可能过载——本实现不做过采样真峰值检测。
2. 峰值与响度在不同数值刻度上独立计算：峰值走原始整数绝对值，响度走滤波后浮点能量。标签里的 `*_PEAK` 与 `*_GAIN` 之间没有任何一致性校验。

---

## 4. 标签读写（阶段②）

### 4.1 五个标签与写入格式

标签名常量（`replaygain.c:71-75`），即 VorbisGain/vorbisgain 传统命名：

| 标签 | printf 格式（`:56-58`） | 示例 |
|---|---|---|
| `REPLAYGAIN_REFERENCE_LOUDNESS` | `%s=%2.1f dB` | `REPLAYGAIN_REFERENCE_LOUDNESS=89.0 dB` |
| `REPLAYGAIN_TRACK_GAIN` | `%s=%+2.2f dB` | `REPLAYGAIN_TRACK_GAIN=-7.32 dB` |
| `REPLAYGAIN_TRACK_PEAK` | `%s=%1.8f` | `REPLAYGAIN_TRACK_PEAK=0.98765432` |
| `REPLAYGAIN_ALBUM_GAIN` | `%s=%+2.2f dB` | （同上） |
| `REPLAYGAIN_ALBUM_PEAK` | `%s=%1.8f` | （同上） |

注意增益标签带显式正负号与单位后缀 ` dB`，峰值是无单位纯小数。写入前临时切换到 `"C"` locale（`append_tag_()`，`:109-115`），保证小数点不被本地化。五个标签最坏占用空间由 `GRABBAG__REPLAYGAIN_MAX_TAG_SPACE_REQUIRED = 190` 字节给出（`:62-69`）。

### 4.2 写入路径：先删后加、幂等替换

- 每个 store 函数先 `vorbiscomment_remove_entries_matching` 删除同名旧标签，再 append 新值（`:374-377`、`:388-398`、`:408-418`），因此重复执行结果幂等，"Existing ReplayGain tags will be replaced"（`man/metaflac.md:189`）。
- 文件没有 Vorbis Comment 块时，`store_to_file_pre_()` 自动新建一个并插到元数据链尾部（`:451-469`）。
- 写盘前 `FLAC__metadata_chain_sort_padding()` 并用 padding 就地写入（`store_to_file_post_()`，`:486-487`），可选保留文件 mtime/权限（`:479-499`）。
- 两套编排：
  - `flac --replay-gain`：编码循环内逐文件写 reference + track 两个标签（`main.c:1775-1786`），全部文件成功后再批量回写 album 两个标签（`main.c:575-589`）；
  - `metaflac --add-replay-gain`：独立两遍扫描（先全部解码分析，再一次性写齐 5 标签，`operations.c:618-645`）；`--scan-replay-gain` 只打印不写（`:642-644`）；`--remove-replay-gain` 展开成 5 个 `OP__REMOVE_VC_FIELD`（`options.c:669-682`）。

### 4.3 读取与匹配语义

`grabbag__replaygain_load_from_vorbiscomment()`（`:606`）：

- 字段匹配经 `FLAC__metadata_object_vorbiscomment_entry_matches()`，实现是 `FLAC__STRNCASECMP` 且要求 `=` 前长度严格相等（`src/libFLAC/metadata_object.c:1420-1426`）——**字段名大小写不敏感**（Vorbis Comment 规范要求 ASCII 小写大写等价），但不会匹配前缀名；
- 值解析 `parse_double_()`（`:582-604`）：截取 `=` 后内容到 32 字节栈缓冲，`strtod()` 在 C locale 下解析；`strtod` 允许跳过前导空白、容忍尾部垃圾（如 `" dB"` 后缀被自然忽略），完全无法解析才失败；
- reference 标签缺失时默认填 89.0（`:621`）；解析出的 `*reference` 在 `flac/decode.c` 中**只接收不使用**（`decode.c:1776` 后再无引用）——参考响度标签纯属信息性记录，增益数学不依赖它；
- 重复同名字段时取 `find_entry_from(..., offset=0)` 命中的**第一个**；
- 峰值解析后若 `< 0.0` 直接判失败（`:644-645`）；峰值为 0.0 本身合法，但会使峰值防削波失效（见 6.2）。

---

## 5. 增益模式、预增益与缩放因子（阶段③的前半段）

### 5.1 规格字符串与默认值

解码侧规格类型 `replaygain_synthesis_spec_t`（`src/flac/decode.h:33-39`）含五个量：是否应用、album/track 选择、限幅模式（NONE/PEAK/HARD）、噪声整形等级、预增益 dB。

默认值（`main.c:605-609`）：**0 dB 预增益、album 增益、HARD 限幅、LOW 噪声整形**，即 man 页所说的 `0aLn1`（`man/flac.md:706`）。

`--apply-replaygain-which-is-not-lossless[=spec]` 解析（`main.c:767-791`），格式 `[preamp][a|t][l|L][n0..3]`：

- **不带参数**：保持上述默认；
- **带参数**：先把限幅重置为 NONE、整形重置为 NONE（`:771-772`），再按出现的字母逐项打开。因此 `=3` 表示 3 dB 预增益、album、**无限幅、无抖动**（`man/flac.md:707`）；
- `a`=专辑增益，`t`=音轨增益；`l`=峰值限幅，`L`=硬限幅；`n0..3`=噪声整形等级；非法字符报 usage error。

### 5.2 album/track 的选择与自动回退

读取时按 `use_album_gain` 选择 `REPLAYGAIN_ALBUM_*` 或 `REPLAYGAIN_TRACK_*`。若首选的一对标签缺失/解析失败且 `strict=false`，递归以**另一种模式、strict=true** 重试（`replaygain.c:651-652`）——即缺专辑标签自动回退音轨标签，反之亦然（man 页 `flac.md:691-694` 明确承诺此行为）。strict 重试本身失败才整体返回 false。

### 5.3 缩放因子计算

`grabbag__replaygain_compute_scale_factor(peak, gain, preamp, prevent_clipping)`（`replaygain.c:657-669`）是全部电平决策的汇合点：

```c
gain += preamp;                              /* 预增益：dB 域线性叠加 */
scale = (float) pow(10.0, gain * 0.05);      /* dB→线性：10^(dB/20)，注意结果强转 float */
if (prevent_clipping && peak > 0.0) {
    max_scale = 1.0 / peak;                  /* 峰值防削波上限 */
    if (scale > max_scale) scale = max_scale;
}
```

- **预增益**没有任何范围检查地直接加在标签增益上（用户责任）；正预增益抬升输出电平与削波风险，负预增益压低电平；
- 注意 INFO 日志打印的是加预增益**之前**的标签 gain（`decode.c:1815`），预增益只体现在最终 scale 里；
- `(float)` 强转使 scale 只有单精度，这是链路上第二处单精度截断（第一处在分析滤波）。

调用处（`decode.c:1812`）把 `limiter == RGSS_LIMIT__PEAK` 映射为 `prevent_clipping=true`；HARD 模式不在此处理，而在样本循环内处理（第 7 节）。两种限幅可独立开关，互不为前提。

---

## 6. 两种限幅机制与失真边界

### 6.1 硬限幅（HARD，`L`，默认开启）：tanh 6 dB 软拐点

在 `FLAC__replaygain_synthesis__apply_gain()` 的样本循环内（`replaygain_synthesis.c:374-380`），信号先归一化到 `[-1, 1)`：

```c
if (hard_limit) {
    if (sample < -0.5) sample = tanh((sample + 0.5) / 0.5) * 0.5 - 0.5;
    else if (sample > 0.5) sample = tanh((sample - 0.5) / 0.5) * 0.5 + 0.5;
}
```

- `|sample| ≤ 0.5`（即不超过满刻度一半，约 -6 dBFS）时**完全透传**，零染色；
- 超出 ±0.5 后进入 `tanh` 软压缩段，渐进趋平于 ±1.0 而非直角削断——所以 man 页称之为"6dB hard limiter that kicks in when the signal approaches full-scale"（`flac.md:695-697`）。它消除的是硬削波式高次谐波，代价是对越界样本引入非线性谐波失真；拐点处一阶连续，失真量随超调量增大而增大。

### 6.2 峰值防削波（PEAK，`l`）：增益域的提前封顶

与硬限幅不同，PEAK 模式不碰样本，而是在 5.3 的公式里把 scale 钳到 `1/peak`。其边界语义：

- 保证 `输出峰值 ≈ 输入峰值 × scale ≤ 1.0`（在标签峰值真实可信的前提下）；
- 这是"宁可少增益也不削波"：当 ReplayGain 要求的提升会把峰值推过满刻度时，实际增益被压低，响度归一化目标让位于不削波；
- 两个失效条件值得注意：
  1. `peak > 0.0` 才生效（`replaygain.c:663`）——峰值标签缺失回退或解析为 0 时，防护静默失效；
  2. 防护信任标签里的峰值。若文件在打标签后被修改、或标签由第三方工具写错，`1/peak` 可能给出错误上限（该判断的实际后果未实测，见第 12 节）；
- 当 `peak > 1.0`（源素材本身已超数字满刻度，整数 PCM 下不常见，但浮点转码或外来标签可能出现），`max_scale < 1`，即使增益标签为 0 dB 也会被衰减。

### 6.3 最终整数硬削波：无条件安全网

无论是否启用上述任一限幅，量化后还有一道整数域硬削波（`replaygain_synthesis.c:386-389`），阈值取自 `hard_clip_factors_[target_bps]`（`:303-337`，即 `-2^(bps-1)`）：

```c
if (val64 >= -hard_clip_factor)      val32 = -(hard_clip_factor + 1);   /* 正满幅 2^(bps-1)-1 */
else if (val64 < hard_clip_factor)   val32 = hard_clip_factor;          /* 负满幅 -2^(bps-1) */
```

这是不对称二补码范围的直接钳位，属于直角硬削波，产生最严重的谐波失真。它是最后防线：NONE 限幅 + 大正预增益时，所有越界样本都在这里被削平。**输出永不溢出整数格式，但"不溢出"不等于"无失真"**——失真边界由 6.1/6.2 决定在哪一层被吸收。

三层防护的优先级可概括为：

```
PEAK 模式：在增益计算处封顶（改变全程电平，无样本级失真）
HARD 模式：在浮点样本处 tanh 压缩（仅越界段非线性失真）
安全网：  整数钳位（直角削波，最后手段，始终存在）
```

---

## 7. 抖动、噪声整形与位深转换（阶段③的后半段）

### 7.1 入口参数与位深约束

`apply_gain()`（`replaygain_synthesis.c:301`）的关键参数：`source_bps`（解码帧 bps）、`target_bps`、`scale`、`hard_limit`、`do_dithering`、`DitherContext`。

`flac` 编排处（`decode.c:1449-1462`）：

- `source_bps = bps`；`target_bps = bps + shift`，其中 `shift = (bps%8) ? 8-(bps%8) : 0`（`decode.c:1250`）——WAVE/AIFF/RAW 容器要求字节对齐位深，非字节 bps（如 12-bit）向上补齐到 16-bit；
- 前置校验（`decode.c:1792-1798`）：只接受 **4–24 bps** 的源；32-bit FLAC 明确拒绝应用 ReplayGain（合成器内部断言 `target_bps < 32`，`replaygain_synthesis.c:364`）；
- `do_dithering = (noise_shaping != NOISE_SHAPING_NONE)`（`decode.c:1460`）：n1/n2/n3 都启用抖动，n0 不启用。

### 7.2 浮点缩放与定点化

每个样本（`replaygain_synthesis.c:372`、`:346`）：

```c
multi_scale = scale / 2^(source_bps-1);   /* 把整数输入归一化到 [-1,1) 与 scale 合并，省一次乘法 */
sample = (double)input[i] * multi_scale;  /* double 精度处理 */
/* 可选 tanh 限幅 */
sample *= 2147483647.0;                   /* 抬到 32-bit 满刻度定点域 */
```

即整个增益/限幅/抖动在 `double` 精度下完成，最后才落到目标位深。

### 7.3 抖动上下文与幅度

`init_dither_context()`（`:203-225`）：

- 噪声整形系数表 `F[] = {F44_0, F44_1, F44_2, F44_3}`，即 16 阶 FIR，F44_0 全零；表内注释给出各阶的带权/不带权 SNR（如 F44_1：SNR(w)=4.84 dB，`:139`；F44_3：SNR(w)=15.38 dB，`:175`）——这些数值移植自 WaveGain，本仓库未复测；
- 抖动幅度查表：`indx = clamp(bits - 11 - shaping, 0, 9)`，`Dither = 0.01 * default_dither[indx] / 2^bits`（`:213-223`），幅度表 `{92,92,88,84,81,78,74,67,0,0}`。**推论（源码可证）**：24-bit 且任何整形等级下 `indx ≥ 9`，抖动幅度为 0——高位深下抖动名义上关闭，但整形 >0 时误差反馈环仍在运行；
- `Mask = (-1) << (32-bits)`（量化掩膜）、`Add = 0.5 * ((1<<(32-bits))-1)`（舍入偏置）。

随机源 `random_int_()`（`:83-110`）是两个互素周期多项式计数器的 XOR，总周期约 1.84×10¹⁹（`:78-81` 注释），确定性、无外部种子。

### 7.4 两条量化路径（`dither_output_()`，`:244-268`）

- **无抖动（n0）**：`ROUND64(d, Sum)` 直接舍入。`ROUND64`（`:227-238`）利用 double 魔数加 `Add` 实现就近舍入——即 n0 不是截断而是**无随机化的四舍五入**，量化误差与信号相关，低电平时可能产生失真模式；
- **有抖动且 shaping=0**：等分布（矩形 PDF）抖动 `random_equi_()`，并与上一个随机数做差（`:251-255`），构成简单的一阶高通结构。注意：flac CLI 在 n0 时传 `do_dithering=false`，因此这条分支在当前命令行组合下不可达，只对库的其他调用方有意义；
- **有抖动且 shaping=1/2/3**（`:257-263`）：三角 PDF 抖动（两个等分布随机数相加，`:121-124`），叠加 16 阶 FIR：
  - 用 `DitherHistory` 对抖动本身滤波；
  - 用 `ErrorHistory` 把量化误差经 `FilterCoeff` 反馈回输入端（典型误差反馈噪声整形），把量化噪声推往高频/人耳不敏感频段；
  - 历史按 `(i + LastHistoryIndex) % 32` 循环索引，块间通过 `LastHistoryIndex` 保持连续（`:426`），分块处理不产生边界爆音。

### 7.5 移位、打包与无符号偏移

```c
val64 = dither_output_(...) >> conv_shift;   /* conv_shift = 32 - target_bps，:338、:383 */
/* 整数硬削波（6.3）*/
if (unsigned_data_out) uval32 ^= (1 << (target_bps-1));   /* 有符号→无符号偏置，:391-393 */
/* 按 little/big endian 写入 8/16/24-bit 字节（:395-423），支持交错多声道 */
```

返回值是写出的字节数 `wide_samples * channels * target_bps/8`（`:428`）。目标位深只支持 8/16/24 三档字节对齐输出（switch 无 default，其他 bps 断言拦截，`:365`）。

### 7.6 不应用 ReplayGain 时的位深处理

作为对照：未启用增益时，子字节对齐只做纯左移（`decode.c:1443-1446`），其余走直通 memcpy/截断路径（`:1465` 起），**无抖动、无整形、无任何电平改变**，保持无损解码语义。

---

## 8. 输出电平与失真边界的合成公式

把整条链路对单个样本 `x`（源整数）写成一条式子：

```
scale   = min( 10^((gain_tag + preamp)/20),  1/peak_tag )     ← 仅 PEAK 模式有 min；peak>0 才生效
x_n     = x / 2^(source_bps-1) * scale                        ← double 域
若 HARD: x_n = L_tanh(x_n)         （仅 |x_n|>0.5 段，tanh 软压缩）
x_q     = Q_ns( x_n * (2^31-1) + dither )                     ← Q_ns = 舍入+噪声整形/掩膜到 target_bps
x_out   = clip_int(x_q, [-2^(b-1), 2^(b-1)-1])               ← 无条件硬削波
```

输出电平由**三个独立旋钮的代数和**决定：标签增益（响度分析的产物）、预增益（用户加性偏移）、峰值封顶（只减不增的安全钳）。失真边界则按"软的优先、硬的兜底"分层：

| 情形 | 电平变化 | 失真来源 |
|---|---|---|
| PEAK 封顶生效 | 全程增益被压低 | 无新增失真（线性衰减），但响度目标未达成 |
| HARD 且样本越 ±0.5 | 越界段被 tanh 压缩 | 软拐点非线性谐波，无直角削波 |
| NONE 限幅且样本超满幅 | 增益全额应用 | 整数硬削波（最严重）+ 量化误差 |
| 任何降位深（target_bps < 32 定点域） | — | n0：相关舍入误差；n1-3：被整形的随机化量化噪声 |
| 抖动幅度为 0（如 24-bit） | — | 无随机化收益，仅剩误差反馈（shaping>0 时） |

另需记住：分析与合成在**不同精度**下工作——分析滤波是 `float` 单精度、scale 中间值也强转 `float`，而合成样本处理是 `double`；标签本身把增益截断到 0.01 dB、峰值截断到 8 位小数。这三处量化共同构成"分析值→标签→重放值"之间的数值漂移。

---

## 9. 缺失或异常标签的兼容行为（汇总）

| 场景 | 源码行为 | 位置 |
|---|---|---|
| 无 Vorbis Comment 块（写入时） | 自动新建 VC 块插入元数据链 | `replaygain.c:451-469` |
| 首选 album/track 标签缺失 | 自动回退另一类（strict 递归重试）；两类都没有才失败 | `replaygain.c:651-652` |
| 完全无可用增益/峰值标签 | WARNING 后该流**不应用增益、继续解码**（非致命）；`--warnings-as-errors` 时升级为中止 | `decode.c:1777-1783`、`man/flac.md:710-712` |
| 标签值无法 `strtod` 解析 | 同上，视为加载失败 | `replaygain.c:640-643` |
| 峰值 `< 0` | 加载失败，走回退/警告路径 | `replaygain.c:644-645` |
| 峰值 `= 0.0` | 加载成功，但 PEAK 防削波因 `peak > 0.0` 条件**静默不生效** | `replaygain.c:663` |
| 增益超出 `[-90, +90] dB` | WARNING"gain value doesn't make sense"，不应用增益（防浮点溢出的合理性闸） | `decode.c:1800-1808` |
| reference 标签缺失 | 默认 89.0；该值不参与任何增益计算 | `replaygain.c:621`、`decode.c:1776` |
| 字段名大小写不同（如 `replaygain_track_gain`） | 正常匹配（大小写不敏感，`=` 前长度须相等） | `metadata_object.c:1425` |
| 值带尾部垃圾/单位/空白 | `strtod` 容忍，前缀数字部分生效 | `replaygain.c:598-603` |
| 同名字段重复 | 取第一个匹配项 | `metadata_object.c:1429-1433` |
| 源 bps 为 32 | 拒绝应用增益，WARNING 后直通解码 | `decode.c:1792-1798` |
| 源 bps 超出 4–24 或为 0 | 同上拒绝 | `decode.c:1784-1798` |
| 分析时采样率不被支持 | 编码/metaflac 直接 ERROR 中止（不产生标签） | `encode.c:1689-1692`、`operations.c:597-600` |
| 分析时声道数非 1/2 | 直接 ERROR 中止 | `encode.c:1685-1688`、`operations.c:601-604` |
| 素材太短，凑不齐一个 50ms RMS 窗口 | 增益得到哨兵 `-24601` dB 并**照常写入标签**（无二次校验）；播放侧再被 ±90 dB 闸门拦截而不应用 | `replaygain_analysis.h:34`、`:529-530`；拦截见 `decode.c:1801` |
| 多文件专辑参数（速率/bps/声道）不一致 | metaflac 显式检查并拒绝；**flac 编码侧无对应检查**——`InitGainAnalysis` 仅首文件调用一次（`encode.c:1693-1698`），分析器是全局单例且滤波器按首文件速率固化，后续异速率文件仍用旧滤波器分析，结果静默错误（man 页 `flac.md:487-489` 以"All input files must have the same..."把一致性责任留给用户）；单文件内部若帧间参数变化，grabbag 解码回调会置 error（`replaygain.c:270-281`） | `operations.c:583-595`、`encode.c:1693-1698` |
| 帧间 bps 不一致且开启续错解码 | ReplayGain 开启时直接 ABORT（无法安全续算） | `decode.c:1291-1294` |

---

## 10. 三层责任划分（响度 / 标签 / 播放）

| 责任 | 归属模块 | 保证什么 | 不保证什么 |
|---|---|---|---|
| **响度测量** | `replaygain_analysis.c` + grabbag 包装 | 按 ReplayGain 1 规范（IIR 等响度 + 50ms RMS + 95 百分位 + 粉噪校准）给出 dB 增益与样本峰值；确定性、与容器无关 | 不保证真峰值；不保证跨实现数值逐位一致（float 精度、系数表版本）；极短素材产出无意义哨兵值 |
| **标签承载** | grabbag + libFLAC 元数据 | 标准化 5 字段、幂等替换、locale 安全、大小写不敏感读取、album/track 回退 | 不校验标签与音频的实际一致性；标签可被任意外部工具删除/篡改；reference 字段仅文档性 |
| **播放应用** | `replaygain_synthesis.c` + flac 解码编排 | 按标签+用户规格决定线性电平；提供峰值封顶/tanh 软限幅/整数削波三道防线；抖动+噪声整形控制降位深失真；异常标签时安全回退到无损直通 | 不改变 libFLAC 解码结果本身；非无损（man 页与运行时 WARNING 双重声明，`decode.c:1816`）；不支持 32-bit 输出；信任标签峰值 |

一句话边界：**分析层只产数字，标签层只搬数字，播放层才对声音负责**；播放层的所有保护都以标签可信为前提，且任何保护都无法把"位深降低"变成无损。

---

## 11. 实测覆盖现状

仓库内与 ReplayGain 相关的自动化测试只有 `test/test_replaygain.sh`：

- 用 `src/test_streams/main.c` 生成的各采样率测试音（`rpg-tone-*.wav`，`test_streams/main.c:1302`、`:1474+`），跑 `metaflac --scan/--add-replay-gain`，断言 13 个基速及 48k 谐波速率下的**增益 dB 值**（`test_replaygain.sh:92-141`，期望值如 44100 Hz → -14.17 dB）；
- 即只覆盖**阶段①的响度算法与阶段②的标签写入/列出**；
- `grep` 全 `test/` 目录，**没有任何用例调用 `--apply-replaygain-which-is-not-lossless`**：缩放因子、两种限幅、抖动/噪声整形、位深打包、异常标签回退等阶段③行为零自动化覆盖；
- 峰值统计的数值正确性同样没有断言（测试只 grep `GAIN=` 标签）。

---

## 12. 尚未取得实测结果的判断

以下结论由源码逻辑推出，本次未构造信号实际运行验证，引用时应保持该限定：

1. **inter-sample 削波风险**：样本峰值防削波不能保证 D/A 重建后不削波——源码确无真峰值检测，但实际限带素材的超调量未测。
2. **tanh 限幅的实际失真指标**（THD、主观听感、拐点平滑度）：代码只给出曲线公式，未做谐波分析。
3. **噪声整形系数表的 SNR 数值**（F44_1/2/3 注释中的 4.84/10.06/15.38 dB 等）：移植自 WaveGain 的历史数据，本仓库构建中无复测。
4. **24-bit 下抖动幅度为 0 的实际可闻影响**：`default_dither[9]=0` 是源码事实，但误差反馈环单独运行时的噪声底表现未测。
5. **单精度（float）截断在极端长素材/极端增益下的累积误差**：分析滤波器与 scale 强转的数值漂移量级未测。
6. **错误/过期峰值标签的后果**：PEAK 模式按不可信标签算出错误 `1/peak` 的实际输出未测；仅确认代码无一致性校验。
7. **`-24601` 哨兵标签的端到端行为**：源码显示它会被写入、再在播放侧被 ±90 dB 闸门拦截，但未实际跑通"极短文件→标签→解码"全链。
8. **n0 就近舍入（无抖动）在低电平段的失真模式**，以及库调用方使用"dithering+shaping=0"等分布分支时的效果：CLI 不可达该分支，无测试。
9. **非字节对齐 bps（如 12/20-bit）经 `shift` 补到 16/24-bit 输出时的实际电精度**：代码路径明确，但未做样本级比对。
10. **locale 与畸形标签的容错实测**：逗号小数、超长值（32 字节截断）、多 `=` 等输入的解析结果未逐一实测，仅依据 `strtod`/`safe_strncpy` 语义推断。
11. **flac 编码异参数多文件的静默错误量级**：源码确认缺少跨文件速率/bps/声道一致性检查（与 metaflac 不同），但混用参数时专辑/标题增益的实际偏差未实测。

---

## 13. 关键源码索引

| 主题 | 文件:行 |
|---|---|
| 分析器接口与参考响度/哨兵常量 | `include/share/replaygain_analysis.h:34,47` |
| 滤波器系数表、速率适配、降采样 | `src/share/replaygain_analysis/replaygain_analysis.c:104,156-263,303-338` |
| RMS 窗口、直方图、95 百分位、PINK_REF | 同上 `:108-114,480-485,519-544` |
| title/album 直方图语义 | 同上 `:547-573` |
| grabbag 输入归一化与样本峰值 | `src/share/grabbag/replaygain.c:134-236` |
| 标签名/格式/locale/写入与回读 | 同上 `:56-75,93-121,353-421,582-655` |
| 缩放因子、预增益、峰值封顶 | 同上 `:657-669` |
| 抖动上下文、幅度表、随机源 | `src/share/replaygain_synthesis/replaygain_synthesis.c:83-124,203-225` |
| 两种抖动路径与 16 阶整形 | 同上 `:244-268` |
| 增益应用主循环、tanh、硬削波、打包 | 同上 `:301-429` |
| CLI 规格解析与默认值 | `src/flac/main.c:605-609,767-791` |
| 解码侧标签加载、校验闸、scale 装配 | `src/flac/decode.c:1774-1820` |
| 解码侧 apply_gain 调用与无损对照路径 | `src/flac/decode.c:1250,1443-1462` |
| 编码期分析编排 | `src/flac/encode.c:1684-1698,2186-2192`；`src/flac/main.c:575-589,1775-1786` |
| metaflac add/scan/remove | `src/metaflac/operations.c:557-650`；`src/metaflac/options.c:663-682` |
| VC 字段大小写不敏感匹配 | `src/libFLAC/metadata_object.c:1420-1426` |
| 行为规格（man 页） | `man/flac.md:365,673-712`；`man/metaflac.md:186-205` |
| 现有测试覆盖边界 | `test/test_replaygain.sh:92-141` |
