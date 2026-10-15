# 02 · 生成流程（Generation）

> **规范性文档。** 本文件规定 Generator 如何把 `Stub + Payload` 合成为单个 Overlay 文件。
> 核心原则：**生成是纯字节操作，不依赖外部工具，且必须确定（same input ⇒ byte-identical output）**。

## 1. 输入

Generator 接受三类输入：

1. **Stub**：预编译的原生可执行程序文件（ELF / PE / Mach-O）。
2. **Payload**：文件集合，每项为三元组 `(逻辑路径, 内容, 权限)`。
   - 逻辑路径：相对路径，不含 `/` 开头、不含目录项。
   - 内容：任意字节。
   - 权限：POSIX 风格 `mode`（如 `0755`、`0644`）。
3. **选项**：`archive_format`（TAR / Raw）、`compression`（none / gzip / zstd / custom）。

## 2. 生成步骤（规范顺序）

Generator MUST 按如下顺序执行。偏离该顺序产生的文件不符合本规范。

### 步骤 1 · 收集与排序

1. 收集全部 `(逻辑路径, 内容, 权限)`。
2. **按逻辑路径的字典序（UTF-8 字节序）排序**——这是强制的确定性来源。
3. 记录 `file_count`。

> 排序使输出可复现，并让 `Raw` 格式的偏移计算稳定。

### 步骤 2 · 构建 Archive 字节流

1. 按 `archive_format` 把排序后的文件打包为 Archive 字节流：
   - **TAR**：标准 ustar，512 字节对齐；
   - **Raw**：按顺序直接拼接各文件内容。
2. 若 `compression != 0`，对整个 Archive 字节流执行一次压缩，得到“压缩后 Archive”。
3. 记录三个值：
   - `archive_offset = stub_len`；
   - `archive_size = 压缩后字节数`；
   - `archive_size_raw = 解压后（原始 Archive）字节数`；
   - `archive_checksum = CRC-64/XZ(压缩后 Archive 字节流)`。

> 注意：`archive_checksum` 校验的是**压缩后**的字节流（即真正写入文件的字节），而非解压后的逻辑流。读取方先校验压缩流，再解压，再逐条目校验 `data_checksum`。

### 步骤 3 · 计算每个条目的 `data_offset`

- **Raw**：`data_offset = Σ(前序文件 data_size)`，可直接静态计算。
- **TAR**：按解包后的逻辑序号定位（见 `01-file-format.md` §4.1），保证同一 Archive 内唯一且可寻址。
- 同时计算每个条目的 `data_checksum = CRC-64/XZ(原始文件内容)` 与 `mode`。

### 步骤 4 · 构造 Trailer

1. 依次写出 `HEADER`（§3.1 的 52 字节）、`FILE_ENTRIES`（§3.2，按排序顺序）、`TRAILER_TAIL`（§3.3）。
2. 计算 `trailer_size`（实际长度）。
3. 计算 `header_checksum`（前 52 字节的 CRC-64/XZ 低 32 位）填入 HEADER。
4. 计算 `trailer_checksum`（整个 Trailer 除自身外的 CRC-64/XZ 低 32 位）填入 TAIL。
5. 写入末尾 `magic_repeat`。

> 注意 HEADER 内的 `archive_offset / archive_size / ...` 等字段在校算 `header_checksum` 时已确定，只有 `header_checksum` 本身在校算时不参与。

### 步骤 5 · 拼接输出

1. 输出文件 = `Stub ‖ 压缩后 Archive ‖ Trailer`，逐字节顺序写出。
2. 不做任何额外填充或改动；输出即最终可执行文件。
3. 平台化后处理（可选）：
   - **Linux**：`chmod +x`；
   - **Windows**：保持 `.exe`；
   - **macOS**：`codesign` 签名（注意：签名可能改动尾部，须在签名后**重新校验**魔数与校验和仍成立，或在签名前完成 Overlay 合成、签名仅作用于 Stub 段——实现者选择并固定一种）。

## 3. 确定性（强制）

- 同一 `Stub + Payload + 选项` 必须产生**字节级相同**的输出。
- 为此 MUST：
  - 排序（步骤 1）；
  - TAR 与压缩器不写入时间戳、随机数、可变元数据（TAR 头 `mtime` 固定为 0，gzip 头 `mtime` 字段固定为 0，zstd 使用固定压缩参数）；
  - 字节操作全程可复现。
- 生成器 SHOULD 提供“自检”：生成后按 `03-runtime.md` 的定位与校验流程读回，确认 `header_checksum / trailer_checksum / archive_checksum` 全部通过。

## 4. 错误处理

Generator 遇到以下情况 MUST 拒绝并报错，不得产出半成品：

- Payload 中存在重名逻辑路径；
- 逻辑路径非法（`/` 开头、含目录项、含 `..`）；
- 选项非法（未知 `archive_format` / `compression`）；
- 压缩或打包失败。
