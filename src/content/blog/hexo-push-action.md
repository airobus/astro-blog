---
author: airobus
pubDatetime: 2024-12-01T15:22:00Z
modDatetime: 2024-12-01T09:12:47.400Z
title: 利用GitHub Actions实现Hexo博客的自动化部署
slug: "112275"
featured: true
draft: false
tags:
  - hexo
  - github actions
description:
  Automating Hexo Blog Deployment with GitHub Actions.
--- 

# 背景

关于hexo博客我有两个仓库：A：Hexo源码仓库，B：Hexo编译后的静态网页仓库。

使用Hexo博客框架时，我们经常面临以下挑战：

1. 每次更新博客需手动执行`hexo deploy`、`hexo algolia`等许多命令，操作繁琐
2. 需要管理两个仓库：一个用于Hexo源码，另一个用于编译后的静态网页
3. 手动部署过程耗时且容易出错
4. 希望快速部署静态文件，而非等待Hexo框架的较长部署时间

为解决这些问题，我们可以利用GitHub Actions实现自动化部署流程。只需将代码推送到仓库，剩余的工作就可以由GitHub Actions自动完成。这不仅简化了工作流程，还提高了效率和一致性。

# 简单介绍一下相关知识

## Hexo博客框架

Hexo是一个快速、简洁且高效的博客框架。它使用Markdown（或其他渲染引擎）解析文章，在几秒内，即可利用靓丽的主题生成静态网页。Hexo的主要特点包括：

- 超快速度：Node.js所带来的超快生成速度，让上百个页面在几秒内完成渲染。
- 支持Markdown：Hexo支持GitHub Flavored Markdown的所有功能，甚至可以整合Octopress的大多数插件。
- 一键部署：只需一条指令即可部署到GitHub Pages, Heroku或其他网站。
- 丰富的插件：Hexo拥有强大的插件系统，安装插件可以让Hexo支持 Jade, CoffeeScript。

然而，尽管Hexo本身已经相当便捷，但在实际使用中仍存在一些痛点，这就是我引入GitHub Actions的原因。

## GitHub Actions深入解析

GitHub Actions是GitHub提供的持续集成和持续部署（CI/CD）平台，它可以自动化软件开发工作流程。使用GitHub Actions，我们可以构建端到端的持续集成和持续部署capabilities直接在您的仓库中。

### GitHub Actions的本质与工作原理

GitHub Actions的工作流程是基于事件驱动的。当特定事件发生在您的仓库中时，例如推送代码或创建pull请求，可以触发一个工作流。工作流由一个或多个作业组成，这些作业可以按顺序运行，也可以并行运行。

每个作业都在一个运行器（runner）环境中运行，可以是GitHub托管的运行器，也可以是自己托管的运行器。作业包含一系列步骤，可以运行命令或使用预定义的操作（Actions）。

### GitHub Actions的核心优势

1. 与GitHub无缝集成：GitHub Actions直接集成在GitHub仓库中，无需额外的工具或服务。
2. 丰富的预设Actions：GitHub Marketplace提供了大量现成的Actions，可以直接在工作流中使用。
3. 灵活的自定义能力：支持Docker容器和多种编程语言环境，可以根据需求自定义工作流。
4. 慷慨的免费额度：对于公开仓库，GitHub Actions是完全免费的；私有仓库每月有2000分钟的免费额度。

# 实现Hexo博客自动化部署的详细步骤

## 配置GitHub仓库

1. 创建一个新的GitHub仓库或使用现有的Hexo博客仓库。
2. 确保仓库中包含完整的Hexo源文件，包括`_config.yml`、`package.json`等配置文件。

## 创建GitHub Actions工作流

1. 在仓库根目录创建`.github/workflows`文件夹。
2. 在该文件夹中创建`hexo-deploy.yml`文件。

## 编写自动化脚本

在`hexo-deploy.yml`文件中，添加以下内容：

```yaml
name: Hexo Action

on:
  push:
    branches:
      - master  # 或者是你的主分支名称
  workflow_dispatch:  # 允许手动触发

jobs:
  build:
    name: Hexo Action Job
    runs-on: ubuntu-latest

    steps:
      # 检出代码
      - name: Checkout repository
        uses: actions/checkout@v3

      # 设置 Node.js 环境
      - name: Set up Node.js
        uses: actions/setup-node@v3
        with:
          node-version: 18
          cache: 'npm'

      # 配置 Git
      - name: Configure Git
        run: |
          git config --global user.name "${{ github.actor }}"
          git config --global user.email "你的邮箱"

      # 安装依赖
      - name: Install dependencies
        run: |
          npm install -g hexo-cli
          npm install

      # 清理和构建
      - name: Clean and Build
        run: |
          hexo clean
          hexo generate
          hexo algolia

      # 构建输出上传为 artifact
      - name: Upload build output
        uses: actions/upload-artifact@v4
        with:
          name: hexo-build-output
          path: ./public

      # 直接使用环境变量设置 Git 凭证
      - name: Set Git credentials
        run: |
          git config --global credential.helper store
          echo "https://${{ github.actor }}:${{ secrets.DEPLOY_TOKEN }}@github.com" > ~/.git-credentials

    # 设置远程仓库的 URL 以使用 PAT
      - name: Set remote URL
        run: |
          git remote set-url origin https://${{ github.actor }}:${{ secrets.DEPLOY_TOKEN }}@github.com/airobus/airobus.github.io.git

      # 部署
      - name: Deploy
        run: hexo deploy
        env:
          DEPLOY_TOKEN: ${{ secrets.DEPLOY_TOKEN }}
      
      # 发送tg
      - name: Send TG message
        uses: appleboy/telegram-action@master
        with:
          to: ${{ secrets.TELEGRAM_TO }}
          token: ${{ secrets.TELEGRAM_TOKEN }}
          message: |
            Push Hexo-Blog Today
            Repository: ${{ github.repository }}
            See changes: https://github.com/${{ github.repository }}/commit/${{github.sha}}
```

这个工作流程会在每次推送到主分支时触发，自动构建Hexo站点并push到GitHub Pages和Vercel。工作流程的主要步骤包括：

1. 检出代码并设置Node.js环境
2. 安装Hexo CLI和项目依赖
3. 执行清理和构建
4. 更新Algolia搜索索引
5. push到GitHub Pages
6. 自动部署到Vercel（通过Vercel的GitHub集成）

## 配置必要的secrets

在GitHub仓库的Settings > Secrets中添加以下secret：

1. `DEPLOY_TOKEN`：GitHub Personal Access Token，用于部署到GitHub Pages
   - 访问 GitHub Settings > Developer settings > Personal access tokens
   - 创建新token，确保勾选 `repo` 权限
   - 复制token并添加到仓库secrets中

2. `ALGOLIA_ADMIN_KEY`：Algolia搜索服务的管理密钥
   - 在Algolia控制台获取Admin API Key
   - 用于更新搜索索引

3. Telegram通知配置（可选）：
   - `TELEGRAM_TO`：Telegram chat ID
   - `TELEGRAM_TOKEN`：Telegram Bot API token

## Algolia搜索配置

要使用Algolia搜索功能，需要进行以下配置：

1. 在Hexo的`_config.yml`中添加Algolia配置：
```yaml
algolia:
  applicationID: '你的Application ID'
  apiKey: '你的Search-Only API Key'
  adminApiKey: '你的Admin API Key'
  indexName: '你的索引名称'
  chunkSize: 5000
```

2. 安装Algolia依赖：
```bash
npm install hexo-algolia --save
```

## Vercel部署配置

Vercel部署会自动触发，但需要进行以下设置：

1. 在Vercel中导入你的Hexo仓库
2. 配置构建命令：
   - Build Command: `hexo generate`
   - Output Directory: `public`
3. 添加必要的环境变量
4. 启用自动部署功能

# 优势与效果

实施GitHub Actions自动化部署Hexo博客后，您将体验到以下优势：

1. 部署时间显著减少：从手动部署的几分钟缩短到自动部署的几十秒
2. 工作流程简化：无需手动执行部署命令，提交代码即可触发部署
3. 更新博客便利性提高：可以在任何设备上编辑文章，无需配置Hexo环境
4. 多平台同步部署：同时更新GitHub Pages和Vercel，提供更好的访问体验
5. 自动化搜索索引：通过Algolia自动更新搜索功能

# 注意事项与优化建议

1. 缓存优化：
   - 使用GitHub Actions的cache功能缓存npm依赖
   - 缓存`.deploy_git`目录加快部署速度
   - 设置合理的缓存过期时间

2. 错误处理：
   - 添加部署失败的重试机制
   - 设置超时时间
   - 配置错误通知

3. 安全性建议：
   - 定期轮换密钥和token
   - 使用最小权限原则配置access token
   - 启用分支保护，避免意外推送

4. 性能优化：
   - 使用并行任务处理多个部署目标
   - 优化构建过程，减少不必要的依赖
   - 合理配置Algolia索引更新策略

5. 监控建议：
   - 定期检查GitHub Actions运行日志
   - 监控Vercel部署状态
   - 设置关键指标告警

# 总结

通过利用GitHub Actions实现Hexo博客的自动化部署，我们成功解决了手动部署的诸多痛点。这不仅大大提高了博客维护的效率。随着持续集成和持续部署理念在个人博客领域的应用，我们可以期待更多创新的自动化解决方案，进一步优化博客管理流程。
