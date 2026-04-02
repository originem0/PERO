# PERO

通过 Claude Code 驱动的个人学习系统。每个学科一个目录，`CLAUDE.md` 是唯一的状态文件——进入目录启动 Claude Code 即可续接学习。

**PERO = Priming → Encoding → Reference → Retrieval**

## 学科

| 目录 | 主题 | 状态 |
|------|------|------|
| `linux-server` | Linux 服务器基础 | 8/8 模块完成，待费曼检验 |
| `python-for-ai` | Python（面向 AI） | 3/14 模块完成 |
| `large-language-model` | 大语言模型原理 | 3/7 模块完成 |
| `computer-principles` | 计算机原理 | 模块 1 进行中 |
| `sql-base` | 关系数据库与 PostgreSQL | 模块 1 进行中 |
| `javascript` | JavaScript | Phase 0 完成 |
| `computer-networking` | 计算机网络 | Phase 0 未完成 |
| `makemore-practice` | makemore 代码实践 | 初始化 |
| `ai-era-engineering` | AI 时代工程判断力 | 初始化 |
| `linear-algebra` | 线性代数 | 初始化 |
| `english` | 英语精读（非 PERO 格式） | Chapter 1 段落 17 |

## 使用

```bash
# 进入任意学科目录，启动 Claude Code
cd python-for-ai
claude

# 开始或继续学习
> /PEROlearn

# 检验理解
> /PEROfeynman
```

## Skills 依赖

Skills 通过 [agent-skills](https://github.com/originem0/agent-skills) 仓库管理，不包含在本仓库内。

```bash
git clone git@github.com:originem0/agent-skills.git ~/agent-skills
cd ~/agent-skills && bash install.sh
```

## 多机同步

本仓库通过 GitHub 同步学习进度。每次 session 结束后 CLAUDE.md 会被更新，提交推送即可。

```bash
# 学习前拉取
git pull

# 学习后提交
git add -A && git commit -m "session: python 模块4" && git push
```
