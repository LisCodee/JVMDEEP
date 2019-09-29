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

第9和10字节保存的常量池数组大小是0x26，十进制是38，说明该字节码文件共包含38个常量池元素，JVM规定不使用第0个元素，因此实际上是50个常量池元素。

- 第一个常量池元素
> 从第十一字节开始就是常量池数组，每一个元素都是以tag位标开始，tag位标只占一个字节。

第十一字节对应的是0xA，也就是10，通过上面两张表可以知道，这代表的是CONSTANT_Methodref_info，组成如下：
- tag：u1
- index：u2，指向声明方法的类描述符CONSTANT_Class_info的索引项
- index:u2，指向名称及类型描述符CONSTANT_NameAndType_info的索引项
接下来第十二、十三个字节合起来表示index，值为9（#9 = Class              #34            // java/lang/Object），第十四十五个字节为第二个index，值为0x1A，也就是26（ #26 = NameAndType        #16:#17        // "<init>":()V）
