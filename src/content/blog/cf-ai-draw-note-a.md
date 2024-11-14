---
author: robus
pubDatetime: 2024-11-14T15:22:00Z
modDatetime: 2024-11-14T09:12:47.400Z
title: 基于 Cloudflare 实现 AI 绘图，记录笔记
slug: "112269"
featured: true
draft: false
tags:
  - cloudflare
  - ai
  - worker
description:
  Use AI to draw and record notes in Cloudflare.
--- 

# github 仓库

- [点击访问](https://github.com/1137882300/cf-ai-pic)

# 效果图

![AI 绘图](https://p.robus.cloudns.be/raw/ai-drawing-24-11-14_compressed.png)

# 功能特点

- **文本生成图片**: 通过自然语言描述生成独特的图片
- **智能提示词优化**: 
  - 使用 LLaMA 模型优化用户输入的提示词
  - 实时优化状态显示
  - 自动更新输入框内容
- **多种模型**: 支持选择不同的生成模型（默认、艺术风格、写实风格）
- **图片管理**:
  - 图片下载功能
  - 图片缩放功能
  - 实时生成状态显示
- **响应式设计**: 完美支持桌面端和移动端

# 准备工作

- 一个 Cloudflare 账号
- 一个 Cloudflare Pages 项目