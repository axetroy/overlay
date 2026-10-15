# 标准测试向量（Test Vectors）

> 本目录是 `SPEC/04-conformance.md` 的配套材料：一组**固定输入**生成、可复现的 Overlay 文件，及其预期解析结果。它是 Overlay“语言无关、可互操作”主张的**可执行证明**。

## 目标向量矩阵（SPEC/04 §2 的最小集）

`test-vectors/` 中 MUST 至少包含（由 Generator 用**固定**输入生成，保证可复现）：

| 向量 | archive_format | compression | 特点 | 状态 |
|:---|:---|:---|:---|:---|
| `minimal.ovl` | TAR | 0 | 单文件、最小体积 | ⏳ 待参考实现落地 |
| `tar-gzip.ovl` | TAR | gzip | 默认压缩 | ⏳ 待参考实现落地 |
| `raw.ovl` | Raw | 0 | 定长切片 | ✅ 已提供，自校验通过 |
| `corrupt-trailer.ovl` | 任意 | 任意 | 故意破坏 `trailer_checksum`（负例） | ✅ 已提供 |

## 目录结构

```
test-vectors/
├── README.md            # 本文件
├── raw.ovl              # Raw + 无压缩（占位 Stub，已自校验通过）
├── corrupt-trailer.ovl  # 负例：篡改 raw.ovl 的 trailer_checksum
├── minimal.ovl          # 最小文件（TAR + 无压缩 + 单文件）—— 待参考实现落地
├── tar-gzip.ovl         # TAR + gzip —— 待参考实现落地
└── expected/
    ├── raw.json
    ├── corrupt-trailer.json
    ├── minimal.json      # 待生成
    └── tar-gzip.json     # 待生成
```

> 当前已提供且**自校验通过**的是 `raw.ovl` 与其负例 `corrupt-trailer.ovl`（见下节“已提供向量”）。`minimal.ovl` / `tar-gzip.ovl` 需要真实 Stub 与 TAR 打包，由 `reference-impl/` 落地某语言 Generator 后按本 README 规则产出并补充对应 `expected/*.json`。

## 已提供向量（自校验通过）

| 向量 | 类型 | 说明 |
|:---|:---|:---|
| `raw.ovl` | 正例 | Raw + 无压缩，占位 Stub（`OVERLAY-DEMO-STUB!`，18 字节）。与 `expected/raw.json` 成对，定位/校验/文件集**全部自校验通过**。 |
| `corrupt-trailer.ovl` | 负例 | 由 `raw.ovl` 将尾部 `trailer_checksum` 最低字节异或 `0x01` 得来。读取方 MUST 定位成功但 `trailer_checksum` 校验失败并拒绝（`expected/corrupt-trailer.json`：`expect: "reject:trailer_checksum"`）。 |

> 占位 Stub 说明：`raw.ovl` 的 Stub 段非真实可执行程序，仅用于演示字节布局与校验和；真实可执行向量由参考实现产出。

## 生成规则（强制）

- 每个 `.ovl` MUST 由某个 Generator 用**固定输入**（固定 Stub 字节、固定 Payload 内容、固定选项）生成。
- 生成输入 MUST 记录在对应 `expected/<name>.json` 的 `input` 字段（见下），保证可复现。
- 生成结果 MUST 满足 `SPEC/04-conformance.md` §3 的全部断言（正例）或被正确拒绝（负例）。

## `expected/<name>.json` 结构

```json
{
  "name": "tar-gzip",
  "input": {
    "stub_sha256": "……",
    "archive_format": "tar",
    "compression": "gzip",
    "files": [
      {"name": "a.txt", "size": 100, "sha256": "……", "mode": "0644"},
      {"name": "b.cfg", "size": 240, "sha256": "……", "mode": "0644"}
    ]
  },
  "expect": "ok",
  "file_size": 123456,
  "trailer_offset": 123400,
  "archive_offset": 512,
  "archive_size": 122888,
  "archive_size_raw": 131072,
  "files": [
    {"name": "a.txt", "size": 100, "data_offset": 0,  "mode": "0644", "sha256": "……"},
    {"name": "b.cfg", "size": 240, "data_offset": 100, "mode": "0644", "sha256": "……"}
  ]
}
```

- 正例：`expect` 为 `"ok"`；负例：`expect` 为 `"reject:<reason>"`（如 `"reject:trailer_checksum"`）。
- `files` 顺序 MUST 与生成时的逻辑路径字典序一致。

## 使用方式

1. **生成方自检**：产出 `.ovl` 后，按 `SPEC/03-runtime.md` 的定位与校验流程读回，确认与 `expected/*.json` 一致。
2. **跨实现互操作**：对每个 `.ovl` 运行每个候选 Stub，比对 `expected/*.json`；对每个候选 Generator 的产物运行每个候选 Stub，结果 MUST 一致（`SPEC/04-conformance.md` §4）。

## 新增向量

- 覆盖新的 `archive_format` / `compression` 组合、或新的负例（损坏 `header_checksum`、未知 `version` 等）时，新增 `.ovl` 与同名 `expected/*.json`。
- 保持**确定性**：输入记录在 `input` 字段；同一输入重生成须字节级一致。
