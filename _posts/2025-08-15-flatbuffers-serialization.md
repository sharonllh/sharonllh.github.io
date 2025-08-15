---
layout: post
title: FlatBuffers的序列化和反序列化
date: 2025-08-15 +0800
tags: [FlatBuffers, 序列化]
categories: [FlatBuffers]
---

在定义好Flatbuffers的schema，并且编译生成C#代码后，就可以进行序列化和反序列化了。本文以下面的schema为例，介绍如何进行序列化和反序列化。

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

## 序列化

### 规则

Flatbuffers的序列化比较麻烦，不能像JSON那样一步到位序列化整个对象，而是需要手动把每个字段写入buffer中，完成序列化。

不同类型的字段，写入buffer的规则是不一样的：
- 如果字段类型是大小固定的，在父对象序列化时直接把数据写入buffer。大小固定的类型包括：
    - 标量类型（如`int`、`double`、`bool`）
    - `enum`
    - `struct`
- 如果字段类型是可变大小的，需要提前序列化，把数据写入buffer，然后在父对象序列化时把offset写入buffer。可变大小的类型包括：
    - `string`
    - `vector`
    - `table`
    - `union`

编译生成的C#代码中，包含把每个字段写入buffer的方法。

### 步骤和例子

下面对`Monster`进行序列化。步骤如下：

1. 构造一个`FlatBufferBuilder`，其背后是一个buffer。初始大小设为1024个字节，如果不够，会自动扩容。
    ```cs
    // Construct a Builder with 1024 byte backing array
    FlatBufferBuilder builder = new FlatBufferBuilder(1024);
    ```

2. 观察`Monster`，找出所有可变大小的字段，包括：
    - `name:string`
    - `inventory:[ubyte]`
    - `weapons:[Weapon]`
    - `equipped:Equipment`
    - `path:[Vec3]`

3. 序列化可变大小的字段，并保留offset。
    - 序列化`name:string`：
        ```cs
        // Serialize name - string
        StringOffset nameOffset = builder.CreateString("Orc");
        ```

    - 序列化`inventory:[ubyte]`：
        ```cs
        // Serialize inventory - [ubyte]
        VectorOffset inventoryOffset = Monster.CreateInventoryVector(builder, new byte[] { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 });
        ```
        或者
        ```cs
        Monster.StartInventoryVector(builder, 10);
        for (int i = 9; i >= 0; i--)
        {
            builder.AddByte((byte)i);
        }
        VectorOffset inventoryOffset = builder.EndVector();
        ```

    - 序列化`weapons:[Weapon]`：
        ```cs
        // Serialize weapons - [Weapon]
        StringOffset weapon1NameOffset = builder.CreateString("Sword");
        Offset<Weapon> weapon1Offset = Weapon.CreateWeapon(builder, weapon1NameOffset, 3);
        StringOffset weapon2NameOffset = builder.CreateString("Axe");
        Offset<Weapon> weapon2Offset = Weapon.CreateWeapon(builder, weapon2NameOffset, 5);
        VectorOffset weaponsOffset = Monster.CreateWeaponsVector(builder, new Offset<Weapon>[] { weapon1Offset, weapon2Offset });
        ```
        或者
        ```cs
        StringOffset weapon1NameOffset = builder.CreateString("Sword");
        Offset<Weapon> weapon1Offset = Weapon.CreateWeapon(builder, weapon1NameOffset, 3);
        StringOffset weapon2NameOffset = builder.CreateString("Axe");
        Offset<Weapon> weapon2Offset = Weapon.CreateWeapon(builder, weapon2NameOffset, 5);
        Monster.StartWeaponsVector(builder, 2);
        builder.AddOffset(weapon2Offset.Value);
        builder.AddOffset(weapon1Offset.Value);
        VectorOffset weaponsOffset = builder.EndVector();
        ```

    - 序列化`path:[Vec3]`：
        ```cs
        // Serialze path - [Vec3]
        Monster.StartPathVector(builder, 2);
        Vec3.CreateVec3(builder, 1, 2, 3);
        Vec3.CreateVec3(builder, 4, 5, 6);
        VectorOffset pathOffset = builder.EndVector();
        ```

4. 序列化`Monster`，对于大小固定的字段，直接把数据写入buffer，对于大小可变的字段，把offset写入buffer。
    ```cs
    // Start serialization of Monster
    Monster.StartMonster(builder);

    // Add pos - Vec3
    Monster.AddPos(builder, Vec3.CreateVec3(builder, 1, 2, 3));

    // Add hp - short
    Monster.AddHp(builder, 300);

    // Add name - string
    Monster.AddName(builder, nameOffset);

    // Add inventory - [ubyte]
    Monster.AddInventory(builder, inventoryOffset);

    // Add color - Color
    Monster.AddColor(builder, Color.Red);

    // Add weapons - [Weapon]
    Monster.AddWeapons(builder, weaponsOffset);

    // Add equipped - Equipment
    Monster.AddEquippedType(builder, Equipment.Weapon);
    Monster.AddEquipped(builder, weapon2Offset.Value);

    // Add path - [Vec3]
    Monster.AddPath(builder, pathOffset);

    // Finish serialization of Monster
    Offset<Monster> monsterOffset = Monster.EndMonster(builder);
    ```

5. 结束序列化，并获取字节序列。
    ```cs
    // Finish the builder
    builder.Finish(monsterOffset.Value);

    // Get the bytes
    byte[] bytes = builder.SizedByteArray();
    ```

### 注意点

#### 关于数组的序列化

- 既可以用`CreateXXXVector`方法来一次性构建，也可以用`StartXXXVector`和`EndVector`方法来增量构建。
- 然而，并不是所有数组都有`CreateXXXVector`方法，得看元素的类型。
- 一次性构建的时候，按照正常的顺序添加元素；增量构建的时候，按照相反的顺序添加元素。

#### 关于union的序列化

上面的例子中，union类型的字段`equipped`中，实际包含的值是`Weapon`类型的。这个值需要在序列化`Monster`之前先序列化，由于这个值复用了`weapons`字段中的值，所以没有重复序列化。

在序列化`Monster`时，对于union字段，需要同时把以下信息写入buffer：
- 实际值的类型
- 实际值的offset

## 反序列化