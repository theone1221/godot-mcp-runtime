# TODO — 融合 godot-stagehand 能力到 runtime

本分支（`feat/stagehand-fusion`）的目标：**在 runtime 的零足迹架构上，补齐它缺失的交互原语**，
不改变它已验证可靠的部分。

上游：`Erodenn/godot-mcp-runtime`（MIT，4.0.1 基线）· 借鉴：`mrf/godot-stagehand`（MIT，0.5.3）。

---

## 0. 约束（先读，违反任一条后面都要返工）

1. **改动优先新增文件，不深改既有函数体。** 上游仍在动（4.x 还会有小版本），
   `mcp_bridge.gd` 是 1994 行的热点文件，改得越深 `git merge upstream/main` 冲突面越大。
   新能力放 `src/scripts/core/*.gd`，`mcp_bridge.gd` 只加转发接线。
2. **autoload 只能有一个。** 不引入 `stagehand_server.gd`，它要求常驻 autoload + WebSocket，
   会破坏 runtime「退出即清、`project.godot` 干净」的核心优势。
3. **只依赖 7 个公开交互工具**（`run_project` / `stop_project` / `get_ui_elements` /
   `simulate_input` / `take_screenshot` / `get_debug_output` / `run_script`），不碰内部实现。
4. **两侧都要改。** GDScript 实现 + TypeScript schema（`src/tools/runtime-tools.ts`）。
   只改 `.gd` 的话 agent 根本调不到新动作。
5. **MIT 署名。** 从 stagehand 借鉴的代码，文件头注明
   `Adapted from mrf/godot-stagehand (MIT)`，并保留其原始版权行。
6. **任何动作必须回证据。** 本次实测已证明「工具返回 `ok:true`」不等于「事情发生了」，
   新动作的返回值必须能被当断言依据。
7. **每步跑 `npm test`。** 基线：`3728 passed / 0 failed`，473 skipped（集成测试需 Godot）。
   绿了才允许下一刀。

---

## 1. 实测依据（这些数字决定了优先级，不是推测）

| 实测结论 | 出处 |
|---|---|
| `simulate_input` 只有 7 种动作，**无 drag** | 4.0.1 的 `inputSchema` 枚举 |
| 手拼 press + 8×motion + release，**全部回 `ok:true`，`_get_drag_data` 从未触发** | 应用侧 `_gui_input` 日志 |
| 根因：runtime 构造的 motion **`button_mask = 0`**，等于「没按住鼠标在移动」 | 同上 |
| `run_script` 手搓带 `MOUSE_BUTTON_MASK_LEFT` + 非零 `relative` 可完整触发 `GET_DRAG_DATA` / `CAN_DROP` / `DROP` | app2 工程 `probe_drag.gd` |
| `relative = Vector2.ZERO` 时**不触发**（判定依赖相对位移累积） | 同上（两个变体对照） |
| `drop` 的 `at_position` 实测 `(10539, 10079)`，正确值约 `(70, 45)` → **落点坐标不可信** | 同上 |
| `wait` 只有 `ms` / `frames`，**无条件等待** | `inputSchema` |
| `{"frames":1}` 精确（146→147），但 headless 帧率与墙钟不成比例（frame 583/606/667 对应 elapsed 增量仅 ~17ms） | 对比 `get_debug_output` 与返回值 |
| Godot 把 disabled 的 Button **仍报为 hovered** → 只看 topmost 会把 disabled 判成通过 | runtime 源码注释；stagehand 因此误报 `success:true` |
| runtime 走 `Input.parse_input_event()`（缓冲一帧，经 DisplayServer 像素重映射） | `src/scripts/mcp_bridge.gd` |
| stagehand 走 `Viewport.push_input(ev, true)`（跳过重映射，但 headless 下需修 `Window.size` 的 64×64 stub） | `addons/stagehand/core/input_simulator.gd` |
| 两家都做「先发 motion → 读 hovered → 再按键」，但 stagehand 缺 disabled 检查 | 两边源码对照 |

---

## 2. 待做

### P0 — 条件等待（先做，收益最直接）

runtime 只有定长 sleep，所有断言都得靠猜时长。这是「agent 能不能自己判断跑完了」的地基。

- [ ] 新增 `src/scripts/core/waiter.gd`
  - `wait_until(predicate_callable, max_frames, max_ms)` → `{settled, frames_used, elapsed_ms, last_value}`
  - `wait_for_signal(node, signal_name, timeout_ms)` → `{emitted, args, waited_ms}`
  - `wait_for_property(node, prop, expected, timeout_ms)` → `{changed, from, to, waited_ms}`
  - `wait_for_node(path|selector, state = visible|exists|enabled, timeout_ms)`
  - **超时必须回 `settled:false`，不能静默继续**
- [ ] `mcp_bridge.gd`：`simulate_input` 增加 `action: "wait_for"`（含 `condition` 子类型与超时参数），只做转发
- [ ] `src/tools/runtime-tools.ts`：同步 action 枚举 + 新增字段 schema（`condition` / `signal` / `node` / `property` / `timeout_ms` / `max_frames`）
- [ ] 单测：`tests/` 下补 waiter 的纯逻辑单测（不依赖 Godot 的部分）
- [ ] 实测：app2 工程里发一个自定义信号，验证 `wait_for_signal` 真的等到、且超时能回 `settled:false`

### P1 — 拖拽

- [ ] 新增 `src/scripts/core/drag.gd`
  - `drag(from, to, steps = 12, step_frames = 1)`，**强制**：motion 带 `BUTTON_MASK_LEFT` + 非零 `relative`；
    按下前先发一次 `NOTIFICATION_VP_MOUSE_ENTER`（GUT issue #790 的坑）
  - 返回证据：`{get_drag_data, can_drop, drop, dropped_data, steps, total_frames, rejected_at}`
  - **不返回、不采信 `at_position`**（实测不可信）；落点断言只能靠 `dropped_data` + 业务状态反推
- [ ] `mcp_bridge.gd`：`simulate_input` 增加 `action: "drag"`（`from_element` / `to_element` / `from` / `to` / `steps`）
- [ ] `src/tools/runtime-tools.ts`：drag 的 schema
- [ ] 实测：`probe_drag.gd` 工程里三条回归 —
  ① 正常拖拽三个回调都触发；② 被遮挡的目标 → `can_drop` 为 false 且有原因；
  ③ 步数为 1（低于阈值）→ 明确报告未触发，而不是假装成功

### P2 — 感知增强（快照加语义）

- [ ] `get_ui_elements` 补 `role`（由 Control 子类派生：`Button` → `button`、`CheckBox` → `checkbox`、
      `LineEdit` → `textbox` …），参照 stagehand `core/accessibility_tree.gd`
- [ ] 补 `accessible_name` / `class` / `visible_in_tree` / `clipped_by`（祖先链裁切）
- [ ] 选择器：支持 `class:` / `name:` / `group:` / `text:` / `meta:` 与链式 `A >> B`
      （可只做解析 + 在 `simulate_input` 的 `element` 字段里接受，不必改工具名）
- [ ] **不做**：把遮挡写进快照。遮挡取决于绘制顺序，不是静态属性，只能靠 `click_element` 拒绝时暴露

### P3 — 证据与回归（可选，晚于 P0/P1）

- [ ] 动作录制 / 回放（参考 `input_recorder.gd`，543 行）
- [ ] 断言层：把 `hit` / `signals` / `changes` 转成断言输出，支持 `--exit-code-on-failure`
      （参照 stagehand 的 `exit 5 = 真回归` 契约）
- [ ] 视觉回归：`take_screenshot` 已有 `likelyBlank` 统计，补基线 diff（像素差百分比 + 输出 diff 图）

---

## 3. 明确不做

| 不做 | 原因 |
|---|---|
| 搬 `stagehand_server.gd`（1029 行，WebSocket + 常驻 autoload） | 破坏零足迹；runtime 的临时注入是核心优势 |
| 亚帧时序精度 | 引擎侧限制（1 帧量化），要测时序得在引擎内按谱面预排事件 |
| `_draw` 自绘控件的语义命中 | 无 Control 树可枚举，只能坐标 + 业务状态 |
| 手感 / 观感评价 | 属于视觉 diff 汇报范围，判断交给人 |
| 改 `click_element` 既有语义 | 它当前是最可靠的一条路径（四类不可达分别拒绝且在 `frame:0` 就拒绝），别动 |
| 未验证就把 stagehand 设成阻断门禁 | 它 0 issue / 0 外部用户，你的流水线会成为第一个真实用户；先只报警 |

---

## 4. 验收方式

统一用 `D:/tempfile/tool/agent-ui-verify/app2`（干净沙箱，runtime 注入 autoload 后可回收）。

**判据一律用应用侧日志，不用工具返回值。** 本次验证已证明两者会不一致：
每个"不该发生"的按钮挂 `pressed` 打印 `（不该发生）`，那行出现即判定作弊/回归。

- P0：自定义信号 → `wait_for_signal` 回 `emitted:true`；超时用例回 `settled:false` 且耗时可解释
- P1：`GET_DRAG_DATA` / `CAN_DROP` / `DROP` 三条标记按序出现在 `get_debug_output` 里
- P2：`get_ui_elements` 返回里 `role` 不为空，且与控件类型一致
- P3：故意改坏一处布局，视觉 diff 应报出偏离百分比

---

## 5. 已知未验证的假设（动手时优先证伪）

1. `push_input(ev, true)` 在 headless 下的 `Window.size` 64×64 stub 问题，**是否也影响 runtime 的 `parse_input_event` 路线**（只影响需要精确坐标的场景？还是全部？）
2. `wait_for_signal` 在桥接往返期间是否会漏掉信号（信号在两次 RPC 之间发出）
3. 拖拽阈值像素（`gui/timers/drag_threshold` 类）在不同 Godot 小版本是否一致
4. 4.7 的 `Control.accessibility_drag()` / `accessibility_drop()` 能否作为「拖放逻辑」的独立测试通道，与「手势能否触发」解耦

---

## 6. 进度

| 阶段 | 状态 |
|---|---|
| 建分支 / 记录约束 | ✅ |
| P0 conditional wait | ⬜ |
| P1 drag | ⬜ |
| P2 semantics | ⬜ |
| P3 evidence | ⬜ |
