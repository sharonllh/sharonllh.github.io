---
layout: post
title: 常用的Git命令
date: 2024-03-02 +0800
tags: [Git命令]
categories: [Git]
---

### 设置用户名和邮箱

```bash
git config --global user.name "Your Name"
git config --global user.email "your.email@example.com"
```

使用`--global`可以让设置应用于整个系统，如果只想对某一个仓库进行设置，就忽略`--global`。

### 查看用户名和邮箱

```bash
git config user.name
git config user.email
```

### 复制一个文件并修改后，如何对比两个文件

```bash
git diff --no-index <old_file_path> <new_file_path>
```

使用`--word-diff`可以按单词变化高亮，而不是按行高亮。
