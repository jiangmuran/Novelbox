# Android Phase 3 Alignment

本文用于更正 PR #2 的第三阶段方向对齐。上一版文档误把 PR #2 中的 `android-native/` 并行工程当成当前 Android 主线，这是错误的。

当前真正推进的 Android 原生复刻主线是：

```text
android-app/
```

也就是在现有 Android 包内推进 `NativeActivity + Compose` 原生化，而不是另起 `android-native/` 工程。`android-native/` 可以作为参考资料，但不应作为第三阶段施工主线，也不应直接合并替换当前 `android-app/`。

## 当前施工主线

当前 Android 主线状态以 `wt1` 分支为准：

- 已完成 `NativeActivity` 拆分。
- 已完成 `android-app` 内的原生入口，应用不再正常进入 WebView。
- 已完成 Room v2：
  - `ProjectEntity`
  - `ChapterEntity`
  - `ProjectMaterialEntity`
  - `CharacterCardEntity`
  - `SessionEntity.projectId`
- 已完成：
  - `ProjectRepository`
  - `ProjectContextProvider`
  - `ProjectViewModel`
  - `NovelMaterialsPanel`
- 已完成“小说资料”轻面板。
- 已完成当前作品资料注入聊天 prompt。
- UI 方向要求继续靠拢 Web 版，而不是 Material 3 重设计，也不是另起 `android-native/`。

因此第三阶段的核心不是“继续完善 PR #2 的 `android-native/`”，而是把 Web 已验证的产品模型映射进 `android-app/` 这条原生主线。

## Web 版已经稳定、可以复刻的能力

以下能力已经可以作为 Android 复刻依据，但不要照抄 Web 代码：

- 交流模式和圆桌模式是同一创作会话的两个视图。
- 一个创作会话只有一个固定主创。
- 主创在交流/圆桌之间不变，记忆不变，模型配置不变。
- 圆桌模式中额外出现议员席位和写手。
- 议员按勾选顺序依次发言。
- @ 其他议员可以影响后续发言队列，但不能串消息。
- prompt 中必须明确标注谁说了什么。
- 写手不是普通议员，写手输出应进入正文区域。
- 正文/稿纸是创作中心，不是普通聊天附件。
- 每个创作者可以拥有独立模型配置，默认继承全局配置。
- 预设主创是模板应用，不是普通可见提示词编辑。
- 跑团模式是独立会话类型，不能和交流/圆桌互相切换。
- 导入 TXT、章节结构化、作品资料、角色卡、世界观、大纲、伏笔，都属于“作品/项目资料”体系，而不是聊天消息本身。

## android-app 已经具备的基础

第三阶段应该复用这些已有能力：

- 原生入口已经存在，不需要再保留“先进入 WebView”的思路。
- Room v2 已经有作品级数据结构。
- `SessionEntity.projectId` 已经把会话和作品项目连接起来。
- `ProjectContextProvider` 已经能把当前作品资料注入聊天 prompt。
- `NovelMaterialsPanel` 已经作为小说资料轻面板存在。
- `ProjectRepository` / `ProjectViewModel` 已经是作品资料方向的基础设施。

这意味着 Android 侧不应该先从“聊天应用”扩展，而应该从“作品项目 + 创作者会话 + 圆桌协作”收敛。

## android-app 还缺的 Web 核心功能

第三阶段前后，`android-app` 还缺这些关键能力：

- 创作者身份模型：主创、议员、写手、预设主创还没有统一抽象。
- 圆桌席位模型：当前需要区分“创作者本人”和“在某个圆桌里的席位”。
- 主创固定规则：同一会话的交流/圆桌必须使用同一个主创。
- 写手规则：写手应继承主创文风与模型配置，只在圆桌/正文写作中出现。
- 创作者记忆池：创作者跨会话、跨圆桌的记忆和参会记录。
- 模型绑定：全局默认配置 + 单创作者覆盖配置。
- 圆桌发言调度：顺序发言、@ 调整队列、停止/恢复、上下文标注。
- 正文沉淀：写手输出、用户采纳、正文版本或章节归档。
- 预设主创：模板应用、头像、隐藏内置提示词。
- 跑团会话类型：新建会话时选择普通创作或跑团，创建后不可互切。

## 第三阶段最小闭环

第三阶段不要试图一次补齐所有 UI。先把领域骨架和最小可用体验做稳。

### Phase 3A: 对齐会话和项目关系

目标：明确会话属于某个作品项目，作品资料是 prompt 的稳定来源。

建议：

- 保留并强化 `SessionEntity.projectId`。
- `ProjectContextProvider` 继续负责组装作品资料。
- 资料注入要可控，至少支持：
  - 当前作品简介
  - 当前章节
  - 角色卡
  - 世界观
  - 大纲
  - 伏笔
- 聊天 prompt 中不要直接塞 UI 面板里的调试文本。

验收：

- 新建会话可以绑定当前作品。
- 切换作品后，新会话默认绑定新作品。
- 当前作品资料能进入主创请求上下文。
- 单元测试覆盖 `ProjectContextProvider` 的资料拼装。

### Phase 3B: 新增 CreatorIdentity

目标：把“主创/议员/写手/预设主创”从普通聊天助手中拆出来。

建议新增 Room/entity 或 Kotlin domain model：

```kotlin
enum class CreatorRoleType {
    MAIN_CREATOR,
    COUNCILOR,
    WRITER,
    PRESET
}

data class CreatorIdentity(
    val id: String,
    val name: String,
    val roleType: CreatorRoleType,
    val avatarUri: String?,
    val systemPrompt: String,
    val modelBindingId: String?,
    val memoryPoolId: String?,
    val sourceTemplateId: String?,
    val createdAt: Long,
    val updatedAt: Long,
)
```

和现有体系的关系：

- `SessionEntity` 应能绑定一个主创 `mainCreatorId`。
- 新建普通创作会话时，如果没有指定主创，就自动创建或绑定默认主创。
- 议员不是主创的替代品，是可以入席圆桌的创作者身份。

验收：

- 每个创作会话都有且只有一个主创。
- 主创信息可被聊天 prompt 使用。
- UI 上不要再把主创叫“AI助手”。

### Phase 3C: 新增 RoundtableSeat

目标：圆桌里参与发言的是“席位”，席位引用创作者。

建议：

```kotlin
data class RoundtableSeat(
    val seatId: String,
    val sessionId: String,
    val creatorId: String,
    val order: Int,
    val speakingEnabled: Boolean,
    val materialScope: MaterialScope,
    val joinedFromSessionId: String?,
    val createdAt: Long,
)
```

为什么需要席位：

- 同一个创作者可以被拉进多个圆桌。
- 创作者本体记忆要统一。
- 每个圆桌内的参会记录要独立。
- 删除席位不等于删除创作者。

验收：

- 参会人列表显示席位顺序。
- 主创可固定显示，并可设置是否参与本轮发言。
- 不能把本会话主创作为外部议员重复拉入。
- 议员按顺序逐个发言。

### Phase 3D: WriterAgent 接入正文

目标：写手不是普通议员，而是正文整理机。

建议：

- 写手可作为 `CreatorIdentity(roleType = WRITER)`，也可以作为 session 内派生对象。
- 写手默认继承主创模型配置。
- 写手 prompt 不展示给普通用户。
- 写手读取：
  - 当前会话上下文
  - 当前作品资料
  - 主创文风摘要
  - 当前正文
- 写手输出进入正文区域，而不是只作为普通聊天气泡。

验收：

- 圆桌模式中可 @ 写手继续写。
- 写手输出视觉上和议员发言不同。
- 写手输出能追加到正文。
- 正文区域靠拢 Web 的稿纸体验。

### Phase 3E: ModelBinding

目标：全局默认配置之外，创作者可以覆盖模型。

建议：

```kotlin
data class ModelBinding(
    val id: String,
    val providerId: String?,
    val baseUrl: String?,
    val apiKeyAlias: String?,
    val model: String?,
    val temperature: Double?,
    val maxTokens: Int?,
    val contextTokenBudget: Int?,
    val unlimitedOutput: Boolean?,
)
```

规则：

- 默认继承全局模型配置。
- 主创可覆盖。
- 议员可覆盖。
- 写手默认继承主创。
- 预设/隐藏主创也走同一套配置，只隐藏内置提示词。

验收：

- prompt 请求构建时能解析“最终模型配置”。
- 未配置覆盖时走全局默认。
- 修改全局默认不应强行覆盖已经单独配置过的创作者。

### Phase 3F: MemoryPool 和参会记录

目标：先做最小记忆闭环，不要一开始做复杂智能记忆。

建议实体：

- `CreatorMemoryEntity`
- `CreatorParticipationRecordEntity`

最低字段：

- `creatorId`
- `projectId`
- `sessionId`
- `roundtableId` 或 `roundId`
- `sourceMessageId`
- `summary`
- `rawExcerpt`
- `createdAt`
- `invalidatedAt`

验收：

- 创作者能看到自己参与过哪些圆桌。
- 圆桌发言后能生成或记录参会摘要。
- 删除席位不删除记忆。
- 删除创作者才删除其记忆。

### Phase 3G: 跑团作为独立 SessionKind

目标：数据层先把模式边界立住。

建议：

```kotlin
enum class SessionKind {
    CREATIVE,
    ROLEPLAY
}
```

规则：

- `CREATIVE` 可在交流/圆桌视图之间切换。
- `ROLEPLAY` 是独立会话类型。
- 两者创建后不可互切。
- 跑团的 TXT/章节结构化应复用 `Project/Chapter/Material` 体系。

验收：

- 新建会话入口能选择创作或跑团。
- 普通创作会话看不到跑团专属推进按钮。
- 跑团会话看不到圆桌切换入口。

## UI 对齐原则

Android UI 要靠拢 Web 版，而不是 Material 3 重设计。

优先复刻：

- 干净、低干扰、贴近创作。
- 普通交流界面简洁。
- 圆桌像“稿纸盖在群聊上”。
- 正文位于视觉中心。
- 参会人/资料/模型配置是二级界面，不要铺满主聊天。
- 对话气泡贴合文字，不要大块空白。
- 流式输出不要整屏闪烁。

Material 3 可以作为组件库，不应主导产品风格。

## PR #2 可吸收的部分

`android-native/` 不作为主线，但里面有些想法可以吸收：

- Kotlin + Compose 的可测试纯逻辑 kernel 思路。
- `Roundtable.parseMentions` 的 @ 解析测试。
- 圆桌 pending queue / resume 的状态机思路。
- `OpenAIClient` 流式 SSE 解析思路。
- `SessionTransfer` 的 JSON 导入导出思路。
- `MemoryExtractor` 的“从近期对话抽取记忆”思路。
- 单元测试覆盖领域逻辑，而不是只测 UI。

吸收方式：

- 读思路，重写到 `android-app/` 当前架构里。
- 不直接搬目录。
- 不引入独立 app id。
- 不让 `android-native/` 成为新的主线。

## PR #2 不要合并的部分

不要合并：

- `android-native/` 整个并行工程目录。
- 独立 app id。
- Material You 重设计方向。
- `PersonaStore` 作为正式创作者系统。
- 以 `Persona / RoundtableConfig` 为核心的旧领域模型。
- 任何会绕开当前 `Project/Chapter/ProjectMaterial/CharacterCard/Session.projectId` 体系的代码。

如果需要参考代码，请按功能拆读，不要整体 merge。

## 推荐 android-app 包结构

基于当前 `android-app/` 主线，建议第三阶段逐步形成：

```text
android-app/app/src/main/java/com/qinglan/chatnovel/
  data/
    room/
      ProjectEntity
      ChapterEntity
      ProjectMaterialEntity
      CharacterCardEntity
      SessionEntity
      CreatorEntity
      RoundtableSeatEntity
      CreatorMemoryEntity
      ModelBindingEntity
    repository/
      ProjectRepository
      CreatorRepository
      RoundtableRepository
      MemoryRepository
  domain/
    project/
      ProjectContextProvider
    creator/
      CreatorIdentity
      ModelBindingResolver
    roundtable/
      RoundtableOrchestrator
      MentionResolver
      WriterAgent
    memory/
      MemoryRetriever
      MemoryWriter
  ui/
    native/
      NativeActivity
    chat/
    roundtable/
    materials/
      NovelMaterialsPanel
    creator/
    manuscript/
```

实际命名以当前 `android-app/` 代码风格为准。

## 第三阶段完成标准

第三阶段完成后，至少应满足：

- `android-app/` 原生入口仍是主入口。
- 新建/打开会话时能绑定作品项目。
- 当前作品资料能稳定进入 prompt。
- 每个创作会话有固定主创。
- 圆桌有席位模型，不再只是普通成员列表。
- 写手能向正文沉淀内容。
- 创作者模型配置有默认继承和单独覆盖。
- 创作者记忆池有最小写入和检索能力。
- 跑团会话类型在数据层明确存在。
- UI 继续靠拢 Web 版，而不是转向 Material You 重设计。

第三阶段不要追求一次实现所有交互。只要领域骨架立住，第四阶段再追圆桌视觉、预设主创界面、复杂记忆管理和跑团阅读体验。
