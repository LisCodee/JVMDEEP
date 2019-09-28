# Java执行引擎工作原理：方法调用
- 方法调用如何实现
- 函数指针和指针函数
- CallStub源码详解

## 1 方法调用如何实现
> 计算机核心三大功能：方法调用、取指、运算
### 1.1 真实机器如何实现方法调用

- 参数入栈。有几个参数就把几个参数入栈，此时入的是调用者自己的栈
- 代码指针（eip）入栈。以便物理机器执行完调用函数之后返回继续执行原指令
- 调用函数的栈基址入栈，为物理机器从被调用者函数返回做准备
- 为调用方法分配栈空间，每个函数都有自己的栈空间。

```
// 一个进行求和运算的汇编程序
main:
    //保存调用者栈基地址，并为main()函数分配新栈空间
    pushl   %ebp
    movl    %esp, %ebp
    subl    $32,  %esp          //分配新栈空间，一共32字节

    //初始化两个操作数，一个是5，一个是3
    movl    $5,   20(%esp)
    movl    $3,   24(%esp)

    //将5和3压栈（参数入栈）
    movl    $24(%esp), %eax
    movl    %eax, 4(%esp)
    movl    20(%esp),  %eax
    movl    %eax, (%esp)
    
    //调用add函数
    calladd
    movl    %eax, 28(%esp)      //得到add函数返回结果

    //返回
    movl    $0,   %eax
    leave
    ret

add:
    //保存调用者栈基地址，并为add()函数分配新栈空间
    pushl   %ebp
    mov     %esp, %ebp
    subl    $16,  %esp

    //获取入参
    movl    12(%ebp), %(eax)
    movl    8(%ebp),  %(edx)

    //执行运算
    addl    %edx,  %eax
    movl    %eax,  -4(%ebp)

    //返回
    movl    -4(%ebp), %eax
    leave
    ret
```
> 我们先来了解一下栈空间的分配，在Linux平台上，栈是向下增长的，也就是从内存的高地址向低地址增长，所以每次调用一个新的函数时，新函数的栈顶相对于调用者函数的栈顶，内存地址一定是低方位的。

栈模型.png

#### 完成add参数压栈后的main()函数堆栈布局

```
//初始化两个操作数，一个是5，一个是3
    movl    $5,   20(%esp)
    movl    $3,   24(%esp)

    //将5和3压栈（参数入栈）
    movl    $24(%esp), %eax
    movl    %eax, 4(%esp)
    movl    20(%esp),  %eax
    movl    %eax, (%esp)
```

完成add参数复制栈布局.png

- 返回值约定保存在eax寄存器中
- ebp：栈基地址
- esp：栈顶地址
- 相对寻址：28(%esp)相对于栈顶向上偏移28字节
- 方法内的局部变量分配在靠近栈底位置，而传递的参数分配在靠近栈顶的位置

#### 调用add函数时的函数堆栈布局

调用add的堆栈布局.png

- 在调用函数之前，会自动向栈顶压如eip，以便调用结束后可正常执行原程序
- 执行函数调用时，需要手动将ebp入栈

#### 物理机器执行函数调用的主要步骤
- 保存调用者栈基址，当前IP寄存器入栈
- 调用函数时，在x86平台上，参数从右到左依次入栈
- 一个方法所分配的栈空间大小，取决于该方法内部的局部变量空间、为被调用者所传递的入参大小
- 被调用者在接收入参时，从8(%ebp)处开始，往上逐个获得每一个入参参数
- 被调用者将返回的结果保存到eax寄存器中，调用者从该寄存器中获取返回值。


### 1.2 C语言函数调用


```
//一个简单的带参数求和函数调用
#include<stdio.h>
int add(int a, int b);
int main() {
	int a = 5;
	int b = 3;
	int c = add(a, b);
	return 0;
}
int add(int a,int b) {
	int z = 1 + 2;
	return z;
}
```


```
//main函数反汇编的代码
int main() {
//参数压栈、分配空间
002D1760  push        ebp       
002D1761  mov         ebp,esp  
002D1763  sub         esp,0E4h 
//以下部分代码不需要注意
002D1769  push        ebx  
002D176A  push        esi  
002D176B  push        edi  
002D176C  lea         edi,[ebp-0E4h]  
002D1772  mov         ecx,39h  
002D1777  mov         eax,0CCCCCCCCh  
002D177C  rep stos    dword ptr es:[edi]  
002D177E  mov         ecx,offset _F08B5E04_JVM1@cpp (02DC003h)  
002D1783  call        @__CheckForDebuggerJustMyCode@4 (02D120Dh)  

//main函数正式代码部分
	int a = 5;
002D1788  mov         dword ptr [a],5  
	int b = 3;
002D178F  mov         dword ptr [b],3  
	int c = add(a, b);
002D1796  mov         eax,dword ptr [b]  
	int c = add(a, b);
002D1799  push        eax  
002D179A  mov         ecx,dword ptr [a]  
002D179D  push        ecx  
002D179E  call        add (02D1172h)  
002D17A3  add         esp,8  
002D17A6  mov         dword ptr [c],eax  
	return 0;
002D17A9  xor         eax,eax  
}
//add函数汇编代码
int add(int a,int b) {
//参数压栈、分配空间
002D16F0  push        ebp  
002D16F1  mov         ebp,esp  
002D16F3  sub         esp,0CCh  
//一下部分代码不需要注意
002D16F9  push        ebx  
002D16FA  push        esi  
002D16FB  push        edi  
002D16FC  lea         edi,[ebp-0CCh]  
002D1702  mov         ecx,33h  
002D1707  mov         eax,0CCCCCCCCh  
002D170C  rep stos    dword ptr es:[edi]  
002D170E  mov         ecx,offset _F08B5E04_JVM1@cpp (02DC003h)  
002D1713  call        @__CheckForDebuggerJustMyCode@4 (02D120Dh)  

//add函数正式代码部分
	int z = 1 + 2;
002D1718  mov         dword ptr [z],3  
	return z;
002D171F  mov         eax,dword ptr [z]  
}
```
- 其实这就是物理机器函数调用是的步骤

#### 有参数传递场景下的C程序函数调用机制
- 压栈
- 参数传递顺序
- 读取入参

### JVM函数调用机制
```
//一个简单的求和函数
package cn.leishida;

public class Test {
	public static void main(String[] args) {
		add(5,8);
	}
	public static int add(int a,int b) {
		int c = a+b;
		int d = c + 9;
		return d;
	}
}
```

```
//编译成字节码的内容
Classfile /G:/workspace/JVM/src/cn/leishida/Test.class
  Compiled from "Test.java"
public class cn.leishida.Test
  minor version: 0
  major version: 52
  flags: ACC_PUBLIC, ACC_SUPER
Constant pool:
   #1 = Methodref          #4.#15         // java/lang/Object."<init>":()V
   #2 = Methodref          #3.#16         // cn/leishida/Test.add:(II)I
   #3 = Class              #17            // cn/leishida/Test
   #4 = Class              #18            // java/lang/Object
   #5 = Utf8               <init>
   #6 = Utf8               ()V
   #7 = Utf8               Code
   #8 = Utf8               LineNumberTable
   #9 = Utf8               main
  #10 = Utf8               ([Ljava/lang/String;)V
  #11 = Utf8               add
  #12 = Utf8               (II)I
  #13 = Utf8               SourceFile
  #14 = Utf8               Test.java
  #15 = NameAndType        #5:#6          // "<init>":()V
  #16 = NameAndType        #11:#12        // add:(II)I
  #17 = Utf8               cn/leishida/Test
  #18 = Utf8               java/lang/Object
{
  public cn.leishida.Test();
    descriptor: ()V
    flags: ACC_PUBLIC
    Code:
      stack=1, locals=1, args_size=1
         0: aload_0
         1: invokespecial #1                  // Method java/lang/Object."<init>":()V
         4: return
      LineNumberTable:
        line 3: 0

  public static void main(java.lang.String[]);
    descriptor: ([Ljava/lang/String;)V
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=1, args_size=1
         0: iconst_5
         1: bipush        8
         3: invokestatic  #2                  // Method add:(II)I
         6: pop
         7: return
      LineNumberTable:
        line 5: 0
        line 6: 7

  public static int add(int, int);
    descriptor: (II)I
    flags: ACC_PUBLIC, ACC_STATIC
    Code:
      stack=2, locals=4, args_size=2
         0: iload_0
         1: iload_1
         2: iadd
         3: istore_2
         4: iload_2
         5: bipush        9
         7: iadd
         8: istore_3
         9: iload_3
        10: ireturn
      LineNumberTable:
        line 8: 0
        line 9: 4
        line 10: 9
}
SourceFile: "Test.java"
```
> 现在我们还不用了解这些字节码的具体含义和作用，但我们知道机器只认识机器码的，所以JVM一定存在一个边界，边界的一边是C，一边是机器码。换言之，C可以直接调用汇编命令。

#### 指针函数与函数指针

##### 函数指针：指向一个函数的首地址（指针符号连同函数名一起被括住）
void (*fun)(int a, int b);  //声明一个函数指针
void add(int x,int y);      //声明一个函数原型
fun = add                   //将add函数首地址赋值给fun


```
#include<stdio.h>
int (*addPointer)(int a, int b);
int add(int a, int b);

int main(){
	int a = 5;
	int b = 3;
	addPointer = add;//初始化函数指针，使其指向add首地址
	int c = addPointer(a, b);
	printf("c = %d\n", c);
	return 0;
}
int add(int a, int b) {
	int c = a + b;
	return c;
}
```


##### 指针函数：返回值是一个指针类型(指针符号没有被括号括住)
int *fun(int a,int b);

```
#include<stdio.h>
int* add(int a, int b);
int main() {
	int a = 5;
	int b = 3;
	int* c = add(a, b);
	printf("c = %p\n", c);
	return 0;
}

int* add(int a, int b) {
	int c = a + b;
	return &c;
}
```

### 1.3 JVM内部分水岭call_stub
> call_stub在JVM内部具有举足轻重的意义，这个函数指针正式JVM内部C程序与机器指令的分水岭。JVM在调用这个函数之前，主要执行C程序，而JVM通过这个函数指针调用目标函数之后，就直接进入了机器指令的领域。


```
//call_stub指针原型：
#define CAST_TO_FN_PTR(func_type,value) ((func_type),(castable_address(value)))
static CallStub call_stub()                              
{ return CAST_TO_FN_PTR(CallStub, _call_stub_entry); }
```
宏替换后：

```
static CallStub call_stub()                              
{ return (CallStub))(castable_address(_call_stub_entry)); }
```
> 在执行函数对应的第一条字节码指令之前，必须经过call_stub函数指针进入对应例程，然后在目标例程中出发对Java主函数第一条字节码指令的调用。

这段代码之前的(CallStub)实际上就是转型，我们先来看一下CallStub的定义

```
// Calls to Java
  typedef void (*CallStub)(
    address   link,
    intptr_t* result,
    BasicType result_type,
    methodOopDesc* method,
    address   entry_point,
    intptr_t* parameters,
    int       size_of_parameters,
    TRAPS
  );
```
由这段定义我们可以知道，CallStub定义了一个函数指针，他指向一个返回类型是void的，有8个入参的函数。
那么这个函数又是怎么调用的呢？

```
void JavaCalls::call(JavaValue* result, methodHandle method, JavaCallArguments* args, TRAPS) {
//省略部分代码
// do call
  { JavaCallWrapper link(method, receiver, result, CHECK);
    { HandleMark hm(thread);  // HandleMark used by HandleMarkCleaner

      StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );

      result = link.result();  // circumvent MS C++ 5.0 compiler bug (result is clobbered across call)
      // Preserve oop return value across possible gc points
      if (oop_result_flag) {
        thread->set_vm_result((oop) result->get_jobject());
      }
    }
  } // Exit JavaCallWrapper (can block - potential return oop must be preserved)
    //省略部分代码...
}
```
可以看到，JVM直接调用了call_stub并传入了8个入参，可是我们再看call_stub定义的时候是没有入参的，这是为什么呢。
```
static CallStub call_stub()                              
{ return (CallStub))(castable_address(_call_stub_entry)); }
//我们将这一行展开
{
    //声明一个指针变量
    CallStub functionPointer;
    //调用castable_address
    address returnAddress = castable_address(_call_stub_entry);
    functionPointer = (CallStub)returnAddress;
    
    //返回CallStub类型的变量
    return functionPointer;
}
```
实际上，JVM调用的是call_stub()函数所返回的函数指针变量，而CallStub在定义时就规定了这种函数指针类型有8个入参，所以JVM在调用这种类型的函数指针前也必须传入8个入参。

```
typedef uintptr_t     address_word; //unsigned integer which will hold a //pointer， 
//except for some implementations of a C++，linkage pointer to function. 
//Should never need one of those to be placed in thisype anyway.
inline address_word  castable_address(address x)              
{ return address_word(x) ; }
```
而uintptr_t实际上就是无符号整数，我们接着分析call_stub函数，实际上就是下面
```
static CallStub call_stub()                              
{ return (CallStub))(unsigned int(_call_stub_entry)); }
```
所以这个函数的逻辑总体包含两部
- 将_call_stub_entry转化为Unsigned int类型
- 将_call_stub_entry转换后的类型在转换为CallStub这一函数指针类型。

> 可是这个函数指针究竟指向哪里呢？其实在JVM初始化过程中就将_call_stub_entry这一变量指向了某个内存地址，在x86 32位Linux平台上，JVM初始化过程存在这样一条链路，这条链路从JVM的main函数开始，调用到init_global()这个全局数据初始化模块，最后在调用到StubRoutines这个例程生成模块，最终在stubGenerator_x86_32.cpp:generate_initial()函数中对_call_stub_entry变量初始化。如下图


```
StubRoutines::_call_stub_entry=generate_call_stub(StubRoutines::_call_stub_return_address);
```
这一句可以说是JVM最核心的功能，不过在分析之前我们先要弄清楚call_stub的入参：

```
StubRoutines::call_stub()(
        (address)&link,
        // (intptr_t*)&(result->_value), // see NOTE above (compiler problem)
        result_val_address,          // see NOTE above (compiler problem)
        result_type,
        method(),
        entry_point,
        args->parameters(),
        args->size_of_parameters(),
        CHECK
      );
```
参数名|含义
---|---|
link|这是一个连接器
result_val_address|函数返回值地址
result_type|函数返回值类型
method()|JVM内部所表示的Java对象
entry_point|JVM调用Java方法例程入口，经过这段例程才能跳转到Java字节码对应的机器指令运行
args->parameters()|Java方法的入参集合
args->size_of_parameters()|Java方法的入参数量
CHECK|当前线程对象

- 连接器link：

> 连接器link的所属类型是JavaCallWrapper，实际上是在Java函数的调用者与被调用者之间打起了一座桥梁，通过这个桥梁，我们可以实现堆栈追踪，得到整个方法的调用链路。在Java函数调用时，link指针江北保存到当前方法的堆栈中。
- method（）

> 是当前Java方法在JVM内部的表示对象，每一个Java方法在被JVM加载时，都会在内部为这个Java方法建立函数模型，保存一份Java方法的全部原始描述信息。包括：
    * Java函数的名称，所属类
    * Java函数的入参信息，包括入参类型、入参参数名、入参数量、入参顺序等
    * Java函数编译后的字节码信息，包括对应的字节码指令、所占用的总字节数等
    * Java函数的注解信息
    * Java函数的继承信息
    * Java函数的返回信息

- entry_point：
>是继_call_stub_entry例程之后的又一个最主要的例程入口
> JVM调用Java函数时通过_call_stub_entry所指向的函数地址最终调用到Java函数。而通过_call_stub_entry所指向的函数调用Java函数之前，必须要先经过entry_point例程。在这个例程里面会真正从method（）对象上拿到Java函数编译后的字节码，通过entry_point可以得到Java函数所对应的第一个字节码指令，然后开始调用。
- parameters：
> 描述Java函数的入参信息，在JVM真正调用Java函数的第一个字节码指令之前，JVM会在CallStub（）函数中解析Java函数的入参，然后为Java函数分配堆栈，并将Java函数的入参逐个入栈
- size_of_parameters:
> Java函数的入参个数。JVM根据这个值计算Java堆栈空间大小。

### _call_stub_entry例程
_call_stub_entry的值是generate_call_stub()函数赋值的，现在我们来分析一下。

```
  address generate_call_stub(address& return_address) {
    StubCodeMark mark(this, "StubRoutines", "call_stub");
    address start = __ pc();

    // stub code parameters / addresses
    assert(frame::entry_frame_call_wrapper_offset == 2, "adjust this code");
    bool  sse_save = false;
    const Address rsp_after_call(rbp, -4 * wordSize); // same as in generate_catch_exception()!
    const int     locals_count_in_bytes  (4*wordSize);
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
    const Address result        (rbp,  3 * wordSize);
    const Address result_type   (rbp,  4 * wordSize);
    const Address method        (rbp,  5 * wordSize);
    const Address entry_point   (rbp,  6 * wordSize);
    const Address parameters    (rbp,  7 * wordSize);
    const Address parameter_size(rbp,  8 * wordSize);
    const Address thread        (rbp,  9 * wordSize); // same as in generate_catch_exception()!
    sse_save =  UseSSE > 0;

    // stub code
    __ enter();
    __ movptr(rcx, parameter_size);              // parameter counter
    __ shlptr(rcx, Interpreter::logStackElementSize); // convert parameter count to bytes
    __ addptr(rcx, locals_count_in_bytes);       // reserve space for register saves
    __ subptr(rsp, rcx);
    __ andptr(rsp, -(StackAlignmentInBytes));    // Align stack

    // save rdi, rsi, & rbx, according to C calling conventions
    __ movptr(saved_rdi, rdi);
    __ movptr(saved_rsi, rsi);
    __ movptr(saved_rbx, rbx);
    // save and initialize %mxcsr
    if (sse_save) {
      Label skip_ldmx;
      __ stmxcsr(mxcsr_save);
      __ movl(rax, mxcsr_save);
      __ andl(rax, MXCSR_MASK);    // Only check control and mask bits
      ExternalAddress mxcsr_std(StubRoutines::addr_mxcsr_std());
      __ cmp32(rax, mxcsr_std);
      __ jcc(Assembler::equal, skip_ldmx);
      __ ldmxcsr(mxcsr_std);
      __ bind(skip_ldmx);
    }

    // make sure the control word is correct.
    __ fldcw(ExternalAddress(StubRoutines::addr_fpu_cntrl_wrd_std()));

#ifdef ASSERT
    // make sure we have no pending exceptions
    { Label L;
      __ movptr(rcx, thread);
      __ cmpptr(Address(rcx, Thread::pending_exception_offset()), (int32_t)NULL_WORD);
      __ jcc(Assembler::equal, L);
      __ stop("StubRoutines::call_stub: entered with pending exception");
      __ bind(L);
    }
#endif

    // pass parameters if any
    BLOCK_COMMENT("pass parameters if any");
    Label parameters_done;
    __ movl(rcx, parameter_size);  // parameter counter
    __ testl(rcx, rcx);
    __ jcc(Assembler::zero, parameters_done);

    // parameter passing loop

    Label loop;
    // Copy Java parameters in reverse order (receiver last)
    // Note that the argument order is inverted in the process
    // source is rdx[rcx: N-1..0]
    // dest   is rsp[rbx: 0..N-1]

    __ movptr(rdx, parameters);          // parameter pointer
    __ xorptr(rbx, rbx);

    __ BIND(loop);

    // get parameter
    __ movptr(rax, Address(rdx, rcx, Interpreter::stackElementScale(), -wordSize));
    __ movptr(Address(rsp, rbx, Interpreter::stackElementScale(),
                    Interpreter::expr_offset_in_bytes(0)), rax);          // store parameter
    __ increment(rbx);
    __ decrement(rcx);
    __ jcc(Assembler::notZero, loop);

    // call Java function
    __ BIND(parameters_done);
    __ movptr(rbx, method);           // get methodOop
    __ movptr(rax, entry_point);      // get entry_point
    __ mov(rsi, rsp);                 // set sender sp
    BLOCK_COMMENT("call Java function");
    __ call(rax);

    BLOCK_COMMENT("call_stub_return_address:");
    return_address = __ pc();

#ifdef COMPILER2
    {
      Label L_skip;
      if (UseSSE >= 2) {
        __ verify_FPU(0, "call_stub_return");
      } else {
        for (int i = 1; i < 8; i++) {
          __ ffree(i);
        }

        // UseSSE <= 1 so double result should be left on TOS
        __ movl(rsi, result_type);
        __ cmpl(rsi, T_DOUBLE);
        __ jcc(Assembler::equal, L_skip);
        if (UseSSE == 0) {
          // UseSSE == 0 so float result should be left on TOS
          __ cmpl(rsi, T_FLOAT);
          __ jcc(Assembler::equal, L_skip);
        }
        __ ffree(0);
      }
      __ BIND(L_skip);
    }
#endif // COMPILER2

    // store result depending on type
    // (everything that is not T_LONG, T_FLOAT or T_DOUBLE is treated as T_INT)
    __ movptr(rdi, result);
    Label is_long, is_float, is_double, exit;
    __ movl(rsi, result_type);
    __ cmpl(rsi, T_LONG);
    __ jcc(Assembler::equal, is_long);
    __ cmpl(rsi, T_FLOAT);
    __ jcc(Assembler::equal, is_float);
    __ cmpl(rsi, T_DOUBLE);
    __ jcc(Assembler::equal, is_double);

    // handle T_INT case
    __ movl(Address(rdi, 0), rax);
    __ BIND(exit);

    // check that FPU stack is empty
    __ verify_FPU(0, "generate_call_stub");

    // pop parameters
    __ lea(rsp, rsp_after_call);

    // restore %mxcsr
    if (sse_save) {
      __ ldmxcsr(mxcsr_save);
    }

    // restore rdi, rsi and rbx,
    __ movptr(rbx, saved_rbx);
    __ movptr(rsi, saved_rsi);
    __ movptr(rdi, saved_rdi);
    __ addptr(rsp, 4*wordSize);

    // return
    __ pop(rbp);
    __ ret(0);

    // handle return types different from T_INT
    __ BIND(is_long);
    __ movl(Address(rdi, 0 * wordSize), rax);
    __ movl(Address(rdi, 1 * wordSize), rdx);
    __ jmp(exit);

    __ BIND(is_float);
    // interpreter uses xmm0 for return values
    if (UseSSE >= 1) {
      __ movflt(Address(rdi, 0), xmm0);
    } else {
      __ fstp_s(Address(rdi, 0));
    }
    __ jmp(exit);

    __ BIND(is_double);
    // interpreter uses xmm0 for return values
    if (UseSSE >= 2) {
      __ movdbl(Address(rdi, 0), xmm0);
    } else {
      __ fstp_d(Address(rdi, 0));
    }
    __ jmp(exit);

    return start;
  }
```

> 这段代码的主要作用就是生成机器码，使用C语言动态生成，弄懂这段指令的逻辑，对理解JVM的字节码执行引擎至关重要。

- 1 pc()函数

```
 address start = __ pc();
```
pc()函数定义：

```
address pc() const{
    return _code_pos;
}
```
> JVM启动过程中，会生成许多例程，例如函数调用、字节码例程、异常处理、函数返回等。每一个例程的开始都是address start = __pc();（JVM的例程都会写入JVM堆内存）。偏移量_code_pos指向上一个例程的最后字节的位置，新的例程开始时会先保存这个起始位置，以便JVM通过CallStub这个函数指针执行这段动态生成的机器指令。

- 2 定义入参

```
// stub code parameters / addresses
    assert(frame::entry_frame_call_wrapper_offset == 2, "adjust this code");
    bool  sse_save = false;
    const Address rsp_after_call(rbp, -4 * wordSize); // same as in generate_catch_exception()!
    const int     locals_count_in_bytes  (4*wordSize);
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
    const Address result        (rbp,  3 * wordSize);
    const Address result_type   (rbp,  4 * wordSize);
    const Address method        (rbp,  5 * wordSize);
    const Address entry_point   (rbp,  6 * wordSize);
    const Address parameters    (rbp,  7 * wordSize);
    const Address parameter_size(rbp,  8 * wordSize);
    const Address thread        (rbp,  9 * wordSize); // same as in generate_catch_exception()!
    sse_save =  UseSSE > 0;
```
> 一个函数的堆栈空间大体上可以分为3部分：
> - 堆栈变量区：
保存方法的局部变量，或者对数据的地址引用（指针），如果没有局部变量，则不分配本区
>- 入参区域
如果当前方法调用了其他方法，并且传了参数，那么这些入参会保存在调用者堆栈中，就是所谓的压栈
>- ip和bp区：一个是代码段寄存器，一个是堆栈栈基急促安琪，一个用于恢复调用者方法的代码位置，一个用于恢复调用方法的堆栈位置。这部分前面有提到过

入参|位置
---|---|
(address)&link:连接器|8(%ebp)
result_val_address:返回地址|12(%ebp)
result_type:返回类型|16(%ebp)
method():方法内部对象|20(%ebp)
entry_point:Java方法调用入口例程|24(%ebp)
parameters():Java方法的入参|28(%ebp)
size_of_parametres():Java方法的入参数量|32(%ebp)
CHECK:当前线程|36(%ebp)

如果按N=4字节来寻址，那我们可以这样写
入参|位置|C++类标记
---|---|---|
(address)&link:连接器|2N(%ebp)|Address link(rbp, 2N)
result_val_address:返回地址|3N(%ebp)|Address link(rbp, 3N)
result_type:返回类型|4N(%ebp)|Address link(rbp, 4N)
method():方法内部对象|5N(%ebp)|Address link(rbp, 5N)
entry_point:Java方法调用入口例程|6N(%ebp)|Address link(rbp, 6N)
parameters():Java方法的入参|7N(%ebp)|Address link(rbp, 7N)
size_of_parametres():Java方法的入参数量|8N(%ebp)|Address link(rbp, 8N)
CHECK:当前线程|9N(%ebp)|Address link(rbp, 9N)

如此一来，generate_call_stub函数前面十几行的类声明就不难理解了

```
    const Address mxcsr_save    (rbp, -4 * wordSize);
    const Address saved_rbx     (rbp, -3 * wordSize);
    const Address saved_rsi     (rbp, -2 * wordSize);
    const Address saved_rdi     (rbp, -1 * wordSize);
```
这四个相对于rbp偏移量为负，说明这四个参数位置在CalStub（）函数的堆栈内部，而这四个变量用于保存调用者的信息，后面会详细分析。

- 3 CallStub：保存调用者堆栈

这部分逻辑从这行代码开始

```
 // stub code
    __ enter();
```
> 这行代码在不同的硬件平台对应不同的机器指令。

enter()的定义：
```
push(rbp);
mov(rbp,rsp);
```
很明显是把rbp压栈的操作，前面已经讲过了。

- 4 CallStub：动态分配堆栈
> JVM为了能够调用Java函数，需要在运行期指导一个Java函数的入参大小，然后动态计算出所需要的堆栈空间。由于物理机器不能识别Java程序，因此JVM必然要通过自己作为中间桥梁连接到Java程序，并让Java被调用的函数的堆栈能够计生在JVM的CallStub（）函数堆栈。
想要实现寄生在CallStub的函数堆栈中，我们就要对它的空间进行扩展，物理机器为扩展堆栈提供了简单指令：

```
sub operand,%esp
```
> 在JVM内部，一切Java对象实例及其成员变量和成员方法的访问，最终皆通过指针得以寻址，同理，JVM在传递Java函数参数时所传递的也只是Java入参对象实例的指针，而指针的宽度在特定的硬件平台上是一样的。


```
    __ movptr(rcx, parameter_size);              // parameter counter
    __ shlptr(rcx, Interpreter::logStackElementSize); // convert parameter count to bytes，将参数长度转化为字节，保存到rcx
    
    __ addptr(rcx, locals_count_in_bytes);       // reserve space for register saves，保存rdi,rsi,rbx,mxcsr四个寄存器的值
    __ subptr(rsp, rcx);    //扩展堆栈空间，rcx是之前计算出的参数的字节
    __ andptr(rsp, -(StackAlignmentInBytes));    // Align stack内存对齐
```
至此，JVM完成了动态堆栈内存分配！！

- 5 CallStub：调用者保存
> esi、edi、ebx属于调用者函数的私有数据，在发生函数调用之前，调用者函数必须将这些数据保存起来，以便被调用函数执行完毕从新回到调用者函数中时能够正常运行。JVM直接将其保存到了被调用者函数的堆栈中。

```
// save rdi, rsi, & rbx, according to C calling conventions
    __ movptr(saved_rdi, rdi);
    __ movptr(saved_rsi, rsi);
    __ movptr(saved_rbx, rbx);
    //...这部分是保存mxcsr的，属于Intel的SSE技术
```

- 6 CallStub：参数压栈
接下来JVM要做的事将即将被调用的Java函数入参复制到剩余的堆栈空间。
采取基址+变址
```
 // pass parameters if any
    BLOCK_COMMENT("pass parameters if any");
    Label parameters_done;
    __ movl(rcx, parameter_size);  // parameter counter参数数量，控制循环次数
    __ testl(rcx, rcx);          //判断参数数量是否为0，如果是，直接跳过参数处理
    __ jcc(Assembler::zero, parameters_done);

    // parameter passing loop

    Label loop;
    // Copy Java parameters in reverse order (receiver last)
    // Note that the argument order is inverted in the process
    // source is rdx[rcx: N-1..0]
    // dest   is rsp[rbx: 0..N-1]

    __ movptr(rdx, parameters);          // parameter pointer，第一个入参地址
    __ xorptr(rbx, rbx);

    __ BIND(loop);
    
```
此时物理寄存器状态：
寄存器名称|指向
---|---
edx|parameters首地址
ecx|Java函数入参数量
现在开始循环将Java函数参数压栈

```
// get parameter
    __ movptr(rax, Address(rdx, rcx, Interpreter::stackElementScale(), -wordSize));
    __ movptr(Address(rsp, rbx, Interpreter::stackElementScale(),
                    Interpreter::expr_offset_in_bytes(0)), rax);          // store parameter
    __ increment(rbx);
    __ decrement(rcx);
    __ jcc(Assembler::notZero, loop);
```
至此，Java函数的3个参数全部被压栈，离函数调用越来越近

### 7 CallStub：调用entry_point例程
> 前面经过调用者框架栈帧保存（栈基），堆栈动态扩展、现场保存、Java函数参数压栈这一系列处理，JVM终于为Java函数的调用演完前奏，而标志开始的则是entry_point例程。


```
  // call Java function
    __ BIND(parameters_done);
    __ movptr(rbx, method);           // get methodOop
    __ movptr(rax, entry_point);      // get entry_point
    __ mov(rsi, rsp);                 // set sender sp
    BLOCK_COMMENT("call Java function");
    __ call(rax);
```
到目前为止各个参数的位置
参数|保存位置
---|---
link|仍在堆栈中8(%ebp)
result_val_address|仍在堆栈中12(%ebp)
result_type|仍在堆栈中16(%ebp)
method|ebx寄存器
entry_point|eax寄存器
parameters|edx寄存器
size_of_parameters|ecx寄存器
CHECK|仍在堆栈中32(%ebp)

物理寄存器保存的重要信息
寄存器名|指向|
---|---
edx|parameters首地址
ecx|Java函数入参数量
ebx|指向Java函数，即Java函数所对应的的method对象
esi|CallStub栈顶
eax|entry_point例程入口
> 此时eax已经指向了entry_point例程入口，只需要call一下就可以赚到entry_point例程，去执行entry_point例程

 - 8 获取返回值
 > 调用完entry_point例程之后会有返回值，CallStub会获取返回值并继续处理。

```
// store result depending on type
    // (everything that is not T_LONG, T_FLOAT or T_DOUBLE is treated as T_INT)
    __ movptr(rdi, result);
    Label is_long, is_float, is_double, exit;
    __ movl(rsi, result_type);
```
> 上面的代码存储了结果和结果类型，JVM会将这两个值分别存进edi和esi这两个寄存器，这是基于约定的技术实现方式。

call_stub例程的讲解至此告一段落，置于JVM内部如何从C/C++程序完成Java函数的调用，会在之后讲解完entry_point例程后揭开真相。