# Sephiria Auto Combat（自动索敌 / 自动攻击）

版本 **v1.0.3** · 插件 GUID `com.sephiria.autocombat`

**整个战斗房间**内自动锁定**距离最近的敌人**，并自动朝它挥动**当前武器**进行普攻。目标一直在动，锁定就跟着动 —— **不是**锁住一个就不放。

- **不自动走位**，移动完全由你自己控制
- **不放技能 / 魔法**，只用当前装备的武器普攻
- **不改存档、不改数值、不写任何网络同步变量**

## 功能

- **自动索敌**：你所在的这个战斗房间里有敌对单位时，自动锁定**最近**的那个并持续跟随 —— 有更近的就换。
- **自动攻击**：走**游戏原版的武器攻击链路**（普攻、连招、冲刺攻击），伤害、动画、连段判定、服务器校验都是原版的。
- **自动面向**：会让角色朝向锁定的目标，你按鼠标打出去的每一刀也都飞向它。
- **攻击触发距离按武器实测**，并且**随武器状态实时跟随**（神器、铁砧附魔、武器形态切换都会改判定框，详见下文）。
- **快捷键有屏幕提示**：按下 F5 / F6 会像游戏原生提示一样在屏幕上显示一行当前状态。

## 本版修复（v1.0.3）

**症状**：自动攻击会去打**还没落地**的小怪（从天而降、还悬在半空），以及**还没真正现身**的小怪
（屏幕上只有一个召唤圈或爆炸预警，本体根本看不见）。

**根因**：敌人出场时游戏会先把它**抬到高空并关掉 AI**，再播一段出场动画，最后才让它落地。
四种出场方式都是这么做的（`AvatarSpawn_Apperance_Fall` 把高度设成 `20`，
`_SummonFx` / `_SummonCurcle` / `_Explosion` 设成 `133` 并关掉重力）。
要命的是**"抬起来"这一步发生在单位进入战斗列表之前**，所以它一出现就已经是
「悬空 + 可锁定 + 没死 + 在房间内」的状态，而原来的筛选只看了死亡 / 可选中 / 阵营 / 距离，
**一个高度都没看**。

**修法**：锁敌时**跳过还没就位的单位**。判据不是自己定的，而是抄游戏自己的近炸引信
（`BulletMoveModule_CrossbowMine`）筛目标用的那一条：高度高于阈值、或物体还没进场景，就不算可用目标。
阈值 `5` 也是照抄它的 —— 正常站在地上的单位高度是 `0`，战斗中被击飞一般不到 `3`，
所以**只会咬到出场中的怪**。两处都加了：候选扫描时直接跳过（它不参与"谁最近"的比较），
已锁定的目标每帧复验时也会摘掉。

场景里如果**只有**还没就位的怪，那就是**完全停手**，不会朝它挥空，等它落地再打。
实测一局：11 次跳过（高度 `20.0` 的空降、`133.0` 的召唤各有），每一次之后几行内同一只怪都会
重新出现 `Locked` / `Switched to` —— 落地即恢复攻击。

### v1.0.2 — 法术模式下自动攻击冻结

**症状**：开着自动攻击切到**法术攻击模式**，自动攻击就**再也不动了** —— 直到目标死亡或离开房间才恢复。

**根因**：游戏自己会在好几个地方**悄悄松开攻击键**，而 mod 完全不知道。最典型的就是进法术模式：
`PlayerInputController.HandleOnActivateCastMode` → `CastStop(100)` → `PlayerAvatar.AttackButtonUp()`。
mod 内部仍然以为"键是按住的"，于是每个环节都认为一切正常（对近战 / 远程来说按住本来就是对的操作），
角色就站着不动了 —— 日志里也看不出来，因为那一行显示的还是"正在攻击"。

**同一条根因还有两个触发方式**（本版一并修掉）：

- **你自己点一下鼠标**：非法术模式下松开开火键会走 `HandleOnFire` 的松手分支 → `CastStop(100)`，
  和法术模式是同一行代码。也就是说以前"打着打着随手点一下鼠标"，自动攻击也会永久停住。
- **任何 UI 接管 / 剧情封锁输入**（打开面板、Boss 登场、过场）：`PlayerInputController.OnDisable()`
  里同样会松开一次。

**修法**是一道**反向看门狗**：只要 mod 认为自己按着键，就每 `FireHeartbeatSeconds`（默认 `0.25` 秒）
**回头核实一次游戏自己的开火标志**；发现游戏已经松了，就丢掉自己那份错误记账，**当帧**重新决策 ——
该打就重新按下，该让路就让路。它只做"对账"，不改变任何攻击判定。

**另外新增** `PauseInCastMode`（默认开）：法术模式下主动收手。理由不是"保险"，而是游戏此刻就是这么设计的 ——
进模式时它已经把武器槽停掉了，而且模式内开火键的语义变成了"放法术"。收手还顺带**不再改道你在法术模式下瞄准的法术**
（以前锁着敌人时，你在法术模式里手瞄的魔法也会被打到锁定目标身上）。退出法术模式的**当帧**自动恢复攻击。

## 快捷键

| 按键 | 作用 |
|---|---|
| **F5** | 自动索敌 开 / 关 |
| **F6** | 自动攻击 开 / 关 |

两个键都可以在配置文件里改成别的键。**没有总开关**，想彻底停用请卸载本 mod（或把两个开关都关掉）。

> 自动攻击依赖锁定目标：F5 关掉之后没有目标，F6 开着也不会攻击（但会把手上的按键干净地松开）。
>
> 每次按键都会在屏幕上显示一行提示，例如 `F5 自动索敌 开`。它走的是**游戏自己的系统消息**（无背景、游戏像素字体），不是叠加层。嫌挡视线可以把 `[General] ShowHotkeyToast` 设为 `false`。

## 攻击触发距离是怎么算出来的

"能看到多远"和"能打到多远"是两件事。锁上房间那头的怪**不等于**你会隔着半个房间挥空刀 —— 超出武器的实际可达距离时 mod 会停手，等你走过去再继续打。

这个"可达距离"是**从武器自身的数据实测**的（`[Range] MeasureFromProjectile = true`，默认开），不是手写的估算表：

| 武器类型 | 取值来源 |
|---|---|
| **近战** | 游戏真正生成的那块攻击判定（`MeleeCollision`）的几何尺寸：圆形取 `radius`；矩形 / 胶囊取 `\|offset\| + 尺寸的半对角`。**再加上手到身体的距离**（判定体生成在 `shoulder.swingPoint` 上，每帧重新锚回角色位置，这段必须算进去）。最后按你的 `WeaponRange` 属性放大，跟游戏自己算判定框的方式一样。 |
| **远程** | 子弹**自己在消失前能飞多远**：优先用模块上写的 `maxDistance` / `maxLaserDistance`；否则用 `速度 × 存活时间`（子弹到点即销毁，这就是它的物理极限）；弓的箭矢扫描是游戏里硬编码的 15 格。**子弹不吃 `WeaponRange` 属性**，所以这里也不会乘 —— 否则会给出一个"子弹根本飞不到"的假承诺。 |

实测值还会加上 `TargetBodyRadius`（默认 `0.75`）：距离是**中心到中心**量的，而子弹只要够到敌人身体边缘就算打得到。

### 为什么它需要"实时跟随"

**武器的射程不是常量。** 每一把武器的普攻其实是"按当前状态换一整套开火数据"：

| 武器 | 会切换到的数据集 |
|---|---|
| 剑盾 | 元素套（`CurrentAllElementalSet`） |
| 刀 | 插刀 / 过热 / 月蚀 |
| 弩 | 压缩弹药 / 强化压缩弹药 |
| 大剑 | MP 消耗攻击 |
| 长棍 | 滚动 / 水晶 / 召唤 / 双重连击 |

**神器与铁砧附魔开关的正是这些集合**，而它们的判定框各不相同。所以 mod 会按 `[Range] ProfileRefreshSeconds`（默认 0.5 秒）重新问一次武器"你现在实际会打出去什么"，换到新数据集时射程立刻跟上 —— 不会出现"强化了反而打不到"。

如果某把武器的实测数字不合你的预期，用 `[Range] RangeOverrides` 直接压一个值，例如 `WeaponSimple_Crossbow=18`。哪把武器解析成了什么、依据是什么，日志里都会写（见「排错」）。

## 前置条件

必须先装好 **BepInEx 6（Unity Mono x64）**。

判断是否已装：打开游戏根目录，同时存在下面三项即已装好：

```
winhttp.dll
doorstop_config.ini
BepInEx\
```

如果你装过带 BepInEx 的整合包，那就已经具备了，直接跳到安装步骤。

## 安装

1. 关闭游戏。
2. 把压缩包里的 `BepInEx` 文件夹**直接拖进游戏根目录**（即 `steamapps\common\Sephiria\`），提示合并时选"是"。
3. 启动游戏。

装好后应该长这样：

```
steamapps\common\Sephiria\
├─ BepInEx\
│  ├─ plugins\SephiriaAutoCombat.dll
│  └─ config\com.sephiria.autocombat.cfg
```

## 卸载

删掉 `BepInEx\plugins\SephiriaAutoCombat.dll` 即可（配置文件可以留着，也可以一并删掉）。

## 配置

配置文件：`BepInEx\config\com.sephiria.autocombat.cfg`
改完保存，重进游戏生效。压缩包里的 cfg 只是**默认值备份**；如果它和你机器上的不一致，以游戏实际生成的那份为准。

> **注意**：BepInEx **不会**覆盖你已存在的旧值。升级本 mod 后，新增的配置项会自动补进你的 cfg，但**默认值变了的旧项不会自动更新** —— 需要手动改（安装包里的 cfg 就是当前推荐值）。

### Keys（快捷键）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `ToggleAutoTargetKey` | `F5` | 自动索敌开关的键，填 Input System 的键名（如 `F5`、`G`） |
| `ToggleAutoAttackKey` | `F6` | 自动攻击开关的键 |

### Targeting（索敌）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `LockRangeMode` | `Room` | `Room` = 锁定范围是**你所在的整个战斗房间**（默认）。`Radius` = 按半径算 |
| `RoomMargin` | `2` | 仅 `Room`：房间矩形每边往外扩多少格仍算"在房间里"（覆盖门口和墙厚）。`0` = 严格按房间矩形 |
| `TargetRadius` | `10` | **仅半径模式**（或房间读不出来时的兜底）用的半径下限。实际半径 = `max(这个值, 武器范围 × LockRadiusMargin + LockRadiusSlack)` |
| `SwitchHysteresis` | `0` | **防抖死区**。`0` = 纯最近，谁近锁谁。填 `1.5` = 新目标要比当前目标近 1.5 格以上才抢得走 |
| `LostDistanceFactor` | `1.4` | 仅半径模式的保留倍数：目标跑出锁定圈后不会立刻丢，还留着 |
| `ReleaseSeconds` | `1.5` | 目标跑出保留范围后，还留着锁定多久。丢掉之后下一次扫描按"最近"重新选 |
| `ScanInterval` | `0.1` | 每隔多久重新扫一次目标（这就是锁定跟随最近的刷新频率；瞄准和攻击仍是每帧执行） |
| `RequireInBattle` | `false` | `true` = 只有游戏标记你「战斗中」时才锁敌；`false` = 范围里有敌人就锁 |
| `ExcludeDummy` | `true` | 是否排除训练假人 / 稻草人（它们会把「战斗中」标志拉起来） |
| `PreferBoss` | `false` | `true` = 有 boss 就优先锁 boss（**会覆盖"永远锁最近"**）。`false` = 一律锁最近的 |
| `LogTargetChanges` | `true` | 锁定 / 切换 / 放弃目标时打日志 |
| `IgnoreSpawnPending` | `true` | 跳过**还没就位**的敌人（未落地 / 还没现身，见「本版修复（v1.0.3）」）。`false` = 恢复成「能锁就打」，不管它在多高的地方 |
| `SpawnPendingHeight` | `5` | 高度超过这个值就算「还没就位」。高度是俯视角的**高度轴**（`transform.position.z`），`5` 抄自游戏自己的选敌条件。正常地面怪是 `0`，被击飞一般不到 `3`，所以只咬出场动画 |

### Range（按武器决定攻击范围）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `MeasureFromProjectile` | `true` | **从武器自身数据实测可达距离**（推荐，见上）。`false` = 退回下面的手写表 |
| `ProfileRefreshSeconds` | `0.5` | 每隔多少秒完整重测一次手上的武器。神器 / 铁砧 / 武器状态会换掉实际开火数据，这一步就是跟随它们的。`0` = 只在换武器或 `WeaponRange` 属性变化时重测（更省，但可能错过数据集切换） |
| `TargetBodyRadius` | `0.75` | 加在实测值上，够到敌人身体边缘就算在射程内。`0` = 用原始实测值。只对实测值生效 |
| `AutoWeaponRange` | `true` | 实测失败时的兜底：按内置武器表（匕首 4 / 剑盾 5 / 刀 6 / 大剑·杖 6.5 / 弓 15 / 弩 16）。`false` = 不用表 |
| `MeleeRadius` | `5` | 实测失败且不在表里的近战武器，用这个兜底 |
| `RangedRadius` | `14` | 实测失败且不在表里的远程武器，用这个兜底 |
| `WeaponRadiusScale` | `1` | 整体倍率，`1` = 不缩放 |
| `ApplyWeaponRangeStat` | `true` | 近战范围按你的 `WeaponRange` 属性缩放（`1 + 属性/100`），跟游戏自己算判定框一致。对硬覆盖的 `AttackRadius` 不生效 |
| `RangeOverrides` | （空） | 逐武器覆盖，格式 `武器类名=范围`，多条用 `;` 隔开。例：`WeaponSimple_Bow=18; WeaponSimple_Katana=7`。**优先级最高**，用来纠正不合理的实测值 |
| `LockRadiusMargin` | `1.15` | 仅半径模式：锁敌半径 = `max(TargetRadius, 攻击范围 × 此值 + LockRadiusSlack)` |
| `LockRadiusSlack` | `1` | 仅半径模式：在上面基础上额外加的固定距离 |
| `LogWeaponProfile` | `true` | 武器解析结果**变化时**打一行「武器 / 节奏 / 攻击范围 / 来源」，方便你核对（不变时不打，避免刷屏） |

### Attack（出招节奏）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `AttackMode` | `Auto` | `Auto` = 按武器自动（推荐）；`Hold` = 一直按住；`Repeat` = 固定节奏「按 → 松」 |
| `PressDuration` | `0.15` | 仅 `Repeat`：每次按住的时长（秒） |
| `PressInterval` | `0.25` | 仅 `Repeat`：两次开始按键之间的间隔（秒） |
| `AttackRadius` | `0` | **硬覆盖**：填正数则**所有武器**都用这个攻击距离，会绕过上面的实测和表。`0` = 关闭（推荐） |
| `AttackOnlyWithWeapon` | `true` | 手上没武器时不去按攻击键 |
| `ChargeReleaseRatio` | `0.999` | 蓄力武器（弓）：蓄力到这个比例就松手。`0.999` = 拉满 |
| `ChargeTimeoutSeconds` | `2.5` | 蓄力武器：按钮最多按住多久。读不到蓄力进度时也用这个值当按住时长 —— 保证任何情况下都不会一直按着 |
| `PulseRecycleSeconds` | `0.25` | 蓄力武器：松手放箭后，隔多久开始下一次蓄力 |

### Aim（瞄准）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `AimAtTarget` | `true` | 把**游戏自己的准星**（`PlayerAvatar.NetworkaimObject`）钉到锁定目标上。**这才是让你自己按鼠标打出去的攻击也飞向敌人的开关**：每把武器都是「一按下就出招」，方向在按下的那一刻就定了，不钉准星的话角色会转身对着敌人、但刀仍然朝鼠标飞。设 `false` 则只标定目标、完全不干预瞄准 |
| `ForceFacing` | `true` | 同时把 `aimedPositionClientside` 也指向目标，让肩部和角色朝向跟过去，按住连击的后几刀也跟得住移动中的目标。出现瞄准表现异常可以关掉 |

### Safety（让位）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `YieldToGame` | `true` | 对话 / 过场动画时让位，不抢输入 |
| `PauseWhenInputBlocked` | `true` | 游戏屏蔽角色输入时（读盘、菜单、剧情）暂停 |
| `PauseWhenUiOpen` | `true` | 打开游戏内面板时暂停 |
| `PauseInSafeMode` | `false` | 游戏把角色标记为"安全模式"时是否停手。**默认关闭是故意的**：游戏只用它来**拒绝非敌对目标的伤害**，而本 mod 只锁敌对目标 —— 开着它一刀都拦不住，只是拒绝挥向游戏本来就允许打的东西。开场新手村事件期间它一直是开的，所以以前会看起来"刚进游戏自动攻击就废了"。想要保险可以把 `PauseInSafeMode` 设回 `true` |
| `OnlyLocalPlayer` | `true` | 只操作本地玩家角色 |
| `ReleaseRetrySeconds` | `2` | 游戏超过这么久还没接受松手指令（角色不能移动时它会静默忽略），就升级处理：打警告，能强制清位就强制清 |
| `HardReleaseFallback` | `true` | 最后的兜底：直接清掉游戏内部的 `isFireButtonDown` / `isAttackButtonDown` 开关。**只有主机 / 离线模式可以**（纯客户端不拥有那份状态，只能靠不停重发松手） |
| `PauseInCastMode` | `true` | 法术攻击模式期间停手（见 v1.0.2 修复）。游戏在进模式时会自己把武器槽停掉，模式内开火键的语义是「放法术」，所以让路才是顺着的做法，而且能让 mod 不再改道你手瞄的法术。**退出模式的当帧自动恢复攻击**。想让它法术模式下也照样挥武器就设 `false` |
| `FireHeartbeatSeconds` | `0.25` | mod 认为自己按着攻击键时，多久回头核实一次游戏自己的开火标志（反向看门狗，见 v1.0.2 修复）。`0` = 关闭核实。改大更安静、更省，改小发现得更快；正常玩没必要动 |

### General（常规）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `AutoTargetEnabled` | `true` | 启动时自动索敌就是开的 |
| `AutoAttackEnabled` | `true` | 启动时自动攻击就是开的 |
| `ShowHotkeyToast` | `true` | 快捷键按下时在屏幕上显示一行提示（走游戏原生系统消息）。`false` = 屏幕完全干净，只剩日志 |
| `MessageSeconds` | `2.5` | 提示停留多久（秒）。`2.5` 与游戏原生提示一致 |
| `HeartbeatSeconds` | `15` | 每隔多少秒打一行状态摘要（含当前武器、锁定范围、目标、当前是否在攻击），`0` = 关闭 |
| `DebugLog` | `false` | 详细日志（每次扫描、每次按键、每次松手），排查问题用 |

### Diagnostics（诊断）

| 配置项 | 默认 | 说明 |
|---|---|---|
| `LogWeaponProbe` | `true` | 每种武器首次装备时，打印**实测到的攻击距离以及它是怎么算出来的**（近战判定框 / 子弹速度 × 寿命），以及原始判定框尺寸和蓄力 / 连招布局。觉得某把武器的触发距离不对，就把这行日志发出来 —— 这是它可诊断而不是靠猜的原因。日常觉得吵可以关掉 |

## 排错

日志文件：`BepInEx\LogOutput.log`

- **完全没反应**：
  1. 先在日志里搜 `Sephiria Auto Combat`，确认 mod 加载成功（会打印版本、两个开关、攻击模式、瞄准开关、锁定范围、射程来源、刷新周期、快捷键）。
  2. 按 F5 / F6 看有没有屏幕提示 + `Auto-target ON/OFF` 的日志。**都没有**说明快捷键没生效（可能被别的 mod 占用，或者配置里键名写错了）—— 换一个键再试。
  3. 把 `DebugLog` 改成 `true`，再看是否出现 `Switched to <敌人名>`。
- **"自动攻击失效了"**：日志里直接搜 `not attacking:`。每个能中断攻击的分支都会**自报家门**，不会让你猜：
  - `auto-target is switched off, so there is nothing to attack` → F5 被关了
  - `no target` → 范围内没有敌对目标
  - `target is out of range (d=4.2 > 3.5, measured WeaponSimple_Katana)` → **超出武器实际射程**，走过去即可；觉得数值不对就用 `RangeOverrides` 压
  - `yielded: a game panel is open` / `a conversation is playing` / `the game is blocking player input` → 让位判定，分别是 `PauseWhenUiOpen` / `YieldToGame` / `PauseWhenInputBlocked`
  - `yielded: safe mode` → 见上面 `PauseInSafeMode`
  - `the game is not accepting attack input (CanMove false)` → 游戏此刻不接受角色输入
  这些只在**状态变化时**打一行（并带 1 秒限流），所以不会刷屏。把这几行发出来就能定位到底哪一条在挡。
- **自动攻击卡住不动了**：日志里搜 `re-synced`，这是反向看门狗在工作（正常情况下应当**每次都能自愈**）：
  - `... while the mod still held it (magic cast mode is active) - re-synced` / `(the game blocked avatar input)` 是**警告**级别：游戏把武器收回去了（法术模式会让位、等它结束自动继续）。
  - `... (the player released the fire button) - re-synced` 是**信息**级别（限流 5 秒）：就是你自己点的鼠标，属于常规事件。
  - 如果**搜不到** `re-synced`、自动攻击却依然卡住，把这一整段日志发出来 —— 说明还有一条没覆盖到的路径。
- **法术模式下自动攻击停手**：这是 `PauseInCastMode` 的预期行为，日志里会写 `not attacking: yielded: magic cast mode is active`。想让它法术模式下也挥武器，把 `PauseInCastMode` 改成 `false`。
- **想知道锁的是哪个范围**：日志里搜 `lock area =`。房间模式会打房间矩形的坐标和尺寸，例如 `lock area = room (12,-8)..(46,20) 34x28`；进新房间、或从房间走到走廊时会各打一行。如果显示的是 `radius` 而不是 `room`，说明这一层的房间没读出来（会有一条警告说明原因），把日志发我。
- **触发距离不对 / 想知道依据**：日志里搜 `rangeFrom=`。会写成 `rangeFrom=BulletMoveModule_... 15.0/s x 0.90s` 或 `rangeFrom=MeleeCollision_Rectangle ...` 这种"数字 + 出处"的形式；来自状态切换后的数据集会标 `+state set`。拿它和实测感受对一下；不对就用 `RangeOverrides` 压。
- **换神器 / 铁砧强化后打不到了**：正常情况下本版会自动跟随（`ProfileRefreshSeconds`）。日志里 `weapon <类名> ... attackRange=x.x (was y.y)` 那行就是变化记录，把这个发出来即可。若刷新周期被设成了 `0`，改回 `0.5` 或更大。
- **怪就站在旁边却不打**：先搜日志 `still spawning`。有这行说明它**还在出场动画里**（未落地或还没现身），
  这是 v1.0.3 的预期行为 —— 几行之后它会重新出现 `Locked` / `Switched to`，落地就开打。
  如果它**明显已经站在地上**、日志却还在刷这行，把 `SpawnPendingHeight` 调大（例如 `8`）、
  或者把 `IgnoreSpawnPending` 设成 `false`，并把那段日志发出来。
- **打的不是我想要的那个敌人**：日志里搜 `Switched to` —— 每次换目标都会打一行 `Switched to X (dist ..., was Y)`。如果换得太频繁，把 `SwitchHysteresis` 调大（例如 `1.5`）；如果它迟迟不换，确认 `PreferBoss` 是 `false`。
- **弓不射箭 / 一直拉不开**：日志里 `kind=ChargedRanged` 才是被识别成了蓄力武器。如果显示 `kind=Ranged` 说明你这把弓没有蓄力字段，那就应该按住即射；把日志发出来。
- **出现 `attack button was stuck held`** 警告：游戏在角色不能移动时忽略了松手指令，mod 已经重试 / 兜底清位了。如果反复出现请把日志发出来。
- **打完怪还会挥一两下**：这是"按住连击"的固有延迟（当前那一刀要挥完），不是没松手。日志里 `attack release` 之后就不会再起新刀了。
- **和别的 mod 冲突**：本 mod 的 4 个 Harmony 补丁全部只做**只读 + 重定向瞄准**，不修改任何返回值、不写任何网络同步变量，冲突概率很低。如果怀疑冲突，把 `AimAtTarget` 和 `ForceFacing` 都改成 `false` 再试。

## 已知限制

- **锁定范围是"房间矩形"，不是"可达区域"**：游戏没有视线检查，所以如果某层的房间矩形把墙外的空间也算进去了（少见），仍然可能锁到一个打不到的敌人。因为 mod 不走位，实际影响只是"朝向"和日志。真遇到就把 `LockRangeMode` 改成 `Radius`。
- **走廊 / 非房间区域没有房间数据**：站在走廊里时会退回按半径锁（默认 10）。走廊里本来就很少打起来，影响很小。
- **多人联机**：每个装了本 mod 的客户端各自控制自己的角色。攻击走的是游戏原版的服务器指令，服务器会照常校验，别的玩家看不到任何异常。房间数据是各客户端用同一个种子本地生成的，所以锁定范围在主机和客户端上是一致的。
  - 如果你是**纯客户端**（不是主机），游戏内部那个"按住"开关在服务器那一份实例上，客户端碰不到。mod 会用不停重发松手指令的方式修，这条路是有效的；但 `HardReleaseFallback` 那层兜底只在主机 / 离线生效，日志里会说明。**瞄准重定向是客户端本地生效的，主机 / 纯客户端都一样有效。**
- **出场中的敌人不会被打**（v1.0.3 起）：未落地 / 还没现身的怪会被跳过，落地后自动开始打 —— 这是有意为之。
  万一某个特殊敌人天生一直悬在半空，它就会被一直跳过，那种情况把 `SpawnPendingHeight` 调小或设 `0`，
  或直接关掉 `IgnoreSpawnPending`。
- **同时面对多个敌人时会来回换目标**：这是"始终锁最近"的固有结果。想让它"认准一个打"就把 `SwitchHysteresis` 调到 `1.5 ~ 3`。
- **不会微调走位**：目标横向跑动时，攻击方向是在每次出招瞬间取的，所以极快速横移的敌人可能偶尔落空 —— 这是「不自动走位」这个前提的必然结果。
- **训练假人默认排除**：想打假人练手请把 `ExcludeDummy` 改成 `false`。
- **法术模式之外**，你用技能键（0~7 号技能槽）放的法术，如果此刻锁着敌人，落点仍会被改道到锁定目标上（法术和武器共用同一个瞄准位置）。法术模式内不受影响（本版起 mod 整段让位）。
- **`PauseInCastMode` 默认开启**：法术模式期间自动攻击会停手（有日志说明），退出模式当帧恢复。不喜欢就设 `false`。
- **`PauseInSafeMode` 默认关闭** 是基于代码逻辑（游戏只对非敌对目标用 `safeMode` 拒绝伤害）判断的，不是实机逐一验证。若你在安全区里发现打了不该打的东西，把它设回 `true`。

---

## English quick version

**Sephiria Auto Combat v1.0.3**

Auto-locks the **nearest** hostile enemy **inside the whole battle room you are standing in**,
and drives the vanilla weapon attack chain at it. It never moves your character, never casts
skills, never writes any SyncVar or save data.

- **F5** toggles auto-targeting, **F6** toggles auto-attacking (change them under `[Keys]`).
  Each press reports itself on screen through the game's own system-message element.
- Requires **BepInEx 6 (Unity Mono x64)** already installed.
- Install: copy the bundled `BepInEx` folder into the game root and merge.
- Uninstall: delete `BepInEx\plugins\SephiriaAutoCombat.dll`.
- Config: `BepInEx\config\com.sephiria.autocombat.cfg`
- Log: `BepInEx\LogOutput.log` (set `DebugLog = true` for detail; search `not attacking:`,
  `lock area`, `rangeFrom`).

**The lock area** is the whole battle room (default `LockRangeMode = Room`). The rectangle is
read out of the game's own floor data - the same rectangle the game tests against when it
reveals a room on the map. Enemies across the room are valid targets, and it is still always
the nearest one. A floor whose rooms cannot be read (or a corridor) falls back to a radius and
says so in the log - it never silently does nothing.

**The attack trigger range is measured, not estimated** (`MeasureFromProjectile = true`),
and it is re-measured while you play (`ProfileRefreshSeconds`, default 0.5s). That matters
because a weapon's reach is not a constant: artefacts, an anvil enchantment, a weapon form
change and the weapon's own per-swing states (elemental sets, blade overheat, eclipse,
compressed ammo, MP attacks) all swap the fire data the weapon actually fires, and those sets
have different hitboxes. Melee: the real hitbox the game spawns (`MeleeCollision` offset +
size) plus the hand-to-body offset, scaled by your `WeaponRange` stat exactly the way the game
scales its own hitbox. Ranged: how far the weapon's own projectile can travel before it is
destroyed (speed x lifetime, or an explicit max distance, or the bow's hard-coded 15-unit
arrow scan). Projectiles ignore `WeaponRange`, so measured ranged ranges do not use it either -
that would promise a shot the bullet cannot make. Override any weapon with
`[Range] RangeOverrides` (e.g. `WeaponSimple_Crossbow=18`); the log prints the measurement and
its source.

**Aim redirection** (`AimAtTarget = true`): the game's own aim transform
(`PlayerAvatar.NetworkaimObject`, normally driven by the mouse) is pinned onto the locked
enemy. This is what makes *your own* attacks hit the target: every weapon swings the instant
the attack button goes down, along whatever direction that transform produced, so without it
the character turns to face the enemy while the swing still flies at the cursor. The dodge
direction is deliberately left alone.

**The game also releases the attack button behind the mod's back** (v1.0.2). Entering magic
cast mode kills the weapon attack itself (`HandleOnActivateCastMode` -> `CastStop(100)` ->
`PlayerAvatar.AttackButtonUp`), so do a manual click while auto-attack is running (the fire
release runs the same code) and any UI takeover or cutscene (`PlayerInputController.OnDisable`).
The mod used to keep believing the button was held, which made every part of it report
"attacking" while the character stood still for the rest of the fight. A reverse watchdog now
re-checks the game's own fire flag every `FireHeartbeatSeconds` (default 0.25s) and re-syncs on
the spot - re-pressing if attacking is still allowed, standing down if the condition is a yield.
`PauseInCastMode` (default on) stands the mod down for the whole of cast mode, which is what the
game expects and also keeps it from redirecting the spell you aim by hand; auto-attack resumes
the frame cast mode ends.

**Enemies that have not landed yet are not targets** (v1.0.3). Spawned enemies are lifted to
height 20 (falling) or 133 (summon / explosion) with their AI disabled *before* they enter the
combat list, so the old filter - death, targetable flag, faction, distance - happily locked onto
something still mid-air or not visible at all. The lock now skips a unit whose altitude is above
`Targeting.SpawnPendingHeight` (default 5, the same cutoff the game's own proximity mine uses),
both while scanning and while re-validating a held target. If such a unit is the only thing
around, the mod stops attacking rather than swinging at air.

**Diagnosable, not guesswork**: every path that can stop attacking reports why - switched off,
no target, out of range (with the distance and the measured limit), yielded (naming which
rule), game not accepting input. The log prints one line when the reason changes.

**Safety**: every stop path funnels through one release function, a watchdog re-sends the
button release until the game confirms it, and host/offline can force-clear the weapon's
internal latch - so the character never keeps swinging in an empty room. The mod yields during
dialogs, cutscenes, menus and while the game blocks your input. `PauseInSafeMode` is off by
default, deliberately: the game only uses `safeMode` to deny damage to *non*-hostile targets,
and this mod only ever locks hostiles.

**Known limits**: no pathing (out-of-range targets are simply not attacked), room rectangles
are not line-of-sight (walls are not checked), and facing multiple enemies at similar ranges
makes the lock follow whichever is closest (raise `SwitchHysteresis` to commit to one).
