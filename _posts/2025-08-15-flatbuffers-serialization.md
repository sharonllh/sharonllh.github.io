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

### 步骤

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

#### 关于可变大小字段的序列化

可变大小的字段必须在`Monster.StartMonster(builder)`之前就序列化，否则运行时会报错。

举个例子，假如不事先序列化`name:string`：
```cs
// Add name - string
Monster.AddName(builder, builder.CreateString("Orc"));
```

运行时会报错：
```
Unhandled exception. System.Exception: FlatBuffers: object serialization must not be nested.
   at Google.FlatBuffers.FlatBufferBuilder.NotNested()
   at Google.FlatBuffers.FlatBufferBuilder.CreateString(String s)
   at FlatBuffersTest.Program.Serialize() in XXX\FlatBuffersTest\Program.cs:line 67
   at FlatBuffersTest.Program.Main(String[] args) in XXX\FlatBuffersTest\Program.cs:line 10
```

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

Flatbuffers的反序列化不会把整个buffer完全解析、转换成内存对象，而是可以只读取需要的字段。对于那些不需要的字段，FlatBuffers不会去读取或拷贝到内存中，因此可以减少内存开销、提高性能。

### 步骤

反序列化的步骤如下：

1. 获取root object的视图。需要注意的是，数据都在buffer中，并没有发生任何复制。
    ```cs
    // Get an view to the root object inside the buffer
    Monster monster = Monster.GetRootAsMonster(new ByteBuffer(bytes));
    ```

2. 读取需要的字段。读取的方法来自于编译生成的C#代码，不同类型的字段需要不同的读取方法。

    - 读取`short`字段：
        ```cs
        // Deserialize hp - short
        short hp = monster.Hp;
        Console.WriteLine($"hp={hp}");
        ```

    - 读取`string`字段：
        ```cs
        // Deserialize name - string
        string name = monster.Name;
        Console.WriteLine($"name={name}");
        ```
    
    - 读取`enum`字段：
        ```cs
        // Deserialize color - Color
        Color color = monster.Color;
        Console.WriteLine($"color={color}");
        ```
    
    - 读取`struct`字段：
        ```cs
        // Deserialize pos - Vec3
        Vec3 pos = monster.Pos.Value;
        float x = pos.X;
        float y = pos.Y;
        float z = pos.Z;
        Console.WriteLine($"pos=[{x}, {y}, {z}]");
        ```
    
    - 读取数组字段：
        ```cs
        // Deserialize inventory - [ubyte]
        byte[] inventory = new byte[monster.InventoryLength];
        for (int i = 0; i < inventory.Length; i++)
        {
            inventory[i] = monster.Inventory(i);
        }
        Console.WriteLine($"inventory=[{string.Join(", ", inventory)}]");
        ```
    
    - 读取`union`字段：
        ```cs        
        // Deserialize equipped - Equipment
        Equipment equippedType = monster.EquippedType;
        if (equippedType == Equipment.Weapon)
        {
            Weapon equipped = monster.Equipped<Weapon>().Value;
            ////Weapon equipped = monster.EquippedAsWeapon();
            Console.WriteLine($"equipped=({equipped.Name}, {equipped.Damage})");
        }
        ```

### 注意点

#### 关于零拷贝（zero-copy）的理解

Flatbuffers反序列化时，会在原始的buffer上根据offset来读取字段，并直接返回，这个过程不会产生内存拷贝。

然而，在上面的示例代码中，我们会把读取的字段存储到临时变量中，这个会产生内存拷贝。

#### 关于数组的读取

读取数组字段时，需要用到两个方法：
1. `XXXLength`：读取数组大小
2. `XXX(i)`：读取索引为`i`的元素

#### 关于union的读取

读取`union`字段时，需要用到两个方法：
1. `XXXType`：读取值的类型
2. `Equipped<XXX>()`或`EquippedAsXXX()`：读取值

#### 关于nullable字段

在C#中，很多类型的字段会被编译生成nullable类型，在读取的时候需要注意，必要时使用`.Value`来获取实际的值。

## 完整代码示例

完整的序列化和反序列化代码如下：

```cs
internal class Program
{
    static void Main(string[] args)
    {
        byte[] bytes = Serialize();
        Console.WriteLine($"Serialize: \n{Convert.ToBase64String(bytes)}");

        Console.WriteLine("\nDeserialize: ");
        Deserialize(bytes);
    }

    static byte[] Serialize()
    {
        // Construct a Builder with 1024 byte backing array
        FlatBufferBuilder builder = new FlatBufferBuilder(1024);

        // Serialize name - string
        StringOffset nameOffset = builder.CreateString("Orc");

        // Serialize inventory - [ubyte]
        Monster.StartInventoryVector(builder, 10);
        for (int i = 9; i >= 0; i--)
        {
            builder.AddByte((byte)i);
        }
        VectorOffset inventoryOffset = builder.EndVector();

        ////VectorOffset inventoryOffset = Monster.CreateInventoryVector(builder, new byte[] { 0, 1, 2, 3, 4, 5, 6, 7, 8, 9 });

        // Serialize weapons - [Weapon]
        StringOffset weapon1NameOffset = builder.CreateString("Sword");
        Offset<Weapon> weapon1Offset = Weapon.CreateWeapon(builder, weapon1NameOffset, 3);
        StringOffset weapon2NameOffset = builder.CreateString("Axe");
        Offset<Weapon> weapon2Offset = Weapon.CreateWeapon(builder, weapon2NameOffset, 5);
        VectorOffset weaponsOffset = Monster.CreateWeaponsVector(builder, new Offset<Weapon>[] { weapon1Offset, weapon2Offset });

        ////StringOffset weapon1NameOffset = builder.CreateString("Sword");
        ////Offset<Weapon> weapon1Offset = Weapon.CreateWeapon(builder, weapon1NameOffset, 3);
        ////StringOffset weapon2NameOffset = builder.CreateString("Axe");
        ////Offset<Weapon> weapon2Offset = Weapon.CreateWeapon(builder, weapon2NameOffset, 5);
        ////Monster.StartWeaponsVector(builder, 2);
        ////builder.AddOffset(weapon2Offset.Value);
        ////builder.AddOffset(weapon1Offset.Value);
        ////VectorOffset weaponsOffset = builder.EndVector();

        // Serialze path - [Vec3]
        Monster.StartPathVector(builder, 2);
        Vec3.CreateVec3(builder, 1, 2, 3);
        Vec3.CreateVec3(builder, 4, 5, 6);
        VectorOffset pathOffset = builder.EndVector();

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

        // Finish the builder
        builder.Finish(monsterOffset.Value);

        // Get the bytes
        byte[] bytes = builder.SizedByteArray();

        return bytes;
    }

    static void Deserialize(byte[] bytes)
    {
        // Get an view to the root object inside the buffer
        Monster monster = Monster.GetRootAsMonster(new ByteBuffer(bytes));

        // Deserialize pos - Vec3
        Vec3 pos = monster.Pos.Value;
        float x = pos.X;
        float y = pos.Y;
        float z = pos.Z;
        Console.WriteLine($"pos=[{x}, {y}, {z}]");

        // Deserialize mana - short
        short mana = monster.Mana;
        Console.WriteLine($"mana={mana}");

        // Deserialize hp - short
        short hp = monster.Hp;
        Console.WriteLine($"hp={hp}");

        // Deserialize name - string
        string name = monster.Name;
        Console.WriteLine($"name={name}");

        // Deserialize inventory - [ubyte]
        byte[] inventory = new byte[monster.InventoryLength];
        for (int i = 0; i < inventory.Length; i++)
        {
            inventory[i] = monster.Inventory(i);
        }
        Console.WriteLine($"inventory=[{string.Join(", ", inventory)}]");

        // Deserialize color - Color
        Color color = monster.Color;
        Console.WriteLine($"color={color}");

        // Deserialize weapons - [Weapon]
        Weapon[] weapons = new Weapon[monster.WeaponsLength];
        for (int i = 0; i < weapons.Length; i++)
        {
            weapons[i] = monster.Weapons(i).Value;
        }
        Console.WriteLine($"weapons=[{string.Join(", ", weapons.Select(w => "(" + w.Name + ", " + w.Damage + ")"))}]");

        // Deserialize equipped - Equipment
        Equipment equippedType = monster.EquippedType;
        if (equippedType == Equipment.Weapon)
        {
            Weapon equipped = monster.Equipped<Weapon>().Value;
            ////Weapon equipped = monster.EquippedAsWeapon();
            Console.WriteLine($"equipped=({equipped.Name}, {equipped.Damage})");
        }

        // Deserialize path - [Vec3]
        Vec3[] path = new Vec3[monster.PathLength];
        for (int i = 0; i < path.Length; i++)
        {
            path[i] = monster.Path(i).Value;
        }
        Console.WriteLine($"path=[{string.Join(", ", path.Select(p => "(" + p.X + ", " + p.Y + ", " + p.Z + ")"))}]");
    }
}
```

运行结果如下：
```
Serialize:
IAAAAAAAGgAwACQAAAAiABwAAAAYABYAEAAPAAgABAAaAAAALAAAAFAAAAAAAAABPAAAAAAAAAB0AAAAgAAAAAAALAEAAIA/AAAAQAAAQEACAAAAAACAQAAAoEAAAMBAAACAPwAAAEAAAEBAAgAAACQAAAAEAAAA7P///wAABQAEAAAAAwAAAEF4ZQAIAAwACAAGAAgAAAAAAAMABAAAAAUAAABTd29yZAAAAAoAAAAAAQIDBAUGBwgJAAADAAAAT3JjAA==

Deserialize:
pos=[1, 2, 3]
mana=150
hp=300
name=Orc
inventory=[0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
color=Red
weapons=[(Sword, 3), (Axe, 5)]
equipped=(Axe, 5)
path=[(4, 5, 6), (1, 2, 3)]
```

## 注意点

### 关于默认值

下面讨论关于默认值的规则，以字段`mana:short = 150`为例。

1. 如果在序列化时没有设置值，在反序列化时读取，将会得到默认值。
    - 序列化时，没有调用`Monster.AddMana()`来设置值。
    - 反序列化时，调用`short mana = monster.Mana`，将会得到默认值`150`。
    - 源代码如下：
        ```cs
        public short Mana { get { int o = __p.__offset(6); return o != 0 ? __p.bb.GetShort(o + __p.bb_pos) : (short)150; } }
        ```

2. 如果在序列化时设置的值等于默认值，这个值其实并不会写入buffer中，跟没设置值是一个效果。
    - 序列化时，调用`Monster.AddMana(builder, 150)`，设置的值等于默认值。
    - 修改schema中的默认值，即`mana:short = 200`，重新编译。
    - 反序列化时，调用`short mana = monster.Mana`，将会得到新的默认值`200`。
    - 源代码如下：
        ```cs
        public void AddShort(int o, short x, int d) { if (ForceDefaults || x != d) { AddShort(x); Slot(o); } }
        ```

3. 设置`builder.ForceDefaults = true`，如果在序列化时设置的值等于默认值，可以让这个值也写入buffer中。

4. 为了保证兼容性，应该避免修改默认值。

### 关于字符串的存储方式

在Flatbuffers中，字符串是以UTF-8格式存储的。源代码如下：

```cs
public StringOffset CreateString(string s)
{
    if (s == null)
    {
        return new StringOffset(0);
    }
    NotNested();
    AddByte(0);
    var utf8StringLen = Encoding.UTF8.GetByteCount(s);
    StartVector(1, utf8StringLen, 1);
    _bb.PutStringUTF8(_space -= utf8StringLen, s);
    return new StringOffset(EndVector().Value);
}
```

### 关于required属性

被标记为`required`的字段，在序列化的时候必须设置值，否则运行时会报错。

以`equipped:Equipment (required)`为例，如果在序列化的时候没有调用`Monster.AddEquipped()`，运行时会报错：
```
Unhandled exception. System.InvalidOperationException: FlatBuffers: field 22 must be set
   at Google.FlatBuffers.FlatBufferBuilder.Required(Int32 table, Int32 field)
   at MyGame.Sample.Monster.EndMonster(FlatBufferBuilder builder) in XXX\FlatBuffersTest\MyGame\Sample\Monster.cs:line 74
   at FlatBuffersTest.Program.Serialize() in XXX\FlatBuffersTest\Program.cs:line 91
   at FlatBuffersTest.Program.Main(String[] args) in XXX\FlatBuffersTest\Program.cs:line 10
```

需要注意，错误信息中只会告诉我们是`field 22`没有设置，而不会告诉我们具体的字段名。那我们怎么知道`field 22`对应的是那个字段呢？

首先，看schema是行不通的，第22行根本就不是`required`字段。

其实呢，可以查看编译生成的C#代码`Monster.cs`，在里面搜索`22`，会看到如下代码：
```cs
public static Offset<MyGame.Sample.Monster> EndMonster(FlatBufferBuilder builder) {
    int o = builder.EndTable();
    builder.Required(o, 22);  // equipped
    return new Offset<MyGame.Sample.Monster>(o);
  }
```

通过代码的注释，就可以知道`22`对应的字段是`equipped`了。

### 关于id属性

对于`table`的字段，可以添加`id`属性，从而指定该字段在buffer中的offset。

如果不指定`id`属性，会自动根据字段定义的顺序，从0开始编号。这种情况下，不能改变字段的顺序，也不能在`table`中间添加新字段，否则会导致错误。

举个例子：
1. 序列化时，根据原来的`Monster`的schema，得到一个byte array。
2. 修改`Monster`的schema，把`hp`字段移动到最后，即：
    ```cs
    table Monster {
        pos:Vec3;
        mana:short = 150;
        name:string;
        friendly:bool = false (deprecated);
        inventory:[ubyte];
        color:Color = Blue;
        weapons:[Weapon];
        equipped:Equipment (required);
        path:[Vec3];
        hp:short = 100;
    }
    ```
3. 重新编译生成C#代码。
4. 把步骤1中得到的byte array进行反序列化，运行时会报错：
    ```
    Deserialize:
    pos=[1, 2, 3]
    mana=150
    hp=44
    Unhandled exception. System.ArgumentOutOfRangeException: Specified argument was out of the range of valid values.
        at Google.FlatBuffers.ByteBuffer.AssertOffsetAndLength(Int32 offset, Int32 length)
        at Google.FlatBuffers.ByteBuffer.ReadLittleEndian(Int32 offset, Int32 count)
        at Google.FlatBuffers.ByteBuffer.GetInt(Int32 index)
        at Google.FlatBuffers.Table.__string(Int32 offset)
        at MyGame.Sample.Monster.get_Name() in XXX\FlatBuffersTest\MyGame\Sample\Monster.cs:line 25
        at FlatBuffersTest.Program.Deserialize(Byte[] bytes) in XXX\FlatBuffersTest\Program.cs:line 125
        at FlatBuffersTest.Program.Main(String[] args) in XXX\FlatBuffersTest\Program.cs:line 20
    ```

为了避免这种情况，最好给`table`的字段添加`id`属性，如下所示：
```
table Monster {
  pos:Vec3 (id:0);
  mana:short = 150 (id:1);
  hp:short = 100 (id:2);
  name:string (id:3);
  friendly:bool = false (deprecated, id:4);
  inventory:[ubyte] (id:5);
  color:Color = Blue (id:6);
  weapons:[Weapon] (id:7);
  equipped:Equipment (required, id:9);
  path:[Vec3] (id:10);
}
```

需要注意：
- `id`属性要么都不加，要么都加，不能只给部分字段加，否则编译会报错：
    ```
    error: either all fields or no fields must have an 'id' attribute
    ```
- `id`属性需要从`0`开始递增，不能重复，否则编译会报错：
    ```
    error: field id's must be consecutive from 0, id 2 missing or set twice, field: friendly, id: 1
    ```
- `union`字段需要在上一个`id`的基础上`+2`，其他字段都只要`+1`，否则编译会报错：
    ```
    error: field id's must be consecutive from 0, id 8 missing or set twice, field: equipped_type, id: 7
    ```

添加`id`属性后，可以改变字段的顺序（保持`id`不变），也可以在`table`中间添加新的字段（`id`递增），都不会影响字段在buffer中的offset，因此不会影响序列化和反序列化的结果。例如：
```
table Monster {
  pos:Vec3 (id:0);
  mana:short = 150 (id:1);
  test:short = 50 (id:11);
  name:string (id:3);
  friendly:bool = false (deprecated, id:4);
  inventory:[ubyte] (id:5);
  color:Color = Blue (id:6);
  weapons:[Weapon] (id:7);
  equipped:Equipment (required, id:9);
  path:[Vec3] (id:10);
  hp:short = 100 (id:2);
}
```

### 关于deprecated属性

被标记为`deprecated`的字段，编译生成的C#代码中，没有该字段的函数，因此无法序列化、反序列化该字段。

把`required`字段标记为`deprecated`时，一般不会有问题，除非使用了可选验证器。

以`Monster`为例，旧的schema为`equipped:Equipment (required)`，新的schema为`equipped:Equipment (required, deprecated)`。考虑以下两种情况：

1. 序列化时用旧的schema，反序列化时用新的schema，不会有问题。
    - 序列化时，必须设置`equipped`字段的值。
    - 反序列化时，没有关于`equipped`字段的函数，因此无法读取值。
    - 反序列化时，如果使用了可选验证器`Monster.VerifyMonster()`，验证会成功。

2. 序列化时用新的schema，反序列化时用旧的schema，不会有问题。
    - 序列化时，没有关于`equipped`字段的函数，因此无法设置值。
    - 反序列化时，如果读取`equippedType`，值为`Equipment.NONE`。
    - 反序列化时，如果使用了可选验证器`Monster.VerifyMonster()`，验证会失败。

## 参考
[FlatBuffers Docs](https://flatbuffers.dev/tutorial/)