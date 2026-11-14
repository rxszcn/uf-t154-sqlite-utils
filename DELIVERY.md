# 类型检测静默跳过修复交付说明

## 一、根因链（为什么「类型永远修不好、退出码 0、排障无从下手」）

现场：首批遗留管道用 `--no-detect-types` 建了全 TEXT 表；次日增量管道按默认（开启检测）
重跑，期望 `id/qty/price` 被修正为 INTEGER/REAL，实际列类型纹丝不动，退出码 0，无任何输出。

完整链路（`sqlite_utils/cli.py`）：

1. **入口分派**：`sqlite-utils insert` 进入 `insert()`，转调
   `insert_upsert_implementation()`。CSV/TSV 分支在 `cli.py:1344-1347` 构造
   `TypeTracker` 并用 `tracker.wrap(docs)` 包裹行迭代器——检测确实在跑。
2. **记录表是否已存在**：`_insert_docs()` 在插入前用
   `table_existed_before_insert = table in db.table_names()` 记下基线（`cli.py:1181`）。
   次日场景下该值为 `True`。
3. **插入正常完成**：`insert_all()` 按既有 TEXT 表结构原样写入，不报错。
4. **守卫条件为假**：修复前的守卫是

   ```python
   if (tracker is not None
       and not table_existed_before_insert   # ← 次日场景为 False
       and db.table(table).exists()):
       db.table(table).transform(types=detected_types)
   ```

   `not table_existed_before_insert` 为 `False`，整个 `transform` 被跳过。
   `tracker.types` 已经算出正确的 `{'id': 'integer', ...}`，但从未被读取。
5. **无任何信号**：被跳过的分支里没有任何 `click.echo`、`warnings.warn` 或返回
   值，命令正常返回、退出码 0、stdout/stderr 全空。所以运维侧看到的是
   「跑了、成功了、什么都没变」。

**这个跳过是刻意的，不是 bug。** 原注释写明原因：对已存在表做 `transform`
会重写列类型并重建表、搬移数据，像 ZIP 码 `01234` 这类带前导零的 TEXT 值一旦
被搬进 INTEGER 列就会变成 `1234`，静默损坏数据。因此默认不碰既有表是正确保护。

**真正缺的是可观测性与逃生舱**：调用方既无法知道「跳过已发生」，也无从在确认
安全后主动应用检测类型。这就是「类型永远修不好、排障无从下手」的直接原因。

## 二、可观测能力（默认路径，行为逐字不回退）

默认不加任何新参数时：

- **表由本次命令创建**：行为与修复前 `sqlite_utils` 完全一致——插入后
  `transform(types=tracker.types ∪ --type 覆盖)`，stdout/stderr 均无输出。
- **表已存在（默认路径）**：仍然**不** `transform`，列类型与数据逐字节不变。
  新增的唯一副作用是一条 **stderr** 警告：

  ```
  Warning: detected column types for table "orders" were not applied because the
  table already exists (detected-types-skipped). Detected: id=integer, qty=integer,
  price=float, zip=integer. Use --apply-detected-types to apply them.
  ```

设计要点：

| 决定 | 理由 |
| --- | --- |
| 写 **stderr** 而非 stdout | `insert` 的 stdout 是管道数据通道（如与其他命令串联）。把告警混进 stdout 会破坏既有下游。stderr 是终端用户可见、且不污染数据流的正确通道。 |
| 稳定标记 `detected-types-skipped` | 可编程读取：`... 2>&1 \| grep -q detected-types-skipped`。标记是固定 ASCII 词，不含路径、不含检测值，脚本可依赖；同一条消息里的人类可读部分可自由调整。常量导出为 `sqlite_utils.cli.DETECTED_TYPES_SKIPPED`，测试与下游代码可直接 import，不必硬编码字符串。 |
| 消息里带上检测到的类型 | 「排障无从下手」的一半原因是不知道为什么该修正。直接列出 `列=检测类型`，运维一眼能看出「我的 id 本应 INTEGER」。 |
| 仅在**确实有检测结果**时告警 | 若本次输入没有任何行（headers-only CSV），`tracker.types` 为空，此时不告警——没有可跳过的内容，告警只会是噪音。空表场景既有测试 `test_insert_csv_headers_only` 依旧原样通过。 |
| 结构化返回值 | `insert_upsert_implementation()` 以及 `insert`/`upsert` 命令现在返回 `sqlite_utils.utils.TypeDetectionOutcome`。Python 调用方可用 `CliRunner().invoke(..., standalone_mode=False).return_value` 直接拿到 `reason`、`detected_types`、`applied_types`、`skipped`，不必解析文本。这是第二条可编程读取通道，且不依赖 stderr 格式。 |

`TypeDetectionOutcome.reason` 的取值：
`table_created`（新建表并应用）、`applied_to_existing_table`（开关开启并已应用）、
`already_matches`（开关开启但无需变更）、`skipped_existing_table`（默认跳过）、
`nothing_detected`（有表但无检测结果）、`no_rows`（未走到该逻辑）。

## 三、显式开关：命名与默认值

**`--apply-detected-types`，默认关闭（`is_flag=True`，不设 `default=True`）。**

命名理由：

- 选项名要回答「做什么」，而不是「再检测一次」。检测本来就默认开着，缺的一直
  **应用**这一步。`--apply-detected-types` 直接描述行为，不会与既有的
  `--no-detect-types`（描述检测开关）混淆。
- 与既有选项构成清晰的三态语义：
  `--no-detect-types`（不检测）→ 默认（检测但不碰既有表）→ `--apply-detected-types`
  （检测并改造既有表）。三者互相独立、可组合，`--no-detect-types --apply-detected-types`
  自然退化为无操作（无检测结果可应用），已用测试固定。
- 刻意**不**复用/复活已被移除的 `-d/--detect-types`：那个选项在 4.0 被明确删除
  （`test_detect_types_flag_removed` 固定了「用它会报 No such option」），复活会
  改变那个既有契约，且语义（「检测」而非「应用」）也不准确。

默认关闭的理由：开关开启意味着 `transform` 会重写既有数据。前导零损坏是真实且不可
逆的，绝不能因为「默认更聪明」而替用户做这个决定。保护必须是默认态，opt-in 必须
是显式的。

开启后的行为细节：

- 只对**已存在且检测亲和性确实不同**的列做 transform
  （`_detected_types_that_differ`）。全部相同则完全不 transform，不会为主键/索引
  带来无谓的重建风险。
- 检测出的 `float` 先经 `COLUMN_TYPE_MAPPING` 归一化为 `REAL`，与建表路径一致，
  避免出现 `FLOAT` 这种较弱的具体类型。
- `--type col TYPE` 覆盖仍然最高优先级：`detected_types.update(column_type_overrides)`
  在这一步之前就已合并，显式覆盖的列不会被检测值改写。
- 不输出跳过告警（用户明确要求了应用，没必要再提示跳过）。
- 与 `--no-detect-types` 组合时无检测结果，安全空转。

## 四、不回退的既有契约（均有测试固定）

- `test_insert_csv_detect_types_leaves_existing_table_alone`：既有表 `zip` 保持
  `TEXT`，前导零值原样保留——**未改一字**，原样通过。
- `test_upsert_csv_detect_types_leaves_existing_table_alone`：同上，upsert 形态——
  原样通过。
- `test_insert_csv_detect_types_new_table`：新表仍按检测值建列——原样通过。
- `test_insert_detect_types` / `test_upsert_detect_types` /
  `test_csv_detect_types_creates_real_columns` / `test_insert_no_detect_types` /
  `test_upsert_no_detect_types`：全部原样通过。
- `test_detect_types_flag_removed`：`-d/--detect-types` 仍然报 `No such option`——原样通过。
- 新建表路径的 stdout/stderr 仍为空，所有断言 `result.output == ...` 的既有测试不受影响
  （告警只走 stderr，且仅发生在既有表跳过分支）。
- 全量基线 `1497 passed, 16 skipped` → 修复后 `1507 passed, 16 skipped`
  （新增 10 个用例，零回归）。

## 五、补齐的测试（`tests/test_cli_insert.py`）

默认跳过形态：

- `test_skip_emits_machine_readable_signal`（insert/upsert 两形态）：断言
  `stdout == ""`、stderr 含 `detected-types-skipped`、含 `already exists`、
  含 `--apply-detected-types`，且列类型确实未变。
- `test_type_detection_outcome_is_programmatically_readable`：断言 skipped /
  applied / created 三种 `return_value` 的 `reason`、`detected_types`、
  `applied_types`、`skipped` 字段。

显式应用形态：

- `test_apply_detected_types_to_existing_table`（insert/upsert 两形态）：全 TEXT 表
  经开关后变为 `{id:int, qty:int, price:float, zip:int}`，既有行被转换。
- `test_apply_detected_types_preserves_type_overrides`：`--type zip text` 优先于检测值。
- `test_apply_detected_types_noop_when_already_matching`：无需变更时不 transform、无输出。
- `test_apply_detected_types_with_no_detect_types_is_inert`：与 `--no-detect-types` 组合无害。
- `test_apply_detected_types_leaves_leading_zeros_alone_when_detected_as_text`：
  检测结果为 TEXT 时不会强行数字化。
- `test_apply_detected_types_new_table_matches_default`：新表路径下开关不改变任何行为，
  schema 仍为 `INTEGER/REAL`。

## 六、改动文件

| 文件 | 改动 |
| --- | --- |
| `sqlite_utils/cli.py` | 守卫改为可观测；新增 `DETECTED_TYPES_SKIPPED`、`_detected_types_that_differ()`、`--apply-detected-types` 选项；`insert`/`upsert` 返回 `TypeDetectionOutcome` |
| `sqlite_utils/utils.py` | 新增 `TypeDetectionOutcome` 数据类 |
| `tests/test_cli_insert.py` | 新增 10 个用例（两形态对照 + 边界） |
| `docs/cli.rst` | 说明跳过告警、机器可读标记、`--apply-detected-types` 及其风险 |
| `docs/changelog.rst` | 4.2.2 (unreleased) 条目 |
| `docs/cli-reference.rst` | `cog` 重新生成，补入新选项帮助文本 |

## 七、验证

```
pytest:    1507 passed, 16 skipped（基线 1497 + 新增 10）
black:     65 files unchanged
flake8:    0
mypy:      Success: no issues found in 64 source files
pyright:   0 errors, 0 warnings
cog --check: 全部通过
```

`ty check` 报出的 1 条诊断（`db.py:4791` 冗余 `cast`）在原始快照上即存在
（已用 `git stash` 对比确认），与本次改动无关。
