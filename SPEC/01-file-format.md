# 01 · 文件格式（File Format）

> **规范性文档。** 本文件定义 Overlay 文件的字节级布局：整体结构、Trailer 各字段、Archive 的两种格式（TAR / Raw）、压缩编码，以及全部校验和。
> 通用约定：除“UTF-8 文本”“gzip 流”“zstd 流”外，**所有字段一律小端序（little-endian）**；字符串均无长度前缀、由相邻字段界定。

## 1. 整体布局

一个 Overlay 文件由三段顺序拼接而成：

```
+--------+----------------------+------------------+
|  Stub  |       Archive        |      Trailer     |
| (PE/   |  (Payload 打包+压缩, |  (元数据, 定长   |
|  ELF/  |   可选压缩)          |   头部+变长列表) |
|  MachO)|                      |                  |
+--------+----------------------+------------------+
 0      stub_len            stub_len+arch_len  EOF
```

- **Stub**：原生可执行程序本体。加载器只读取其头与节表覆盖的范围；`Stub` 之后多出的字节（Archive + Trailer）对加载器透明，不影响进程启动。
- **Archive**：Payload 打包（TAR 或 Raw）后的字节流，可再经一次整体压缩。
- **Trailer**：追加在文件最末尾的元数据，是 Stub 定位与读取 Archive 的唯一依据。

### 1.1 不变量

- Trailer 的头部必须能从文件**尾部**被定位（见 `03-runtime.md` §2 的标准定位算法）。
- `Archive` 与 `Trailer` 均不得写入时间戳、随机数等可变数据（确定性要求，见 `02-generation.md`）。
- 文件总长 = `stub_len + archive_stored_len + trailer_len`。

## 2. 魔数（Magic）

Trailer 头部以 4 字节魔数开头：

| 字节 | 值 | ASCII |
|:---|:---|:---|
| b0 | `0x4F` | `O` |
| b1 | `0x56` | `V` |
| b2 | `0x4C` | `L` |
| b3 | `0x59` | `Y` |

即魔数字节序列为 `4F 56 4C 59`（`OVLY`），长度 4。

> 命名取 “OverLAY”。4 字节魔数的碰撞概率极低，足以防止把普通文件误判为 Overlay 文件，以及尾部被截断/损坏后的误定位。

## 3. Trailer 布局

Trailer 由三部分顺序组成：

```
+---------------------+---------------------+---------------------------+
|  HEADER  (固定 52B) |  FILE_ENTRIES (变长)|  TAILER_TAIL (固定 8B)    |
+---------------------+---------------------+---------------------------+
```

- `HEADER` 固定 52 字节。
- `FILE_ENTRIES` 共 `file_count` 项，每项变长（含文件名字节）。
- `TRAILER_TAIL` 固定 8 字节，位于整个文件的**最后 8 字节**。

### 3.1 HEADER（固定 52 字节）

| 偏移 | 大小 | 字段 | 说明 |
|:---|:---|:---|:---|
| 0  | 4 | `magic` | `4F 56 4C 59` |
| 4  | 2 | `version` | 规范主版本，当前为 **1**（u16，小端） |
| 6  | 2 | `flags` | 保留标志位，当前**必须为 0**（u16，小端）。未来可标记扩展能力，见 `05-extensions.md` |
| 8  | 2 | `archive_format` | 归档格式选择（u16）：`0` = TAR，`1` = Raw。当前仅这两个值；其它值保留、必须报错 |
| 10 | 2 | `compression` | 压缩编码（u16）：`0` = 无，`1` = gzip，`2` = zstd，`3` = 自定义 |
| 12 | 8 | `archive_offset` | Archive 起始偏移 = `stub_len`（u64，小端） |
| 20 | 8 | `archive_size` | **压缩后**的 Archive 字节数（u64，小端） |
| 28 | 8 | `archive_size_raw` | **解压后**的 Archive 字节数（u64，小端；未压缩时等于 `archive_size`） |
| 36 | 4 | `file_count` | 文件条目数（u32，小端） |
| 40 | 8 | `archive_checksum` | Archive **压缩后字节流**的 CRC-64/XZ（u64，小端） |
| 48 | 4 | `trailer_size` | 整个 Trailer 的字节数，含头部、文件清单与尾部（u32，小端） |
| 52 | 4 | `header_checksum` | 本 HEADER 前 52 字节（偏移 0–51）的校验和，低 32 位（u32，小端） |

> `header_checksum` 覆盖偏移 `0 .. 51` 共 52 字节，**不含**自身（偏移 52–55）。

### 3.2 FILE_ENTRIES（变长）

`file_count` 个条目，每个条目字段如下：

| 偏移(相对) | 大小 | 字段 | 说明 |
|:---|:---|:---|:---|
| 0 | 2 | `name_length` | 文件名字节数（u16，小端） |
| 2 | `name_length` | `name` | 相对路径，UTF-8 字节（不含目录项，不含 `/` 开头） |
| `2+name_length` | 8 | `data_offset` | 该文件内容在 **解压后 Archive** 中的偏移（u64，小端） |
| `2+name_length+8` | 8 | `data_size` | 文件原始大小（u64，小端） |
| `2+name_length+16` | 8 | `data_checksum` | 文件**原始内容**的 CRC-64/XZ（u64，小端） |
| `2+name_length+24` | 2 | `mode` | 权限，POSIX 风格（u16，小端），如 `0755`、`0644` |

- 条目的存储顺序见 `02-generation.md` §2（按逻辑路径字典序）。
- `data_offset` 的语义因 `archive_format` 而异，见 §4。

### 3.3 TRAILER_TAIL（固定 8 字节，位于文件最后 8 字节）

| 偏移(相对文件尾) | 大小 | 字段 | 说明 |
|:---|:---|:---|:---|
| 末尾-8 | 4 | `magic_repeat` | 魔数重复 `4F 56 4C 59`，用于自下而上的尾部扫描定位 |
| 末尾-4 | 4 | `trailer_checksum` | 整个 Trailer（不含本字段）的校验和，低 32 位（u32，小端） |

> `trailer_checksum` 覆盖从 Trailer 起点到 `magic_repeat` 之前的全部字节，**不含**自身。

## 4. Archive 数据格式

`archive_format` 决定 `Archive` 段内部的字节语义，以及 `FILE_ENTRIES` 中 `data_offset` 的解读方式。

### 4.1 格式 0：TAR（默认）

- Payload 打包为标准 **TAR** 归档（POSIX ustar，每文件一个条目，512 字节对齐）。
- 所有文件以**相对路径**存入；文件权限由 `FILE_ENTRIES` 中的 `mode` 携带。
- **`data_offset` 语义**：文件在 TAR 归档解包后的**逻辑序号位置**（便于 Stub 用统一定位方式访问）。具体语义由 Stub 实现决定；规范只要求：同一 Archive 内偏移**唯一且可寻址**。
- 优点：三平台可移植、`tar` 命令可直接调试、单文件自包含。

### 4.2 格式 1：Raw（定长切片）

- `Archive` 是一个连续的原始字节池：所有文件内容按 `FILE_ENTRIES` 顺序直接拼接，**每个条目固定长度 = `data_size`**。
- 偏移可**预先静态计算**：`data_offset = Σ(前序文件 data_size)`。
- **`data_offset` 语义**：文件内容在字节池中的**字节偏移**（相对解压后 Archive 起点）。
- 优点：字节级可精确定位、`seek + read` 即用、性能最优；适合“单文件 + 静态偏移”场景。

## 5. 压缩编码（`compression`）

对**整个 Archive 字节流**（TAR 或 Raw）做一次整体压缩；Trailer 永不压缩。

| 值 | 编码 | 说明 |
|:---|:---|:---|
| 0 | 无 | 原样存放，`archive_size == archive_size_raw` |
| 1 | gzip | 默认压缩，POSIX 兼容，三平台可解（`zlib` / `zlib-ng` / `gzip` CLI） |
| 2 | zstd | 更高压缩比与更快解压（`libzstd`） |
| 3 | 自定义 | 实现者自定义，须在 `05-extensions.md` 中登记格式与解码方法 |

- `archive_size` 始终指**压缩后**字节数；`archive_size_raw` 指**解压后**字节数。
- `compression == 0` 时二者必须相等。

## 6. 校验和

| 校验和 | 覆盖范围 | 算法 | 存储宽度 |
|:---|:---|:---|:---|
| `header_checksum` | HEADER 前 52 字节 | CRC-64/XZ，取低 32 位 | 4 |
| `trailer_checksum` | 整个 Trailer（不含自身） | CRC-64/XZ，取低 32 位 | 4 |
| `archive_checksum` | Archive **压缩后**字节流 | CRC-64/XZ | 8 |
| `data_checksum`（每条目） | 该文件**原始**内容 | CRC-64/XZ | 8 |

- 默认算法 **CRC-64/XZ**：多项式 `0xC96694822660B932`，反转多项式 `0xC96C5795D7870F42`，初值与终值 `0xFFFFFFFFFFFFFFFF`，refin/refout 均为 true。标准校验值：`CRC-64/XZ("123456789") = 0x995DC9BBDF1939FA`。
- 实现者也可以用 **SHA-256 前 8 字节**代替 CRC-64；规范允许通过 `flags` 中的保留位标记所用算法，但**默认必须支持 CRC-64/XZ**（见 `05-extensions.md`）。
- 对 64 位 `archive_checksum` / `data_checksum`，直接存 8 字节小端；对 32 位 `header_checksum` / `trailer_checksum`，取 64 位结果的**低 32 位**。

> 校验和用于**检测意外损坏/截断**，**不提供**对抗性完整性保护；需要防篡改时应另行签名（macOS 签名等），见 `03-runtime.md` §5 与白皮书 §7。

## 7. 完整性校验顺序（读取方）

读取方（Stub）应按以下顺序校验（详见 `03-runtime.md`）：

1. `trailer_checksum` —— 确认 Trailer 自身未被损坏；
2. `header_checksum` —— 确认 HEADER 字段可信；
3. `archive_checksum` —— 确认压缩后的 Archive 完整；
4. 逐条目 `data_checksum`（按需，读取某文件时校验其内容）—— 确认单文件内容。

## 8. 字段取值与约束（速查）

- `version == 1`（当前）。
- `archive_format ∈ {0, 1}`；`compression ∈ {0,1,2,3}`。
- `compression == 0 ⇒ archive_size == archive_size_raw`。
- 每个 `data_size`、`name_length`、`trailer_size`、`file_count` 非负。
- 所有条目 `data_offset + data_size ≤ archive_size_raw`。
- `trailer_size` 必须等于实际 Trailer 长度（`52 + Σ条目长度 + 8`）。
