# 04 · 一致性测试（Conformance）

> **规范性文档。** 本文件给出判断“一个生成器 / 一个 Stub 是否符合 Overlay v1 规范”的判据。
> 配套材料：`test-vectors/`（标准测试向量）与 `expected/`（预期解析结果）。

## 1. 一致性等级

| 角色 | 最小一致性（Level A） | 完全一致性（Level B） |
|:---|:---|:---|
| **Generator** | 能产出 `trailer_checksum / header_checksum` 全部自洽的 Overlay 文件（TAR + 无压缩） | 支持全部 `archive_format`、`compression`、确定性输出、自检读回 |
| **Stub / SDK** | 能定位并解析 Trailer、`readFile` 成功（TAR + 无压缩） | 支持全部格式与压缩、逐项 `data_checksum` 校验、`extractAll` 写 `mode` |

> Level A 是“能互读最小集”；Level B 是“全特性可用”。

## 2. 标准测试向量

`test-vectors/` 中 MUST 至少包含（由 Generator 用**固定**输入生成，保证可复现）：

| 向量 | archive_format | compression | 特点 |
|:---|:---|:---|:---|
| `minimal.ovl` | TAR | 0 | 单文件、最小体积 |
| `tar-gzip.ovl` | TAR | gzip | 默认压缩 |
| `raw.ovl` | Raw | 0 | 定长切片 |
| `corrupt-trailer.ovl` | 任意 | 任意 | 故意破坏 `trailer_checksum`（负例） |

每个向量 MUST 与 `expected/<name>.json` 成对提供，JSON 至少含：

```json
{
  "name": "tar-gzip",
  "file_size": 123456,
  "trailer_offset": 123400,
  "archive_offset": 512,
  "archive_size": 122888,
  "archive_size_raw": 131072,
  "files": [
    {"name": "a.txt", "size": 100, "data_offset": 0, "mode": "0644", "sha256": "..."}
  ],
  "expect": "ok | reject:<reason>"
}
```

## 3. 判据（Pass/Fail）

对每一对 `(vector, expected)`，测试程序 MUST 通过以下断言才判为 **PASS**：

1. **定位**：能从文件尾定位到 Trailer，且 `trailer_offset` 与 `expected.trailer_offset` 一致。
2. **魔数**：Header 前 4 字节与末尾 4 字节均为 `4F 56 4C 59`。
3. **校验和**：`header_checksum` 与 `trailer_checksum` 通过；`archive_checksum` 通过。
4. **文件集**：`listFiles()` 返回的 `(name, size, mode)` 集合与 `expected.files` 完全一致（顺序按 §2 排序）。
5. **内容**：`readFile(name)` 的字节 SHA-256 与 `expected.files[].sha256` 一致。
6. **负例**：`corrupt-*` 向量 MUST 被**拒绝**并报告对应失败项（`expect` 以 `reject:` 开头）。

任一断言失败即 **FAIL**，并报告失败的步骤号。

## 4. 跨实现互操作性（Interop）

- **Generator ↔ Stub 交叉**：由实现 A 的 Generator 产出的文件，MUST 能被实现 B 的 Stub 正确读取（Level B 之间）。
- 测试方法：对每个 `test-vectors/*.ovl`，运行**每个候选 Stub**，结果 MUST 与 `expected` 一致；对每个候选 Generator 的产物，MUST 能被每个候选 Stub 读取。
- 这是规范“语言无关、可互操作”主张的可执行证明。

## 5. 确定性测试

- 同一输入连续生成 N 次（N ≥ 2），输出字节 MUST 完全一致（`02-generation.md` §3）。
- 断言：`sha256(out1) == sha256(out2) == ...`。

## 6. 报告格式

一致性测试工具 MUST 输出结构化结果：

```
[PASS|FAIL] <vector>  step=<定位|魔数|header|trailer|archive|files|content|neg|determinism>
```

任一 FAIL 即整体不合规；全部 PASS 方可声明“符合 Overlay v1”。
