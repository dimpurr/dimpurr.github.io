---
title: 将 Hexo CI 迁移到 Github Actions
tags:
---

两年前配置了 Travis CI 的自动部署，按说应该把 hexo 分支自动部署到主分支上来实现 Github Pages ，但是惊闻 Travis CI 停止了免费服务。因此需要换到自带的 Github Actions 上。

在 Github Marketplace 找到了如下 Action ： https://github.com/marketplace/actions/hexo-action