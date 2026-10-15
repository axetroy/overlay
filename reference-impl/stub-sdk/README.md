# 读取 SDK（Stub SDK）

> 占位骨架。按语言子目录存放：`stub-sdk/<lang>/`。当前仓库不内置实现代码。

## 职责

在运行时从“自身 Overlay 文件尾部”读取并解包 Archive。行为 MUST 遵循 `SPEC/03-runtime.md`：

- 获取自身路径（平台 API，`03-runtime.md` §2）；
- 从文件尾自下而上定位 Trailer（§3 标准算法）；
- 提供 `listFiles / readFile / extractAll`；
- 按 `01-file-format.md` §7 的顺序校验 `trailer_checksum → header_checksum → archive_checksum → data_checksum`；
- 失败时**明确报告**哪一项失败（§5）。

## 各语言子目录的 `README.md` 必备字段

| 字段 | 说明 |
|:---|:---|
| 语言 / 最低版本 | 如 “Rust ≥ 1.70” |
| 构建 | 一条命令即可复现 |
| 一致性等级 | Level A / B（`SPEC/04-conformance.md` §1） |
| 支持矩阵 | 能读哪些 `archive_format` / `compression`；不支持者 MUST 报错而非误读 |
| 自检 | 对 `test-vectors/` 的运行结果（PASS/FAIL 摘要） |

## 添加实现

在 `stub-sdk/` 下建 `<lang>/` 子目录并填写上表；对 `test-vectors/` 全部向量运行并保留结果，须通过 `SPEC/04-conformance.md` §3 判据（含负例拒绝）。
