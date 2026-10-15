# 参考实现（Reference Implementations）

> 本目录是**占位骨架**，当前仓库不内置任何语言的实现代码。Overlay 是**语言无关**的规范，参考实现**可以**用任意语言编写；本目录只提供组织约定与验收口径。

## 目录约定

```
reference-impl/
├── generator/<lang>/     # 生成器：<lang> 语言实现
│   └── README.md
└── stub-sdk/<lang>/      # 读取 SDK：<lang> 语言实现
    └── README.md
```

- 每种语言一个子目录（如 `generator/go/`、`stub-sdk/rust/`、`generator/c/`）。
- 每个子目录 MUST 自带 `README.md`，声明：语言与版本要求、构建方式、对应的一致性等级（Level A / B，见 `SPEC/04-conformance.md`）、以及该实现**支持**的 `archive_format` / `compression` 子集。

## 验收口径

任何参考实现 MUST：

1. 通过 `SPEC/04-conformance.md` §3 的全部断言（至少 Level A）；
2. 通过确定性测试（`04-conformance.md` §5）；
3. 在 `test-vectors/` 上产出/读取结果与 `expected/` 一致；
4. 在跨实现互操作矩阵（`04-conformance.md` §4）中通过交叉测试。

## 添加新实现

1. 建 `reference-impl/<component>/<lang>/`；
2. 填 `README.md`（支持矩阵 + 构建说明）；
3. 提交其生成的 `test-vectors` 产物（若该实现是生成器）或其对该目录向量的读取结果；
4. 更新 `04-conformance.md` §4 的互操作矩阵。

> 新增实现不改变规范本身；规范只描述“应该是什么样”，实现是证明。
