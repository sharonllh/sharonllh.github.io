---
layout: post
title: FlatBuffers的schema介绍
date: 2025-08-07 +0800
tags: [FlatBuffers, schema]
categories: [FlatBuffers]
---

FlatBuffers是谷歌开源的一个跨平台的序列化库，适用于游戏开发等性能要求较高的场景。最近在项目中尝试引入FlatBuffers，在此边学习边记录一下。本文介绍最基本的FlatBuffers schema。

## Schema文件结构

FlatBuffers的schema文件以`.fbs`结尾，可以同时定义多个schema文件。

在一个schema文件中，通常包含以下几个部分：
1. 命名空间
2. `include`语句：用于引入其他schema文件
3. 类型定义
4. `root_type`语句：用于指定序列化的根类型

下面是一个例子，有两个schema文件，其中第二个schema引用了第一个schema中的类型。

1. `weapon.fbs`文件
    ```
    namespace MyGame.Sample;

    table Weapon {
        name:string;
        damage:short;
    }
    ```

2. `monster.fbs`文件

    ```
    include "weapon.fbs";

    namespace MyGame.Sample;

    enum Color : byte {
        Red = 0,
        Green = 1,
        Blu = 2
    }

    union Equipment {
        Weapon
    }

    struct Vec3 {
        x:float;
        y:float;
        z:float;
    }

    table Monster {
        pos:Vec3;
        mana:short = 150;
        hp:short = 100;
        name:string;
        friendly:bool = false (deprecated);
        inventory:[ubyte];
        color:Color = Blue;
        weapons:[Weapon];
        equipped:Equipment (required);
        path:[Vec3];
    }

    root_type Monster;
    ```

## Schema类型定义

### 内置类型

- 数值类型
    - `byte`/`ubyte`
    - `short`/`ushort`
    - `int`/`uint`
    - `long`/`ulong`
    - `float`
    - `double`
- 布尔类型：`bool`
- 字符串：`string`
- 数组：`[type]`

其中，数值和布尔类型是标量（scalar），即大小固定、不包含内部子结构的原子类型。

### 自定义类型

#### 枚举类型

例子：
```
enum Color : byte {
    Red = 0,
    Green = 1,
    Blu = 2
}
```

注意点：
- 必须指定底层的整数，否则编译时会报错：
    ```
    error: must specify the underlying integer type for this enum (e.g. ': short', which was the default).
    ```
- 不支持C#中的flags enum

#### struct类型

struct类型是FlatBuffers中定义对象的一种方式，适用于定义永远不会变的简单类型。相比于table，它使用更少的内存，访问也更快。

例子：
```
struct Vec3 {
    x:float;
    y:float;
    z:float;
}
```

注意点：
- struct类型的字段只能是标量或者其他struct，否则会报错：
    ```
    error: structs may contain only scalar or struct fields
    ```
- struct类型的字段不能设置默认值，否则会报错：
    ```
    error: default values are not supported for struct fields, table fields, or in structs.
    ```
- struct类型的字段是required的，必须设置值。

#### table类型

table类型是Flatbuffers中另一种定义对象的方式，可以进行添加、淘汰（而非删除）字段等操作，而不影响前后兼容性。

FlatBuffers中的`root_type`只能是table类型。

例子：
```
table Monster {
    pos:Vec3;
    mana:short = 150;
    hp:short = 100;
    name:string;
    friendly:bool = false (deprecated);
    inventory:[ubyte];
    color:Color = Blue;
    weapons:[Weapon];
    equipped:Equipment (required);
    path:[Vec3];
}
```

##### 关于默认值
- 标量类型的默认值是`0`，其他类型的默认值是`null`。
- 可以给标量类型设置自定义的默认值，如`mana:short = 150`。
- 不可以给非标量类型设置自定义的默认值，否则会报错：
    ```
    error: Default values for strings and vectors are not supported in one of the specified programming languages
    ```

##### 关于`required`属性
- 只有非标量类型的字段才能设置为`required`，否则会报错：
    ```
    error: only non-scalar fields in tables may be 'required'
    ```
- 设置为required的字段，必须要设置值。

##### 关于nullable类型

对于标量类型，如果把默认值设为`null`，编译后就会生成nullable类型。

例如`mana:short = null`，生成的C#代码为`public short? Mana`。

#### union类型

例子：
```
union Equipment {
    Weapon
}
```

注意点：
- union类型中，只能包括table和struct类型的字段，否则会报错：
    ```
    error: type referenced but not defined (check namespace): string, originally at: monster.fbs:15
    ```
- 编译后，会生成以下枚举类型，表示实际存储的值的类型。`NONE`是自动添加的，表示没有存储的值。
    ```cs
    public enum Equipment : byte
    {
        NONE = 0,
        Weapon = 1,
    };
    ```
- 在table中，可以定义union类型的字段，如`equipped:Equipment (required)`。编译后，会生成以下代码：
    ```cs
    public MyGame.Sample.Equipment EquippedType
    public TTable? Equipped<TTable>() where TTable : struct, IFlatbufferObject
    public MyGame.Sample.Weapon EquippedAsWeapon()
    ```
    - `EquippedType`属性：表示实际存储的值的类型。
    - `Equipped<TTable>`函数：获取实际存储的值，类型为`TTable`。
    - `EquippedAsWeapon`函数：获取实际存储的值，类型为`Weapon`。

## 编译Schema文件

下载`flatc`编译器，并在控制台运行下列命令，就可以把schema文件编译成C#代码：

```bash
flatc --csharp monster.fbs weapon.fbs
```

需要注意的是，编译时必须指定所有的schema文件。如果只指定了`monster.fbs`，那只会生成`Monster`类型，而不会生成`Weapon`类型，从而出现错误。

## 参考
[FlatBuffers Docs](https://flatbuffers.dev/tutorial/)