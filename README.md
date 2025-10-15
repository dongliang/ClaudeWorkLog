# Claude 协作工作日志

这个项目用于保存与 Claude 每次协作对话的记录和提炼的知识。

## 📁 项目结构

```
ClaudeWorkLog/
├── README.md              # 项目说明和索引
├── _templates/            # 模板文件
│   └── session-template.md  # 单次协作记录模板
├── sessions/              # 协作记录(按日期命名)
│   └── YYYY-MM-DD_标题.md
└── insights/              # 提炼的知识和最佳实践
    └── 主题.md
```

## 🎯 项目目标

1. **完整备份** - 保存与 Claude 协作的完整工作过程
2. **方便查证** - 日后可以回顾具体是如何解决问题的
3. **提炼知识** - 总结新思路、新方法、最佳实践

## 📝 使用方式

### 备份对话

当完成一次有价值的协作后,对 Claude 说:
```
备份对话
```

Claude 会:
1. 自动提炼对话内容
2. 使用模板生成结构化记录
3. 保存到 `sessions/` 目录
4. 提醒你进行 git 提交

### 手动记录

也可以手动使用模板创建记录:
1. 复制 `_templates/session-template.md`
2. 重命名为 `sessions/YYYY-MM-DD_标题.md`
3. 填写内容

### 提炼知识

当发现某个主题积累了多次协作经验后:
1. 在 `insights/` 目录创建主题文件
2. 从多个 session 中提取共性知识
3. 形成可复用的最佳实践

## 🔍 检索方式

### 按标题查找
- sessions 文件使用描述性标题命名
- 在文件管理器中快速定位

### 按标签查找
- 每个 session 底部有标签
- 使用文本搜索工具(如 vscode, grep)按标签检索

### 查看精华
- 直接浏览 `insights/` 目录
- 查看提炼后的最佳实践和知识点

## 📊 统计信息

- **总协作次数**: 0
- **最近更新**: 2025-01-15

## 🏷️ 常用标签

### 技术栈
- `#csharp` `#dotnet` `#python` `#javascript`
- `#database` `#redis` `#postgresql`

### 问题类型
- `#bugfix` `#feature` `#refactor` `#optimization`
- `#architecture` `#debugging`

### 领域
- `#backend` `#frontend` `#devops` `#testing`

---

**创建日期**: 2025-01-15
**维护者**: dongliang
