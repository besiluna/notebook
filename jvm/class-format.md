# class文件格式

> 每个class文件中都包含一个类或接口的定义信息，本节内容主要介绍class文件的结构及其组成。

class文件采用一种类似C语言结构体的伪结构来描述和存储数据，这种伪结构中的数据内容称为数据项。[《Java虚拟机规范》][jvm_spec_8]中定义了两种数据类型来表示这些数据项：

- 无符号数：*u1*、*u2*、*u4*，它们分别表示占用无符号的1个字节、2个字节、4个字节存储空间的数据项。无符号数是class文件中最基本的数据类型，可以表示数字、索引引用、`UTF-8`编码构成的字符串值。
- 表：包含0个或多个数据项，占用的字节大小并不确定，是由无符号数或其他表组成的复合结构。表的命名通常以 *_info* 结尾。

class文件由一组8个字节的字节流组成，其中包含若干个数据项（无符号数或表），每个数据项都按照严格的顺序和结构进行排列，中间没有任何分隔，超过8个字节的数据项按照高位在前的顺序分割为若干8个字节存储。

> **说明：** class文件并非专指以 *.class* 结尾的文件，它也可能由代码动态生成，并直接被类加载器加载到内存中。

## class文件结构

class文件的整体结构如下所示：

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

### 魔数

*magic* 称为魔数，占用4个字节，用来标识这是一个可以被Java虚拟机识别的class文件，默认值：`0xCAFEBABE`。

### 版本号

*minor_version* 和 *major_version* 分别表示class文件的次版本号和主版本号，分别占用两个字节。

如果一个class文件的 *minor_version* 值为 *m*，*major_version* 值为 *M*，则该class文件的版本号表示为 *M.m*。Java虚拟机实现必须向下兼容低版本的class文件，但是不能执行高于自身版本的class文件。

每个JDK版本都有其对应的class文件版本号，Java中class文件的版本号是从45.0开始的，JDK1.0.2中的Java虚拟机实现能够支持的class文件版本范围是：45.0 ~ 45.3，而到了JDK8，对应的class文件版本号为52.0。

### 常量池集合

*constant_pool_count* 占用2个字节，表示常量池集合中数据项数量，实际值为常量池数据项数量-1。如果*constant_pool_count* 值为20，则常量池中实际的数据项数量为19。

*constant_pool[]* 表示常量池集合，其中每一项都是一个常量池表 *cp_info* 结构的数据项，常量池表是class文件中第一个表类型的数据项，常量池集合的索引下标范围是从1到 *constant_pool_count* - 1，而class文件中的其他任何表结构都是从0开始索引的。

### 访问标志

*access_flags* 表示该类或接口的访问信息，其值为各标志值按位或的结果，占用2个字节。*access_flags* 总共有16个标志位可以使用，在JDK8中只用了8个，如下表所示：

| 标志名称       | 值     | 说明                                                         |
| -------------- | ------ | ------------------------------------------------------------ |
| ACC_PUBLIC     | 0x0001 | 表示`public`类型，可以从其程序包外部进行访问                 |
| ACC_FINAL      | 0x0010 | 表示`final`类型，没有子类                                    |
| ACC_SUPER      | 0x0020 | 表示是否允许使用 *invokespecial* 字节码指令，该指令的语义在JDK1.0.2发生过改变，JDK1.0.2之后编译出来的class文件中，这个标志都必须为真 |
| ACC_INTERFACE  | 0x0200 | 表示这是一个接口                                             |
| ACC_ABSTRACT   | 0x0400 | 表示`abstract`类型，抽象类和接口中，该标志都为真             |
| ACC_SYNTHETIC  | 0x1000 | 表示该类或接口由编译器生成                                   |
| ACC_ANNOTATION | 0x2000 | 表示这是一个注解                                             |
| ACC_ENUM       | 0x4000 | 表示这是一个枚举                                             |

**示例：** 定义一个如下的`ClassFormat`类，然后使用`javac --release 8 ClassFormat.java`命令进行编译

```java
public class ClassFormat {
}
```

再使用`javap -v ClassFormat.class`命令查看生成的class文件，可以发现其访问标志值为：0x0021=0x0001|0x0020，因此该类的访问标志为：ACC_PUBLIC 和 ACC_SUPER

```text
flags: (0x0021) ACC_PUBLIC, ACC_SUPER
```

### 类信息

*this_class* 表示当前类，占用2个字节，该数据项的值是一个索引，指向常量池表中一个 *CONSTANT_Class_info* 结构的数据项，表示该class文件所定义的类或接口。

*super_class* 表示父类，占用2个字节，该数据项的值是0或者一个有效的索引值，指向常量池中一个 *CONSTANT_Class_info* 结构的数据项，表示该class文件所定义的类的父类。如果 *super_class* 的值为0，则这个类肯定是`java.lang.Object`。

### 接口集合

*interfaces_count* 表示该class文件所定义的类或接口的父类接口数量，占用2个字节。如果该类没有实现任何接口，则该数据项的值为0，后面紧接着的接口索引集合不占用任何字节。

*interfaces[]* 表示接口索引集合，该数据项中的每一项都是一个占用2字节的有效索引值，指向常量池中一个 *CONSTANT_Class_info* 结构的数据项，表示该class文件所定义的类或接口的父类接口。

### 字段集合

*fields_count* 表示该class文件所定义的类或接口中声明的字段（成员属性）的数量，占用2个字节。

*fields[]* 表示字段表集合，其中每一个数据项都是一个字段表 *field_info* 结构的数据项。字段表集合只包含该class文件所定义类或接口中的字段，并不包含从父类或接口中继承的字段。

### 方法集合

*methods_count* 表示该class文件所定义的类或接口中声明的方法的数量，占用2个字节。

*methods[]* 表示方法表集合，其中每一项都是一个方法表 *method_info* 结构的数据项。

### 属性集合

*attributes_count* 表示该class文件所定义的类或接口中所有属性的数量，占用2个字节。

*attributes[]* 表示属性表集合，其中的每一项都是一个属性表 *attribute_info* 结构的数据项。

## 内部名称形式

类、接口、方法、字段、本地变量、形式参数的名称在class文件中都通过 *CONSTANT_Utf8_info* 结构表示。

### 二进制名称

类或接口的名称在class文件中表示为一种全限定形式，称为二进制名称。

由于历史原因，class文件中二进制名称的内部形式跟[Java语言规范][jls]中的表示有所区别，class文件中的二进制名称用`/`替换了`.`。例如：`java.lang.Thread`在class文件中的内部名称形式为`java/lang/Thread`。

### 非全限定名称

方法、字段、本地变量、形式参数的名称在class文件中存储为非全限定名。非全限定名通常只包含一个简单的名称，不能包含：`.`、`;`、`[`、`/`这些字符。例如：定义一个`method()`方法，则其在class文件中的内部名称形式就是`method`。

> **说明：** `<init>`和`<clinit>`是例外，这两个方法名是javac编译器在代码生成阶段生成的。

## 描述符

描述符用来表示字段或方法的类型，在class文件中是一个`UTF-8`编码的字符串。

### 字段描述符

字段描述符分为三种类型：

- 基本类型：用特定的单个 *ASCII* 字符表示

- 对象类型：用 *LclassName;* 这种形式表示，其中，*L* 和 *;* 必不可少，*className* 为类名或接口名的内部二进制名称形式

- 数组类型：用 *[* 表示数组，后面是具体数组元素类型

各类型字段描述符表示如下所示：

| 描述符表示  | 类型     | 说明                                                        |
| ----------- | -------- | ----------------------------------------------------------- |
| B           | byte     | 表示字节类型                                                |
| C           | char     | 表示字符类型                                                |
| D           | double   | 表示双精度类型                                              |
| F           | float    | 表示单精度类型                                              |
| I           | int      | 表示整型                                                    |
| J           | long     | 表示长整型                                                  |
| S           | short    | 表示短整型                                                  |
| Z           | boolean  | 表示布尔类型                                                |
| [           | 数组     | 表示一维数组，多维数组用多个 [ 表示，后面是具体数组元素类型 |
| LclassName; | 引用类型 | 表示引用类型，className 是类名的二进制名称形式              |

> **说明：** 基本类型`int`的字段描述符表示为：`I`；引用类型`Object`的字段描述符表示为：`Ljava/lang/Object;`；数组类型`Object[]`的字段描述符表示为：`[Ljava/lang/Object;`，数组类型最多支持255个维度。

### 方法描述符

方法描述符包含两种描述符：

- 参数描述符：参数描述符（0个或多个）用字段描述符表示

- 返回类型描述符：用字段描述符表示方法返回类型，其中 *V* 表示返回类型为`void`

**示例：** 如下方法

```java
Object m(int i, double d, Thread t) {
}
```

其方法描述符表示为：

```text
(IDLjava/lang/Thread;)Ljava/lang/Object;
```

## 常量池

常量池中的每一个数据项都是一个表类型的数据结构，每个数据结构的第一个字节都是 *tag* 标志位，用来标识这是哪一种类型的数据结构。

```
cp_info {
    u1 tag;
    u1 info[];
}
```

Java虚拟机指令实际上并不依赖运行时的类实例、接口或数组，而是依赖常量池中这些表类型的符号信息。

JDK8的常量池中包含如下不同结构的表类型数据项：

| 类型                             | tag标志值 | 说明                   |
| -------------------------------- | --------- | ---------------------- |
| CONSTANT_Class_info              | 7         | 类或接口的符号信息     |
| CONSTANT_Fieldref_info           | 9         | 字段符号信息           |
| CONSTANT_Methodref_info          | 10        | 类方法符号信息         |
| CONSTANT_InterfaceMethodref_info | 11        | 接口方法符号信息       |
| CONSTANT_String_info             | 8         | 字符串类型信息         |
| CONSTANT_Integer_info            | 3         | 整型字面量             |
| CONSTANT_Float_info              | 4         | 浮点型字面量           |
| CONSTANT_Long_info               | 5         | 长整型字面量           |
| CONSTANT_Double_info             | 6         | 双精度浮点型字面量     |
| CONSTANT_NameAndType_info        | 12        | 字段或方法部分符号信息 |
| CONSTANT_Utf8_info               | 1         | UTF-8编码的字符串      |
| CONSTANT_MethodHandle_info       | 15        | 方法句柄信息           |
| CONSTANT_MethodType_info         | 16        | 方法类型信息           |
| CONSTANT_InvokeDynamic_info      | 18        | 动态方法调用点         |

### 类或接口

*CONSTANT_Class_info* 结构表示一个类或接口符号信息

```
CONSTANT_Class_info {
    u1 tag;
    u2 name_index;
}
```

*name_index* 的值是一个索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示一个类或接口的全限定名。

### 字段、方法和接口方法

*CONSTANT_Fieldref_info* 结构表示一个字段符号信息

*CONSTANT_Methodref_info* 结构表示一个类方法符号信息

*CONSTANT_InterfaceMethodref_info* 结构表示一个接口方法符号信息

它们的结构相似：

```
CONSTANT_Fieldref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
 
CONSTANT_Methodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
 
CONSTANT_InterfaceMethodref_info {
    u1 tag;
    u2 class_index;
    u2 name_and_type_index;
}
```

*class_index* 的值是一个索引，指向常量池中一个 *CONSTANT_Class_info* 结构的数据项，表示一个包含该字段或方法的类或接口类型。

*name_and_type_index* 的值是一个索引，指向一个 *CONSTANT_NameAndType_info* 结构的数据项，表示该字段或方法的名称和描述符。

### 字符串

*CONSTANT_String_info* 结构表示一个`String`类型的常量

```
CONSTANT_String_info {
    u1 tag;
    u2 string_index;
}
```

*string_index* 的值是一个索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项。

### 数值常量

*CONSTANT_Integer_info* 和 *CONSTANT_Float_info* 结构都表示一个4字节的数值常量。

```
CONSTANT_Integer_info {
    u1 tag;
    u4 bytes;
}
 
CONSTANT_Float_info {
    u1 tag;
    u4 bytes;
}
```

*CONSTANT_Integer_info* 结构中的 *bytes* 值是一个`int`常量值，按照高位在前的顺序存储。

*CONSTANT_Float_info* 结构中的 *bytes* 的值是一个按照 *IEEE 754 floating-point single format* 格式表示的浮点型常量，也按照高位在前的顺序存储。

*CONSTANT_Long_info* 和 *CONSTANT_Double_info* 结构都表示一个8字节的数值常量。

```
CONSTANT_Long_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
 
CONSTANT_Double_info {
    u1 tag;
    u4 high_bytes;
    u4 low_bytes;
}
```

class文件的常量表中所有的8字节的常量都占用2个索引位置，例如：常量表中第 *i* 个常量为 *CONSTANT_Long_info* 或 *CONSTANT_Double_info* 结构，则该class文件的常量表中不存在值为 *i+1* 的索引，下一个数据项的索引值为 *i+2*。

*CONSTANT_Long_info* 或 *CONSTANT_Double_info* 结构所表示的值为：

```
((long) high_bytes << 32) + low_bytes
```

其中，*CONSTANT_Double_info* 结构的值按照 *IEEE 754 floating-point double format* 格式表示。

### 名称和类型

*CONSTANT_NameAndType_info* 结构表示一个字段或方法的名称和描述符。

```
CONSTANT_NameAndType_info {
    u1 tag;
    u2 name_index;
    u2 descriptor_index;
}
```

*name_index* 的值是一个索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示一个字段或方法的名词。

*descriptor_index* 的值也是一个索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示一个字段或方法的描述符。

### UTF8

*CONSTANT_Utf8_info* 结构用来表示一个`String`常量值。

```
CONSTANT_Utf8_info {
    u1 tag;
    u2 length;
    u1 bytes[length];
}
```

*length* 表示字符串的字节长度，占用2个字节。

*bytes[]* 包含了字符串的字节数组，数组长度为 *length*。

字符串的内容使用UTF-8缩略编码表示，UTF-8缩略编码可以使用1个字节表示`\u0001`到`\u007f`之间的字符，使用2个字节表示`\u0080`到`\u07ff`之间的字符，使用3个字节表示`\u0080`到`\uffff`之间的字符。

> **说明：** *bytes[]* 数组的长度为 *length*，而 *length* 占用两个字节，所能表示的最大数值为65535，因此Java虚拟机中所有的字符串（包括类名、字段名、方法名等）最大占用64K存储空间。如果定义的字符串超过64k，javac编译时会报错，提示常量字符串过长。

### 方法句柄

*CONSTANT_MethodHandle_info* 结构表示一个方法句柄。

```
CONSTANT_MethodHandle_info {
    u1 tag;
    u1 reference_kind;
    u2 reference_index;
}
```

*reference_kind* 的值范围为1-9之间的整数，表示不同的方法句柄的类型。

*reference_index* 的值是一个索引，指向常量表中的一个数据项：

- 当 *reference_kind* 值为1，2，3或4时，该索引值肯定指向一个 *CONSTANT_Fieldref_info* 结构数据项
- 当 *reference_kind* 值为5或8时，该索引值肯定指向一个 *CONSTANT_Methodref_info* 结构数据项，且如果值为8，则 *CONSTANT_Methodref_info* 结构表示的方法的名称必须为`<init>`
- 当 *reference_kind* 值为6或7，且class文件版本小于52.0（即JDK8）时，该索引值肯定指向一个 *CONSTANT_Methodref_info* 结构数据项；而当class文件版本大于等于52.0时，该索引值肯定指向一个 *CONSTANT_Methodref_info* 或 *CONSTANT_InterfaceMethodref_info* 结构数据项
- 当 *reference_kind* 值为9时，该索引值肯定指向一个CONSTANT_InterfaceMethodref_info结构数据项

### 方法类型

*CONSTANT_MethodType_info* 结构表示一个方法类型。

```
CONSTANT_MethodType_info {
    u1 tag;
    u2 descriptor_index;
}
```

*descriptor_index* 的值是一个索引，指向常量表中一个 *CONSTANT_Utf8_info* 结构数据项，表示一个方法描述符。

### 动态执行方法

*CONSTANT_InvokeDynamic_info* 结构由 *invokedynamic* 指令使用，表示一个启动方法，动态执行的名称、参数、返回类型和可选称为的静态参数的常量。

```
CONSTANT_InvokeDynamic_info {
    u1 tag;
    u2 bootstrap_method_attr_index;
    u2 name_and_type_index;
}
```

*bootstrap_method_attr_index* 的值必须是class文件中引导方法表的 *bootstrap_methods[]* 中的有效索引。

*name_and_type_index* 的值是一个索引，指向常量池中的一个 *CONSTANT_NameAndType_info* 结构数据项，表示动态执行方法的名称和描述符。

## 字段表

字段表中的每一项都是一个 *field_info* 结构的数据项，一个class文件中不能有两个字段的名称和描述符都相同，如果一个class文件中两个字段的描述符不相同，那么字段名称可以相同（Java语言中不管字段类型、修饰符是否相同，都不允许存在相同的字段名）。

字段表中的字段只包含该类或接口中的字段，不包含从父类或接口中继承的字段。

字段表的结构如下：

```
field_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

### 访问标志

*access_flags* 表示该字段的访问权限和属性，跟类或接口的访问标志一样，字段访问标志的值也是各标志值按位或的结果。

字段有如下访问和属性标志：

| 标志名称      | 标志值 | 说明                     |
| ------------- | ------ | ------------------------ |
| ACC_PUBLIC    | 0x0001 | 声明为`public`           |
| ACC_PRIVATE   | 0x0002 | 声明为`private`          |
| ACC_PROTECTED | 0x0004 | 声明为`protected`        |
| ACC_STATIC    | 0x0008 | 声明为`static`           |
| ACC_FINAL     | 0x0010 | 声明为`final`            |
| ACC_VOLATILE  | 0x0040 | 声明为`volatile`         |
| ACC_TRANSIENT | 0x0080 | 声明为`transient`        |
| ACC_SYNTHETIC | 0x1000 | 表示字段由编译器自动产生 |
| ACC_ENUM      | 0x4000 | 标识字段是否为枚举       |

### 名称索引

*name_index* 表示名称索引，值是一个有效的索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示该字段的非限定名称。

### 描述符索引

*descriptor_index* 表示描述符索引，值是一个有效的索引，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示该字段的描述符。

### 字段属性

*attributes_count* 表示该字段的额外属性的数量。

*attributes[]* 表示该字段的属性表集合，其中每一项都是一个有效索引值，指向一个属性表 *attribute_info* 结构的数据项，一个字段可以有与之关联的任意数量的额外属性。

## 方法表

方法表中的每一项都是一个 *method_info* 结构的数据项，表示一个具体的方法（包括每个实例的初始化方法`<init>`和类或接口的初始化方法`<clinit>`）。一个class文件中不能有两个方法的名称和描述符都相同。

如果父类中的方法没有被重写，则方法表中不会包含父类的方法信息，但是可能会包含编译器自动添加的方法，最常见的是类或接口构造器`<clinit>:()V`和实例构造器`<init>:()V`方法。

*method_info* 的结构和 *field_info* 的结构相同，具体如下：

```
method_info {
    u2             access_flags;
    u2             name_index;
    u2             descriptor_index;
    u2             attributes_count;
    attribute_info attributes[attributes_count];
}
```

### 访问标志

*access_flags* 表示该方法的访问权限和属性，方法的访问标志和字段的访问标志不同：

| 标志名称         | 标志值 | 说明                         |
| ---------------- | ------ | ---------------------------- |
| ACC_PUBLIC       | 0x0001 | 声明为`public`               |
| ACC_PRIVATE      | 0x0002 | 声明为`private`              |
| ACC_PROTECTED    | 0x0004 | 声明为`protected`            |
| ACC_STATIC       | 0x0008 | 声明为`static`               |
| ACC_FINAL        | 0x0010 | 声明为`final`                |
| ACC_SYNCHRONIZED | 0x0020 | 声明为`synchronized`         |
| ACC_BRIDGE       | 0x0040 | 表示是由编译器生成的桥接方法 |
| ACC_VARARGS      | 0x0080 | 表示该方法有可变参数         |
| ACC_NATIVE       | 0x0100 | 声明为`native`               |
| ACC_ABSTRACT     | 0x0400 | 声明为`abstract`             |
| ACC_STRICT       | 0x0800 | 声明为`strictfp`             |
| ACC_SYNTHETIC    | 0x1000 | 表示方法由编译器自动产生     |

### 名称索引

*name_index* 表示方法的名称索引，值是一个有效的索引，指向常量表中一个 *CONSTANT_Utf8_info* 结构的数据项，表示方法的非限定名称。

### 描述符索引

*descriptor_index* 表示方法的描述符索引，值是一个有效的索引，指向常量表中一个 *CONSTANT_Utf8_info* 结构的数据项，表示方法的描述符。

### 方法属性

*attributes_count* 表示该方法的额外属性的数量。

*attributes[]* 表示属性表索引集合，其中每一项都是一个有效索引值，指向一个属性表 *attribute_info* 结构的数据项，一个方法可以有与之关联的任意数量的额外属性。

方法中的具体代码经过编译器编译成字节码指令后，存放在 *Code* 属性中。

## 属性表

class文件，字段表，方法表和 *Code* 属性中都可以拥有自己的属性表，用于描述某些场景的专有信息。

属性表具有如下通用结构：

```
attribute_info {
    u2 attribute_name_index;
    u4 attribute_length;
    u1 info[attribute_length];
}
```

对于所有属性表来说，*attribute_name_index* 的值都必须是一个有效的16位无符号索引下标，指向常量池中一个 *CONSTANT_Utf8_info* 结构的数据项，表示该属性的名称。*attribute_length* 表示紧接着的属性信息所占用的字节长度。

JDK8中预定义了23种属性：

| 属性名称                                                     | 使用位置                                 | class版本 | 说明                                                         |
| ------------------------------------------------------------ | ---------------------------------------- | --------- | ------------------------------------------------------------ |
| SourceFile                                                   | ClassFile                                | 45.3      | 源文件名称                                                   |
| InnerClasses                                                 | ClassFile                                | 45.3      | 内部类列表                                                   |
| EnclosingMethod                                              | ClassFile                                | 49.0      | 仅当一个类为局部类或匿名类时才拥有这个属性                   |
| SourceDebugExtension                                         | ClassFile                                | 49.0      | 存储额外调试信息                                             |
| BootstrapMethods                                             | ClassFile                                | 51.0      | 用于保存 *invokedynamic* 指令所引用的引导方法限定符          |
| ConstantValue                                                | field_info                               | 45.3      | 由`final`关键字定义的常量                                    |
| Code                                                         | method_info                              | 45.3      | Java代码编译成的字节码指令                                   |
| Exceptions                                                   | method_info                              | 45.3      | 方法抛出的异常列表                                           |
| RuntimeVisibleParameterAnnotations,<br>RuntimeInvisibleParameterAnnotations | method_info                              | 49.0      | 用于指明哪些方法参数的注解是运行时可见或不可见的             |
| AnnotationDefault                                            | method_info                              | 49.0      | 用于记录注解元素的默认值                                     |
| MethodParameters                                             | method_info                              | 52.0      | 用于支持将方法名称编译进class文件                            |
| Synthetic                                                    | ClassFile, field_info, method_info       | 45.3      | 标识方法、字段是编译器自动生成的                             |
| Deprecated                                                   | ClassFile, field_info, method_info       | 45.3      | 被声明为`deprecated`的方法或字段                             |
| Signature                                                    | ClassFile, field_info, method_info       | 49.0      | 用于记录泛型签名信息                                         |
| RuntimeVisibleAnnotations,<br>RuntimeInvisibleAnnotations    | ClassFile, field_info, method_info       | 49.0      | 用于指明那些注解是运行时可见或不可见的                       |
| LineNumberTable                                              | Code                                     | 45.3      | Java源码行号与字节码指令的对应关系                           |
| LocalVariableTable                                           | Code                                     | 45.3      | 方法局部变量描述                                             |
| LocalVariableTypeTable                                       | Code                                     | 49.0      | JDK5中新增的属性，用来描述泛型参数化类型                     |
| StackMapTable                                                | Code                                     | 50.0      | JDK6中新增的属性，类型检查验证器用来检查和处理目标方法的局部变量和操作数栈所需要的类型是否匹配 |
| RuntimeVisibleTypeAnnotations,<br>RuntimeInvisibleTypeAnnotations | ClassFile, field_info, method_info, Code | 52.0      | 用于哪些类注解是运行时可见或不可见的                         |

与class文件中的其他数据项不同，[《Java虚拟机规范》][jvm_spec_8]中并没有规定各个属性表在属性表集合中出现的顺序，用户甚至可以在自己实现的前端编译器中向属性表添加自定义的属性，Java虚拟机会忽略掉它无法识别的属性。

以上预定义的属性可以分为三组：

1. 如下5个属性对于Java虚拟机正确地解释class文件至关重要，每一个都必须被Java虚拟机实现正确的识别和读取。
   - *ConstantValue*
   - *Code*
   - *StackMapTable*
   - *Exceptions*
   - *BootstrapMethods*
2. 如下12个属性对于通过Java SE平台的类库正确解释class文件至关重要
   - *InnerClasses*
   - *EnclosingMethod*
   - *Synthetic*
   - *Signature*
   - *RuntimeVisibleAnnotations*
   - *RuntimeInvisibleAnnotations*
   - *RuntimeVisibleParameterAnnotations*
   - *RuntimeInvisibleParameterAnnotations*
   - *RuntimeVisibleTypeAnnotations*
   - *RuntimeInvisibleTypeAnnotations*
   - *AnnotationDefault*
   - *MethodParameters*
3. 如下6个属性是辅助型的，对于Java虚拟机或Java SE平台类库对class文件的正确解释都不重要，但对于工具却非常有用
   - *SourceFile*
   - *SourceDebugExtension*
   - *LineNumberTable*
   - *LocalVariableTable*
   - *LocalVariableTypeTable*
   - *Deprecated*

### ConstantValue

*ConstantValue* 是出现在字段表的属性集合中的一个固定长度的属性，用于表示[常量表达式][jls_15_28]的值。

如果一个字段表中的 *access_flags* 数据项设置了 *ACC_STATIC* 标志，则该字段由字段表中的 *ConstantValue* 属性所表示的值在类加载的初始化阶段完成赋值，这个赋值动作发生在类或接口的实例初始化方法之前。

只有被`static`关键字修饰的字段才有可能存在这项属性。对非静态字段的赋值是在实例构造器的`<init>()`方法中进行的；对于静态字段，既可以在`<clinit>()`方法中赋值，也可以使用 *ConstantValue* 属性赋值。在javac编译器中，如果该字段同时使用`final`和`static`来修饰，并且这个字段的类型是基本类型或`String`类型的话，就将会生成 *ConstantValue* 属性来进行初始化；如果这个字段没有被`final`修饰，或者并非基本类型及字符串类型，则会在`<clinit>()`方法中进行初始化。

*ConstantValue* 属性的结构如下：

```
ConstantValue_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 constantvalue_index;
}
```

*attribute_name_index* 表示属性名称，值必须是常量表中的一个有效索引，指向一个 *CONSTANT_Utf8_info* 结构的数据项，值固定是`ConstantValue`。

*attribute_length* 的值在 *ConstantValue_attribute* 结构中必须是2。

*constantvalue_index* 的值必须是常量表中的一个有效索引，常量表中该索引位置必须是一个基本类型或`String`类型的值：

| 字段类型                        | 常量池中对应数据项类型 |
| ------------------------------- | ---------------------- |
| long                            | CONSTANT_Long_info     |
| float                           | CONSTANT_Float_info    |
| double                          | CONSTANT_Double_info   |
| int, short, char, byte, boolean | CONSTANT_Integer_info  |
| String                          | CONSTANT_String_info   |

### Code

*Code* 属性是方法表 *method_info* 中一个可变长度的属性，用于存储Java虚拟机指令和方法的辅助信息。本地方法和抽象方法的方法表中没有 *Code* 属性。

*Code* 属性的结构如下：

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

*attribute_name_index* 表示属性名称，值必须是常量表中的一个有效索引，指向一个 *CONSTANT_Utf8_info* 结构的数据项，值固定是`Code`。

*attribute_length* 表示属性的长度，由于 *attribute_name_index* 和 *attribute_length* 一共占用 6个字节，所以属性值的长度固定为整个属性表长度减去6个字节。

*max_stack* 表示操作数栈的最大深度。虚拟机运行时需要根据这个值来分配栈帧中的操作数栈深度。

*max_locals* 表示局部变量表所占的槽位（Slot）。每个`double`和`long`类型的局部变量占用2个变量槽，而其他不超过32位的数据类型变量占用1个变量槽。局部变量表中存储的变量包括：方法参数（包括`this`）、方法体中定义的局部变量、`catch`块中定义的异常变量等。*max_locals* 的值并不是方法中所有局部变量所占用的槽位之和，javac编译器会根据变量的作用域计算出方法中同时存在的最大局部变量数量和和类型计算出 *max_locals* 的值。Java虚拟机在运行时会对局部变量表中的变量槽进行重用，以减少栈帧所耗费的内存。

*code_length* 和 *code[]* 用来存储Java源文件编译后生成的字节码。*code_length* 表示字节码的长度，*code[]* 中存储方法中的字节码，每个字节码占用1个字节。当虚拟机读取到一个字节码时就可以找出该字节码对应的虚拟机指令是什么，并能够知道该指令是否需要跟随参数，以及参数应该如何解析。由于单个字节码只占用1个字节，能表示的数值范围是0-255，所以Java虚拟机中实际上最多能拥有256个虚拟机指令。虽然 *code_length* 占用4个字节，但[《Java虚拟机规范》][jvm_spec_8]中明确规定了方法中不能超过65535条字节码指令。

*exception_table_length* 和 *exception_table[]* 保存了方法中显式处理的异常信息。*exception_table_length* 表示*exception_table[]* 中数据项的数量。*exception_table[]* 中的每一项都描述了一个异常处理器，它们的顺序很重要。

*exception_table[]* 具有如下结构：

```
{
    u2 start_pc;
    u2 end_pc;
    u2 handler_pc;
    u2 catch_type;
}
```

*start_pc* 和 *end_pc* 的值都是 *code[]* 中的有效索引位置，用于表明异常处理程序在 *code[]* 字节码数组中的范围：[start_pc, end_pc)。

*handler_pc* 的值是 *code[]* 中的有效索引位置，指向异常处理器的开始位置，该位置的值必须是指向虚拟机指令操作码的索引。

*catch_type* 的值为非零，且必须是常量池中的有效索引位置，指向一个 *CONSTANT_Class_info* 结构的数据项，表示这个异常处理器捕获的异常类型。

### StackMapTable

*StackMapTable* 属性是在JDK6中新增的属性，是属性表中的一个变长属性，在类加载的类型检查验证阶段被用到。

*StackMapTable* 属性的结构如下：

```
StackMapTable_attribute {
    u2              attribute_name_index;
    u4              attribute_length;
    u2              number_of_entries;
    stack_map_frame entries[number_of_entries];
}
```

*attribute_name_index* 表示属性名称，值必须是常量表中的一个有效索引，指向一个 *CONSTANT_Utf8_info* 结构的数据项，值固定是`StackMapTable`。

*attribute_length* 表示 *StackMapTable* 属性的长度，不包含 *attribute_name_index* 和 *attribute_length* 占用的6个字节。

*number_of_entries* 表示 *entries[]* 中的 *stack_map_frame* 数据项的数量。*entries[]* 中的每一项都表示一个栈映射帧。每个栈映射帧都显式或隐式地代表了一个字节码偏移量，用于表示执行到该字节码时局部变量表和操作数栈的验证类型。类型检查验证器会通过检查目标方法的局部变量和操作数栈所需要的类型来确定一段字节码指令是否符合逻辑约束。

> **说明：** 在版本号为50.0（JDK6）及以上的class文件中，如果一个 *Code* 属性中没有 *StackMapTable* 属性，则它实际上包含一个隐式的栈映射，相当于 *number_of_entries* 为0的 *StackMapTable* 属性。

### Exceptions

*Exceptions* 属性的作用是列举出方法中抛出的受查异常，即`throws`关键字后面跟着的异常。*Exceptions* 属性与 *Code* 属性中的异常表 *exception_table* 不同，它用于描述方法中抛出的受检查异常信息，而 *exception_table* 则用于描述方法中捕获的异常信息并控制字节码的跳转。

*Exceptions* 属性的结构如下：

```
Exceptions_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 number_of_exceptions;
    u2 exception_index_table[number_of_exceptions];
}
```

*attribute_name_index* 表示属性名称，值必须是常量表中的一个有效索引，指向一个 *CONSTANT_Utf8_info* 结构的数据项，值固定是`Exceptions`。

*attribute_length* 表示 *Exceptions* 属性的长度，不包含 *attribute_name_index* 和 *attribute_length* 占用的6个字节。

*number_of_exceptions* 表示 *exception_index_table[]* 中数据项的数量。*exception_index_table[]* 中的每一项都是一个指向常量池的有效索引，指向一个 *CONSTANT_Class_info* 结构的数据项，表示受检查异常的类型。

### BootstrapMethods

*BootstrapMethods* 属性是一个可变长度的属性，位于class结构的属性表中，是JDK7中新增的属性，用于保存 *invokedynamic* 指令引用的引导方法限定符。

[《Java虚拟机规范》][jvm_spec_8]中规定如果一个class文件的常量池中出现了 *CONSTANT_InvokeDynamic_info* 类型的常量，则该class文件的属性表中必须同时存在一个 *BootstrapMethods* 属性。

*BootstrapMethods* 属性的结构如下：

```
BootstrapMethods_attribute {
    u2 attribute_name_index;
    u4 attribute_length;
    u2 num_bootstrap_methods;
    {   u2 bootstrap_method_ref;
        u2 num_bootstrap_arguments;
        u2 bootstrap_arguments[num_bootstrap_arguments];
    } bootstrap_methods[num_bootstrap_methods];
}
```

*attribute_name_index* 表示属性名称，值必须是常量表中的一个有效索引，指向一个 *CONSTANT_Utf8_info* 结构的数据项，值固定是`BootstrapMethods`。

*attribute_length* 表示 *BootstrapMethods* 属性的长度，不包含 *attribute_name_index* 和 *attribute_length* 占用的6个字节。

*num_bootstrap_methods* 表示 *bootstrap_methods[]* 中启动方法的数量。*bootstrap_methods[]* 中的每一项都表示一个引导方法，还包含了该引导方法的静态参数序列（可能为空）。

*bootstrap_methods[]* 中的每一项都拥有如下结构：

```
{
    u2 bootstrap_method_ref;
    u2 num_bootstrap_arguments;
    u2 bootstrap_arguments[num_bootstrap_arguments];
}
```

*bootstrap_method_ref* 的值是一个指向常量池中 *CONSTANT_MethodHandle_info* 结构的索引。

*num_bootstrap_arguments* 的值是 *bootstrap_arguments[]* 中数据项的数量。*bootstrap_arguments[]* 中的每一项都是一个指向常量池中的有效索引，且该索引位置必须是如下数据项中的一个：*CONSTANT_String_info*, *CONSTANT_Class_info*, *CONSTANT_Integer_info*, *CONSTANT_Long_info*, *CONSTANT_Float_info*, *CONSTANT_Double_info*, *CONSTANT_MethodHandle_info* 或 *CONSTANT_MethodType_info*。

囿于篇幅，关于属性表中的其他属性，这里就不再一一赘述，感兴趣的读者可以阅读[Chapter 4.7. Attributes][jvm_spec_8_4_7]作更全面的了解。

[jls]: https://docs.oracle.com/javase/specs/jls/se8/html/index.html
[jls_15_28]: https://docs.oracle.com/javase/specs/jls/se8/html/jls-15.html#jls-15.28
[jvm_spec_8]: https://docs.oracle.com/javase/specs/jvms/se8/html/index.html
[jvm_spec_8_4_7]: https://docs.oracle.com/javase/specs/jvms/se8/html/jvms-4.html#jvms-4.7
