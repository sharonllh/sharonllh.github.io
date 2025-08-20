---
layout: post
title: FlatBuffers的规范指南
date: 2025-08-20 +0800
tags: [FlatBuffers, 规范指南]
categories: [FlatBuffers]
---

定义FlatBuffers的schema时，应该遵循以下规范指南：

1. 命名规范

    | Schema元素 | 命名规范 |
    |-----|-----|
    | namespace名称 | UpperCamelCase |
    | enum名称, enum值 | UpperCamelCase |
    | table名称 | UpperCamelCase |
    | struct名称 | UpperCamelCase |
    | union名称 | UpperCamelCase |
    | table和struct的字段名 | snake_case |

    如果不遵循以上命名规范，可能会在编译时报warning，如下：
    ```
    warning: field names should be lowercase snake_case, got: equippedThing
    ```

2. 左花括号`{`位置：不换行

3. 关于空格
    - 缩进为2个空格
    - 定义字段时，名称和类型之间的`:`前后无空格，如`name:string`
    - 定义字段时，默认值的`=`两边加空格，如`hp:short = 100`