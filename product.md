# 产品需求文档 (PRD) - JigShare (拼图快传)

**文档版本**: v6.8  
**更新日期**: 2026-03-26  
**平台**: iOS 17.0+ (iPhone 优先)

---

## 目录

1. [产品概述](#1-产品概述-product-overview)
2. [核心玩法与功能](#2-核心玩法与功能-core-features)
3. [异常、中断与权限管理](#3-异常中断与权限管理-edge-cases)
4. [社交与裂变机制](#4-社交与裂变机制-social--sharing)
5. [UI 设计与界面流程](#5-ui-设计与界面流程-ui-flow)
6. [新手引导](#6-新手引导-ftue)
7. [全局设置与历史记录](#7-全局设置与历史记录-settings--history)
8. [预设游戏模式 (Game Presets)](#8-预设游戏模式-game-presets)
9. [后续迭代储备](#9-后续迭代储备-backlog)
10. [.jig 文件格式](#10-jig-文件格式)
11. [云端功能规划](#11-云端功能规划-cloud-feature-roadmap)
12. [个人资料与账户管理](#12-个人资料与账户管理-profile--account)

> §11.6 为技术实施参考（数据模型、Firestore Rules、Callable 规格、迁移清单）。


---

## 1. 产品概述 (Product Overview)

### 1.1 产品定位

一款基于用户真实照片、主打社交互动的轻量级网格拼图游戏。

### 1.2 核心价值

将传统的「寻找边缘」拼图转变为快节奏的「空间记忆与位置交换」。通过极简的固定网格设计、无缝的隐式动画以及强社交挑战属性，打造低门槛、易裂变的碎片化娱乐体验。

---

## 2. 核心玩法与功能 (Core Features)

### 2.1 游戏设置与图片处理

| **图片来源** | 用户从系统相册选取照片（单张或多张，最多 5 张） |
| **内容限制** | 系统不限制图片内容（纯色、极端长宽比均可），保留熟人间相互恶搞的乐趣 |
| **单图挑战 (Single)** | **经典模式**：用户选择单张图片进行挑战，侧重于单次对局的极致用时竞技 |
| **多图挑战 (Pack)** | **v4.0 新增**：支持一次性选择多张照片组成「挑战包」(Pack)。玩家需顺序完成该包内的所有拼图层级 |
| **比例与切割** | 游戏直接采用照片的原长宽比例，按设定的难度切割成 N×M 网格，每个碎片均为标准矩形 |
| **难度选择** | 在创建界面用户**分别选择行数 (Rows) 和列数 (Columns)**，取值范围均为 1～9，支持任意组合（如 2×3、4×5、9×1 等） |
| **分辨率处理** | **支持原图**：不进行降采样，直接使用照片原始分辨率；**结算态**：拼图成功后，瞬间替换回原画质无损高清照片 |

### 2.2 初始状态与洗牌机制 (Initialization & Shuffling)

| 项目 | 说明 |
|------|------|
| **固定网格** | 所有碎片均在居中的矩形网格内固定排列，无散落碎片，无需收纳槽 |
| **网格间距** | **零间距**：碎片间无缝隙，视觉上拼接为完整照片；胜利时渐变切换至高清原图 |
| **乱序洗牌** | 游戏开始时，系统生成随机种子（Seed），将所有碎片在网格中完全随机打乱位置 |
| **初始翻转** | 洗牌时，每个碎片有 50% 概率处于 180° 翻转状态（上下颠倒）；**不支持** 90°/270° 旋转。若「允许翻转」设置关闭，则跳过翻转随机化，所有碎片初始均为正向 |

### 2.3 游戏内交互 (Gameplay UX - 核心状态机)

用户核心操作：**点击、交换、翻转**。不支持手势拖拽、双指缩放及画布平移。

#### 2.3.1 选中与取消 (Tap to Select/Deselect)

- 点击任意未选中碎片 A → 触发高亮：明亮内边框、轻微放大悬浮（增加阴影）、轻微触觉震动
- 再次点击已选中的碎片 A → 取消选中，恢复初始状态

#### 2.3.2 位置交换 (Swap)

- 碎片 A 已选中时，点击另一块碎片 B
- 碎片 A 和 B 通过弹簧动画互换网格位置（动画参数见 [2.4 动画参数](#24-动画参数-animation-spec)）
- 互换完成后，自动清空选中状态

#### 2.3.3 原位翻转 (Flip 180°)

- **方式一**：双击网格中的任意碎片 → 180° 翻转（正向 ↔ 倒转）
- **方式二**：选中某碎片后，点击顶部悬浮栏 [🔄] 旋转按钮

#### 2.3.4 辅助原图 (Peek)

- 顶部悬浮栏 [👁️] 按钮
- 长按：所有碎片变为半透明，底层显现原图供参考
- 松开：立刻恢复

#### 2.3.5 计时器 (Timer)

- 位置：界面顶部显著位置
- 触发：从用户第一次点击任意碎片开始计时
- 格式：`MM:SS`（例：`05:30`），等宽数字字体防止抖动

### 2.4 动画参数 (Animation Spec)

以下为暂定参数，开发时可微调以优化手感：

| 动画类型 | 参数 | 建议值 | 说明 |
|----------|------|--------|------|
| **位置交换 (Swap)** | Spring | response: 0.45, dampingFraction: 0.8 | 略带弹性，收尾干脆 |
| **选中高亮 (Select)** | 缩放 + 阴影 | scale: 1.12, shadow radius: 20, y: 10 | 强烈的“拿起”悬浮感 |
| **取消选中 (Deselect)** | 过渡 | duration: 0.15s, ease-in | 快速恢复 |
| **180° 翻转 (Flip)** | 过渡 | duration: 0.3s, ease-in-out | 翻转半周 |
| **结算切换至原图** | 过渡 | duration: 0.5s, ease-in-out | 缝隙消失 + 渐变 |
| **胜利庆祝 (Confetti)** | 粒子 + 延迟 | duration: 2.5s | 庆祝动画 + 结果页延迟 |

> 注：iOS 弹簧可用 `Animation.spring(response: 0.45, dampingFraction: 0.8)` 等价实现。

### 2.5 胜利判定与结算

| 项目 | 说明 |
|------|------|
| **胜利条件** | 每个碎片都在正确索引位置，且翻转状态全部为 0°（未翻转） |
| **表现反馈** | **庆祝序列**：拼好最后一块 → 瞬间切换至高清原图（隐藏网格线）→ 屏幕爆发**彩带粒子 (Confetti)** → 显示“Perfect Fit!”徽章 → 延迟约 2.5s 后平滑弹出结算面板 |
| **单图计时** | 结算页显著展示本次挑战的 `MM:SS` 成绩，并与历史最高纪录对比 |
| **关键帧反馈** | 交换碎片时，若其中一块恰好回到正确位置：触发极轻微「咔哒」Haptic + 绿色描边高亮该碎片约 0.55s；设置开关关闭时不触发；胜利时不额外触发（胜利 Haptic 优先） |
| **正确锁定 (Correct Lock)** | **v2.8 新增**：当碎片处于绝对正确的位置且方向正确时，触发清脆触觉反馈（UI 呈现绿色边框强调），并锁定该碎片，使其无法再被移动。默认关闭。 |
| **错误提示 (Error Hint)** | **v2.8 新增**：游戏页面添加提示按钮。点击后，随机将 1 个处于错误位置（位置错或方向错）的碎片短暂标红。默认关闭（未开启时隐藏或置灰）。 |
| **禅系模式 (Zen Mode)** | **v4.2 新增**：无计时器的放松模式。界面顶部隐藏计时器，结算页不展示时间与纪录对比。提供纯粹的拼图放松体验。 |
| **魔法提示 (Magic Hint)** | **v4.3 新增**：智能引导。点击后，系统自动寻找一个位置或方向错误的碎片，并将其移动到正确的位置（或纠正方向）。伴随平滑的交换动画。 |

---

## 3. 异常、中断与权限管理 (Edge Cases)

### 7.1 进度的保存与恢复 (Auto-Save & Continue)

- **自动存档**：游戏过程中自动保存当前网格状态、旋转角度、已消耗时间及当前关卡索引（多图模式下）。
- **首页恢复 (Continue)**：**v4.0 优化**：首页（SetupView）根据是否存在有效存档动态显示「继续游戏 (Continue Last Game)」按钮。若无存档，该按钮完全隐藏。
- **国际化**：恢复功能的提示语（如「继续上次进度」）完整支持英、中、西三语。

### 7.2 历史记录
- **自动保存**：**v6.5 优化**：单图及多图本地游戏（非挑战）在完成后自动写入历史记录。**挑战游戏完成后不再自动保存至历史记录**，用户可在挑战中心查看其成绩。
- **记录类型**：区分单图记录与多图包记录。
- **视觉增强**：**v6.5 优化**：移除了历史列表中的“📦 图包 (Pack)”与“🏆 挑战 (Challenge)”勋章，使界面更加清爽。
- **日期精度**：**v6.5 优化**：历史列表中的日期精确到分钟（例如：`Mar 26, 2:33 PM`）。
- **再次游玩 (Play Again)**：**v6.4 新增**：单图与多图详情页均提供「Play Again」入口，以 Modal Sheet 进入游玩页；沿用该条记录保存的游戏规则（Peek/Flip/Ghost/CorrectLock/ErrorHint、Zen、每关行列数等），**每局使用新的随机 seed**（非同一盘面重播，亦非与好友成绩对比的挑战模式）。
- **多图关卡规则持久化**：多图包 `LevelRecord` 除每关用时与行列外，同步保存每关 `allowPeek` / `allowFlip` / `ghostModeEnabled` / `correctLockEnabled` / `errorHintEnabled`，旧存档缺省字段时解码为安全默认值，保证向后兼容。

---

## 4. 社交与裂变机制 (Social & Sharing)

### 4.1 发起挑战 (Sender)

1. 玩家在**结算页**或**游戏进行中**点击「挑战朋友」（游戏页右上角分享按钮）
2. 弹出**挑战寄语输入 Sheet**，用户可输入一句话（可选，限 50 字），点击「Share」确认
3. 系统在内存中构建后缀为 `.jig` 的自定义数据，封装内容：
   - 压缩版照片数据（JPEG 0.88，最大边 1024px，强制 RGB 去除 Alpha）
   - 网格参数 (N×M)、随机种子 (Seed)
   - 发起者用时 (`senderTime`)
   - 发起者昵称 (`senderName`，优先取设置页填写的昵称，否则使用设备名)
   - 挑战寄语 (`message`，可选)
   - 游戏规则快照：`allowFlip`、`allowPeek`、`keyFrameHapticEnabled`、`ghostModeEnabled`
4. 唤起 iOS 系统分享面板 (Share Sheet)，通过 iMessage 或微信发送

### 4.2 接受挑战 (Receiver)

1. 朋友点击 `.jig` 文件，通过 iOS 文件关联机制唤起 App
2. App 解析文件后，展示全屏**「挑战书」视图 (ChallengeLetterView)**，而非直接进入游戏：
   - 信封入场动效 → 展开 → 卡片滑入
   - 主文案：「{senderName} challenged you」（无昵称时「You received a puzzle challenge」）
   - 副文案：「They solved it in {time}」（无用时时「Can you complete it?」）
   - 挑战寄语（若有），斜体引用样式展示
   - 模糊图片预览条（左上角 1/4 区域，高斯模糊，制造悬念）
3. 用户点击「Accept Challenge」→ 过渡动画 → 进入游玩页
4. **公平竞技保障**：使用相同 Seed 初始化网格；`allowFlip`、`allowPeek`、`keyFrameHapticEnabled`、`ghostModeEnabled` 均以文件值覆盖本地设置
5. **结果实时反馈**：接收方完成挑战后，挑战列表（SocialHubView）立即展示「胜利 (VICTORY)」或「失败 (DEFEAT)」勋章。
6. **挑战书增强**：用户打开挑战书时，若双方均有成绩，直接以大字体展示对决结果（Victory/Defeat），并提供「查看对决结果」详情入口。
7. 提供「返回游戏」按钮，方便用户从挑战详情一键返回列表。
8. 提供「挑战朋友」按钮，引导其转化为发起者（社交裂变）。

---

## 5. UI 设计与界面流程 (UI Flow)

### 5.1 视觉语言

- **壳层 UI（导航、列表、表单、登录等）**：**波普风格 (Pop Art)**——高饱和色、粗黑描边、硬边投影；详见 **`UIUX-DESIGN.md` v4.0** 与代码 `PopArtTokens.swift`。
- **游玩区**：中间彩色照片网格仍为 **视觉焦点**；顶部 HUD 等以可读性与 PRD 动效为准，不必与壳层同一套圆角/毛玻璃叙述。
- *历史表述*：早期文档中的「现代极简、毛玻璃」已让位于壳层波普化；若文内个别章节仍出现旧措辞，以 `UIUX-DESIGN.md` 为准。

### 5.2 游玩页布局

- 游玩区为紧凑占满安全区域的 LazyVGrid（无需拖拽空间）
- **支持横屏**：布局随屏幕方向自适应，竖屏/横屏均可游玩

### 5.3 核心路径

```
新挑战页（SetupView）
├── [选择照片]
├── 分别选择行数、列数（1～9，实时预览切割位置）
├── [开始挑战]
├── NavigationBar [🕐 历史记录]  ── Push ──▶  历史记录列表页
│                                               └── tap ──▶  历史详情页
├── NavigationBar [✉️ 挑战]      ── Push ──▶  收到的挑战列表（SocialHubView）
└── NavigationBar [👤 个人中心]  ── Sheet ──▶  Profile Hub (SettingsView)

profileSection（在 SettingsView 中）
├── 个人资料：头像、昵称、好友码
├── 社交入口：好友列表 (Friends)
└── 账户操作：退出登录 / 删除账户（仅登录可见）

游玩页（GameView）
├── 顶部毛玻璃悬浮栏：[← 返回] | 计时器(HH:MM:SS) | [👁️ 查看原图] | [🔄 翻转] | [💡 提示] | [📤 分享]
│   注：allowPeek=false 时 👁️ gray out；allowFlip=false 时 🔄 隐藏；errorHintEnabled=false 时 💡 隐藏
│   [📤 分享] 按钮始终可见，触发「挑战寄语 Sheet → Share」与结算页逻辑完全一致
└── 中央占满屏幕的紧凑网格

结算页（ResultScreen）
├── 完整高清原图展示
├── **单图成绩**：显著展示本次用时及与「最佳成绩」对比（仅单图模式可见）
├── **多图预览**：呈现所有挑战包图片的缩略图及各关卡用时（v4.0 新增，多图模式可见）
├── 胜利用时（总时长及分关卡时长）
├── [挑战朋友] 按钮 → 生成 .jig v5 文件分享
└── [保存照片] 按钮 → **选择性保存 (Selective Save)**：点击后弹出列表，由用户自主勾选需要保存到相册的照片（v4.0 优化）
- **Zen Mode 结算**：隐藏胜利用时及对比，主文案显示“轻松，愉悦”。

挑战书视图（ChallengeLetterView，全屏覆盖）
├── 信封入场动效（由小放大，弹性）
├── 信封拆开（封口切换为 `envelope.open.fill`）
├── 卡片滑入（**高清图片预览** + 文案逐行展示）
├── 文案：主标题 + **对决结果 (VICTORY/DEFEAT)** + 挑战寄语（若有）+ 难度 badge
└── [Accept Challenge] / [Play Again] | [View Duel Results] | [Return to Game]
```

---

## 6. 新手引导 (FTUE)

首次进入游玩页时，界面覆盖半透明黑色遮罩，按顺序（或同时）出现以下高亮与手势动效提示：

| 序号 | 提示内容 | 演示方式 |
|------|----------|----------|
| 1 | 拿起与移动 | 点击一块碎片使其“浮起”，再次点击另一块完成交换 |
| 2 | 查看原图 | 提示长按顶部 [👁️] 图标，在底层显现原图辅助 |
| 3 | 旋转与翻转 | 指引使用顶部 [🔄] 按钮或**双击**碎片进行 180° 翻转 |
| 4 | 错误提示 | 介绍顶部 [💡] 灯泡图标，点击可定位放错位置的碎片 |
| 5 | **魔法提示** | **v4.3 新增**：介绍魔棒图标，点击可智能寻找一块错位碎片归位 |
| 6 | 发起挑战 | 引导玩家在完成后使用 [📤] 分享拼图给朋友 |

**触发规则 (Granular Tutorial)**：
- **细粒度追踪**：系统记录每个步骤是否已观看。仅当对应游戏规则（如 `allowFlip`）开启且用户从未看过该步骤时才显示。
- **动态显示**：已有玩家若遇到新上线功能（如魔法提示），只会看到该新功能的引导，不会重复展示基础操作。
- **退出条件**：点击“开始游戏”或点击遮罩外部（如适用）即可进入游戏模式。
- **手动重看**：用户可在「设置」中随时点击「观看游戏教程」完整重温。

---

## 7. 全局设置与历史记录 (Settings & History)

### 7.1 全局设置

通过新挑战页右上角 `gearshape.fill` 按钮进入，以 Modal Sheet 形式呈现。

| 类别 | 设置项 | 默认值 | 说明 |
|------|--------|--------|------|
| **游戏体验** | 触觉反馈 | 开 | 全局控制所有 Haptic 反馈 |
| **游戏体验** | 音效 | 开（预留） | Phase 1 可见但无实际效果 |
| **游戏规则** | 允许查看原图（Peek） | 开 | 关闭时游戏内 👁️ 按钮 gray out；接受挑战时以 `.jig` 文件中的值为准 |
| **游戏规则** | 允许翻转拼图（Flip） | 开 | 关闭时洗牌不翻转，翻转按钮隐藏；接受挑战时以 `.jig` 文件中的值为准 |
| **游戏规则** | 幽灵位模式（Ghost Mode） | 关 | 开启后所有碎片以灰度显示（去除色彩提示），仅影响渲染，不改变胜利判定；进入游戏时锁定；接受挑战时以文件值为准 |
| **游戏规则** | 正确锁定（Correct Lock） | 关 | **v2.8 新增**：碎片归位后自动锁定不可移动，伴随触觉反馈与绿色边框 |
| **游戏规则** | 错误提示（Error Hint） | 关 | **v2.8 新增**：启用后游戏内显示「提示」按钮，点击可红标 1 个错误碎片 |
| **游戏体验** | 关键帧震动（Key Frame Haptic） | 关 | 交换后碎片回到正确位置时触发轻微咔哒 Haptic + 绿色高亮；需全局触觉反馈同时开启才生效；接受挑战时以文件值为准 |
| **默认难度** | 默认行数 / 列数 | 3 / 3 | 即时同步至新挑战页的 Slider |
| **个人资料** | 个人资料 (Profile) | — | **v5.3 优化**：整合个人头像、昵称、好友码、好友列表入口以及账户操作（登出/注销） |
| **应用偏好** | 语言 / 语言 (Language) | 跟随系统 | 支持 English / 中文 / Español；即改即生效 |
| **应用偏好** | 外观 / 深色模式 | 跟随系统 | 跟随系统 / 浅色 / 深色 |
| **游戏规则** | 禅系模式（Zen Mode） | 关 | **v4.2 新增**：启用后隐藏计时器，结算页不计时间 |
| **危险区** | 清除游戏进度 | — | 仅删除自动存档，不影响历史记录 |
| **教程** | 观看游戏教程 | — | **v4.4 新增**：以全屏弹窗形式完整展示教程序列 |

> **游戏规则生效时机**：规则在**进入游戏时**读取并锁定，中途修改设置不影响当前局。接受挑战时以 `.jig` 文件中的值为准，**完整覆盖**本地设置（`allowFlip`、`allowPeek`、`keyFrameHapticEnabled`、`ghostModeEnabled`）。

### 7.2 历史记录

通过新挑战页右上角 `clock.arrow.circlepath` 按钮进入，Push 导航。

- **写入时机**：仅在胜利结算时记录，未完成的游戏不计入。
- **视觉增强**：列表页展示挑战图片的实时缩略图，取代通用图标。
- **对决状态**：云端挑战记录详情中展示「VICTORY / DEFEAT」实时对决结果（基于双方成绩对比）。
- **记录内容**：完成时间、难度（行×列）、用时、游戏规则快照（Peek/Flip/Ghost/CorrectLock/ErrorHint、Zen、多图时每关规则）、挑战赛对比（若适用）。
- **再次游玩**：详情页在「分享」上方提供 **Play Again**（`arrow.clockwise`）；以 Sheet 打开 `GameView`，重玩为**自由练习**（不携带好友 `senderTime`），盘面与 seed 与当时完成局不同。
- **图片存储**：缩略图（200pt，质量 0.5）+ 详情图（1024pt，质量 0.85），存于 `Documents/history/`。
- **存储上限**：无自动上限，由用户手动管理。

| 删除方式 | 触发 | 二次确认 |
|---------|------|---------|
| 逐条删除（Swipe） | 列表页左滑 Cell | 否（iOS 原生惯例） |
| 逐条删除（编辑模式） | 右上角「编辑」按钮 | 否 |
| 详情页删除 | 详情页右上角 `trash.fill` | 是 |
| 全部删除 | 列表页底部红色按钮 | 是 |

---

## 8. 预设游戏模式 (Game Presets)

**v2.8 新增**：为了简化设置流程并提供差异化挑战，新增预设模式。选择预设模式会自动调整各项子规则。首页（SetupView）的模式选择卡片会实时响应底层规则的变化（无论是在首页切换还是在设置页修改），确保 UI 选中的状态与实际生效的规则逻辑一致。

### 8.1 新手模式 (Rookie Mode)
旨在提供最轻松的入门体验。
- **难度**: 强制设置为简单 (3x3)
- **查看原图 (Peek)**: 允许
- **翻转拼图 (Flip)**: **不允许** (碎片始终正向)
- **幽灵位模式 (Ghost Mode)**: **关** (全彩显示)
- **关键帧震动**: **开**
- **正确锁定**: **开**
- **错误提示**: **开**

### 8.2 地狱模式 (Hell Mode)
极高难度的终极挑战。
- **难度**: 强制设置为简单 (3x3) *[注：用户要求设置为简单模式，可能是为了在小规模内体现规则的变态]*
- **查看原图 (Peek)**: 允许
- **翻转拼图 (Flip)**: **允许**
- **幽灵位模式 (Ghost Mode)**: **开** (全灰度)
- **关键帧震动**: **关**
- **正确锁定**: **关**
- **错误提示**: **关**

---

## 9. 后续迭代储备 (Backlog)

| 阶段 | 内容 |
|------|------|
| **Phase 2** | 音效系统实装（交换/翻转/胜利轻量音效，可静音） |
| **Phase 2** | 历史详情「精确 seed 重播」：与已实现之「同规则新局」区分，需持久化 seed 字段（可选） |
| **Phase 2** | 实现 `HomeView`，将双图标入口迁移至首页 |
| **Phase 3** | **Cloud Foundation**: 接入云端后端 (Firebase/Supabase) |
| **Phase 3** | **Universal Links**: 支持短链接跳转，取代大体积 .jig 文件 |
| **Phase 4** | **Social & Gamification**: 好友与邀请系统、国家/地区排行榜 |

---

## 10. .jig 文件格式

> **文档分工说明**：文件格式的**逻辑结构与字段语义**在 PRD 中定义，作为产品级「共享契约」；具体实现细节（编码方式、压缩库选型等）在 SDD 中决定。

### 10.1 文件结构（逻辑层，v3）

二进制布局：`Magic(4) | Version(1) | MetaSize(4) | Meta(JSON) | ImageData`

| 字段 | 类型 | 说明 |
|------|------|------|
| **magic** | 4 字节 | 固定 `JIG1`，用于文件识别 |
| **version** | 1 字节 | 格式版本号，**当前为 `4` (单图) 或 `5` (多图)** |
| **metaSize** | 4 字节大端序 | JSON 元数据长度 |
| **meta** | 变长 JSON | 见下方 `JigsawMetadata` 字段 |
| **imageData** | 变长 | JPEG 压缩照片数据（质量 0.85，最大边 1024px） |

**`JigsawMetadata` JSON 字段 (v3/v4 单图)**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | Int | 3 或 4 |
| `rows` | Int | 行数 N (1-9) |
| `cols` | Int | 列数 M (1-9) |
| `seed` | UInt32 | 随机种子 |
| `senderTime` | Double? | 发起者用时 |
| `allowFlip` | Bool | 是否允许翻转 |
| `allowPeek` | Bool | 是否允许查看原图 |
| `correctLockEnabled` | Bool | 正确锁定 |
| `errorHintEnabled` | Bool | 错误提示 |
| `isZenMode` | Bool? | v4.2 新增：禅系模式 |

**`MultiJigsawMetadata` JSON 字段 (v5 多图)**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `version` | Int | 5 |
| `levels` | Array | 挑战关卡数组，每个元素为 `JigsawMetadata` |
| `senderTotalTime` | Double? | 发起者总用时 |
| `senderName` | String? | 发起者昵称 |
| `message` | String? | 挑战寄语 |
| `isZenMode` | Bool? | v4.2 新增：禅系模式 |
| `packName` | String? | 图包名称（可选） |

### 10.2 解析规则

- `magic` ≠ `JIG1` → 解析失败，Toast「文件损坏或格式无效」
- `version` 不在 `{3, 4, 5}` 范围内 → **拒绝解析**，Toast「请更新到最新版本」；第一版不做降级回退
- `rows`、`cols` 超出 1～9 → 解析失败
- `imageData` 为空 → 解析失败

### 10.3 游戏规则与挑战文件

接收方打开 `.jig` 文件时，以文件中的 **全部规则字段** 为准，完整覆盖接收方本地设置，保证双方游戏条件完全一致（公平竞技原则）：

| 字段 | 覆盖说明 |
|------|---------|
| `allowFlip` | 翻转规则，含洗牌翻转概率与游戏内按钮显隐 |
| `allowPeek` | 查看原图权限 |
| `keyFrameHapticEnabled` | 关键帧震动（全局触觉总开关仍遵循接收方设置） |
| `ghostModeEnabled` | 幽灵位灰度渲染 |
| `correctLockEnabled` | 正确锁定逻辑 |
| `errorHintEnabled` | 错误提示功能显隐 |

### 10.4 与 SDD 的衔接

SDD 中补充：JPEG 压缩参数、文件扩展名与 UTI 注册、`JigFileError` 错误处理、`AppRouter` 路由机制等实现细节。

---

## 11. 云端功能规划 (Cloud Feature Roadmap)

本节由资深后端专家设计，详细描述了未来接入 Firebase 后的核心社交、分享及竞技功能。云端挑战的产品目标与约束见 **§11.5**，技术实施参考（字段定义、Rules 草案、Callable 规格、迁移清单）见 **§11.6**。

### 11.1 好友与其邀请系统 (Social / Invitations)

本系统平衡了隐私与互动，支持非实时的挑战流转。

- **UID 与身份标识**：
    - 使用 **Firebase Auth** 集成 Apple Sign-In 为每个用户生成唯一 **UID**。
    - **UID 搜索与二维码**：用户可通过搜素好友 UID 或直接扫描其个人名片二维码建立联系。
- **轻量级好友关系**：
    - 客户端维护一个常用的 **挑战名单 (Recipients)**，同步至 Firestore 的好友子集合中，无需重复输入 UID。
- **挑战邀请与推送 (FCM)**：
    - **发起邀请**：用户 A 为 B 发起挑战时，App 将加密后的图片与元数据上传至云端，并关联 `recipientId`。
    - **通知唤起**：通过 **Cloud Messaging (FCM)** 向 B 的设备发送静默或带有挑战寄语的通知。
    - **快速进入**：B 点击通知，App 自动通过 Universal Link 逻辑定位至该云端挑战。

### 11.2 隐私驱动的分享与生命周期 (Sharing & Lifecycle)

核心原则是：**内容安全、阅后即焚（可选）与数据合规**。

- **内容安全自动审计 (Safe Search)**：
    - **自动触发**：所有上传至云端的图片在生成挑战后，立即通过 **Google Cloud Vision API** 进行多维度（成人、暴力、医疗等）内容识别。
    - **门控机制**：仅在该自动化审计返回「安全」结果后，该挑战才对接收者可见。未通过审计的挑战将保持不可读状态。
- **1 周自动清理 (TTL)**：
    - **元数据**：所有上传至云端的挑战元数据默认具有 **1 周生命周期**，到期后自动从 Firestore 逻辑删除。
    - **物理清理**：由后端 **Scheduled Cloud Function** 自动执行每小时的存储审计，物理删除 Storage 中所有孤立的、已过期的图片资源。
- **私密分享模式**：
    - **点对点发送**：所有挑战均为点对点发送，或通过私密链接转发，确保拼图内容仅在熟人间可见。
- **Universal Links**：
    - 取代 `.jig` 文件发送，App 自动生成短链接。未安装 App 的用户点击后引导至 App Store，已安装用户一键进入。

### 11.3 全球/本地排行榜与校验 (Leaderboards & Anti-Cheat)

提供多维度的竞技排行，并从后端确保成绩的真实性。

- **多级地理位置排行榜**：
    - **全球/国家/地区**：排行榜支持下钻（例如：全球排行 -> 中国区排行 -> 马萨诸塞州排行/MA 州排行）。
    - **自动定位**：获取用户注册时的地理位置，自动将其成绩归档至对应级别的榜单。
- **防作弊验证机制 (Cheat Protection)**：
    - **操作日志上报**：单局胜利后，客户端上报交换路径流水。
    - **云端重演 (Cloud Replay)**：后端利用种子重演逻辑，自动剔除逻辑不符的成绩，确保榜单权威性。

### 11.4 本地模式与网络连通性探测 (Local Mode & Connectivity Monitor)

**v5.2 新增**：为了降低新用户门槛并适配防火墙限制地区（如中国大陆），应用引入了非强制登录的「本地模式」。

- **免登录游玩 (Guest Mode)**：
    - 用户可以在登陆页点击「以游客身份继续」直接进入应用首页，享受完整的本地图片裁剪与单机游玩体验。
    - 随时可以在「设置」或触发云端功能时（如发送挑战给好友、查看收到的挑战）转化注册/登录为正式云端账户。
- **智能区域探测 (Connectivity Monitor)**：
    - **探测机制**：App 通过网络监测和心跳包静默探测 Firebase 认证服务端点的连通性。
    - **UI 适配 (门控)**：
        - 若探测失败：云端相关入口（如好友列表、云端挑战流）点击后会友好提示「云服务在您所在的地区不可用，请使用本地模式」。
        - 若探测成功但未登录：入口点击后提示「需要登录」，并包含一键前往登录的捷径。

### 11.5 云端挑战数据模型与演进 (Cloud Challenge Model & Evolution)

本节描述云端挑战（Firestore `challenges`）的产品目标、与本地 `.jig` 挑战的差异，以及实现应对齐的约束。**大型重构**应以本节为产品单一事实来源，并与 Security Rules、Cloud Functions（审核、TTL）一并评审。技术细则与 Rules 草案见 **§11.6**。

#### 11.5.1 目标能力（产品）

| 能力 | 说明 |
|------|------|
| **Challenge ID** | 每条云端挑战具有稳定、可引用的标识（默认即 Firestore 文档 ID）。可选扩展**短码/邀请码**便于口头分享，需全局唯一并建索引。 |
| **Creator（发起人）** | 永久记录原始发起人 `creatorId`（及展示用 `creatorName`），与后续参与者区分；榜单与 UI 需能标识「发起人成绩」。 |
| **多人参与** | 支持多名被挑战者；每名参与者在子集合 `results/{uid}` 中独立记录成绩（最佳用时、尝试次数、完成状态等），语义对齐现有「对决」展示。 |
| **多图挑战 (Pack)** | 云端元数据应对齐 **v5 多图**（多关卡、每关可不同行列与种子等）；Storage 支持多对象或等价打包策略，删除时需清理全部关联对象。 |
| **按 ID 加入** | 用户输入 ID（或扫描/深度链接）后能拉取挑战并游玩，前提是 **Security Rules** 允许该用户读取该文档与图片资源。 |

#### 11.5.2 与本地 `.jig` 挑战的关系

- **本地挑战**（`ReceivedChallenge` + Documents 内 `.jig`）：文件内嵌图片与 `MultiJigsawMetadata`，无 Firestore 文档。
- **云端挑战**：图片与元数据在 **Storage + Firestore**；客户端打开时下载并组装为与本地一致的游玩载荷。**两种入口不得在文档中混称为同一套「Challenge ID」语义**，除非显式说明「从云端同步到本地的视图模型」。

#### 11.5.3 审核与可见性（与 §11.2 衔接）

- 挑战创建后可进入 **`checking`（待审核）** 状态；仅当自动化内容安全审计通过后转为 **`approved`**，接收方列表查询方可命中（与 Cloud Vision / Cloud Function 设计一致）。
- 若线上规则未将状态从 `checking` 更新为 `approved`，接收方「收到的挑战」可能为空；发布前必须验证**全链路状态机**。

#### 11.5.4 分享链：完成页再分享给第三人（现状 vs 目标）

**产品可选目标**：第三人应加入**同一条**云端挑战（同一 Challenge ID），**发起人不变**，仅扩展被挑战者集合（或参与者集合）。

| 路径 | 现状（实现约束） | 目标（文档建议） |
|------|------------------|------------------|
| **应用内「发给好友」（云端）** | 调用创建接口会 **新建** Firestore 文档；**当前用户**成为新文档的 **creator**；**不会**复用上一局的 `cloudChallengeId`，也**不会**向原文档追加 recipient。 | 提供「邀请至同一挑战」能力：在服务端或客户端对**已有** `challengeId` 执行 **追加 recipient**（或受控的 join），并固定 **creatorId** 不可因转发而改写。 |
| **系统分享（.jig 文件）** | 生成新的 `.jig`，元数据中的发送方为**当前分享者**；**不包含** Firestore Challenge ID。 | 若需链式云端对决，应通过 **Universal Link / 短链 + challengeId** 补充，而非仅依赖文件传递。 |

**文档约束**：在「同一 Challenge ID + creator 不变」能力落地之前，产品说明与客服话术不得承诺「转发后仍是同一场云端对决」。

#### 11.5.5 按 Challenge ID「搜索」与查找复杂度

| 方式 | 客户端复杂度 | 主要依赖 |
|------|----------------|----------|
| **完整文档 ID 精确查询** | **低～中低**：输入框 + 调用按 ID `getDocument`；导航至挑战书/游玩与现有拉取流程类似。 | **Security Rules**：是否允许非 recipient 凭 ID 读取；若否，需 Cloud Function 代理或规则扩展。 |
| **短码映射同一挑战** | **中**：输入短码 → 按字段查询或 Callable 解析；需 **Firestore 单字段索引**（或后端生成唯一码）。 | 服务端唯一性、防枚举策略。 |
| **列表内模糊搜 / 全文搜** | **高（通常超出纯客户端）** | Firestore 不适合模糊搜索；需 Algolia / Typesense / 自建搜索服务等。 |

**阶段性建议**：首期可实现「输入完整 ID 加入」+ **Universal Link**；短码与「Challenges 页搜索框」可作为二期；全文搜索非 MVP。

#### 11.5.6 反滥用与配额（提醒）

- 每日发起云端挑战次数等配额应在产品层明确（**当前限制为每日 3 次**）。
- **UI 反馈**：设置页面应显示今日剩余可用次数，格式为 `(可用数量 / 总数量)`。
- 删除挑战时需删除 Firestore 文档及 **全部** Storage 对象（多图场景下不得只删单路径）。

#### 11.5.7 界面入口与文案区分（云端定向挑战）

为避免用户混淆「新开一局云端挑战」与「让多人加入同一条云端挑战」，**必须在 UI 层拆分入口并重命名**：

| 入口 | 用户意图 | 建议文案方向（可本地化） |
|------|----------|---------------------------|
| **新建一条云端挑战** | 创建新的 Firestore 挑战文档，当前用户为 **creator**，可指定好友接收 | 如：**「发起新挑战」**、**「新建云端挑战」**；避免与下项共用「挑战朋友」等笼统说法。 |
| **同一场挑战再邀请一人** | 在**已有** `challengeId` 上 **追加** `recipient`，**creator 不变** | 如：**「邀请好友加入这场挑战」**、**「加入同一场对决」**；出现在完成页、挑战详情或「收到的挑战」上下文。 |

**配套能力**：追加参与者由 **`addRecipient`** 流程完成（推荐服务端 **HTTPS Callable**）；成功后向 **新被邀请用户发送 FCM**，引导其打开「收到的挑战」。Callable 的技术含义与去重规格见 **§11.6.1**。

#### 11.5.8 Weekly 公开挑战（产品摘要）

- **成绩规则**：每位登录用户在当周挑战中 **仅有一次有效成绩**，以 **首次提交** 为准，**不支持**「多次尝试取最短用时」覆盖。
- **榜单信任**：周榜 **信任客户端** 上报的用时；不在 MVP 对 Weekly 做服务端重演验算（与定向私域挑战的防作弊策略可区分）。
- 详细数据模型与 Firestore Rules 见 **§11.6.4**。

### 11.5.9 导航栏角标动态更新 (Navigation Badge)

为了确保用户不错过任何挑战，主界面的「挑战」按钮配有动态橙色角标（Badge）：

- **逻辑定义**: 角标显示的是待处理的「新挑战」总数。
- **包含内容**:
    - **收到的挑战**: 包含通过文件导入的本地挑战，以及云端收到的尚未完成的挑战。
    - **发出的挑战**: 遵循用户习惯，若用户发出了挑战但自己尚未完成（且处于 `approved` 状态），该挑战也会计入角标，提醒用户参与竞技。
- **页面内引导**: 在「挑战」列表页内，针对「收到的挑战」和「发出的挑战」两个 Tab，同样显示各自对应的未完成数量子角标，帮助用户快速定位。
- **即时性保障**: 应用采用并发预加载机制，在用户进入「挑战」页面前即已静默同步完成状态，确保首页及子页面角标数字准确且无跳变。

### 11.6 技术实施参考（Cloud Challenge Implementation Reference）

**文档性质**：iOS / Cloud Functions / Rules 的**实施基线**；与 §11.5 产品描述一致，冲突时以 §11.5（产品）为准。  
**状态**：可据此开工；细则以 Firebase 控制台与 staging 验证为准。

#### 11.6.1 设计原则与 Callable 说明

1. **单一事实来源**：挑战规则（关卡、种子、图片引用）与参与者集合只在挑战文档（或 Weekly 父文档）维护一份；成绩在子集合，不在父文档内嵌大 map。
2. **最小权限**：读、写分离；定向挑战默认非公开，不可被未授权 UID 枚举或读取。
3. **可扩展性**：Schema 带 `schemaVersion`，新字段向后兼容；多图用 `levels[]` 对齐 v5。
4. **规则复杂度可控**：若「仅允许修改 `recipients` 数组」在 Rules 中难以无损表达，对 `addRecipient` 使用 **HTTPS Callable** 校验调用者身份，由服务端写入，并下发 **FCM** 通知新 recipient。

**Callable 是什么**：Firebase 的 HTTPS Callable 函数——客户端通过 `functions.httpsCallable("functionName")` 调用，SDK 自动附带 Firebase Auth ID Token，服务端可验证 `request.auth.uid`。与客户端直接 `updateDocument` 相比，业务规则集中、便于审计与推送；代价是多一跳延迟与 Functions 计费。

**`addRecipient` 是否会产生重复 recipient？不会**，原因分两层：

| 层级 | 说明 |
|------|------|
| **Firestore** | 使用 `FieldValue.arrayUnion(newUid)` 更新 `recipients` 时，同一字符串 UID 在数组中只会出现一次，重复 `arrayUnion` 不会造出重复元素。 |
| **Callable（必选逻辑）** | 在写入前校验：`newUid != creatorId`；`newUid` **不在**现有 `recipients` 中。若已在列表中，**不再发送 FCM**，并返回明确结果（如 `alreadyRecipient: true`），避免用户误以为操作失败或重复骚扰。 |

**结论**：Callable 负责权限、去重判断与单次 FCM；存储层 `arrayUnion` 作为兜底防重复。客户端应展示 Callable 返回的业务码（已存在 / 成功 / 禁止邀请自己等）。

#### 11.6.2 定向挑战数据模型：`challenges/{challengeId}`

**已定产品决策**：

| 决策 | 说明 |
|------|------|
| **不向陌生人开放邀请码/Challenge ID 搜索** | 加入定向挑战的路径限于：推送/深度链接、应用内「收到的挑战」列表、以及由 creator 或现有 recipients 主动把他人加入。 |
| **谁可以扩展参与者** | **仅 `creatorId` 与当前 `recipients` 列表中的 UID** 可以更新挑战文档，将新 UID 加入 `recipients`。第三方不能通过「搜 ID」自助加入。 |

**字段建议（演进后）**：

| 字段 | 类型 | 说明 |
|------|------|------|
| `schemaVersion` | `int` | 从 `1` 起，便于迁移。 |
| `kind` | `string` | 固定 `"directed"`，与 Weekly 区分。 |
| `creatorId` | `string` | 发起人 UID，**不可变**。 |
| `creatorName` | `string` | 展示快照（可选）。 |
| `recipients` | `array<string>` | 被邀请参与者 UID；**不包含 creator**。 |
| `imageURLs` / `storagePaths` | `array` | 多图；单图时 length=1。 |
| `meta` | `map` | 与 `MultiJigsawMetadata` 对齐的子集（`levels[]` 每关 rows/cols/seed/规则位）。 |
| `status` | `string` | `checking` \| `approved` \| `rejected`。 |
| `message` | `string?` | 寄语。 |
| `createdAt` / `expiresAt` | `timestamp` | TTL 与清理任务一致。 |
| ~~`results`（文档内嵌）~~ | — | **删除或仅作只读聚合缓存**；权威数据在子集合 `results`。 |

子集合 `challenges/{challengeId}/results/{uid}`：保留 `bestTime`、`attemptsCount`、`lastAttemptAt`、`status`；多图需总分 + 分关可再扩字段。

**读写意图**：

| 操作 | 允许主体 | 备注 |
|------|----------|------|
| **读文档 + 读子集合成绩** | `creatorId`、`recipients` 中 UID；审核服务账号 | 不满足「公开可读」。 |
| **创建** | 已认证用户 | 创建者为 `creatorId`。 |
| **更新 `recipients`（追加 UID）** | **仅** `creatorId` 或**已在** `recipients` 中的 UID | 推荐只经 Callable；不允许任意用户通过 ID 自助加入。 |
| **写自己的 `results/{uid}`** | `request.auth.uid == uid`，且该 UID 为 `creatorId` 或 `recipients` 之一 | |
| **删除** | `creatorId` 或管理员 | 多图需删多条 Storage 路径。 |

**`addRecipientToChallenge` Callable 规格**：

| 项目 | 约定 |
|------|------|
| **入参** | `challengeId: string`，`newUid: string` |
| **身份** | `context.auth.uid` 必须为 **`creatorId`** 或 **`recipients` 内已有 UID** |
| **校验顺序** | ① 读取文档，不存在则失败；② `status` 若要求仅 `approved` 可邀人则在此拒绝；③ `newUid == creatorId` → 拒绝；④ `newUid` 已在 `recipients` → **不写库**，返回 `alreadyRecipient: true`，**不发 FCM** |
| **写入** | `update`：`recipients: FieldValue.arrayUnion(newUid)` |
| **通知** | 仅当**本次实际新增**时，向 `newUid` 发送 **FCM**（payload 含 `challengeId`） |
| **返回** | `{ ok: true, alreadyRecipient?: boolean }` 或等价错误码 |

**生产环境推荐**：客户端 **禁止**直接 `update` 挑战文档的 `recipients`；一律调用该 Callable，由服务端写入并发 FCM。

#### 11.6.3 Firebase Security Rules 草案

以下为示意，部署前必须在 Firebase 控制台实测；复杂字段更新建议走 Callable。

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {

    function isSignedIn() {
      return request.auth != null;
    }

    function isDirectedParticipant(challenge) {
      return isSignedIn() && (
        request.auth.uid == challenge.data.creatorId ||
        request.auth.uid in challenge.data.recipients
      );
    }

    // ---- 定向挑战 ----
    match /challenges/{challengeId} {
      allow read: if isSignedIn() && isDirectedParticipant(resource);

      allow create: if isSignedIn()
        && request.resource.data.creatorId == request.auth.uid
        && request.resource.data.kind == 'directed';

      // 更新：仅 creator 或已有 recipient；且禁止擅自改 creatorId / kind
      allow update: if isSignedIn() && isDirectedParticipant(resource)
        && request.resource.data.creatorId == resource.data.creatorId
        && request.resource.data.kind == resource.data.kind;

      allow delete: if isSignedIn() && request.auth.uid == resource.data.creatorId;

      match /results/{userId} {
        allow read: if isSignedIn() && isDirectedParticipant(get(/databases/$(database)/documents/challenges/$(challengeId)));
        allow create, update: if isSignedIn()
          && request.auth.uid == userId
          && isDirectedParticipant(get(/databases/$(database)/documents/challenges/$(challengeId)));
      }
    }

    // ---- Weekly 公开挑战 ----
    match /weeklyChallenges/{periodId} {
      allow read: if isSignedIn();

      match /scores/{userId} {
        allow read: if isSignedIn();
        // 仅首次写入：禁止 update（与 §11.6.4「只保留第一次」一致）
        allow create: if isSignedIn()
          && request.auth.uid == userId
          && !exists(/databases/$(database)/documents/weeklyChallenges/$(periodId)/scores/$(userId));
        allow update, delete: if false;
      }
    }

    // users 等其余集合保持现有策略
  }
}
```

**注意**：
- **定向挑战 `update`**：实施阶段建议改为**仅服务端（Callable 使用的 Admin SDK）可写**，客户端 deny update；`recipients` 变更**只**经 §11.6.2 Callable。若暂保留旧客户端，需用 `diff().affectedKeys()` 等严格限制，维护成本高。
- 定向挑战**读**规则已排除「任意 UID 凭 documentId 试探读取」，与「不允许邀请码查找」一致。
- `get()` 父文档在子集合规则中会增加读次数，需评估。

#### 11.6.4 Weekly 公开挑战：`weeklyChallenges/{periodId}`

与定向挑战**分集合**更清晰，Rules 更简单，避免 `arrayContains` / `kind` 组合索引泛滥。

**父文档字段**：

| 字段 | 说明 |
|------|------|
| `periodId` | 如 `2026-W12` 或 ISO 周；与文档 ID 一致。 |
| `title` | 展示用标题。 |
| `startsAt` / `endsAt` | 周窗口；客户端与排行榜查询用。 |
| `status` | `draft` \| `live` \| `archived`（便于提前配置、结束后只读榜）。 |
| `imageURLs` / `storagePaths` | 当周题目资源。 |
| `meta` | 与 v5 对齐的关卡配置。 |
| `schemaVersion` | `int` |

**成绩子集合**（路径：`weeklyChallenges/{periodId}/scores/{uid}`）：

**产品规则（已定）**：不允许「周内多次提交、取最短用时」；**仅保留第一次提交**的成绩。重玩或再次通关**不得覆盖**已有文档（服务端 Rules 禁止 `update`，仅允许 `create` 且 `!exists`）。

| 字段 | 说明 |
|------|------|
| `time`（或 `submittedTime`） | **首次**提交的总用时（秒）；写入后不可变。 |
| `submittedAt` | 服务器时间戳（建议 `serverTimestamp`），用于争议排查。 |
| `displayNameSnapshot` | 可选，便于榜单展示。 |

**信任模型（已定）**：周榜信任客户端上报的用时，不在 MVP 对 Weekly 做 Cloud Replay 等服务端验算。  
**排名**：客户端对已提交文档按 `time` 升序排序即可；若数据量大再考虑聚合文档或分页查询。

**读写权限**：

| 操作 | 允许主体 |
|------|----------|
| **读父文档** | 任意**已登录**用户（`request.auth != null`）。 |
| **读他人成绩** | 已登录用户可读（用于榜单）。 |
| **写自己的 `scores/{uid}`** | 仅 `request.auth.uid == uid`；在 `status == live` 且当前时间在窗口内；**且仅允许首次创建**。 |

#### 11.6.5 Storage Rules、索引与查询

**Storage Rules（示意）**：
- **定向挑战路径**（如 `challenges/{creatorUid}/{uuid}.jpg`）：仅 creator 可上传，图片 URL 通过 Firebase token 或短期签名分发；或统一经 Cloud Function 生成下载 URL。
- **Weekly 资源**：`weekly/{periodId}/...` 可对已登录用户只读；写仅限管理员/service account。

**建议索引**：

| 查询 | 集合 | 建议索引 |
|------|------|----------|
| 我收到的定向挑战 | `challenges` | `recipients` array-contains + `status` + `createdAt` |
| 我发出的 | `challenges` | `creatorId` + `createdAt` |
| Weekly 当周榜 | `weeklyChallenges/{id}/scores` | 客户端按 `time` 升序排序；或单字段索引视查询方式而定 |

#### 11.6.6 迁移要点（从当前实现）

1. 为旧文档补 `kind: "directed"`、`schemaVersion: 1`；新字段逐步回填。
2. 去掉文档级 `results` map 的依赖，统一子集合。
3. `ChallengeMeta` → 多关 `meta`：迁移脚本或双读兼容期。
4. Rules 上线前：**staging** 验证 creator/recipient 读、非参与者读应拒绝。
5. 客户端：扩展参与者走明确 API（Callable `addRecipient` + FCM），不再误用「新建挑战」；UI 上区分「新建云端挑战」与「邀请他人加入当前挑战」（见 §11.5.7）。

#### 11.6.7 实施检查清单

**A. 后端 / Firebase**

1. [ ] 部署 Callable `addRecipientToChallenge`（§11.6.2），含：`creatorId`/`recipients` 权限、`newUid` 去重、非 creator、**仅实际新增时 FCM**。
2. [ ] 配置 FCM：新 recipient 通知 payload（`challengeId`、展示文案）；处理 token 缺失。
3. [ ] 更新 Firestore Rules（§11.6.3）：定向挑战读权限；`challenges` 客户端 `update` deny（若采用 Callable 专属路径）；Weekly `scores` 仅首次 `create`。
4. [ ] 审核流水线：`checking` → `approved`，与收件监听一致。
5. [ ] TTL / 定时任务：过期删除 `challenges` 与 Storage 多路径。

**B. iOS 客户端**

1. [ ] 数据模型：对齐 `kind`、`schemaVersion`、多图 `meta`（逐步替换单图 `ChallengeMeta`）。
2. [ ] **新建挑战** vs **同挑战再邀请**：两入口 + 文案（§11.5.7）；再邀请仅调 Callable，不传错 `createChallenge`。
3. [ ] 完成页 / 详情页：挂载「邀请好友加入这场挑战」；成功后可选本地 Toast。
4. [ ] Weekly：提交成绩用一次性 `setData`（失败则提示已提交）；不做本地多次覆盖请求。
5. [ ] 监听与列表：`recipients` array-contains + `status`（§11.6.5）。

**C. 验证**

1. [ ] Staging：非参与者**不得** `get` 定向挑战；参与者可读。
2. [ ] 同一 `newUid` 连续两次 `addRecipient`：第二次**无重复 FCM**、返回 `alreadyRecipient`。
3. [ ] Weekly：同一用户第二次写 `scores/{uid}` 被 Rules **拒绝**。

---

## 12. 个人资料与账户管理 (Profile & Account)

**v5.0 新增**：为了增强社交属性，用户可以自定义其在云端和挑战文件中的身份标识。

- **显示昵称 (Display Name)**：
    - 用户可在「设置 -> 个人资料」中修改。
    - 长度限制：1-20 字符。
    - 存储：同步至 Firebase Auth 及 Firestore `users` 集合。
- **头像 (Avatar)**：
    - 支持从相册选取图片。
    - 自动裁剪为正方形并进行压缩处理。
    - 存储：上传至 Firebase Storage，URL 存入 Firestore。
- **账户关联**：
    - 绑定 Apple/Google 账号，确保个人资料跨设备同步。

---

## 附录：文档修订记录

| 版本 | 日期 | 变更说明 |
|------|------|----------|
| v6.8 | 2026-03-26 | **每日分享限制重构**：修复了分享限制使用 UTC 时间导致跨时区重置不及时的问题。现在统一使用用户本地时间进行每日零点重置。 |
| v6.7 | 2026-03-26 | **挑战角标逻辑完善**：1. 角标包含「收到的」与「已发送未完成」；2. 挑战页 Tab 增加子角标；3. 引入并发预加载机制。 |
| v6.5 | 2026-03-26 | **历史记录优化**：挑战记录不再保存至本地历史；移除历史列表徽章；提高日期显示精度至分钟；调整时间显示位置。 |
| v6.5 | 2026-03-23 | **视觉语言与主设计文档对齐**：§5.1 更新为壳层 **波普 (Pop Art)** + 游玩区照片焦点；指向 `UIUX-DESIGN.md` v4.0。 |
| v6.4 | 2026-03-23 | **历史再次游玩**：§7.2 补充单图/多图详情「Play Again」、多图 `LevelRecord` 游戏规则持久化；§9 Backlog 将「再来一局」调整为可选「精确 seed 重播」后续项。 |
| v6.3 | 2026-03-23 | **文档整合**：将 `docs/CLOUD_CHALLENGE_DATA_MODEL_AND_RULES.md` 全部内容整合至新增 §11.6（技术实施参考），包含设计原则、Callable 规格、字段定义、Rules 草案、Weekly 模型、迁移要点与实施检查清单；删除独立技术文档，所有交叉引用更新至 §11.6。 |
| v6.2 | 2026-03-23 | **Cloud Challenge UX & Weekly**：§11.5.7 拆分「新建云端挑战」与「同场再邀请」入口及文案；§11.5.8 Weekly 首次提交定榜与客户端信任榜单。 |
| v6.1 | 2026-03-23 | **Cloud Challenge PRD**: 新增 §11.5「云端挑战数据模型与演进」：Challenge ID / creator / 多人 / 多图目标；本地与云端语义区分；审核状态与可见性；完成页再分享之现状 vs 目标；按 ID 查找与搜索复杂度及分阶段建议；配额与删除约束。 |
| v5.1 | 2026-03-20 | **User Profile Customization**: 1. 支持修改显示昵称与头像；2. 整合个人资料管理至设置页；3. 修复 LoginView 性能与稳定性问题。 |
| v2.0 | 2026-03-13 | 玩法重构：网格交换版，移除拖拽，新增 FTUE、社交裂变、待完善建议 |
| v2.1 | 2026-03-13 | 难度改为行/列分别选择（1～9）；新增 .jig 文件格式、动画参数；明确不支持无障碍 |
| v2.2 | 2026-03-13 | 零间距网格；支持原图；允许重复挑战；支持横屏 |
| v2.3 | 2026-03-13 | 新增全局设置（游戏规则 Peek/Flip、默认难度、深色模式、存档管理）；新增历史记录功能；`.jig` 格式升至 v2（新增 allowFlip/allowPeek 字段，严格版本校验） |
| v2.4 | 2026-03-14 | 新增国际化支持（英文 / 简体中文 / 西班牙语）；补充 §10.3 可访问性国际化覆盖范围说明 |
| v2.5 | 2026-03-14 | 新增应用内语言切换：首页 🌐 快速入口 + 设置页 Language 分区；更新 §5.3 核心路径、§7.1 设置项表 |
| v2.6 | 2026-03-14 | 新增关键帧震动反馈（Key Frame Haptic）与幽灵位模式（Ghost Mode）；更新 §2.5 胜利判定与结算、§7.1 全局设置表 |
| v2.7 | 2026-03-14 | `.jig` 格式升至 v3：新增 `keyFrameHapticEnabled` / `senderName` / `message` / `ghostModeEnabled` 字段；发起挑战新增「挑战寄语」输入 Sheet；接受挑战新增「挑战书」仪式感视图（ChallengeLetterView）；设置页新增「社交 / 显示昵称」；所有规则字段在挑战模式下完整覆盖本地设置；更新 §4.1/4.2/5.3/7.1/9 |
| v2.8 | 2026-03-14 | 新增「正确锁定」与「错误提示」模式；新增「新手模式」与「地狱模式」预设；`.jig` 格式升至 v4（新增 `correctLockEnabled` / `errorHintEnabled`）；更新目录与各章节 |
| v2.9 | 2026-03-15 | 计时器格式调整为 `MM:SS`（分钟:秒）；游戏进行中新增右上角 `[📤]` 分享按钮，与结算页共用同一套分享逻辑；`.jig` 分享改为全内存构建（JPEG 质量 0.88，`UIGraphicsImageRenderer` 强制 RGB 去 Alpha 通道）；更新 §2.3.5、§4.1、§5.3 |
| v3.0 | 2026-03-17 | **全流程体验优化**：1. 提升最低部署目标至 **iOS 17.0** 以支持高级动画；2. 强化碎片选中视觉反馈（1.12x 缩放 + 深阴影）；3. 新增胜利庆祝序列（原图展示 + 彩带粒子 + 2.5s 延迟结算）；4. 全面升级新手引导。 |
| v4.0 | 2026-03-17 | **多图挑战模式 (Multi-Image Pack)**：1. 支持最多 5 张图片的多关卡挑战；2. 历史记录支持 Master-Detail 结构并自动保存；3. `.jig` 协议升至 v5；4. 首页「继续游戏」按钮动态化；5. 补全英、中、西三语国际化查漏补缺。 |
| v4.1 | 2026-03-17 | **文档修正**：恢复并强化「单图挑战」相关描述，确保双模式文档对等与完整性。 |
| v4.2 | 2026-03-18 | **禅系模式 (Zen Mode)**：1. 新增无计时模式及相关 UI 适配；2. 设置页与新挑战页增加 Zen Mode 开关；3. 历史记录支持 Zen Mode 标记；4. 完善英、中、西三语国际化。 |
| v4.3 | 2026-03-18 | **AI Hint System (Magic Hint)**：1. 新增「魔法提示」功能，智能移动碎片至正确位置；2. 顶部导航栏布局优化，支持更多功能按钮；3. 增加平滑交换动画。 |
| v4.4 | 2026-03-18 | **Tutorial System Enhancement**：1. 教程重构为独立复用组件；2. 实现细粒度教学追踪（仅显示未见过的步骤）；3. 设置页增加「观看游戏教程」入口。 |
| v5.0 | 2026-03-20 | **Cloud Milestone**: 1. 云端好友与 FCM 邀请系统设计；2. 全球/国家/地区多级排行榜；3. 24 小时数据自动清理策略。 |
| v5.1 | 2026-03-20 | **User Profile Customization**: 1. 支持修改显示昵称与头像；2. 整合个人资料管理至设置页；3. 修复 LoginView 性能与稳定性问题。 |
| v5.2 | 2026-03-21 | **Local Mode & Connectivity (Guest Mode)**: 1. 支持免登录「游客模式」直接游玩；2. 引入 Connectivity Monitor 动态探测 Firebase 可达性；3. 对云服务依赖界面进行访问门控与 UI 降级提示。 |
| v5.3 | 2026-03-22 | **UI/UX Optimization & Profile Hub**: 1. 简化首页导航栏，整合为个人中心入口；2. 重构设置页为统一的 Profile Hub，并优化内容分类。 |
| v6.0 | 2026-03-23 | **Challenge Experience Enhancement**: 1. 挑战列表新增图片缩略图与 VICTORY/DEFEAT 对决状态勋章；2. 挑战书视图 (ChallengeLetterView) 流程优化：简化对决结果展示、新增「返回游戏」主按钮、双向开放「查看对决结果」详情。 |
