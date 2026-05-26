# Android Phase 3 Alignment

本文用于对齐 PR #2 的 Android 原生复刻方向。结论先行：`android-native/` 的 Kotlin + Compose 骨架值得保留，但第三阶段不能继续复刻早期 ChatBox 式聊天，而要开始对齐 TBird / Novelbox 已经验证过的核心产品模型：多创作者圆桌创作器。

## 当前定位

- Web 版继续作为试验开发场，负责快速验证交互、提示词、记忆和圆桌流程。
- Android 原生版是正式产品化方向，不能照搬 Web 的 `main.js`，要迁移稳定后的领域模型和规则。
- PR #2 目前是优秀的 Android 起点：已有 Material 3、会话持久化、Persona、圆桌队列、正文 manuscript、流式输出、导入导出和测试。
- PR #2 目前仍偏早期：概念还是 `Persona / RoundtableConfig`，没有完整对齐主创、议员、写手、创作者记忆池、参会记录、预设主创和会话类型。

## 第三阶段目标

第三阶段的目标不是继续补普通聊天功能，而是让 Android 原生工程开始承载 Novelbox 的真实领域语言。

必须对齐的产品事实：

- 一个普通创作会话只有一个固定主创。
- 交流模式和圆桌模式是同一个会话的两个视图，主创不变、记忆不变。
- 圆桌模式额外出现议员席位和写手。
- 写手只在圆桌中负责正文整理与续写，默认继承主创文风和模型配置。
- 议员不是普通聊天助手，而是可被拉入圆桌发言的创作者身份。
- 创作者应拥有跨会话记忆池；被拉入其他圆桌时，仍能追溯自己的参会记录。
- 跑团模式是独立会话类型，不能和交流/圆桌互相切换。

## 推荐领域模型

Android 第三阶段建议先加新模型，不急着删除旧 `Persona`。可以让旧 `Persona` 临时适配到新模型，逐步迁移。

### CreatorIdentity

创作者身份。替代现在过于笼统的 `Persona`。

建议字段：

- `id`
- `name`
- `roleType`: `MAIN_CREATOR | COUNCILOR | WRITER | PRESET`
- `avatarUri`
- `systemPrompt`
- `modelBinding`
- `memoryPoolId`
- `privateSessionId`
- `sourceTemplateId`
- `createdAt`
- `updatedAt`

### SessionKind

会话类型。

- `CREATIVE`: 普通创作会话，可在交流和圆桌视图之间切换。
- `ROLEPLAY`: 跑团会话，独立入口，不能切换为交流/圆桌。

会话创建时确定类型。不要再用一个 `enabled` 开关把跑团塞进普通聊天。

### RoundtableSeat

圆桌席位。圆桌里真正参与发言的是席位，不是直接把创作者塞进会话。

建议字段：

- `seatId`
- `creatorId`
- `order`
- `speakingEnabled`
- `materialScope`
- `joinedFromSessionId`
- `createdAt`

这样以后才能支持“同一个创作者在不同圆桌留下不同参会记录”。

### WriterAgent

写手不是普通议员。建议作为会话内派生角色：

- 默认模型配置继承主创。
- 默认提示词偏工具化：继承文风、整理正文、按用户命令续写。
- 每次进入圆桌时可以根据主创上下文生成/刷新文风摘要。
- 写手输出进入正文稿纸，而不是普通议员气泡。

### MemoryPool

创作者记忆池。

建议先做简单 JSON 存储，不必一开始上 Room。

最低要求：

- 每个 `CreatorIdentity` 有独立记忆池。
- 记忆条目记录来源会话、来源圆桌、来源消息。
- 普通会话分支废弃时，废弃分支的记忆以后要能被清理或标记失效。
- 议员被从圆桌移除，不等于删除创作者记忆。

### ModelBinding

模型绑定。

建议字段：

- `providerId`
- `baseUrl`
- `apiKeyAlias`
- `model`
- `temperature`
- `maxTokens`
- `contextTokenBudget`
- `unlimitedOutput`

默认继承全局配置，但创作者可覆盖。隐藏/预设主创也要走同一套结构，只是隐藏内置提示词。

## 第三阶段施工顺序

### Phase 3A: 数据模型对齐

新增 `CreatorIdentity`, `SessionKind`, `RoundtableSeat`, `ModelBinding`, `MemoryPool`。

验收：

- 新模型有序列化测试。
- 旧 `Persona` 可以映射为 `CreatorIdentity(roleType = COUNCILOR)`。
- 旧 `RoundtableConfig.personaIds` 可以映射为 `RoundtableSeat` 列表。

### Phase 3B: 主创固定

每个 `CREATIVE` 会话必须拥有一个主创。

验收：

- 新建普通创作会话时自动创建/绑定主创。
- 交流视图和圆桌视图显示同一个主创。
- 圆桌不能把本会话主创当作外部议员重复拉入。

### Phase 3C: 圆桌席位与发言顺序

用 `RoundtableSeat` 替代直接操作 persona id 列表。

验收：

- 勾选顺序等于发言顺序。
- 可设置主创是否参与发言。
- 议员按顺序等待前一个说完再发言。
- @ 其他议员只影响队列，不串消息。
- 模型收到的上下文必须标明谁说了什么。

### Phase 3D: 写手与正文稿纸

把 manuscript 从普通字符串升级为正文工作区。

验收：

- 圆桌模式出现写手。
- 写手输出视觉上和普通气泡不同。
- “送入正文”仍可保留，但写手正文应自动进入稿纸。
- 正文视图不要被普通聊天 UI 挤压。

### Phase 3E: 创作者记忆池

先实现最小闭环：可写入、可检索、可展示。

验收：

- 创作者可以看到自己在哪些圆桌发过言。
- 拉其他会话主创为议员时，带着同一身份的压缩记忆和参会记录。
- 删除席位不删除创作者。
- 删除创作者才删除其记忆和相关身份配置。

### Phase 3F: 跑团独立入口

Android 也要跟 Web 最新方向一致：新建会话先选普通创作或跑团。

验收：

- `CREATIVE` 会话可在交流/圆桌视图切换。
- `ROLEPLAY` 会话不可切圆桌。
- 普通创作会话不可切跑团。

## 不要做的事

- 不要把 Web 的 `main.js` 直接翻译成 Kotlin。
- 不要继续扩大 `ChatViewModel`，圆桌调度、记忆、模型配置都应拆成独立 service/kernel。
- 不要把主创、议员、写手都叫 `Persona`，这会导致后面迁移越来越乱。
- 不要把跑团做成普通聊天里的开关。
- 不要把内置提示词暴露在普通设置页。
- 不要把 UI 先做复杂；第三阶段先让领域模型站稳。

## 推荐包结构

```text
android-native/app/src/main/java/com/qinglan/chatnovel/
  model/
    CreatorIdentity.kt
    Session.kt
    RoundtableSeat.kt
    MemoryPool.kt
    ModelBinding.kt
  data/
    CreatorStore.kt
    SessionStore.kt
    MemoryStore.kt
    Migration.kt
  domain/
    roundtable/
      RoundtableOrchestrator.kt
      MentionResolver.kt
      WriterSync.kt
    memory/
      MemoryRetrieval.kt
      MemoryWriter.kt
    modelconfig/
      ModelBindingResolver.kt
  ui/
    chat/
    roundtable/
    creator/
    manuscript/
```

## 和 Web 的对齐方式

Web 只提供稳定规则，不提供要照抄的代码。

当前可参考的 Web 规则：

- 创作者身份与会话主创的关系。
- 圆桌成员按顺序发言。
- @ 可以调整后续发言队列。
- 写手正文和普通气泡视觉不同。
- 每个创作者可有独立模型配置，默认继承全局。
- 隐藏/预设主创是模板应用，不是普通可编辑提示词。

## 第三阶段完成标准

第三阶段完成时，Android 不必拥有 Web 的全部 UI，但必须拥有正确领域骨架：

- 新建普通创作会话后，有固定主创。
- 圆桌里能添加议员席位，而不是简单 persona id。
- 写手作为特殊角色出现，并能写入正文。
- 创作者记忆池存在，并能被圆桌 prompt 检索。
- 跑团会话类型已经在数据层预留或实现。
- 关键逻辑有单元测试，不依赖 Compose UI 才能验证。

做到这里后，第四阶段再追 UI：稿纸盖在群聊上、参会人设置、创作者详情页、预设主创选择界面。
