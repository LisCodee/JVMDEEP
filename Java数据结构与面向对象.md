# Java数据结构与面向对象
- 数据结构的基本含义
- Java数据结构的实现机制
- Java数据结构的字节码格式分析
- 大端与小端

### 数据结构与算法
> 编程就是使用合适的算法处理特定的数据结构，程序就是算法与数据结构的有机结合。Java程序的算法由Java字节码指令所驱动，而数据结构往往会作为算法的输入输出和中间产出，及时输出的是一个简单的数字也是一种数据结构。

- 在Java中不仅将程序算法“字节码”化，连同数据结构一起被“字节码”化。
- Java的数据结构接触了对物理机器的依赖，但是C中定义的结构体被转换成机器指令后，他的类型信息会被彻底抹去，变得不可理解。

### Java类型识别
> Java类在编译器生成的字节码有其特定的组织规律，Java虚拟机在加载类是，对于其生成的字节码信息按照固定的格式进行解析，可以解析出字节码中所存储的类型的结构信息，从而在运行期完全还原出原始的Java类的全部结构。

### Java字节码概述
- class文件的十个组成结构
    * MagicNumber
    * Version
    * Constant-pool
    * Access_flag
    * This_class
    * Super_class
    * Interfaces
    * Fileds
    * Methods
    * Attributes

简单说明
- MagicNumber：用来标识class文件，位于每一个Java class文件的最前面四个字节，值固定为0xCAFEBABE，虚拟机加载class文件时会先检查这四个字节，如果不是这个值则拒绝加载
- Version：由2个长度为2字节的字段组成，分别为MagicVersion和Minor Version个，代表当前class文件的主版本号和次版本号。高版本的JVM可以兼容低版本的
- 常量池：从第九个字节开始，首先是2字节的长度字段constant_pool_count说明常量池包含多少常量，接下来的二进制信息描述这些常量，常量池里放的是字面常量和符号引用
> 字面常量主要包含文本串以及被声明为final的常量，符号引用包好类的接口和全局限定名、字段的名称和描述符、方法的名称和描述符，因为Java语言在编译时没有连接这一步，所有的引用都是运行时动态加载的，所以要把这些引用存在class文件里。符号引用保存的是引用的全局限定名，所以保存的是字符串。
字面常量分为字符串、整形、长整形、浮点型、双精度浮点型

- Access_flag:保存当前类的访问权限
- This_class：保存当前类的全局限定名在常量池的索引
- Super_class: 保存当前类的父类的全局限定名在常量池的索引
- Interfaces：主要保存当前类实现的接口列表，包含interfaces_count和interfaces[interfaces_count]
- Fields:主要保存当前类的成员列表：包含fields_count和fields[files_count]
- Methods：保存当前类的方法列表，包含methods_count和method[methods_count]
- Attributes：主要保存当前类attributes列表，包含attributes_count和attributes[attributes_count]

#### JVM内部的int类型
- 1字节：uint8_t,JVM:u1
- 2字节: uint16_t,JVM:u2
- 4字节：uint32_t,JVM:u4
- 8字节：uint64_t,JVM:u8

#### 常量池与JVM内部对象模型
> 常量池是Java字节码文件中比较重要的概念，是整个Java类的核心所在，他记录了一个Java类的所有成员变量，成员方法和静态变量与静态方法、构造函数等全部信息，包括变量名、方法名、访问表示、类型信息等。

#### OOP-KLASS二分模型
> KLASS用于保存类元信息，而OOP用于表示JVM所创建的类实例对象，KLASS信息被保存在perm永久区，而oop则被分配在heap堆区，同时JVM为了支持反射等技术，必须在OOP中保存一个指针，用于指向其所属的类型KLASS，这样Java开发者便可以基于反射技术，在Java程序运行期获取Java的类型信息。

HotSpot中的oop体系

```
typedef class oopDesc*                            oop;
typedef class   instanceOopDesc*            instanceOop;
typedef class   methodOopDesc*                    methodOop;
typedef class   constMethodOopDesc*            constMethodOop;
typedef class   methodDataOopDesc*            methodDataOop;
typedef class   arrayOopDesc*                    arrayOop;
typedef class     objArrayOopDesc*            objArrayOop;
typedef class     typeArrayOopDesc*            typeArrayOop;
typedef class   constantPoolOopDesc*            constantPoolOop;
typedef class   constantPoolCacheOopDesc*   constantPoolCacheOop;
typedef class   klassOopDesc*                    klassOop;
typedef class   markOopDesc*                    markOop;
typedef class   compiledICHolderOopDesc*    compiledICHolderOop;
```
Hotspot使用klass来描述Java类型信息，定义了如下几种

```
class Klass;
class   instanceKlass;
class     instanceMirrorKlass;
class     instanceRefKlass;
class   methodKlass;
class   constMethodKlass;
class   methodDataKlass;
class   klassKlass;
class     instanceKlassKlass;
class     arrayKlassKlass;
class       objArrayKlassKlass;
class       typeArrayKlassKlass;
class   arrayKlass;
class     objArrayKlass;
class     typeArrayKlass;
class   constantPoolKlass;
class   constantPoolCacheKlass;
class   compiledICHolderKlass;
```
#### oopDesc
> C++中的instanceOopDesc等类最终全部来自顶级父类oopDesc，Java的面向对象和运行期反射的能力便是由其予以提现和支撑。

```
class oopDesc {
  friend class VMStructs;
 private:
  volatile markOop  _mark;
  union _metadata {
    wideKlassOop    _klass;
    narrowOop       _compressed_klass;
  } _metadata;
```
> 抛开友元类VMStructs以及用于内存屏障的_bs，就只剩下了两个成员变量，_mark和_metadata。

- Java类在整个生命周期中，会涉及到线程状态、并发所、GC分代信息等内部表示，这些标识全部打在_mark变量上。
- _metadata用于表示元数据，也就是前面所说的数据结构，起到指针的作用，指向Java类的数据结构被解析后所保存的内存位置。前面所说的instanceOop内部会有一个指针指向instanceKlass便是_metadata，klass是Java类的对等题。

### 大端与小端
- Little-Endian小端：低位字节排放在内存的低位地址
- Big-Endian大端：高位字节排放在内存的低位地址。
- 网络字节序，TCP/IP协议中使用的字节序通常称为网络字节序，使用Big-Endian

大小端产生的本质原因是由于CPU的标准不同，假定一个双字节的寄存器，如果他的左边为高字节端，那么0x0102在寄存器所存的数值就是0x0102，而如果左边是低字节的话就成了0x0201，并最终在内存的存储顺序上反应出来。

#### 大小端问题的避免
- 在单机上使用同一种编程语言读写变量，文件，进行网络通信，所读取的字节与所写入字节序相同。
- 在分布式场景下，使用同一种编程语言，在同样大小端模式的不同机器上所读写的文件与网络信息，字节序相同。

> 然而Java的跨平台特性让它不一定能满足上面所说的，那么JVM是怎么处理大小端问题呢。
- Java所输出的字节信息全部是大端模式，对于小端模式的硬件平台，JVM定义了响应的转换接口。
