# AI Agent Skills 知识库与最佳实践仓库

> **Skill 开发就是新时代的编程。**
> 
> 本仓库致力于沉淀高质量、生产级、跨工具标准的 **AI Agent Skills（操作手册与工作流资产）**。严格遵循“**省·准·稳**”的设计哲学，旨在为大语言模型（如 Antigravity、Claude Code、Cursor、OpenCode、Codex 等）提供精准的专业领域知识、标准工作流（SOP）及质量验收防线。

---

## 📁 仓库目录与技能清单

```text
skills/
├── README.md                   # 仓库使用与说明文档
├── 怎么写好一个 skill.md        # 📘 核心理论：Skill 开发标准化设计指南
└── h5-code-skill/              # 📱 实战技能：移动端 H5 开发与兼容性避坑指南
    ├── SKILL.md                # 主技能入口 (Frontmatter、铁律、工作流、参数路由)
    └── references/             # 按领域划分的深度避坑参考模块
        ├── html-pitfalls.md    # HTML & 系统能力调用篇
        ├── css-pitfalls.md     # CSS & 视觉布局适配篇
        ├── js-pitfalls.md      # JS & 交互逻辑避坑篇
        └── checklist.md        # 交付前质量检查清单 (P0~P3)
```

---

## 🌟 核心内容介绍

### 1. [怎么写好一个 skill.md](./怎么写好一个%20skill.md)
深度解析如何为 AI Agent 设计和编写高水准的 `SKILL.md` 操作手册：
* **核心哲学**：
  * **入口 (Entry)**：构建高命中率的 `description` 关键词网。
  * **省 (Economy)**：主入口严格控制在 500 行以内，复杂知识按需下沉到 `references/`，确定性逻辑下沉到 `scripts/`。
  * **准 (Precision)**：用“该问的问题”引导模型注意力，设立 Iron Laws（铁律）与 Red Flags（刹车反模式），点名 Anti-Patterns。
  * **稳 (Stability)**：显式工作流清单（`⛔ BLOCKING` / `⚠️ REQUIRED`）、决策门禁与可量化的验收清单。

### 2. [h5-code-skill](./h5-code-skill/SKILL.md)
基于大厂三年半移动端开发实战提炼的 **40 条核心坑位与最佳实践**：
* **四大核心铁律**：
  * ❌ 禁止直接使用 `new Date('YYYY-MM-DD')` 解析时间（iOS 必现 NaN），必须统一转换为 `/` 分隔。
  * ❌ 弹窗/抽屉必须配套完整的滚动穿透阻断（`ScrollLock`，记录并恢复 `scrollTop`）。
  * ❌ 视口必须声明 `viewport-fit=cover`，吸底元素必加 `env(safe-area-inset-bottom)` 安全区适配。
  * ❌ 表单输入必须兼顾 iOS 键盘收起失焦重绘与 Android 软键盘挤压吸底元素。
* **分模块知识体系**：
  * `html-pitfalls.md`：系统拨号/短信/拍照、数字键盘唤起、唤醒原生 App、禁用自动识别。
  * `css-pitfalls.md`：动态视口高度 `100dvh`、Retina 1px 细边框、消除点击灰底、弹性滚动与滚动传播阻断。
  * `js-pitfalls.md`：滚动穿透、软键盘异常、现代 300ms 消除、BFCache 恢复、微信环境适配、二维码识别。
  * `checklist.md`：P0 ~ P3 级别的移动端交付前自查与 Code Review 清单。

---

## 🚀 快速上手与部署安装

由于本项目中的 Skills 遵循开放的跨工具 `SKILL.md` 标准，你可以通过以下任一方式将其引入到你的开发工作流中：

### 方式 A：团队项目协同共享（推荐 ⭐⭐⭐⭐⭐）

将本技能库直接放入目标项目的 Agent 发现目录中，提交到 Git 仓库，团队所有成员 `git pull` 后全员自动生效：

```bash
# 进入你的项目根目录
cd your-project

# 方式 1: 直接 Clone 或复制本仓库的技能到项目
mkdir -p .agents/skills
cp -r /path/to/skills/h5-code-skill .agents/skills/

# 方式 2: 作为 Git Submodule 嵌入
git submodule add https://github.com/lanhaizhou/skills.git .agents/skills/my-skills
```

> **兼容路径说明**：
> - **Antigravity / Google Agent**：`.agents/skills/` 或 `.agent/skills/`
> - **Claude Code**：`.claude/skills/`

---

### 方式 B：本机全局配置（跨所有项目随时调用）

将特定 Skill 复制到本机的用户全局配置目录下：

* **Claude Code 全局生效**：
  ```bash
  mkdir -p ~/.claude/skills/
  cp -r h5-code-skill ~/.claude/skills/
  ```

* **Antigravity 全局生效**：
  ```bash
  mkdir -p ~/.gemini/config/skills/
  cp -r h5-code-skill ~/.gemini/config/skills/
  ```

---

## 💡 使用方式

安装完成后，在与 AI Agent 的对话中可以通过以下两种方式触发技能：

1. **显式指令唤起**：
   ```bash
   /h5-code-skill audit          # 对当前页面进行移动端兼容性审查
   /h5-code-skill fix scroll     # 获取生产级防滚动穿透方案
   /h5-code-skill css            # 专注于 CSS 样式与视口适配
   ```
2. **自然语言隐式触发**：
   直接询问相关问题，Agent 会自动对照 description 命中并激活该 Skill：
   > *“帮我写一个移动端底部抽屉弹窗组件，注意避免滚动穿透和 iOS 键盘遮挡。”*

---

## 🛠️ 新增 Skill 规范

若计划向本仓库贡献新的 Skill，请遵循以下规约：
1. 每个 Skill 必须为一个独立文件夹，包含且仅包含一个入口 `SKILL.md`。
2. 必须包含规范的 Frontmatter（`name`, `description`, 可选 `argument-hint`）。
3. `SKILL.md` 正文建议控制在 **500 行以内**，复杂或大篇幅知识必须按领域拆入 `references/` 目录。
4. 明确定义 **Iron Laws（铁律）**、**Red Flags（反模式）** 以及 **Probing Questions（引导注意力的自查问题）**。

---

## 📄 开源协议

本项目基于 [MIT License](https://opensource.org/licenses/MIT) 开源。欢迎提交 Issue 与 Pull Request 共同完善！
