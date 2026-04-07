# 规范驱动 + 需求 Loop AI 编程工作流 Demo

## 目标

用"登录功能"作为载体，在一个 React + TypeScript 项目中，跑通完整的下一代 AI 编程工作流，**以 OpenSpec 作为 Spec 层的核心基础设施**：

```
用户自然语言输入："我要实现一个登录功能"
       ↓
语义检索：匹配 Skill 库中相关的最佳实践
       ↓
AI 对话式补全：针对检索到的 Skill 逐一确认（非静态问卷，而是自然语言反问）
       ↓
[OpenSpec: proposal.md + specs/ + design.md + tasks.md]
       ↓
[代码实现]
       ↓
[OpenSpec: /opsx:verify 规范验证]
```

**核心设计原则**：Loop 的触发不是"打开一张问卷表"，而是用户先表达意图，系统语义召回相关 Skill，再以自然对话的方式逐项确认。这更接近真实的 AI 编程助手体验。

---

<!-- anchor:architecture -->

## 1. Demo 架构设计

### 1.1 整体分层

```
┌─────────────────────────────────────────────────────────┐
│  LAYER 0: 意图输入（自然语言对话入口）                      │
│  IntentInput.tsx                                        │
│  用户输入："我要实现一个登录功能"                           │
└──────────────────────────┬──────────────────────────────┘
                           ↓ 语义检索（关键词 + 标签匹配）
┌─────────────────────────────────────────────────────────┐
│  LAYER 1: Skill 库（最佳实践知识库）                       │
│  src/skills/login-best-practices.ts                     │
│  → 每条 Skill 有 tags/keywords，支持语义检索              │
│  → 检索结果："找到 8 条相关最佳实践"                       │
└──────────────────────────┬──────────────────────────────┘
                           ↓ 检索命中的 Skills → 进入 Loop
┌─────────────────────────────────────────────────────────┐
│  LAYER 2: AI 对话式需求补全 Loop                          │
│  src/workflow/loop-engine.ts                            │
│  → 不是展示问卷，而是 AI 以对话气泡的方式逐项询问            │
│  → "我注意到你要实现登录，通常需要考虑双 Token 机制，        │
│     它可以让 AT 短期有效而不必频繁重新登录，你需要吗？"       │
│  → 用户回复 Yes/No 或追加说明                             │
└──────────────────────────┬──────────────────────────────┘
                           ↓ Loop 完成 → 生成 OpenSpec
┌─────────────────────────────────────────────────────────┐
│  LAYER 3: OpenSpec（规范驱动核心）                         │
│  openspec/                                              │
│  ├── specs/auth-login/spec.md  ← 系统级规范（持久化）       │
│  └── changes/add-login/        ← 本次变更                 │
│      ├── proposal.md           ← 为什么做、做什么           │
│      ├── design.md             ← 技术决策                  │
│      ├── tasks.md              ← 实现任务清单               │
│      └── specs/                ← spec delta               │
│          └── auth-login/spec.md                          │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  LAYER 4: 代码实现（严格对照 Spec）                        │
│  src/components/Login/                                  │
│  → 每个实现对应 Spec 中的一条 Requirement + Scenario       │
└──────────────────────────┬──────────────────────────────┘
                           ↓
┌─────────────────────────────────────────────────────────┐
│  LAYER 5: OpenSpec Verify（闭环验证）                      │
│  /opsx:verify                                           │
│  → AI 对照 spec.md 中的 Requirements 逐条检查实现          │
│  → 在 Demo 的 UI 中可视化展示验证结果                      │
└─────────────────────────────────────────────────────────┘
```

### 1.2 Demo 的两个层面

**A. 工作流演示层（核心，在浏览器中交互）**

- 一个交互式「需求 Loop」页面，模拟 AI 反问 Loop 的全过程
- Loop 完成后展示自动生成的 OpenSpec 文档（proposal + spec delta）
- 最终展示 Check 验证结果面板

**B. 功能实现层（结果验证）**

- 根据 OpenSpec `tasks.md` 实现的登录组件
- 实现双 Token、防抖、表单校验、错误处理等最佳实践
- 在 Demo UI 中对照 `spec.md` 做 Requirement-level 的可视化验证

---

<!-- anchor:phase0 -->

## 2. Phase 0 — 安装 OpenSpec & 初始化

### 2.1 安装 OpenSpec

```bash
npm install -g @fission-ai/openspec@latest
cd /path/to/my-react-app
openspec init
```

执行后，项目根目录会生成：

```
openspec/
├── specs/          ← 系统级规范库（跨 feature 持久化）
└── changes/        ← 每次变更的 proposal/design/tasks/spec-delta
```

### 2.2 OpenSpec 核心工作流命令

| 命令                             | 说明                                  |
| -------------------------------- | ------------------------------------- |
| `/openspec:proposal "add-login"` | 创建本次变更提案，生成 4 个 artifacts |
| `/opsx:verify`                   | 对照 spec.md 逐条验证代码实现         |
| `/opsx:archive`                  | 归档当前变更，更新系统级 specs        |

OpenSpec 的哲学：**先对齐（align），再实现（implement）**。生成提案文档让人和 AI 在写代码前就对目标对齐，这正是我们的 Loop 要解决的问题。

---

<!-- anchor:phase1 -->

## 3. Phase 1 — 搭建基础工程环境

### 3.1 安装前端依赖

```bash
# UI 组件 & 样式
npm install tailwindcss @tailwindcss/vite

# 表单校验（与 OpenSpec Scenario 对应验证用）
npm install zod

# 表单管理
npm install react-hook-form @hookform/resolvers

# 路由（页面切换）
npm install react-router-dom

# 工具函数
npm install clsx
```

### 3.2 整体文件结构

```
my-react-app/
├── openspec/                      ← OpenSpec 规范层（Phase 0 生成）
│   ├── specs/
│   │   └── auth-login/
│   │       └── spec.md            ← 系统级登录规范（归档后持久化）
│   └── changes/
│       └── add-login/
│           ├── proposal.md        ← 变更提案：为什么做、做什么
│           ├── design.md          ← 技术决策：双 Token 方案等
│           ├── tasks.md           ← 实现任务清单（对应代码实现）
│           └── specs/
│               └── auth-login/
│                   └── spec.md    ← spec delta（本次变更的规范）
└── src/
    ├── skills/
    │   └── login-best-practices.ts   ← Skill 库（含 tags/keywords/aiQuestion）
    ├── workflow/
    │   ├── types.ts
    │   ├── intent-matcher.ts         ← 意图检索：用户输入 → 匹配相关 Skills
    │   ├── loop-engine.ts            ← Loop 状态机：idle→matching→confirming→done
    │   └── spec-generator.ts         ← Spec 生成器：Loop 答案 → OpenSpec 文档
    ├── components/
    │   ├── WorkflowDemo/
    │   │   ├── IntentInput.tsx        ← 意图输入框（对话起点）
    │   │   ├── ChatLoop.tsx           ← 对话气泡式 Loop UI（核心组件）
    │   │   ├── SpecViewer.tsx         ← Spec 文档展示（Markdown 渲染）
    │   │   └── VerifyPanel.tsx        ← 验证面板：逐条 Requirement Check
    │   └── Login/
    │       ├── LoginForm.tsx          ← 登录表单（对照 spec tasks 实现）
    │       └── useLoginLogic.ts       ← 登录逻辑 Hook
    └── pages/
        ├── WorkflowPage.tsx           ← 工作流演示主页（三步走）
        └── LoginPage.tsx              ← 最终登录页
```

---

<!-- anchor:phase2 -->

## 4. Phase 2 — Skill 库 + 意图检索 + 对话式 Loop 引擎

### 4.0 整体流程（核心设计）

**用户先表达意图，系统语义召回 Skill，再以 AI 对话气泡方式逐项确认，而不是展示一张静态问卷。**

```
Step 1: 用户输入意图
   → "我要实现一个登录功能"

Step 2: 系统语义检索（关键词 + 标签匹配）
   → 匹配 Skills 库，找到 8 条相关最佳实践
   → UI 展示："✨ 找到 8 条相关最佳实践，开始确认需求..."

Step 3: AI 对话式逐条确认（非问卷列表）
   → 以聊天气泡形式，每次一个问题，带上下文说明原因
   → 用户可以回答 "需要" / "不需要" / 或追加补充说明

Step 4: Loop 完成 → 生成 OpenSpec 文档
   → 自动生成 proposal.md + spec.md + design.md + tasks.md
```

### 4.1 Skill 数据结构（新增 tags/keywords + 对话式问法）

文件：`src/skills/login-best-practices.ts`

```typescript
interface Skill {
  id: string;
  category: "security" | "ux" | "performance" | "architecture";
  tags: string[]; // 语义检索标签，如 ['login', 'auth', '登录', '认证']
  keywords: string[]; // 关键词列表，用于意图匹配
  // AI 对话气泡的提问方式（自然语言，带上下文和原因）
  aiQuestion: string;
  defaultAnswer: boolean; // 推荐默认值
  specRequirement: string; // Loop 选 Yes 时，写入 OpenSpec 的 Requirement 语句
  skipIf?: string[]; // 当某些 skill 为 No 时跳过此问题
}
```

**关键区别：`aiQuestion` 是自然语言，包含上下文和原因：**

```typescript
// ❌ 旧方式（静态问卷式，无上下文）
question: "是否使用双 Token 机制？";

// ✅ 新方式（AI 对话气泡，有上下文和理由）
aiQuestion: `我注意到你要实现登录功能。通常建议使用
双 Token（Access Token + Refresh Token）机制——AT 
短期有效（15分钟），RT 长期有效（7天），这样既安全又
不用频繁重新登录。你的项目需要这个吗？`;
```

**内置 Skills 列表（登录场景）：**

| ID                 | tags（语义检索用）                                  | OpenSpec Requirement                                           |
| ------------------ | --------------------------------------------------- | -------------------------------------------------------------- |
| `dual-token`       | `['login', 'auth', 'token', '登录', '认证']`        | The system SHALL implement Access Token + Refresh Token        |
| `debounce`         | `['login', 'form', 'submit', '登录', '表单']`       | The system SHALL debounce login submissions (300ms)            |
| `form-validation`  | `['login', 'form', 'validate', '校验']`             | The system SHALL validate inputs in real-time via Zod          |
| `password-encrypt` | `['login', 'password', 'security', '密码', '安全']` | The system SHALL encrypt passwords before transmission         |
| `remember-me`      | `['login', 'session', 'persist', '记住我']`         | The system SHALL support persistent sessions via "Remember me" |
| `captcha`          | `['login', 'security', 'captcha', '验证码']`        | The system SHALL require captcha after 3 failed attempts       |
| `error-handling`   | `['login', 'error', 'feedback', '错误']`            | The system SHALL distinguish error types (account/password)    |
| `loading-state`    | `['login', 'ux', 'loading', '加载']`                | The system SHALL show loading indicator during auth            |
| `rate-limiting`    | `['login', 'security', 'brute-force', '限流']`      | The system SHALL lock login after N consecutive failures       |

### 4.2 意图检索引擎

文件：`src/workflow/intent-matcher.ts`

```typescript
interface IntentMatchResult {
  userIntent: string; // 原始用户输入，如 "我要实现一个登录功能"
  detectedFeature: string; // 解析出的功能类型，如 "login"
  matchedSkills: Skill[]; // 语义匹配到的 Skill 列表（按相关度排序）
  confidence: number; // 匹配置信度 0-1
}
```

匹配策略（Demo 中用轻量级关键词匹配模拟语义检索）：

1. 提取用户输入中的关键词（"登录"、"login"、"auth" 等）
2. 与每条 Skill 的 `tags` 和 `keywords` 计算交集得分
3. 得分 > 阈值的 Skill 进入候选列表，按分数降序排列
4. 返回 Top N 条 Skill 作为本次 Loop 的问题队列

> **真实场景扩展**：Demo 中用关键词匹配演示即可；真实产品可替换为向量 embedding 相似度检索。

### 4.3 对话式 Loop 引擎

文件：`src/workflow/loop-engine.ts`

```typescript
type LoopPhase =
  | "idle" // 等待用户输入意图
  | "matching" // 正在检索相关 Skills（带加载动画）
  | "confirming" // AI 逐条对话确认（核心阶段）
  | "generating" // 生成 OpenSpec 文档
  | "done"; // 完成，展示 Spec

interface LoopState {
  phase: LoopPhase;
  userIntent: string;
  matchedSkills: Skill[];
  currentSkillIndex: number;
  answers: Record<string, boolean | string>; // string 支持用户追加说明
  chatHistory: ChatMessage[]; // 完整对话记录（用于展示聊天界面）
  generatedSpec: OpenSpecDraft | null;
}

interface ChatMessage {
  role: "user" | "assistant";
  content: string;
  skillId?: string; // 对应哪个 Skill（用于侧边栏进度高亮）
}
```

**对话示例：**

```
用户:  "我要实现一个登录功能"

AI:    "好的，我找到了 8 条与「登录」相关的最佳实践，
        让我们逐一确认你的需求 ↓"

AI:    "我注意到你要实现登录功能。通常建议使用双 Token
        机制——AT 短期有效（15分钟），RT 长期有效（7天）。
        你的项目需要这个吗？"

用户:  "需要，RT 有效期改成 30 天"

AI:    "好的，记录：双 Token，RT=30天。
        另外，为了防止重复提交，建议给登录按钮加
        300ms 防抖。需要吗？"

用户:  "需要"

...（继续 8 条）

AI:    "需求已全部确认！根据你的选择，我生成了
        OpenSpec 文档，请查看 →"
```

### 4.4 Spec 生成器（Loop 答案 → OpenSpec 文档）

文件：`src/workflow/spec-generator.ts`

根据用户的 Loop 答案，自动生成符合 OpenSpec 格式的文档：

**生成 `proposal.md`：**

```markdown
# Proposal: Add Login Feature

## Why

用户需要一个安全的登录入口来访问系统。

## What's Changing

- 新增登录表单组件
- 实现 [用户选择的特性列表]

## Out of Scope

- [用户选择跳过的特性]
```

**生成 `specs/auth-login/spec.md`（spec delta）：**

```markdown
# auth-login Specification

## Requirements

### Requirement: [Skill.specRequirement]

[Loop 中用户选 Yes 的每一条 Skill]

#### Scenario: [对应场景]

- GIVEN [前置条件]
- WHEN [触发动作]
- THEN [期望结果]
```

**生成 `tasks.md`：**

```markdown
# Implementation Tasks

## Phase 1: Core Auth

- [ ] 1.1 实现双 Token 存储和刷新逻辑
- [ ] 1.2 实现登录 API 调用

## Phase 2: UX

- [ ] 2.1 实现防抖提交
- [ ] 2.2 实现 Zod 表单校验
```

**生成 `design.md`：**

```markdown
# Technical Design

## Token Storage Decision

选择 localStorage 存储 AT（有 XSS 风险但开发方便）vs httpOnly Cookie（更安全）...
```

---

<!-- anchor:phase3 -->

## 5. Phase 3 — 对照 OpenSpec Tasks 实现登录功能

### 5.1 OpenSpec `tasks.md` 驱动实现

对照 `openspec/changes/add-login/tasks.md` 中的每一条 Task 进行实现。每完成一条 Task 可在 Demo 的 VerifyPanel 中标记。

### 5.2 核心实现文件

**`src/components/Login/useLoginLogic.ts`**

- 双 Token 管理（accessToken + refreshToken）
- 防抖 Hook（`useDebounce`）
- 失败次数计数器（限流）
- Remember Me 逻辑

**`src/components/Login/LoginForm.tsx`**

- React Hook Form + Zod 实时校验
- Loading 状态管理
- 细化错误类型展示
- 密码可见切换

---

<!-- anchor:phase4 -->

## 6. Phase 4 — VerifyPanel（对照 OpenSpec spec.md 验证）

### 6.1 核心思想

将 OpenSpec `spec.md` 中的每一条 **Requirement** 和 **Scenario** 解析出来，在 Demo UI 中对照实际代码状态进行可视化展示。

这模拟了 `/opsx:verify` 命令的效果（AI 对照 spec 逐条检查），只不过在 Demo 中以可视化 UI 呈现。

### 6.2 VerifyPanel 数据结构

文件：`src/components/WorkflowDemo/VerifyPanel.tsx`

```typescript
interface VerifyItem {
  requirementId: string;
  requirement: string; // 来自 spec.md 的 Requirement 描述
  scenarios: {
    given: string;
    when: string;
    then: string;
  }[];
  implementedBy: string; // 对应哪个代码文件/函数
  status: "pass" | "fail" | "skip";
}
```

**UI 展示：**

- ✅ Pass：规范要求，已按 Scenario 实现
- ❌ Fail：规范要求，未找到对应实现
- ⏭ Skip：用户在 Loop 中选择了 No
- 最终给出 "规范覆盖率" 百分比

---

<!-- anchor:phase5 -->

## 7. Phase 5 — 整合 UI & 展示效果

### 7.1 页面路由

- `/` → 工作流演示页（三步走：Loop → Spec → Verify）
- `/login` → 最终产物：按照 Spec 实现的登录页

### 7.2 WorkflowPage 四阶段布局（新增意图输入阶段）

```
[阶段 0: 意图输入]       [阶段 1: 对话式 Loop]      [阶段 2: OpenSpec]        [阶段 3: 验证]
┌─────────────────┐     ┌──────────────────────┐   ┌──────────────────┐    ┌────────────────┐
│ � 描述你的需求  │ →→→ │ 🤖 AI 对话气泡        │   │ 📄 proposal.md   │    │ ✅ Verify      │
│                 │     │ ──────────────────── │   │ 📋 spec delta    │    │                │
│  "我要实现一个   │     │ AI: "我找到 8 条相    │   │ 🔧 design.md     │    │ 覆盖率: 92%    │
│   登录功能"     │     │      关实践，逐一确   │   │ ✅ tasks.md      │    │                │
│                 │     │      认..."           │   │                  │    │ ✅ dual-token  │
│ [开始分析 →]   │     │ AI: "是否需要双        │   │ [Markdown 渲染]  │    │ ✅ debounce    │
│                 │     │      Token？AT 15min  │   │                  │    │ ❌ captcha     │
│ 右侧匹配到的     │     │      RT 7天..."       │   │ [复制] [下载]    │    │ ✅ loading...  │
│ Skills 预览      │     │ 用: "需要，RT 30天"   │   │                  │    │                │
│ ✨ 找到 8 条     │     │ AI: "好的！防抖呢？"  │   │                  │    │ [查看登录页 →] │
└─────────────────┘     └──────────────────────┘   └──────────────────┘    └────────────────┘
```

---

<!-- anchor:impl-order -->

## 8. 实施顺序

| 步骤 | 任务                                             | 文件                                          | 优先级 |
| ---- | ------------------------------------------------ | --------------------------------------------- | ------ |
| 0    | 安装 OpenSpec CLI & 初始化                       | `openspec/`                                   | P0     |
| 1    | 安装前端依赖、配置 Tailwind、配置路由            | `package.json`, `vite.config.ts`              | P0     |
| 2    | 定义 Skill 数据库（含 tags/keywords/aiQuestion） | `src/skills/login-best-practices.ts`          | P0     |
| 3    | 实现意图检索引擎                                 | `src/workflow/intent-matcher.ts`              | P0     |
| 4    | 实现对话式 Loop 状态机                           | `src/workflow/loop-engine.ts`                 | P0     |
| 5    | 实现 Spec 生成器（输出 OpenSpec 格式 Markdown）  | `src/workflow/spec-generator.ts`              | P0     |
| 6    | 实现意图输入组件（对话起点）                     | `src/components/WorkflowDemo/IntentInput.tsx` | P1     |
| 7    | 实现对话气泡式 Loop UI                           | `src/components/WorkflowDemo/ChatLoop.tsx`    | P1     |
| 8    | 实现 Spec 文档查看器（Markdown 渲染）            | `src/components/WorkflowDemo/SpecViewer.tsx`  | P1     |
| 9    | 实现 VerifyPanel（对照 spec.md 验证）            | `src/components/WorkflowDemo/VerifyPanel.tsx` | P1     |
| 10   | 实现登录逻辑 Hook（对照 tasks.md 实现）          | `src/components/Login/useLoginLogic.ts`       | P1     |
| 11   | 实现登录表单组件                                 | `src/components/Login/LoginForm.tsx`          | P1     |
| 12   | 手工创建初始 OpenSpec 文档（演示用）             | `openspec/changes/add-login/`                 | P1     |
| 13   | 组装 WorkflowPage（四阶段步骤导向）              | `src/pages/WorkflowPage.tsx`                  | P2     |
| 14   | 组装 LoginPage                                   | `src/pages/LoginPage.tsx`                     | P2     |
| 15   | 更新 App.tsx 路由                                | `src/App.tsx`                                 | P2     |

---

<!-- anchor:key-insight -->

## 9. 核心设计哲学（Demo 要传递的思想）

1. **Skill = 最佳实践知识库**：团队沉淀的经验，以机器可读的方式存储，而非散落在 Wiki 文档里

2. **Loop = 意图驱动的需求对齐自动化**：
   - 用户先用自然语言描述意图（"我要实现登录功能"）
   - 系统语义检索匹配相关 Skill（关键词匹配，可扩展为 embedding 检索）
   - AI 以对话气泡方式逐条确认，而非展示静态问卷
   - 最终输出 OpenSpec 提案（proposal + spec + design + tasks）

3. **OpenSpec = 代码与需求的唯一真相来源**：
   - `spec.md` 中的 GIVEN/WHEN/THEN 是人类可读的规范
   - `tasks.md` 是 AI 实现时的任务清单
   - `proposal.md` 是变更的"为什么"存档
   - 一份 spec 可以跨多个 feature 复用和累积

4. **Verify = 闭环保障**：
   - OpenSpec 的 `/opsx:verify` 让 AI 对照 spec.md 逐条检查代码
   - Demo 的 VerifyPanel 把这个过程可视化
   - 消除"我以为实现了"的幻觉

5. **可扩展性**：
   - `openspec/specs/` 随着项目迭代不断积累，形成越来越完整的系统规范库
   - 新人入职读 `specs/` 就能理解系统；AI 接任务读 `specs/` 就能理解上下文
   - 这套工作流不局限于登录，任何功能（支付、搜索、上传）都可以走 `opsx:proposal` 流程
