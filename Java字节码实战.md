# Java字节码实战
- Java字节码的二进制格式
- 字节码的魔数与版本
- 字节码的常量池
- 字节码的类继承
- 字节码的字段存储
- 字节码的方法格式

> 想要深刻理解JVM执行引擎的机制，就必须对JVM内部的数据结构有深入了解，而要了解JVM内部的数据结构就必须要了解Java字节码。

## 字节码初探
测试用例：

```
public class Test {
    public int a = 3;
    static Integer si = 6;
    String s = "Hello world!";

    public static void main(String[] args) {
        Test test = new Test();
        test.a = 8;
        si = 9;
    }
    private void test(){
        this.a = a;
    }
}
```
我们使用javap -verbose Test来分析上面测试类的字节码文件。

```
Classfile /G:/ThinkInJava/ThinkInJava4th/src/JVM/Test.class
  Last modified 2019-9-29; size 631 bytes
  MD5 checksum 1627651c6c617c5efd2285751ae57f3c
  Compiled from "Test.java"
public class JVM.Test
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #9.#26         // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#27         // JVM/Test.a:I
   #3 = String             #28            // Hello world!
   #4 = Fieldref           #5.#29         // JVM/Test.s:Ljava/lang/String;
   #5 = Class              #30            // JVM/Test
   #6 = Methodref          #5.#26         // JVM/Test."<init>":()V
   #7 = Methodref          #31.#32        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #8 = Fieldref           #5.#33         // JVM/Test.si:Ljava/lang/Integer;
   #9 = Class              #34            // java/lang/Object
  #10 = Utf8               a
  #11 = Utf8               I
  #12 = Utf8               si
  #13 = Utf8               Ljava/lang/Integer;
  #14 = Utf8               s
  #15 = Utf8               Ljava/lang/String;
  #16 = Utf8               <init>
  #17 = Utf8               ()V
  #18 = Utf8               Code
  #19 = Utf8               LineNumberTable
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               test
  #23 = Utf8               <clinit>
  #24 = Utf8               SourceFile
  #25 = Utf8               Test.java
  #26 = NameAndType        #16:#17        // "<init>":()V
  #27 = NameAndType        #10:#11        // a:I
  #28 = Utf8               Hello world!
  #29 = NameAndType        #14:#15        // s:Ljava/lang/String;
  #30 = Utf8               JVM/Test
  #31 = Class              #35            // java/lang/Integer
  #32 = NameAndType        #36:#37        // valueOf:(I)Ljava/lang/Integer;
  #33 = NameAndType        #12:#13        // si:Ljava/lang/Integer;
  #34 = Utf8               java/lang/Object
  #35 = Utf8               java/lang/Integer
  #36 = Utf8               valueOf
  #37 = Utf8               (I)Ljava/lang/Integer;
{
  public int a;
    descriptor: I
    flags: ACC_PUBLIC

  static java.lang.Integer si;
    descriptor: Ljava/lang/Integer;
    flags: ACC_STATIC

  java.lang.String s;
    descriptor: Ljava/lang/String;
    flags:

  public JVM.Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: aload_0
         5: iconst_3
         6: putfield      #2                  // Field a:I
         9: aload_0
        10: ldc           #3                  // String Hello world!
        12: putfield      #4                  // Field s:Ljava/lang/String;
        15: return
      LineNumberTable:
        line 3: 0
        line 4: 4
        line 6: 9

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=2, args_size=1
         0: new           #5                  // class JVM/Test
         3: dup
         4: invokespecial #6                  // Method "<init>":()V
         7: astore_1
         8: aload_1
         9: bipush        8
        11: putfield      #2                  // Field a:I
        14: bipush        9
        16: invokestatic  #7                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
        19: putstatic     #8                  // Field si:Ljava/lang/Integer;
        22: return
      LineNumberTable:
        line 9: 0
        line 10: 8
        line 11: 14
        line 12: 22

  static {};
    descriptor: ()V
    flags: ACC_STATIC
    Code:
      stack=1, locals=0, args_size=0
         0: bipush        6
         2: invokestatic  #7                  // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
         5: putstatic     #8                  // Field si:Ljava/lang/Integer;
         8: return
      LineNumberTable:
        line 5: 0
}
SourceFile: "Test.java"
```
> 每一个const后面的#号后的数字代表了长廊吃向在常量池中的索引，当JVM在解析类的常量池信息是，常量池项的索引与此一致。

Test.class文件的十六进制字节码如图字节码十六进制.png
现在我们来逐项分析。

### 魔数与版本
- 所有class字节码文件的开始4字节都是魔数，并且值固定为0xCAFEBABE，如果不是，则JVM拒绝解析。

- 魔数之后的四个字节是版本信息，前两个字节表示major version，即主版本号，后两个字节表示minor version，即次版本号，这里版本号是0x00000034，是jdk1.8.
    * 1.1（45），1.2（46），1.3（47）...，1.8（52）

### 常量池
> 常量池是.class字节码文件中非常重要和核心的内容，一个Java类中的绝大多数信息都是由常量池保存，尤其是Java类中定义的变量和方法，都由常量池保存。在JVM内存模型中，有一块就是常量池，JVM堆区的常量池就是用于保存每一个Java类所对应的常量池的信息，一个Java应用程序中所包含的所有Java类的常量池组成了JVM堆区中大的常量池。

- 常量池的基本结构
> Java类所对应的常量池主要由常量池数量和常量池数组两部分组成，常量池数量紧跟在版本号后面，占两字节，常量池数组紧随其后。常量池数组的每个元素的第一个数据都是一个u1类型，是**标志位**，占一个字节，JVM解析常量池时，根据u1来获取元素具体类型。

**每一个常量池元素都由tag和数据内容两部分组成。**

#### JVM定义的十一种常量。
编号|常量池元素名称|tag标识|含义
---|---|---|---|
1|CONSTANT_Utf8_info|1|UTF-8编码的字符串
2|CONSTANT_Integer_info|3|整型字面量
3|CONSTANT_Float_info|4|浮点型字面量
4|CONSTANT_Long_info|5|长整型字面量
5|CONSTANT_Double_info|6|双精度浮点型字面量
6|CONSTANT_Class_info|7|类或接口的符号引用
7|CONSTANT_String_info|8|字符串类型的字面量
8|CONSTANT_Fieldref_info|9|字段的符号引用
9|CONSTANT_Methodref_info|10|类中方法的符号引用
10|CONSTANT_InterfaceMethodref_info|11|接口中方法的符号引用
11|CONSTANT_NameAndType_info|12|字段和方法的名称以及类型的符号引用

**类的方法信息、接口和继承信息、属性信息都是定义在NameAndType_Info中的。

#### 常量池元素结构
> 常量池数组的每一种元素的内容都是符合数据结构的，下面给出JVM所定义的常量池中每一种元素的具体结构。

常量池元素结构.png

#### 常量池元素总数量
> 从第九字节开始的一大段字节流都用于描述常量池数组信息，其中，第九、十字节用于描述常量池元素总数量。

第9和10字节保存的常量池数组大小是0x26，十进制是38，说明该字节码文件共包含38个常量池元素，JVM规定不使用第0个元素，因此实际上是37个常量池元素。

#### 第一个常量池元素
> 从第十一字节开始就是常量池数组，每一个元素都是以tag位标开始，tag位标只占一个字节。

第十一字节对应的是0xA，也就是10，通过上面两张表可以知道，这代表的是CONSTANT_Methodref_info，组成如下：
- tag：u1
- index：u2，指向声明方法的类描述符CONSTANT_Class_info的索引项
- index:u2，指向名称及类型描述符CONSTANT_NameAndType_info的索引项

> 接下来第十二、十三个字节合起来表示index，值为9（#9 = Class              #34            // java/lang/Object），第十四十五个字节为第二个index，值为0x1A，也就是26（ #26 = NameAndType        #16:#17        // "<init>":()V）

#### 第二个常量池元素
> 从第十六个字节开始是第二个常量池元素，tag位是0x09，对应的类型是CONSTANT_Fieldref_info紧跟的后四个字节前两个指向class，后两个指向nameAndType。

- index:0x0005:#5 = Class              #30            // JVM/Test
- index:0x001B: #27 = NameAndType        #10:#11        // a:I

#### 第三个常量池元素
> 从第二十一字节开始是第三个常量池元素，tag位是0x08，对应类型是CONSTANT_String_info，第21、 22两个字节是指向字符串字面量的索引
- index:0x001C:  #28 = Utf8               Hello world!

#### 第四个常量池元素
> 从第23字节开始是第四个常量池元素，tag位是0x09，对应类型CONSTANT_Fieldref_info，24 25 26 27四个字节分别是两个index，指向Class和NameAndType

- index:0x0005: #5 = Class              #30            // JVM/Test
- index:0x001D:#29 = NameAndType        #14:#15        // s:Ljava/lang/String;

#### 第五个常量池元素
> 从第28字节开始是第五个，tag位：07，类型：CONSTANT_Class_info，29 30字节是index指向全限定名常量项索引

- index:001E:#30 = Utf8               JVM/Test

#### 第六个常量池元素
> 从第31字节开始，tag位：0x0A,类型：CONSTANT_Methodref_info，32 33 34 35是两个index，分别指向Class和NameAndType
- index:0x0005:#5 = Class              #30            // JVM/Test
- index:0x001A:#26 = NameAndType        #16:#17        // "<init>":()V

#### 第七个常量池元素
> 从36字节开始，tag位：0x0A，类型：CONSTANT_Methodref_info，37 38 39 40是俩个index

- index:0x001F: #31 = Class              #35            // java/lang/Integer
- index:0x0020  #32 = NameAndType        #36:#37        // valueOf:(I)Ljava/lang/Integer;

#### 第八个常量池元素
> 从41字节开始，tag位：0x09，类型：CONSTANT_Fieldref_info， 42 43 44 45是俩index位

- index:0005 #5 = Class              #30            // JVM/Test
- index:0021#33 = NameAndType        #12:#13        // si:Ljava/lang/Integer;

#### 第九个常量池元素
> 从46开始，tag位：0x07，类型：CONSTANT_Class_info， 47 48指向全限定名常量项索引

- index:0x0022:#34 = Utf8               java/lang/Object
**这个常量是父类常量，用于描述父类信息，由于本类没有显示继承其他类，所以默认继承Object类。**

#### 第十个常量池元素
> 从49开始，tag：01，类型：CONSTANT_Utf8_info，是一个utf-8编码的字符串，50 51是length位，52是bytes
- length:0x0001:说明总共占一个字节
- bytes:0x61:十进制97，对应ASCII编码为a

#### 第十一个常量池元素
> 从53开始，tag：01，类型：CONSTANT_Utf8_info，是一个utf-8编码的字符串，54 55是length位，56是bytes

- length:0x0001:说明总共占一个字节
- bytes:0x49:十进制73，对应ASCII编码为I

#### 第十二个常量池元素
> 从57开始，tag：01，类型：CONSTANT_Utf8_info，是一个utf-8编码的字符串，58 59是length位，60 61是bytes

- length:0x0002:说明总共占两个字节
- bytes:0x73,0x69:十进制115,105，对应ASCII编码为s,i即变量si

#### 第十三个常量池元素
> 从62开始，tag：01，类型：CONSTANT_Utf8_info，是一个utf-8编码的字符串，63 64是length位，后面19个字节是bytes

- length:0x0013即19个
- bytes:对应字符串：Ljava/lang/Integer;

**第十二、十三个常量池元素合起来描述了Test类中static Integer si这样的类变量。**

#### 第十四个常量池元素
> tag:01,类型：CONSTANT_Utf8_info

- length:0x0001
- bytes:0x73对应ASCII：s

#### 第十五个常量池元素
> tag:01

- length:0x0012即18字节
- bytes:对应字符串：Ljava/lang/String;

**第十四十五个常量池元素描述了Test类中的s字符串变量**

对于剩下的一些常量池元素我们不在一一分析，但分析方法一样，接下来我们把目光回到.class字节码文件后续的组成部分

## 访问标识与继承信息

### access_flags
> 字节码文件中，常量池数组之后紧跟着的是access_flags结构，该类型为u2，占2字节，access_flag代表访问标志位，用于标注类或接口层次的访问信息，如当前类是类还是接口，是否为public，是否为abstract类型等。

标志名称|标志值|含义
---|---|---|
ACC_PUBLIC|0x0001|是否为public类型
ACC_FINAL|0x0010|是否为final，只有类可以设置
ACC_SUPER|0x0020|是否允许使用invokespecial字节码指令，1.2版本以后为真
ACC_INTERFACE|0x0200|标识这是一个接口
ACC_ABSTRACT|0x0400|是否为abstract类型，对于接口和抽象类为真
ACC_SYNTHETIC|0x1000|标识这个类型并非由用户代码产生
ACC_ANNOTATION|0x2000|标识这是一个注解
ACC_ENUM|0x4000|标识这是一个枚举

本类中的access_flag是0x21，因此他的访问标识既包含ACC_PUBLIC(0x0001)又包含ACC_SUPER(0x0020)

### this_class
> 紧跟着access_flags访问标识之后的是this_class结构，类型是u2，记录当前类的全限定名（包名+类名），指向常量池中对应的索引值

本类中this_class的值是0x0005，根据字节码信息可知#5 = Class              #30            // JVM/Test

### super_class
> 接着的是super_class，类型u2，记录当前类的父类全局限定名，指向常量池索引。

本类中值是0x0009，#9 = Class              #34            // java/lang/Object

### interface
> 紧跟着super_class标识之后的是interfaces_count结构，类型u2，interfaces_count后是一组u2类型数据的集合，描述当前类实现了那些借口，按implements后的顺序从左到右排列在接口索引集合中

本类中值是0x0000，因此没有接口信息。

## 字段信息
### fields_count
> 紧跟着接口，类型u2，记录当前类中所定义的变量总数量，包括成员变量和类变量（静态变量）

本类中值为0x0003，一共包含三个变量，是a si 和s
### field_info fields[fields_count]
> 紧跟着fields_count的是fields结构，长度不确定，记录所定义的各个变量的详细信息，包括变量名、变量类型、访问标识、属性等。

fields结构组成格式
类型|名称|数量
---|---|---|
u2|access_flags|1|
u2|name_index|1|
u2|descriptor_index|1|
u2|attributes_count|1|
attrubute_info|attributes|attributes_count|

- access_flags:标识变量的访问表示，该值可选由JVM规范规定
- name_index：变量的简单名称引用，指向常量池索引
- descriptor_index：变量的类型信息引用，指向常量池索引

access_flags可选项
标志名称|标志值|含义
---|---|---|
ACC_PUBLIC|0x0001|是否为public类型
ACC_PRIVATE|0x0002|是否为private类型
ACC_PROTECTED|0x0004|是否为protected类型
ACC_STATIC|0x0008|是否为static类型
ACC_FINAL|0x0010|是否为final
ACC_VOLATILE|0x0040|是否为volatile
ACC_TRANSIENT|0x0080|是否为transient
ACC_SYNTHETIC|0x1000|是否为编译器自动产生
ACC_ENUM|0x4000|是否为enum

**其中ACC_PUBLIC、ACC_PRIVATE、ACC_PROTECTED三选一，接口字段必须有ACC_PUBLIC、ACC_STATIC、ACC_FINAL标志。**

#### 第一个变量a
> 0x0001 000A 000B 0000分别代表访问标识（ACC_PUBLIC)、名称索引（#10 = Utf8 a）、类型索引（ #11 = Utf8 I就是int）和属性数量（没有属性）

标识符|含义
---|---|
B|基本类型byte
C|基本类型char
D|基本类型double
F|基本类型float
I|基本类型int
J|基本类型long
S|基本类型short
Z|基本类型boolean
V|特殊类型void
L|对象类型如Ljava/lang/Object
> 对于数组类型，每一维使用一个前置的“[”字符来描述，如int[]被记录为[I,String[][]为"[[Ljava/lang/String;"

> 用描述符描述方法时，按照先参数列表，后返回值的顺序，参数列表按照参数的严格顺序放在一组（）内，如方法“String getAll(int id,String name)”描述符为“(I,Ljava/lang/String;)Ljava/lang/String”

#### 变量si和s
> si:0x0008 000C 000D 0000  static   #12 = Utf8   si #13 = Utf8  Ljava/lang/Integer;
> s: 0x0000 000E 000F 0000 无访问标识符 #14 = Utf8 s #15 = Utf8  Ljava/lang/String;

## 方法信息
### methods_count
> 紧跟着fileds的是methods_count结构，u2类型，描述类一共包含多少方法

> 本示例中为0x0004，即Test类一共包含四个方法，可是我们明明只定义了两个方法，这是因为在编译期间，编译器会自动为一个类添加void <clinit>()这样一个方法，该方法的主要作用是执行类的初始化，源程序中所有static类型变量都会在这个方法中完成初始化，被static所包围的程序都在这个方法中执行。同时，JVM为它添加了一个默认构造器void<init>()，所以一共包含四个方法。

### method_info methods[methods_count]
> 紧跟在后面的是methods结构，是一个数组，每一个方法的全部细节都包含在里面，包括代码指令。

methods结构组成格式
类型|名称|数量
---|---|---|
u2|access_flags|1|
u2|name_index|1|
u2|descriptor_index|1|
u2|attributes_count|1|
attrubute_info|attributes|attributes_count|

access_flags可选项
标志名称|标志值|含义
---|---|---|
ACC_PUBLIC|0x0001|是否为public类型
ACC_PRIVATE|0x0002|是否为private类型
ACC_PROTECTED|0x0004|是否为protected类型
ACC_STATIC|0x0008|是否为static类型
ACC_FINAL|0x0010|是否为final
ACC_SYNCHRONIZED|0x0020|是否为SYNCHRONIZED
ACC_BRIDGE|0x0040|是否为编译器产生的桥接方法
ACC_VARARGS|0x0080|是否接收不定参数
ACC_NATIVE|0x0100|是否为native
ACC_ABSTRACT|0x0400|是否为abstract
ACC_STRICTFP|0x0800|是否为strictfp
ACC_SYNTHETIC|0x1000|是否为编译器自动产生


```
Constant pool:
   #1 = Methodref          #9.#26         // java/lang/Object."<init>":()V
   #2 = Fieldref           #5.#27         // JVM/Test.a:I
   #3 = String             #28            // Hello world!
   #4 = Fieldref           #5.#29         // JVM/Test.s:Ljava/lang/String;
   #5 = Class              #30            // JVM/Test
   #6 = Methodref          #5.#26         // JVM/Test."<init>":()V
   #7 = Methodref          #31.#32        // java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
   #8 = Fieldref           #5.#33         // JVM/Test.si:Ljava/lang/Integer;
   #9 = Class              #34            // java/lang/Object
  #10 = Utf8               a
  #11 = Utf8               I
  #12 = Utf8               si
  #13 = Utf8               Ljava/lang/Integer;
  #14 = Utf8               s
  #15 = Utf8               Ljava/lang/String;
  #16 = Utf8               <init>
  #17 = Utf8               ()V
  #18 = Utf8               Code
  #19 = Utf8               LineNumberTable
  #20 = Utf8               main
  #21 = Utf8               ([Ljava/lang/String;)V
  #22 = Utf8               test
  #23 = Utf8               <clinit>
  #24 = Utf8               SourceFile
  #25 = Utf8               Test.java
  #26 = NameAndType        #16:#17        // "<init>":()V
  #27 = NameAndType        #10:#11        // a:I
  #28 = Utf8               Hello world!
  #29 = NameAndType        #14:#15        // s:Ljava/lang/String;
  #30 = Utf8               JVM/Test
  #31 = Class              #35            // java/lang/Integer
  #32 = NameAndType        #36:#37        // valueOf:(I)Ljava/lang/Integer;
  #33 = NameAndType        #12:#13        // si:Ljava/lang/Integer;
  #34 = Utf8               java/lang/Object
  #35 = Utf8               java/lang/Integer
  #36 = Utf8               valueOf
  #37 = Utf8               (I)Ljava/lang/Integer;
```

#### 第一个方法<init>

- access_flags:0x0001:public
- name_index:0x0010: #16 = Utf8               <init>
- descriptor:0x0011:#17 = Utf8               ()V
- attribute_count:0x0001:#1 = Methodref          #9.#26         // java/lang/Object."<init>":()V
所以第一个方法时void init()，是由JVM自动添加的无参构造器

在分析属性信息之前，我们需要先了解attributes这一字段结构组成：
9大属性表集合
属性名称|使用位置|含义
---|---|---|
Code|方法表|Java代码编译成的字节码指令
ConstantValue|字段表|final关键字定义的常量值
Deprecated|类文件、字段表、方发表|被声明为deprecated的方法和字段（不赞成使用的）
Exceptions|方法表|方法抛出的异常
InnerClasses|类文件|内部类列表
LineNumberTale|Code属性|Java源码的行号与字节码指令的对应关系
LocalVariableTable|Code属性|方法的局部变量描述
SourceFile|类文件|源文件名称
Synthetic|类文件、方法表、字段表|表示方法或字段是由编译器自动生成的

9中属性具体结构：
1.Code属性
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|max_stack|1
u2|max_locals|1
u4|code_length|1
u1|code|code_length
u2|exception_table_length|1
exception_info|exception_bale|exception_table_length
u2|attributes_count|1
attribute_info|attributes|attributes_count

- max_stack：操作数栈深度最大值，JVM根据这个值分配栈帧的操作数栈深度
- max_locals:局部变量表所需存储空间，单位为Slot，并不是所有局部变量占用的Slot之和，一个局部变量生命周期结束后分配给其他局部变量
- code_length和code：存放Java源程序经编译后生成的字节码指令

2.ConstantValue属性
> ConstantValue属性通知虚拟机自动为静态变量赋值，只有static变量才能使用
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|constantvalue_index|1

- 其中attribute_length固定为0x00000002，constantvalue_index为常量池字面类型常量索引，只支持基本类型和字符串
**对非static变量的赋值在示例构造函数<init>中进行，如果变量同时被static和final修饰，并且为基本类型或字符串，就生成ConstantValue进行初始化，否则在构造函数<clinit>中初始化**

3.Exception属性
> 该属性列举出方法可能抛出的受查异常（即方法描述是throw关键字后列出的异常），与Code属性平级
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|number_of_exceptions|1
u2|exception_index_table|number_of_exceptions

4.InnerClasses属性
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|number_of_classes|1
u2|inner_classes|number_of_classes

inner_class_info表结构
类型|名称|数量|备注
---|---|---|---|
u2|inner_class_info_index|1|指向常量池Class索引
u4|outer_class_info_index|1|指向常量池Class索引
u2|inner_class_name_index|1|指向常量池utf-8类型索引，为内部类名称，如果为匿名类则为0
u2|inner_name_access_flags|1

> inner_class_info_index和outer_class_info_index指向常量池中CONSTANT_Class_info类型索引，access_flags与类的访问属性的可选值一样

5.LineNumberTable属性
> 用于描述Java源码的行号与字节码行号之间的关系，可以使用-g：none或-g：lines命令关闭或开启，关闭时在抛出异常时不会显示出错的行号，调试时无法按照源码设置断点。
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|line_number_table_length|1
line_number_table_info|line_number_table|line_number_table_length

line_number_table_info属性结构表
类型|名称|数量|备注
---|---|---|---|
u2|start_pc|1|字节码行号
u4|line_number|1|源码行号

6.LocalVariableTable属性
> 用于描述栈帧中局部变量表中变量与Java源码中定义的变量之间的关系，可以使用-g:none或-g:vars命令关闭或开启，关闭时其他人引用这个方法时，所有的参数名称都将都是，调试时调试器无法根据参数名称从上下文中获取参数值
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|local_variable_table_length|1
ocal_variable_table_info|ocal_variable_table|ocal_variable_table_length

local_variable_info结构表
类型|名称|数量|备注
---|---|---|---|
u2|start_pc|1|局部变量的生命周期开始的字节码偏移量
u2|length|1|局部变量作用范围覆盖长度
u2|name_index|1|局部变量名称索引
u2|description_index|1|局部变量描述符
u2|index|1|局部变量在栈帧局部变量表中Slot位置，如果为64位，会占用Slot的index和index+1位置

7.SourceFile属性
> 记录生成这个class文件的源码文件名称，可用-g:none或-g：source关闭或开启,关闭后，在抛出异常时不显示出错误代码所属的文件名
类型|名称|数量
---|---|---|
u2|attribute_name_index|1
u4|attribute_length|1
u2|sourcefile_index|1

8.Deprecated属性和Synthetic属性
> 都属于标志类型的不二属性，只存在有和没有的差别，Deprecated表示该类、方法不推荐使用，可以通过注解进行设置。Synthetic表示该字段或方法不是有Java源码直接生成，而是由编译器自动添加，当然也可以设置访问标志ACC_SYNTHETIC

类型|名称|数量|备注
---|---|---|---
u2|attribute_name_index|1|
u4|attribute_length|1|0x00000000

- 现在我们继续回到第一个方法。
- access_flags:0x0001:public
- name_index:0x0010: #16 = Utf8               <init>
- descriptor:0x0011:#17 = Utf8               ()V
- attribute_count:0x0001:#1 = Methodref          #9.#26         // java/lang/Object."<init>":()V

- 接下来两个字节为0x0012：（#18 = Utf8 Code），正是Code方法表,
- 接下来是u4类型的attribute_length:0x00000030,对应十进制48，
- 接下来是u2类型的max_stack(0x0002)和max_locals（0x0001）对应2和1
- 然后是u4类型的code_length（0x00000010），值是16
- code_length之后是的字节码流是code属性，是Java的精华所在，因为code_length的值是16，所以之后的16个字节都用于描述字节码指令
> JVM是基于栈的指令系统，都属于一元操作数类型，只有一个操作数，一些指令后面不跟操作数，以此来分清楚指令和数据。
> 0x 2A B7 00 01 2A 06 B5 00 02 2A 12 03 B5 00 04 B1
这段指令对应的就是下面的Code中的方法：
```
 public JVM.Test();
    descriptor: ()V
    flags: ACC_PUBLIC
Code:
      stack=2, locals=1, args_size=1
         0: aload_0     //对应2A，将第一个引用类型本地变量推送至栈顶
         1: invokespecial #1                  //B7 Method java/lang/Object."<init>":()V
         4: aload_0 //01
         5: iconst_3    //
         6: putfield      #2                  // 06 Field a:I
         9: aload_0 
        10: ldc           #3                  // String Hello world!
        12: putfield      #4                  // Field s:Ljava/lang/String;
        15: return          //B1
      
```
Code属性中也会引用其他8类属性
- 接下来的四个字节表示attributes_count：0x00000001，表示一共引用了一个其他类型属性。
- 接下来两个字节为：0x0013，十进制为19，对应 #19 = Utf8               LineNumberTable，描述的属性是LineNumberTable
> 包括u2类型attribute_name_index，u4类型attribute_length，u2类型line_number_table_length，u2类型line_number_table
- 其中attribute_name_index：19 = Utf8               LineNumberTable，描述的属性是LineNumberTable
- attribute_length：0x0000000E，表示长度为14，表明LineNumberTable接下来还占14字节，14字节之后则不属于当前属性
- line_number_table_length：0x0003，标记后面line_number_table的数量，所以是3个line_number_table元素
- 接下来的12字节描述了两个line_number_table元素
- 0x00000003：前两字节表示字节码指令偏移位置，值是0，后两字节代表Java源码行号值是3
- 0x00040004：前两字节表示字节码指令偏移位置，值是4，后两字节代表Java源码行号值是4
- 0x00090006：前两字节表示字节码指令偏移位置，值是9，后两字节代表Java源码行号值是6
上面正对应以下：

```
LineNumberTable:
        line 3: 0
        line 4: 4
        line 6: 9
```
到此之后的字节码信息就不属于当前方法的范畴，对于其余的方法不再做详细描述。