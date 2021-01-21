---
layout: post
title: "浅尝第一杯咖啡"
description: "初步探索 Class 文件结构"
cover_url: https://i.loli.net/2020/05/17/xGeAYcSCNK1VwWt.png
cover_meta: illustration by [ArseniXC](https://www.pixiv.net/artworks/53117853)
tags: 
  - Develop
  - Java
---

下方为 Java 虚拟机规范（以下简称规范）中对 Class 文件的描述，Class 文件由多个字段构成。

```
ClassFile {
    u4             magic;
    u2             minor_version;
    u2             major_version;
    u2             constant_pool_count;
    cp_info        constant_pool[constant_pool_count-1];
    u2             access_flags;
    u2             this_class;
    u2             super_class;
    u2             interfaces_count;
    u2             interfaces[interfaces_count];
    u2             fields_count;
    field_info     fields[fields_count];
    u2             methods_count;
    method_info    methods[methods_count];
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

> 其中，`u4` 表示该字段为长度 4 个字节的无符号整数，`u2` 表示该字段为长度 2 个字节的无符号整数。
> 
> Class 文件中的数据为大端表示。

## 字段解析

> Class 文件的魔数由 Java 之父詹姆斯 · 高斯林所设，另一个类似的魔数是对象持久化文件的 `0xCAFEDEAD`，其原因是他常与小伙伴去一个叫圣米歇尔港的地方吃午餐，并将那里称为死亡咖啡（ Cafe Dead ）。
> 
> "鬼知道我都经历了些什么。" —— 詹姆斯 · 高斯林
{:.side-note}

### Magic

Class 文件魔数，取值为 `0xCAFEBABE`。

> 在特定文件格式中，其头部的几个字节被无意或有意地设定为固定的值，因此可以通过该值判断文件格式，这几个字节的内容就被称为魔数。

### Minor Version / Major Version

Class 文件版本号，不同版本的 Java 所编译的 Class 文件版本号不相同，可用于 Jvm 对版本兼容的检查。

> Major Version 不小于 56 时（即 Java 12+），Minor Version 的值被限制在 0 到 65535 之间，而在低版本中 Minor Version 有可能是任何值。

### Constant Pool Count / Constant Pool

常量池（Constant Pool）是 Class 文件中存储常量项（其类型为 `cp_info`）的数组，每一个 Class 文件均包含一个常量池。

Constant Pool Count 其值为常量池中元素的个数**加 1**，需要注意的是**这一数组的索引从 1 开始**。

### Access Flag

标明类的访问权限。

### This Class

该类在常量池中的索引。

### Super Class

该类的超类在常量池中的索引。

### Interfaces Count / Interfaces

Interfaces Count 为该类实现接口的数量，在其后的 Interfaces 数组中包括了该类实现接口的信息，与 This Class 和 Super Class 相同，Interfaces 数组中的信息均为常量池中的索引。

### Fields Count / Fields

Fields Count 为该类成员变量的数量，在其后的 Fields 数组中包括了该类成员变量的信息，同样均为索引。

### Methods Count / Methods

Methods Count 为该类成员方法的数量，在其后的 Methods 数组中包括了该类成员方法的信息，索引。

### Attributes Count / Attributes

Attributes 中定义了该类包含的属性信息，如方法的执行指令、调试信息等，Attributes Count 为属性信息的个数。

## 常量池

在前文提到过，每一个 Class 文件中都存在一个常量池，类型为存储常量项（`cp_info`）的数组，用于存储 Class 文件中常量的相关信息。

> 这里所称的常量与代码层面上的常量含义上稍有不同，常量池中的常量不仅包括代码中的数字、字符或字符串等常量数据，还包括类信息、方法信息和成员变量信息等。

规范中，每一个常量项的格式如下：

```
cp_info {
    u1 tag;
    u1 info[];
}
```

> 这里 `info[]` 数组前的 `u1` 不代表其仅占用 1 个字节，而代表其内容长度根据情况而定，各数据项格式中的其它数组同理。

根据 `tag` 的不同，常量项表示的内容也不同，而 `tag` 的类型如下：

| tag                         | 取值 | 含义                                     |
|:---------------------------:|:------:|:--------------------------------------:|
| CONSTANT_Utf8               | 1      | 存储字符串内容                                |
| CONSTANT_Integer            | 3      | 存储 int 型数据，长度 4 Byte                   |
| CONSTANT_Float              | 4      | 存储 float 型数据，长度 4 Byte                 |
| CONSTANT_Long               | 5      | 存储 long 型数据，长度 8 Byte                  |
| CONSTANT_Double             | 6      | 存储 double 型数据，长度 8 Byte                |
| CONSTANT_Class              | 7      | 表示类或接口的信息                              |
| CONSTANT_String             | 8      | 表示字符串，**本身不存储内容，仅包含指向 Utf8 类型的索引**     |
| CONSTANT_Fieldref           | 9      | 表示成员变量的信息                              |
| CONSTANT_Method             | 10     | 表示成员方法的信息                              |
| CONSTANT_InterfaceMethodref | 11     | 表示接口函数的信息                              |
| CONSTANT_NameAndType        | 12     | 用于描述成员变量或成员方法的相关信息                     |
| CONSTANT_MethodHandle       | 15     | 反射相关，表示 Method Handler 相关信息，Java 7 中引入 |
| CONSTANT_MethodType         | 16     | 反射相关，用于描述成员方法的相关信息，Java 7 中引入          |
| CONSTANT_Dynamic            | 17     | 动态语言功能，表示动态计算中的常量，Java 11 中引入          |
| CONSTANT_InvokeDynamic      | 18     | 动态语言功能，表示动态计算中的命令，Java 7 中引入           |
| CONSTANT_Module             | 19     | 模块化相关，表示模块信息，Java 9 引入                 |
| CONSTANT_Package            | 20     | 模块化相关，表示包信息，Java 9 引入                  |

*本文暂且不讨论最后六项的内容。*

接下来看看规范中各种常见常量项的定义：

```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;  // 字符串长度
    u1 bytes[length];  // 字符串内容
}

CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;  // 数据内容
}

CONSTANT_Float_info {
    u1 tag;
    u4 bytes;  // 数据内容
}

CONSTANT_Long_info {
    u1 tag;
    u4 high_bytes;  // 数据高四位内容
    u4 low_bytes;  // 数据低四位内容
}

CONSTANT_Double_info {
    u1 tag;
    u4 high_bytes;  // 数据高四位内容
    u4 low_bytes;  // 数据低四位内容
}

CONSTANT_Class_info {
    u1 tag;
    u2 name_index;  // 类名，Utf8_info 索引
}

CONSTANT_String_info {
    u1 tag;
    u2 string_index;  // 字符串内容，Utf8_info 索引
}

CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;  // 所属类，Class_info 索引
    u2 name_and_type_index;  // 相关信息，NameAndType_info 索引
}

CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;  // 所属类，Class_info 索引
    u2 name_and_type_index;  // 相关信息，NameAndType_info 索引
}

CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;  // 所属接口，Class_info 索引
    u2 name_and_type_index;  // 相关信息，NameAndType_info 索引
}

CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;  // 方法或成员变量名称，Utf8_info 索引
    u2 descriptor_index;  // 相关类型索引，Utf8_info 索引
}
```

需要特别注意的是前文强调过的两点：

- 常量池数组的索引**从 1 开始**。

- Constant Pool Count 的数值为常量池内元素个数**加 1**。

> **为什么常量池的索引从 1 开始？**
> 
> 常量池中的索引 0 可作为常量池中的"空指针”使用，具有多种用途，较为常见的场景如表示"catch all”的异常捕获，或是未命名的参数，匿名类的内部名称等。
> 
> 或许这也可以从一种说法上解释 Constant Pool Count 的值要比常量池内元素个数多 1 了。

### UTF-8 常量项

这一常量项用于保存字符串数据，除了 `tag` 之外，常量项内还有用于记录字符串长度（单位为字）的 `length` 字段与记录字符串实际内容的 `bytes` 数组。

> 在 Class 文件中，不定长的内容均会有一个字段用于确定内容长度，不然 Jvm 怎么会知道自己应该读多少数据呢？

在其它常量项中所需要表示字符串的部分均通过索引指向 UTF-8 常量项，原因是**节省 Class 文件的空间占用**，举个简单的例子：

``` java
public class Sample {
    private String a = "Hello World!";
    public String b = "Hello World!";
}
```

在 Class 文件中，字符串内容 "Hello World!” 与表示类型的 "Ljava/lang/String;” 均仅有一份数据，避免出现冗余信息。

用 json 来表示一个 UTF-8 常量项如下：

``` json
{
    "index": "#?",
    "type": "utf8",
    "value": ".."
}
```

> 由于 Class 文件中的数据过于套娃，利用 json 有利于理清各项数据的层次关系（？），下文中也会对 Class 文件中的各项数据用 json 表示。
> 
> 所有的索引均用 `"#?"` 格式的字符串表示，如 `"#1"`，`"#2"`。
> 
> 对元素的个数、长度等帮助 Jvm 解析字节码的数据不放在 json 中。
> 
> 由于 UTF-8 常量项类型的特殊性，所有指向 UTF-8 常量项的 Key 值均为大写。

### Integer、Float、Long、Double 常量项

``` json
{
    "index": "#?",
    "type": "integer|float|long|double",
    "value": ..
}
```

存储方式为 big-endian（高字节在前）顺序存储。

> 回想起来，让 8 个字节的常量采用两个常量池条目是一个糟糕的选择。—— Jvm 规范
{:.side-note}

需要注意的是，对于占用 8 个字节的 Long、Double 常量项来说，**它们将占用两个索引的位置，但仅第一个索引是可用的**，比如，一个 Long 常量项的索引为 `"#3"`，则在常量池中下一个常量项的索引为 `"#5"`，而非 `"#4"`。

### Class 常量项

``` json
{
    "index": "#?",
    "type": "class",
    "NAME": "#?"  // 对应字段 name_index
}
```

Class 常量项里的 `NAME`（当然还是 UTF-8 索引）是该类的全限定名。

> 类的全限定名即类名的全称，包括包名。在 Java 源代码中的全限定名与 Class 文件中的全限定名并不相同，两者转换需要将 '.' 与 '/' 相互更替，如 Java 源代码中的 `java.lang.Object` 对应 Class 文件中的 `java/lang/Object`。

### String 常量项

``` json
{
    "index": "#?",
    "type": "string",
    "VALUE": "#?"  // 对应字段 string_index
}
```

String 常量项对应 Java 源代码中的字符串常量，其自身不包含字符串的内容。

> UTF-8 常量与源代码中的字符串常量不作为同一概念，前者表示的数据远比后者更多，而 String 常量项暂可片面地将其与源代码中 String 常量的关系与如 Integer、Float 常量项与源代码中数字常量的关系等同起来。

### Name And Type 常量项

在讲 Field、Method、Interface Method 常量项之前有必要先说明这一常量项：

``` json
{
    "index": "#?",
    "type": "name_and_type",
    "NAME": "#?",  // 对应字段 name_index
    "TYPE": "#?"  // 对应字段 descriptor_index
}
```

`NAME` 是方法或成员变量的名称，构造器（实例初始化方法）的 `NAME` 为 "<init>"。

> 所有被 Method 常量项引用的名称中，如果以 '<'（'\u003c'）开头，则该名称必须为特殊名称 "<init>"，且返回值必须为 void。

`TYPE` 是方法或成员变量的相关属性，数据类型的描述规则如下：

- 原始数据类型的描述为 "B"（byte）、"C"（char）、"D"（double）、"F"（float）、"I"（int）、"J"（long）、"S"（short）、"Z"（boolean）。

- 引用数据类型为 "L" + 全限定名 + ";"，如 "Ljava/lang/String;”。

- 数组为上述类型前加若干 '[' 字符，数量与数组维度相同，如二维 int 数组（int[][]）表示为 "[[I”，一维 String 数组（String[]）表示为 "[Ljava/lang/String;”。

若该常量项用于表示成员变量的属性，则其记录的是成员变量的类型；若该常量项用于表示成员方法的属性，则记录的是 "(" + 传入参数类型 + ")"  + 返回类型，传入参数类型按顺序排列，中间无间隔，如方法 "double add(double a, double b)" 表示为 "(DD)D"。

> 引用数据类型末尾的 ';' 已经很好地起到了分隔的作用。

### Field 常量项

``` json
{
    "index": "#?",
    "type": "field",
    "class": "#?",  // 对应字段 class_index
    "name_and_type": "#?"  // 对应字段 name_and_type_index
}
```

### Method、Interface Method 常量项

``` json
{
    "index": "#?",
    "type": "method|interface_method",
    "class": "#?",  // 对应字段 class_index
    "name_and_type": "#?"  // 对应字段 name_and_type_index
}
```

当该常量项为 Method 时，所指向的 `class` 应为类，而当常量项为 Interface Method 时，所指向的 `class` 应为接口。

## Access Flag

在 Java 中，类、方法、成员变量都有对访问权限的设置，而 Access Flag 用于表示访问权限，取值情况如下，多个访问标志可通过与运算合并：

| 标志名            | 取值情况   | 说明                  |
|:--------------:|:------:|:-------------------:|
| ACC_PUBLIC     | 0x0001 | public              |
| ACC_FINAL      | 0x0010 | final               |
| ACC_SUPER      | 0x0020 | 用于 invokespecial 指令 |
| ACC_INTERFACE  | 0x0200 | 该类为接口               |
| ACC_ABSTRACT   | 0x0400 | 该类为抽象类              |
| ACC_SYNTHETIC  | 0x1000 | 该类由编译器根据情况生成，对源码不可见 |
| ACC_ANNOTATION | 0x2000 | 该类为注解               |
| ACC_ENUM       | 0x4000 | 该类为枚举               |

## This Class / Super Class / Interfaces

This Class 与 Super Class 均为常量池中 Class 常量项的索引，指示该类与超类.

一个类可以实现多个接口，因此接口信息采用数组表示，数组内元素同样为 Class 常量项的索引.

## Fields

Fields 中包含了所有成员变量，根据规范，每一个成员变量的格式如下：

```
field_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

可用 json 表示：

``` json
{
    "access_flag": "public|private|..",  // 对应字段 access_flags
    "NAME": "#?",  // 对应字段 name_index
    "TYPE": "#?", // 对应字段 descriptor_index
    "attributes": [..]  // 对应数组 attributes
}
```

成员变量的 Access Flag 与类中的有些差异，下面是成员变量 Access Flag 取值情况：

| 标志名           | 取值情况   | 说明                    |
|:-------------:|:------:|:---------------------:|
| ACC_PUBLIC    | 0x0001 | public                |
| ACC_PRIVATE   | 0x0002 | private               |
| ACC_PROTECTED | 0x0004 | protected             |
| ACC_STATIC    | 0x0008 | static                |
| ACC_FINAL     | 0x0010 | final                 |
| ACC_VOLATILE  | 0x0040 | volatile              |
| ACC_TRANSIENT | 0x0080 | transient，说明该成员无法被串行化 |
| ACC_SYNTHETIC | 0x1000 | 该成员由编译器根据情况生成，对源码不可见  |
| ACC_ENUM      | 0x4000 | 枚举                    |

`NAME` 与 `TYPE` 和常量池中 Name And Type 常量项无异，均为索引。

`attributes` 是存储类、方法与成员变量中相关属性的数组，数组中的每个元素对应规范中 `attribute_info` 的结构，这一部分将在下文详解。

## Methods

Methods 中包含了所有成员方法，根据规范，每一个成员方法的格式如下：

```
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

可用 json 表示：

``` json
{
    "access_flag": "public|private|..", // 对应字段 access_flags
    "NAME": "#?", // 对应字段 name_index
    "TYPE": "#?", // 对应字段 descriptor_index
    "attributes": [..] // 对应数组 attributes
}
```

下表为成员方法 Access Flag 的取值情况：

| 标志名              | 取值情况   | 说明                   |
|:----------------:|:------:|:--------------------:|
| ACC_PUBLIC       | 0x0001 | public               |
| ACC_PRIVATE      | 0x0002 | private              |
| ACC_PROTECTED    | 0x0004 | protected            |
| ACC_STATIC       | 0x0008 | static               |
| ACC_FINAL        | 0x0010 | final                |
| ACC_SYNCHRONIZED | 0x0020 | synchronized         |
| ACC_BRIDGE       | 0x0040 | 桥接方法，有编译器根据情况生成      |
| ACC_VARARGS      | 0x0080 | 该方法为可变参数方法           |
| ACC_NATIVE       | 0x0100 | native               |
| ACC_ABSTRACT     | 0x0400 | 抽象方法                 |
| ACC_STRICT       | 0x0800 | 精确浮点类型               |
| ACC_SYNTHETIC    | 0x1000 | 该方法由编译器根据情况生成，对源码不可见 |

## 属性

在类、方法、成员变量中均存在一个存储 `attribute_info` 类型的数组，用于存储类、方法、成员变量的相关属性。

规范中，`attribute_info` 的格式如下：

```
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```

与常量池中通过 1 字节的无符号数 `tag` 来区别常量项不同，属性通过名称（索引所指 UTF-8 常量项内容）来区分，这一索引即为 `attribute_name_index`。

`attribute_length` 用于确定 `info` 的长度，`info` 为属性的详细内容。

下表列出了一些重要的属性及其作用：

| 名称                 | 作用                                   |
|:------------------:|:------------------------------------:|
| ConstantValue      | 用于描述常量成员的值，这里所指的常量为源代码层面的常量          |
| Code               | 包含方法编译得到的虚拟机指令，try/catch 语句对应的异常处理表等 |
| Exceptions         | 若一个方法抛出异常或错误，则其 method_info 保留该属性    |
| SourceFile         | 记录 Class 文件对应的 Java 源代码文件名称          |
| LocalVariableTable | 描述方法中局部变量的相关信息                       |

在这里重点分析 Code 属性，即 Java 源代码编译后实际操作内容的所在位置。

### Code

一个 Code 属性的 `attribute_info`，在规范中为以下格式：

```
Code_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 max_stack;
    u2 max_locals;
    u4 code_length;
    u1 code[code_length];
    u2 exception_table_length;
    {   u2 start_pc;
        u2 end_pc;
        u2 handler_pc;
        u2 catch_type;
    } exception_table[exception_table_length];
    u2 attributes_count;
    attribute_info attributes[attributes_count];
}
```

`attributes_name_index` 与 `attributes_length` 为 `attribute_info` 的公有格式，上文已经提及。

`max_stack` 表示 Code 属性所属方法在执行时需要多少操作数栈的栈深。

> Jvm 执行指令时，指令的操作数会存储在一个名为 "操作数栈（Operand Stack）" 的栈中，每一个操作数会根据类型占用 1 到 2 个栈顶的空间。

`max_locals` 表示需要的局部变量空间，每一种类型的局部变量占用的空间大小不同。

`code_length` 记录了 `code` 数组的长度，`code` 数组存储虚拟机指令。

`exception_table_length` 记录了 `exception_table` 数组的长度，`exception_table` 数组中的每一个元素均占 8 个字节，每 2 个字节分别代表：try 语句开始指令位置、try 语句结束指令位置、开始捕获异常的指令位置、捕获异常的名称（UTF-8 常量）。

> pc（Program Count）为 Jvm 中维护的用于代表当前执行指令的变量，在此可视为指令行数，如同源代码中的行号。

`attributes_count` 记录了 `attributes` 的长度，`attributes` 用于表示属性的额外信息。

> ~~禁止套娃。~~

用 json 来表示如下：

``` json
{
    "TYPE": "#?",  // 这里所指向的 UTF-8 常量项内容应为 "Code"
    "stack_deep": ..,  // 对应字段 max_stack
    "local_variables_space": ..,  // 对应字段 max_locals
    "code": [..],  // 对应数组 code
    "exceptions": [..],  // 对应数组 exception_table
    "extra_attributes": [..]  // 对应数组 attributes
}
```

`code` 数组中存储着多条指令，每一条指令均由 1 个字节的指令码与 n 个字节（n 不小于 0）的操作数构成。

> 虽然每条指令是不定长的，但是每一指令的参数个数是确定的，因此虚拟机可以准确读取。

`extra_attributes` 中由两个属性较为常见：

#### LineNumberTable

该额外属性用于 Java 中的调试，它记录着中虚拟机指令位置与源代码行数的对应关系，规范中格式定义如下：

```
LineNumberTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 line_number_table_length;
    {   u2 start_pc;
        u2 line_number;    
    } line_number_table[line_number_table_length];
}
```

额外属性也属于 `attribute_info`，`attributes_name_index` 与 `attribute_length` 为公有格式。

`line_number_table_length` 记录了 `line_number_table` 数组的长度，`line_number_table` 数组中的每一个元素均占 4 个字节，每 2 个字节分别代表：虚拟机指令的位置、对应源代码的行数。

> 虽然源代码中的空行、注释不影响生成代码的执行，但是生成的 Class 文件在内容上会有不同，因此其哈希值也会不同。

#### LocalVariableTable

该额外属性记录了代码中方法内局部变量的相关信息，规范中格式定义如下：

```
LocalVariableTable_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 local_variable_table_length;
    {   u2 start_pc;
        u2 length;
        u2 name_index;
        u2 descriptor_index;
        u2 index;
    } local_variable_table[local_variable_table_length];
}
```

`attributes_name_index` 与 `attribute_length` 为公有格式。

`local_variable_table_length` 记录了 `local_variable_table` 数组的长度，`local_variable_table` 数组中的每一个元素均占 10 个字节，每 2 个字节分别代表：局部变量作用域起始位置、局部变量作用域范围大小（指令数）、局部变量名称（UTF-8 常量项索引）、局部变量类型（UTF-8 常量项索引）、局部变量数组的索引。

> 占用 2 个字节的局部变量（long 与 double），其下一个局部变量的索引 +2。

## Class 文件解析

通过一段简单且经典的 Java 代码进一步理解 Class 文件的结构。

``` java
public class Test {
    public static void main(String[] args) {
        System.out.println("Hello World!");
    }
}
```

> 受限于篇幅（~~懒，制作图表很累~~），这一段代码中没有成员变量，没有异常，其 Class 文件中的内容仅仅覆盖上文所讲的一小部分。

生成 Class 文件后采用十六进制读写工具打开。

> 本文采用 vim 读写 Class 文件，在通过命令 "vim -b <filename>" 打开后，可采用 vim 命令 ":%!xxd" 转换为 16 进制阅读。

![vim -b Test.class]({{ site.assets }}/assets/posts/first-cup-of-coffee/1.png)

对十六进制数据进行整理可以得出下表：

![class file table]({{ site.assets }}/assets/posts/first-cup-of-coffee/2.png)

我们可以看到前 8 个字节所包含的信息：

- Class 文件魔数 0xCAFEBABE。

- 小版本号为 0。

- 大版本号为 57（Java 13）。

接下来的 2 个字节存储了 constant pool count，可以看到是 29，即常量池中包含了 28 个元素，来看一看常量池中所存储的常量项内容：

![class file constant pool]({{ site.assets }}/assets/posts/first-cup-of-coffee/3.png)

> 为了便于识别，图中每个常量项中的 `tag` 下方所标的指示格式为 "索引 - 常量 tag"，其中，U - UTF-8、C - Class、M - Method、F - Field、N&T - Name And Type。每一个 UTF-8 常量项背景为蓝色，其它常量项背景为灰色。

通过 json 来表示这一常量池：

``` json
{
    ..
    "constants": [
        {
            "index": "#1",
            "type": "method",
            "class": "#2",
            "name_and_type": "#3"
        },
        {
            "index": "#2",
            "type": "class",
            "NAME": "#4"
        },
        {
            "index": "#3",
            "type": "name_and_type",
            "NAME": "#5",
            "TYPE": "#6"
        },
        {
            "index": "#4",
            "type": "utf8",
            "value": "java/lang/Object"
        },
        {
            "index": "#5",
            "type": "utf8",
            "value": "<init>"
        },
        {
            "index": "#6",
            "type": "utf8",
            "value": "()V"
        },
        {
            "index": "#7",
            "type": "field",
            "class": "#8",
            "name_and_type": "#9"
        },
        {
            "index": "#8",
            "type": "class",
            "NAME": "#10"
        },
        {
            "index": "#9",
            "type": "name_and_type",
            "NAME": "#11",
            "TYPE": "#12"
        },
        {
            "index": "#10",
            "type": "utf8",
            "value": "java/lang/System"
        },
        {
            "index": "#11",
            "type": "utf8",
            "value": "out"
        },
        {
            "index": "#12",
            "type": "utf8",
            "value": "Ljava/io/PrintStream;"
        },
        {
            "index": "#13",
            "type": "string",
            "VALUE": "#14"
        },
        {
            "index": "#14",
            "type": "utf8",
            "value": "Hello World!"
        },
        {
            "index": "#15",
            "type": "method",
            "class": "#16",
            "name_and_type": "#17"
        },
        {
            "index": "#16",
            "type": "class",
            "NAME": "#18"
        },
        {
            "index": "#17",
            "type": "name_and_type",
            "NAME": "#19",
            "TYPE": "#20"
        },
        {
            "index": "#18",
            "type": "utf8",
            "value": "java/io/PrintStream"
        },
        {
            "index": "#19",
            "type": "utf8",
            "value": "println"
        },
        {
            "index": "#20",
            "type": "utf8",
            "value": "(Ljava/lang/String;)V"
        },
        {
            "index": "#21",
            "type": "class",
            "NAME": "#22"
        },
        {
            "index": "#22",
            "type": "utf8",
            "value": "Test"
        },
        {
            "index": "#23",
            "type": "utf8",
            "value": "Code"
        },
        {
            "index": "#24",
            "type": "utf8",
            "value": "LineNumberTable"
        },
        {
            "index": "#25",
            "type": "utf8",
            "value": "main"
        },
        {
            "index": "#26",
            "type": "utf8",
            "value": "([Ljava/lang/String;)V"
        },
        {
            "index": "#27",
            "type": "utf8",
            "value": "SourceFile"
        },
        {
            "index": "#28",
            "type": "utf8",
            "value": "Test.java"
        }
    ],
    ..
}
```

而常量池后的其它数据则是对 Test 的相关信息说明与用于执行的虚拟机指令：

![class file others]({{ site.assets }}/assets/posts/first-cup-of-coffee/4.png)

> 右方对应颜色的色块用于协助理解数据之间的层级关系，灰色部分为各个方法编译所得的虚拟机指令

接下来的 6 个字节指明了 Test 类的访问权限、类信息与超类信息：

- Test 的访问权限为 `public`。

- Test 对应的 Class 常量项位于索引 `#21`。

- Test 的超类对应的 Class 常量项位于索引 `#2`，即 `java.lang.Object`。

`interfaces count` 与 `fields count` 均为 0，表示 Test 类未实现任何接口且不包含任何方法；`methods` 中存储了两个方法的相关信息：构造器（访问权限为 `public`）、main 方法（访问权限为 `public`，且这是一个静态方法）。在 Class 文件的末尾部分为 `attributes` 数组，内部记录了 Class 文件所对应的源码文件名：Test.java。

至此我们解析完了整一 Class 文件的文件结构，通过 json 进行表示：

``` json
{
    "magic": 3405691582,
    "minor_version": 0,
    "major_version": 57,

    "constants": [
        {
            "index": "#1",
            "type": "method",
            "class": "#2",
            "name_and_type": "#3"
        },
        {
            "index": "#2",
            "type": "class",
            "NAME": "#4"
        },
        {
            "index": "#3",
            "type": "name_and_type",
            "NAME": "#5",
            "TYPE": "#6"
        },
        {
            "index": "#4",
            "type": "utf8",
            "value": "java/lang/Object"
        },
        {
            "index": "#5",
            "type": "utf8",
            "value": "<init>"
        },
        {
            "index": "#6",
            "type": "utf8",
            "value": "()V"
        },
        {
            "index": "#7",
            "type": "field",
            "class": "#8",
            "name_and_type": "#9"
        },
        {
            "index": "#8",
            "type": "class",
            "NAME": "#10"
        },
        {
            "index": "#9",
            "type": "name_and_type",
            "NAME": "#11",
            "TYPE": "#12"
        },
        {
            "index": "#10",
            "type": "utf8",
            "value": "java/lang/System"
        },
        {
            "index": "#11",
            "type": "utf8",
            "value": "out"
        },
        {
            "index": "#12",
            "type": "utf8",
            "value": "Ljava/io/PrintStream;"
        },
        {
            "index": "#13",
            "type": "string",
            "VALUE": "#14"
        },
        {
            "index": "#14",
            "type": "utf8",
            "value": "Hello World!"
        },
        {
            "index": "#15",
            "type": "method",
            "class": "#16",
            "name_and_type": "#17"
        },
        {
            "index": "#16",
            "type": "class",
            "NAME": "#18"
        },
        {
            "index": "#17",
            "type": "name_and_type",
            "NAME": "#19",
            "TYPE": "#20"
        },
        {
            "index": "#18",
            "type": "utf8",
            "value": "java/io/PrintStream"
        },
        {
            "index": "#19",
            "type": "utf8",
            "value": "println"
        },
        {
            "index": "#20",
            "type": "utf8",
            "value": "(Ljava/lang/String;)V"
        },
        {
            "index": "#21",
            "type": "class",
            "NAME": "#22"
        },
        {
            "index": "#22",
            "type": "utf8",
            "value": "Test"
        },
        {
            "index": "#23",
            "type": "utf8",
            "value": "Code"
        },
        {
            "index": "#24",
            "type": "utf8",
            "value": "LineNumberTable"
        },
        {
            "index": "#25",
            "type": "utf8",
            "value": "main"
        },
        {
            "index": "#26",
            "type": "utf8",
            "value": "([Ljava/lang/String;)V"
        },
        {
            "index": "#27",
            "type": "utf8",
            "value": "SourceFile"
        },
        {
            "index": "#28",
            "type": "utf8",
            "value": "Test.java"
        }
    ],

    "access_flag": "public",
    "this_class": "#21",
    "super_class": "#2",

    "interface": [],
    "fields": [],
    "methods": [
        {
            "access_flag": "public",
            "NAME": "#5",
            "TYPE": "#6",
            "attributes": [
                {
                    "TYPE": "#23",
                    "stack_deep": 1,
                    "local_variables_space": 1,
                    "code": [
                        {
                            "command": "aload_0",
                            "arguments": null
                        },
                        {
                            "command": "invoke_special",
                            "arguments": ["#1"]
                        },
                        {
                            "command": "return",
                            "arguments": null
                        }
                    ]
                }
            ],
            "exceptions": [],
            "extra_attributes": [
                {
                    "TYPE": "#24",
                    "line_table": [
                        {
                            "start_pc": 0,
                            "line_number": 1
                        }
                    ]
                }
            ]
        },
        {
            "access_flag": "public & static",
            "NAME": "#25",
            "TYPE": "#26",
            "attributes": [
                {
                    "TYPE": "#23",
                    "stack_deep": 2,
                    "local_variables_space": 1,
                    "code": [
                        {
                            "command": "get_static",
                            "arguments": ["#7"]
                        },
                        {
                            "command": "ldc",
                            "arguments": ["#13"]
                        },
                        {
                            "command": "invoke_virtual",
                            "arguments": ["#15"]
                        },
                        {
                            "command": "return",
                            "arguments": null
                        }
                    ]
                }
            ],
            "exceptions": [],
            "extra_attributes": [
                {
                    "TYPE": "#24",
                    "line_table": [
                        {
                            "start_pc": 0,
                            "line_number": 1
                        },
                        {
                            "start_pc": 8,
                            "line_number": 4
                        }
                    ]
                }
            ]
        }
    ],

    "attributes": [
        {
            "TYPE": "#27",
            "SOURCE_FILE_NAME": "#28"
        }
    ]
}
```

> 实际上，在 JDK 中自带有一个 Class 文件反编译工具 javap，命令格式如下：
> 
> ```bash
> javap <options> <classes>
> ```
> 
> 较为常用的参数如下：
> 
> - -verbose（-v），输出包括行号、局部变量信息、反编译代码、常量池信息等。
> 
> - -l，输出包括行号与局部变量信息。
> 
> - -c，反编译字节码生成虚拟机指令。
> 
> 在这一例子中，通过命令 "javap -verbose Test" 可以得到 Test 的 Class 文件信息：
> 
> ![javap -verbose Test]({{ site.assets }}/assets/posts/first-cup-of-coffee/5.png)

## 参考链接

- [Oracle - Java Virtual Machine Specification Chapter 4. The class File Format](https://docs.oracle.com/javase/specs/Jvms/se13/html/Jvms-4.html)

- [Stack Overflow - Why is the constant pool indexed from 1?](https://stackoverflow.com/questions/56808432/why-is-the-constant-pool-in-java-classfile-indexed-from-1-and-not-0-what-is)
