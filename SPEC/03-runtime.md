# 03 · 运行时行为（Runtime）

> **规范性文档。** 本文件规定 Stub（或其配套 SDK）在运行时如何读取自身文件的 Archive。
> 目标：跨平台、可验证、失败可报告。

## 1. 能力要求

一个符合本规范的 Stub MUST 提供以下能力（可用 SDK 形式封装）：

1. **自身路径获取**：拿到“本 Overlay 文件在磁盘上的完整路径”（§2）。
2. **Trailer 定位**：从文件尾部找到并解析 Trailer（§3）。
3. **文件清单**：`listFiles() -> [ {name, size, offset, mode} ]`。
4. **读取单文件**：`readFile(name) -> bytes`。
5. **解包到目录**：`extractAll(dir) -> 落盘`，并写入 `mode` 权限。
6. **完整性校验**：校验 `trailer_checksum`、`header_checksum`、`archive_checksum`，按需校验 `data_checksum`（§4）。

## 2. 获取自身路径（平台 API）

| 平台 | 推荐 API | 说明 |
|:---|:---|:---|
| Linux | `/proc/self/exe` | 读符号链接，返回绝对路径 |
| Windows | `GetModuleFileNameEx` / `GetModuleFileNameW` | 取主模块全路径 |
| macOS | `_NSGetExecutablePath` | 返回 `CFStringRef`，需转 `char*` |

- 路径 MUST 为绝对路径；
- 读取方 MUST NOT 假设文件名为 `argv[0]`（`argv[0]` 可能是相对路径或被 shell 改写）。

## 3. 标准定位算法（从文件尾自下而上）

> 这是规范的核心：Trailer 永远位于文件**末尾**。读取方从尾部向上定位，无需知道文件总长即可起步，且对“尾部被追加”有天然容错。

```
1. 打开自身文件，取得总长 file_size。
2. 从 file_size - 8 处向前扫描（步长 1 字节，最多扫描到合理的上限，如 4 MiB 或文件头），
   寻找 4 字节 magic_repeat = 4F 56 4C 59。
3. 找到后，从该位置向前推算 Trailer 起始：
       trailer_start = magic_pos - trailer_size
   其中 trailer_size 需先从 Header 读取（此时已能定位到 Header 起点：
   Header 起点 = magic_pos - trailer_size，因为 Header 是 Trailer 的第一部分）。
   实现上：先读 magic_pos-? 处的 trailer_size 字段不可行，故规范约定——
   扫描命中 magic_repeat 后，其紧邻前方的字节即 TAIL 的 trailer_checksum 之前的内容；
   实际可操作序列为：
     a) 命中 magic_repeat 位置 m；
     b) 由 m 向上读取 HEADER 需要 trailer_size：trailer_size 字段位于 m - 8（TAIL 前 4 字节）
        与 HEADER 的偏移 48 处，二者一致，取其一；
     c) trailer_start = m - (trailer_size - 8)。
4. 校验 magic（Header 前 4 字节）== 4F 56 4C 59。
5. 校验 header_checksum（Header 前 52 字节）与 trailer_checksum（整个 Trailer 除自身）。
6. 校验 version：不支持的版本 MUST 报错，不得尝试继续解析。
7. 读取 archive_offset / archive_size / archive_size_raw / compression。
8. 从 archive_offset 读取 archive_size 字节（压缩后 Archive）。
9. 校验 archive_checksum。
10. 若 compression != 0，解压得到原始 Archive（长度必须 == archive_size_raw）。
11. 按 FILE_ENTRIES 读取目标文件：
      Raw：seek 到 archive_base + data_offset，读 data_size；
      TAR：按逻辑序号定位（§4.1）。
12. 校验 data_checksum（按需）。
```

> 定位只依赖**文件尾部**的魔数与 `trailer_size`，因此：
> - 即使 Archive 内部数据全错，只要 Trailer 完好，仍能正确解析清单；
> - 即使文件被其它数据追加在尾部之前（如 macOS 签名段位于 Stub 内部），只要 Trailer 仍在末尾，定位仍成立。

### 3.1 尾部追加的容错

macOS 代码签名可能在可执行段写入签名，**但不会改变文件尾部**（签名信息放在 Mach-O 头/段内，不追加到文件末尾）。因此：

- 生成器在 macOS 上 MUST 保证：签名发生在“Stub 段已定稿、Trailer 已追加”之前，或签名不改动文件尾部；
- 若某实现确实会在文件最末尾追加字节，则本规范**不覆盖**该场景（Trailer 必须仍是最后 8 字节含 magic_repeat 的块）。

## 4. 完整性校验（顺序）

读取方 MUST 按 `01-file-format.md` §7 的顺序校验：

1. `trailer_checksum`（Trailer 自洽）；
2. `header_checksum`（Header 字段可信）；
3. `archive_checksum`（压缩后 Archive 完整）；
4. 逐条目 `data_checksum`（读取某文件时，先定位、读取、再校验，不匹配 MUST 报错）。

任一校验失败 MUST 报告**具体哪一项失败**，不得静默吞掉。

## 5. 失败与降级

- 找不到 magic_repeat：报错“非 Overlay 文件或尾部损坏”。
- `version` 不支持：报错并提示“需要更新版本的读取器”。
- 解压失败 / 长度不符：报错“Archive 损坏”。
- 单文件校验失败：可继续读取其它文件，但 MUST 在 API 上暴露该文件的失败状态。

## 6. 与 SFX 的运行差异

- SFX：解压到临时目录再 exec 子进程；Overlay：**不产生子进程**，Stub 直接 `seek+read` 自身尾部，行为完全自定义。
- Overlay 的“解包”是**读取语义**，可选择仅内存读取或落盘（`extractAll`），不强制创建临时目录。
