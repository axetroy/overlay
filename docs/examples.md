# 使用示例（Examples）

> **信息性文档**：不是规范。演示如何“生成”与“读取” Overlay 文件，并给出一个**字节级可校验**的完整算例。
> 实际实现请参考 `SPEC/01`–`03`；本文件只用于理解与手工核对。

## 1. 概念模型

```
Generator:  Stub ‖ [TAR|Raw 打包] → [gzip|zstd?] → Archive ‖ Trailer = 单个 Overlay 文件
Stub(运行时): 自身尾部自下而上定位 Trailer → 校验 → 定位 Archive → 解压 → seek/read
```

## 2. 完整算例（Raw + 无压缩）

这是一个**真实可字节级复现**的算例，与 `test-vectors/raw.ovl` + `test-vectors/expected/raw.json` 对应。
使用占位 Stub（`OVERLAY-DEMO-STUB!`，18 字节，非真实可执行文件，仅为演示字节布局），两个 Payload 文件：

| 逻辑路径 | 内容 | 字节数 |
|:---|:---|:---|
| `conf/app.ini` | `[app]\nmode = demo\n` | 18 |
| `hello.txt` | `Hello, Overlay!\n` | 16 |

### 2.1 关键数值（按 `SPEC/02-generation.md` 步骤）

- `file_count = 2`；按字典序排序后 `conf/app.ini` 在前、`hello.txt` 在后。
- `archive_offset = stub_len = 18`。
- 各条目 `data_offset`（Raw：前序 `data_size` 之和）：
  - `conf/app.ini` → `0`，`data_size = 18`
  - `hello.txt` → `18`，`data_size = 16`
- `archive_size = 18 + 16 = 34`；无压缩，`archive_size_raw = 34`。
- `archive_checksum = CRC-64/XZ(压缩后 Archive 字节流) = 0x996cb738798c83c6`。
- 条目校验和（原始内容 CRC-64/XZ）：
  - `conf/app.ini` → `0x0b74f73ad1bf3200`
  - `hello.txt` → `0xbe59d5d737dd471e`
- `trailer_size = 56 + 77 + 8 = 141`（HEADER 52 + 两个变长条目共 77 + TAIL 8）。
  - 注：条目长度 = 每个 `2(name_length) + name + 8 + 8 + 8 + 2`，两个条目合计 77 字节。
- `header_checksum = 0xee516900`；`trailer_checksum = 0x4cba682a`。

### 2.2 完整字节布局（文件总长 193）

```
偏移   区域
0x000  ┌──────────────────────────────────┐
       │ Stub (18B)                       │  "OVERLAY-DEMO-STUB!"
0x012  ├──────────────────────────────────┤
       │ Archive (34B, Raw 无压缩)        │  conf/app.ini ‖ hello.txt
0x034  ├──────────────────────────────────┤
       │ Trailer (141B)                   │  HEADER(52) ‖ ENTRIES(77) ‖ TAIL(8)
0x0c1  └──────────────────────────────────┘  = 193 字节
```

Trailer 尾部 8 字节（位于文件 `0x0bf..0x0c0`）：

```
magic_repeat  = 4F 56 4C 59
trailer_cksum = 2a 68 ba 4c   (小端 = 0x4cba682a)
```

### 2.3 手工核对（用 `xxd` / 校验工具）

```
# 查看文件尾 16 字节（应见 magic_repeat + trailer_checksum）
tail -c 16 test-vectors/raw.ovl | xxd

# 用 expected/raw.json 中的字段核对偏移与校验和
#   trailer_offset = 52, archive_offset = 18, archive_size = 34
```

读取方（Stub）按 `SPEC/03` §3 的自下而上算法定位到 `trailer_offset=52`，校验通过即可 `seek+read`。

## 3. 生成（伪代码，语言无关）

```
stub  = read_file("myprog")            # 预编译可执行程序
files = [
  ("conf/app.ini", b"[app]\nmode = demo\n", 0o644),
  ("hello.txt",    b"Hello, Overlay!\n",    0o644),
]
options = { format: RAW, compression: NONE }

out = generator.build(stub, files, options)   # 见 SPEC/02
write_file("dist/myprog.overlay", out)        # 单文件分发物
```

## 4. 读取（伪代码，语言无关）

```
p     = self_path()                     # 平台 API, 见 SPEC/03 §2
trail = locate_trailer(p)               # 自下而上, 见 SPEC/03 §3
check(trail)                            # trailer / header / archive 校验
arch  = read_and_decompress(p, trail)   # archive_offset .. archive_size
for f in trail.files:
    data = arch[f.data_offset .. f.data_offset + f.data_size]
    assert crc64_xz(data) == f.data_checksum
    if f.name == "conf/app.ini": use(data)
```

## 5. 用现成工具调试

- **TAR 格式**：Archive 段是标准 ustar，可直接 `tar -tvf` 调试（需先按 `archive_offset` 截取 Archive 段）。
- **Raw 格式**：纯字节池，`hexdump` / `xxd` 按 `expected/*.json` 的 `data_offset` 对位即可。
- **压缩流**：`gzip` 用 `zlib`/`gunzip`，`zstd` 用 `zstd -d`。

## 6. 常见误区

- 把 `archive_offset` 当成“相对 Archive 的偏移”——它是**相对整个 Overlay 文件**的偏移，等于 `stub_len`。
- 混淆 `archive_size`（压缩后，写入文件的字节数）与 `archive_size_raw`（解压后，逻辑流长度）。
- 假设 `argv[0]` 就是自身路径——MUST 用平台“自身路径” API（`SPEC/03` §2）。
- 期望校验和提供防篡改——它只检测**意外**损坏；防篡改需另加签名（macOS `codesign` 等）。
