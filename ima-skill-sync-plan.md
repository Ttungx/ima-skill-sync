# ima-skill Sync Plan

## 1. 项目概述

创建一个 GitHub 仓库 `ima-skills`，用于：
- 通过 GitHub Actions 定期从腾讯 IM 官方地址同步 `ima-skill` 压缩包
- 以 CCS（opencode skill 仓库）标准格式提供 skill，方便用户在 CCS 中添加仓库后直接同步更新
- 每次内容变更时自动发布 GitHub Release

## 2. 仓库目录结构

```
ima-skills/
├── index.json                        ← CCS 仓库清单（必选）
├── raw/                              ← 存放官方原始 zip 文件
│   └── ima-skills-1.1.7.zip
├── ima-skill/                        ← 解压后的 skill 文件目录
│   ├── SKILL.md                      ← 根 skill（name: ima-skill）
│   ├── ima_api.cjs
│   ├── meta.json
│   ├── knowledge-base/               ← 知识库子模块
│   │   ├── SKILL.md
│   │   ├── references/
│   │   │   └── api.md
│   │   └── scripts/
│   │       ├── cos-upload.cjs
│   │       └── preflight-check.cjs
│   └── notes/                        ← 笔记子模块
│       ├── SKILL.md
│       └── references/
│           └── api.md
└── .github/
    └── workflows/
        └── sync.yml                  ← GitHub Actions 同步工作流
```

## 3. index.json（CCS 仓库清单）

```json
{
  "skills": [
    {
      "name": "ima-skill",
      "files": [
        "SKILL.md",
        "ima_api.cjs",
        "meta.json",
        "knowledge-base/SKILL.md",
        "knowledge-base/references/api.md",
        "knowledge-base/scripts/cos-upload.cjs",
        "knowledge-base/scripts/preflight-check.cjs",
        "notes/SKILL.md",
        "notes/references/api.md"
      ]
    }
  ]
}
```

**说明：**
- `name` 必须匹配正则 `^[a-z0-9]+(-[a-z0-9]+)*$`，此处 `ima-skill` 符合
- `files` 必须包含 `SKILL.md`，按 skill 目录下的相对路径列出所有文件
- 不需要 `description` 字段，opencode 会从 `SKILL.md` 的 frontmatter 读取
- opencode 下载时会拼接 URL: `{repo_url}/{name}/{file}`
  - 例如: `https://raw.githubusercontent.com/{user}/ima-skills/main/ima-skill/SKILL.md`

## 4. GitHub Actions 工作流（sync.yml）

### 4.1 触发条件

```yaml
on:
  schedule:
    - cron: '0 6 */2 * *'     # 每 2 天 UTC 6:00 运行
  workflow_dispatch:           # 支持手动触发
```

### 4.2 完整工作流

```yaml
name: Sync ima-skill
on:
  schedule:
    - cron: '0 6 */2 * *'
  workflow_dispatch:

jobs:
  sync:
    runs-on: ubuntu-latest
    steps:
      # 1. 检出仓库
      - uses: actions/checkout@v4

      # 2. 下载官方 zip
      - name: Download official zip
        run: |
          curl -L -o /tmp/download.zip \
            "https://app-dl.ima.qq.com/skills/ima-skills-1.1.7.zip"

      # 3. SHA256 对比，判断内容是否有变化
      - name: Compare zip hash
        id: hash
        run: |
          NEW_HASH=$(sha256sum /tmp/download.zip | cut -d' ' -f1)
          echo "new_hash=$NEW_HASH"
          if [ -f raw/ima-skills-1.1.7.zip ]; then
            OLD_HASH=$(sha256sum raw/ima-skills-1.1.7.zip | cut -d' ' -f1)
            echo "old_hash=$OLD_HASH"
            if [ "$NEW_HASH" = "$OLD_HASH" ]; then
              echo "changed=false" >> $GITHUB_OUTPUT
              echo "no changes, skipping"
            else
              echo "changed=true" >> $GITHUB_OUTPUT
            fi
          else
            echo "changed=true" >> $GITHUB_OUTPUT
          fi

      # 4. 内容变更时同步文件
      - name: Sync files
        if: steps.hash.outputs.changed == 'true'
        run: |
          # 4a. 更新 raw/ 中的原始 zip
          cp /tmp/download.zip raw/ima-skills-1.1.7.zip

          # 4b. 解压到临时目录
          mkdir -p /tmp/extracted
          unzip -o /tmp/download.zip -d /tmp/extracted/

          # 4c. 使用 rsync 增量更新 ima-skill/ 目录
          #     -a: archive mode（保留权限、时间戳等）
          #     --delete: 删除目标中有但源中没有的文件
          #     只复制有差异的文件，避免 git 全量重写
          rsync -a --delete /tmp/extracted/ima-skill/ ./ima-skill/

      # 5. 提取版本号（从文件名 ima-skills-{version}.zip）
      - name: Extract version
        if: steps.hash.outputs.changed == 'true'
        id: version
        run: |
          FILENAME=$(basename raw/ima-skills-*.zip)
          VERSION=$(echo "$FILENAME" | sed -n 's/^ima-skills-\(.*\)\.zip$/\1/p')
          echo "version=$VERSION" >> $GITHUB_OUTPUT

      # 6. 提交变更到 git
      - name: Commit and push
        if: steps.hash.outputs.changed == 'true'
        run: |
          git config user.name "ima-skill-sync[bot]"
          git config user.email "bot@ima-skill-sync"
          git add raw/ ima-skill/
          git diff --cached --quiet || \
            git commit -m "sync ima-skill v${{ steps.version.outputs.version }} ($(date +%Y-%m-%d))"
          git push

      # 7. 创建 GitHub Release
      - name: Create Release
        if: steps.hash.outputs.changed == 'true'
        env:
          GITHUB_TOKEN: ${{ secrets.GITHUB_TOKEN }}
        run: |
          V=v${{ steps.version.outputs.version }}
          gh release create "$V" \
            --title "$V" \
            --notes "" \
            "raw/ima-skills-${{ steps.version.outputs.version }}.zip"
```

### 4.3 关键设计说明

| 步骤 | 方案 | 原因 |
|------|------|------|
| **去重** | SHA256 对比 raw/*.zip | 官方内容无变化时跳过全部后续步骤，避免空提交 |
| **防 git 膨胀** | `rsync -a --delete` 替代 `rm -rf + mv` | 只更新有差异的文件，git 只跟踪增量，不会每次全量重写目录 |
| **版本来源** | `sed` 从文件名提取 `ima-skills-(.*)\.zip` | 按需求：版本号来自原文件名 |
| **Release** | `gh release create` | 每次内容变化自动发版，zip 作为附件 |
| **跳过空提交** | `git diff --cached --quiet` | 即使 hash 不同但文件内容无实际变化时（极少情况），不产生空提交 |
| **同步频率** | 每 2 天 | 按需求设置 |

## 5. CCS 接入方式

用户在其 opencode 配置（`opencode.json`）中添加：

```json
{
  "skills": {
    "urls": [
      "https://raw.githubusercontent.com/{username}/ima-skills/main/"
    ]
  }
}
```

opencode 的处理流程：
1. 访问 `{url}/index.json` 读取 skill 清单
2. 遍历每个 skill，根据 `files` 列表从 `{url}/{name}/{file}` 下载文件
3. 缓存到 `~/.cache/opencode/skills/{name}/`
4. 扫描 `**/SKILL.md` 加载 skill（包括子目录中的 SKILL.md）

或者在 CCS 的 UI（opencode TUI 的 skill 管理界面）中直接添加仓库 URL。

## 6. 首次初始化步骤

初次搭建仓库时需要手动完成：

### 6.1 获取资源

当前工作目录 `D:\mytmp\opencode\` 下已有所需资源：
- `ima-skills-1.1.7.zip` — 官方原版压缩包
- `.opencode/skills/ima-skill/` — 已解压的 skill 目录

### 6.2 创建 GitHub 仓库

在 GitHub 上创建新仓库（如 `{username}/ima-skills`，public）

### 6.3 准备本地目录

```
ima-skills/                    ← 本地仓库根目录
├── index.json                 ← 按第 3 节内容创建
├── raw/                       ← 将 ima-skills-1.1.7.zip 复制到此
├── ima-skill/                 ← 将 .opencode/skills/ima-skill/ 内容复制到此
└── .github/workflows/
    └── sync.yml               ← 按第 4 节内容创建
```

### 6.4 首次提交

```bash
git init
git add .
git commit -m "init: ima-skill v1.1.7"
git remote add origin https://github.com/{username}/ima-skills.git
git push -u origin main
```

首次 push 后，Action 不会立即触发（只在 schedule 或手动 dispatch 时运行）。可以手动触发一次 `workflow_dispatch` 验证流程。

## 7. 注意事项

1. **URL 硬编码**：官方下载地址 `https://app-dl.ima.qq.com/skills/ima-skills-1.1.7.zip` 中的版本号是硬编码的。如果官方将文件名改为新版本（如 `ima-skills-1.1.8.zip`），需要手动更新 sync.yml 中的下载 URL。
2. **zip 在 git 中**：`raw/ima-skills-1.1.7.zip` 是二进制文件，每次同步都会替换。这是按需求设计的，但需要注意 zip 文件大小增长对 git 仓库的影响。
3. **SKILL.md 权限**：根 `SKILL.md` 的 frontmatter 中描述了环境变量要求（`IMA_OPENAPI_CLIENTID`、`IMA_OPENAPI_APIKEY`），用户需要在 opencode 配置中设置这些环境变量才能正常使用该 skill。
