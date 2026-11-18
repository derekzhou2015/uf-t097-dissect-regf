# 交付说明：FastLeaf 非 ASCII NameHint 按名查找修复

## 1. 问题根因

`lf`（FastLeaf）子键列表的每个条目带一个 4 字节 `NameHint` 加速字段
（`c_regf._CM_INDEX.NameHint`）。旧实现 `FastLeaf.subkey()` 直接对它做 ASCII 解码：

```python
hint = entry.NameHint.rstrip(b"\x00").decode("ascii").lower()
```

注册表对压缩名（`CM_KEY_NODE` 带 `COMP_NAME` 标志，按 Latin-1/单字节存储）并不保证把
非 ASCII 名的提示清零。像 `Ärzte` 这类 Latin-1 名字，其提示会保留原始单字节
（`Ärz` → `b"\xc4rz"`），`0xC4` 超出 ASCII 范围，`decode("ascii")` 直接抛
`UnicodeDecodeError`，并从 `subkey()` → `KeyNode.subkey()` → `RegistryHive.open()`
整条按名/按路径查找链路外泄，导致查找崩溃——而同一键在 `subkeys()` 枚举路径上
（`__iter__` 只读 `entry.Cell`，从不触碰 NameHint）却能完整列出。

旧实现还有第二个隐患：提示一旦"形似合法 ASCII"（例如垃圾值 `b"JUNK"`），它参与的是
查找判等逻辑的一部分；提示与真实名不一致时条目会被直接跳过，提示从"加速器"变成了
"否决器"，可能造成漏命中。

## 2. 修复方案（dissect/regf/regf.py 的 FastLeaf.subkey）

核心原则：**NameHint 只作加速，命中与否只由真实键名复核决定；提示字节永不做字符串解码。**

实现为"提示预筛 + 全量名复核兜底"两段式：

1. 目标名小写后，仅当能编码成纯 ASCII 时才构造 4 字节提示（前 4 字节、`\x00` 填充）；
   含非 ASCII 字符时 `name_hint = None`，完全不使用提示快路径。
2. 第一遍遍历：只有在"目标提示存在 **且** 存储提示是纯 ASCII **且** 非零 **且** 两者
   小写相等性检查不通过"时，才把该条目标记为 `skipped` 延后处理；其余条目立即用真实
   键名做大小写不敏感复核。
3. 第二遍兜底：对所有被延后的条目，逐个打开真实键节点用 `sk.name.lower() == lname`
   复核。非 ASCII 提示、零填充提示、ASCII 垃圾提示都落在这一遍。

### 关键取舍与理由

- **为什么不解码提示，而在字节域处理**：提示本来就是原始字节，解码是崩溃源头。对
  Latin-1 变音字节也不存在可靠的字节级大小写折叠（`bytes.lower()` 只覆盖 ASCII 区，
  `b"\xc4".lower()` 仍是 `b"\xc4"`，而 Python 侧 `"Ä".lower()` 是 `"ä"` = `0xE4`）。
  因此提示快路径严格限定在双方都是纯 ASCII 的情形；非 ASCII 一律走真实名复核，
  真实名比较是 Python `str` 级，大小写语义正确。
- **为什么提示不匹配时是"延后"而不是"丢弃"**：提示是加速元数据而非判定依据。垃圾或
  异常提示不能否决一个真实存在的键（本仓库构造的 `Arzt` 键带 `JUNK` 提示即此场景），
  所以被提示排除的条目必须在第二遍用真实名重新核验，保证零漏命中、零错配。
- **异常封口**：查找路径不再有任何对提示的 `decode`；目标名编码用 `try/except
  UnicodeEncodeError` 收口为 `name_hint=None`。对外的失败语义保持不变——找不到时仍抛
  `RegistryKeyNotFoundError`，不会有 `UnicodeDecodeError` 外泄。
- **复杂度**：典型 ASCII 场景仍是 O(n) 单遍、且多数不匹配在提示层即过滤，保留加速
  效果；最坏情况（全部条目延后）退化为对每个条目做一次真实名比较，与 `IndexLeaf`
  的既有 O(n) 行为同阶，不改变整体量级。
- **枚举路径零改动**：`__iter__`/`count`/`__len__` 未触碰，枚举既有行为保持不变。

## 3. 回归测试

新增测试 `tests/test_regf.py::test_fastleaf_compressed_non_ascii_name_hint`，使用新夹具
`tests/_data/FastLeafHints.gz`（fixture `fastleaf_hints_hive`）。该镜像是一个根键使用
`lf` 列表的最小 hive，三个子键分别覆盖：

| 子键 | 名称存储 | NameHint | 覆盖点 |
| --- | --- | --- | --- |
| `Ärzte` | 压缩 Latin-1（非 ASCII） | `b"\xc4rz\x00"`（原始 Latin-1 字节，旧代码即在此崩溃） | 压缩非 ASCII 名按名命中、大小写不敏感、无解码异常外泄 |
| `Über` | 压缩 Latin-1（非 ASCII） | `b"\x00\x00\x00\x00"`（零填充） | 零提示回退路径 |
| `Arzt` | 压缩 ASCII | `b"JUNK"`（形似合法 ASCII 的垃圾值） | 垃圾提示不漏命中、也不产生错配（`JUNK`/`junk` 等查不到，真实名 `Arzt` 查得到） |

测试同时断言：枚举结果不变、`hive.open(r"Ärzte")` 路径定位可用、不存在的名仍抛
`RegistryKeyNotFoundError`。

已验证该夹具在**旧实现**下于首个条目即抛
`UnicodeDecodeError: 'ascii' codec can't decode byte 0xc4`，证明回归用例确实能拦住此缺陷。

镜像由仓库外独立脚本 `build_fastleaf_hints_hive.py` 生成（按不可信数据隔离要求，生成
工具与仓库分离，使用 `python -I` 运行），数据文件以 gzip 形式随仓库 `tests/_data/`
分发，与现有 `FastLeaf.gz`/`SYSTEM.gz` 的夹具组织方式一致。

## 4. 验收结果（Windows 原生 Python 3.12，可编辑安装，仓库根目录 pytest）

- 压缩形态非 ASCII 名（`Ärzte`/`Über`）按名查找命中，且大小写不敏感：通过
- 查找路径无任何字符串解码异常外泄，未知名仍为 `RegistryKeyNotFoundError`：通过
- 枚举路径行为保持不变（新镜像 + 真实 SYSTEM 镜像均验证枚举/查找一致性）：通过
- 垃圾/异常 NameHint 不造成错配或漏命中，命中以真实名复核为准：通过
- `pytest tests/` 全量 **15 passed**（原 14 + 新增 1），多个随机种子下稳定通过
- `ruff check`、`ruff format --check` 全部通过

## 5. 改动文件

- `dissect/regf/regf.py`：重写 `FastLeaf.subkey()`（提示字节域预筛 + 真实名兜底复核）
- `tests/test_regf.py`：新增 `test_fastleaf_compressed_non_ascii_name_hint`
- `tests/conftest.py`：新增 `fastleaf_hints_hive` fixture
- `tests/_data/FastLeafHints.gz`：新增测试镜像
