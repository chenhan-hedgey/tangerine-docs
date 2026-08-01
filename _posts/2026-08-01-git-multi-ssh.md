---
layout: post
title: "Git 多账号 SSH 配置"
date: 2026-08-01
description: "一台电脑上为多个 GitHub 账号配置不同的 SSH key"
tags: 工具 git
categories: 笔记
---

# Git 多账号 SSH 配置

## 背景

一台电脑上有多个 GitHub 账号时，需要为每个账号配置独立的 SSH key。

## 步骤

### 1. 为每个账号生成 SSH key

```bash
ssh-keygen -t ed25519 -C "账号备注" -f ~/.ssh/id_ed25519_账号名
```

### 2. 配置 ~/.ssh/config

```
Host github.com
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_rsa
    IdentitiesOnly yes

Host github-另一个账号
    HostName github.com
    User git
    IdentityFile ~/.ssh/id_ed25519_另一个账号
    IdentitiesOnly yes
```

### 3. 把各自公钥添加到对应的 GitHub 账号

到 https://github.com/settings/ssh/new 添加。

### 4. 克隆时用对应的 Host

```bash
git clone git@github-另一个账号:用户名/仓库.git
```

## 注意事项

- `IdentitiesOnly yes` 很重要，否则 SSH 会尝试所有 key，可能触发限流
- 已有的 remote 可以用 `git remote set-url` 修改
