# Workflow 仓库

这个仓库包含可复用的 GitHub Actions workflows，供其他仓库引用。

## Claude Code Workflow

### 功能说明

当 issue 评论中包含 `@claude` 时，自动触发 Claude Code 处理评论。

### 使用方法

在其他仓库中创建 `.github/workflows/claude-issue.yml` 文件，内容如下：

```yaml
name: Claude Issue Comment

on:
  issue_comment:
    types: [created]

jobs:
  claude:
    if: |
      (github.event_name == 'issue_comment' && contains(github.event.comment.body, '@claude'))
    uses: your-org/workflow/.github/workflows/claude-issue.yml@main
    secrets:
      OPENAI_API_KEY: ${{ secrets.OPENAI_API_KEY }}
```

**注意：** 请将 `your-org/workflow` 替换为实际的仓库路径（例如：`catvote/workflow`）。

### 配置要求

1. 在目标仓库的 Settings > Secrets and variables > Actions 中添加 `OPENAI_API_KEY` secret
2. 确保仓库有写入 issues 的权限

### 使用示例

在 issue 评论中输入 `@claude` 加上你的问题，例如：

```
@claude 请帮我检查这段代码有什么问题
```

Claude 会自动处理并回复。

