# 生成器（Generator）

> 占位骨架。按语言子目录存放：`generator/<lang>/`。当前仓库不内置实现代码。

## 职责

把 `Stub + Payload + 选项` 合成为单个 Overlay 文件。流程 MUST 遵循 `SPEC/02-generation.md`：

- 按逻辑路径字典序排序；
- 按 `archive_format`（TAR / Raw）打包；
- 按 `compression`（none / gzip / zstd / custom）整体压缩；
- 计算 `data_offset`、各校验和；
- 构造 Trailer 并拼接输出；
- 保证**确定性**（same input ⇒ byte-identical output）。

## 各语言子目录的 `README.md` 必备字段

| 字段            | 说明                                                                        |
| :-------------- | :-------------------------------------------------------------------------- |
| 语言 / 最低版本 | 如 “Go ≥ 1.21”                                                              |
| 构建            | 一条命令即可复现                                                            |
| 一致性等级      | Level A / B（`SPEC/04-conformance.md` §1）                                  |
| 支持矩阵        | `archive_format` ∈ {TAR, Raw}？`compression` ∈ {none, gzip, zstd, custom}？ |
| 自检            | 生成后读回校验的开关/命令                                                   |

## 添加实现

在 `generator/` 下建 `<lang>/` 子目录并填写上表；产物须通过 `SPEC/04-conformance.md` §3 的判据与确定性测试。
