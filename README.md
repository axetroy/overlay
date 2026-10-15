# Overlay

> 一个将**固定可执行程序**与**自定义数据**合成为**单一可执行文件**的**格式规范与参考标准**。
> 语言无关：定义“应该是什么样”，不规定“用什么写”。

Overlay 的本质是 **附加数据（Appended Data / Overlay）**：操作系统的可执行文件加载器只关心文件开头的可执行格式部分（PE / ELF / Mach-O），尾部多出来的字节被忽略。于是把一个预编译的固定程序（Stub）与任意文件集合（Payload）拼成一个文件分发，运行时由 Stub 从自身尾部读取数据。

## 目录结构

```
overlay/
├── spec.md                  # 设计白皮书（设计背景与动机，非规范性）
│
├── SPEC/                    # ★ 权威规范（规范性，语言无关）
│   ├── README.md            #   阅读顺序与文档地图
│   ├── 00-overview.md       #   总纲：术语、文档目录
│   ├── 01-file-format.md    #   字节级布局：魔数/Trailer/Archive/压缩/校验和
│   ├── 02-generation.md     #   生成器流程 + 确定性要求
│   ├── 03-runtime.md        #   运行时行为：自身定位算法 + 校验顺序
│   ├── 04-conformance.md    #   一致性判据：测试向量/Pass-Fail/互操作
│   ├── 05-extensions.md     #   扩展机制：flags/自定义压缩/兼容原则
│   └── 06-versioning.md     #   版本演进策略
│
├── reference-impl/          # 参考实现（占位骨架，当前不内置语言代码）
│   ├── README.md            #   组织约定 + 验收口径
│   ├── generator/README.md #   生成器
│   └── stub-sdk/README.md  #   读取 SDK
│
├── test-vectors/            # 标准测试向量 + 预期解析结果
│   ├── README.md            #   生成规则 + JSON schema
│   ├── raw.ovl              #   可字节级复现的 Raw 无压缩算例（占位 Stub）
│   ├── corrupt-trailer.ovl  #   负例：篡改 trailer_checksum
│   └── expected/
│       ├── raw.json
│       └── corrupt-trailer.json
│
└── docs/
    └── examples.md          # 使用示例 + 字节级算例（信息性）
```

## 快速上手（阅读路线）

1. **设计背景**：`spec.md`（白皮书）。
2. **要读文件**：`SPEC/01-file-format.md`（字节布局）→ `SPEC/03-runtime.md`（定位与校验）。
3. **要写生成器**：`SPEC/02-generation.md`。
4. **要验证实现**：`SPEC/04-conformance.md` + `test-vectors/`。
5. **要看例子**：`docs/examples.md`（含与 `test-vectors/raw.ovl` 对应的字节级算例）。

## 状态

- **规范**：`SPEC/` 为现行 v1 权威规范（v1 冻结）。
- **测试向量**：`raw`（正例）与 `corrupt-trailer`（负例）已提供且**自校验通过**；`minimal`（TAR）与 `tar-gzip` 待参考实现落地后由 Generator 产出。
- **参考实现**：`reference-impl/` 为占位骨架，本仓库**不绑定**任何语言；实现者按 `SPEC/04` 的口径添加并提交向量结果。

## 关于占位 Stub

`test-vectors/raw.ovl` 使用的 Stub 是 18 字节占位文本（`OVERLAY-DEMO-STUB!`），**非真实可执行程序**，仅用于演示字节布局与校验和。真实的、可执行的参考向量需要在 `reference-impl/` 落地某语言实现后，用真正的 Stub 重新生成。

## 不变量（跨文档一致）

- 字节序：除 UTF-8 / gzip / zstd 外**一律小端**（`SPEC/01` §1）。
- 魔数：`4F 56 4C 59`（`OVLY`），HEADER 起点与 TAIL 各出现一次。
- 校验和：默认 **CRC-64/XZ**（poly `0xC96C5795D7870F42`），标准值 `CRC-64/XZ("123456789") = 0x995DC9BBDF1939FA`。
- 生成**必须确定**（同输入字节级相同输出），TAR/压缩器不得写入时间戳或随机数。
