# Java栈帧
- entry_point例程
- 局部变量表创建的机制
- 堆栈与栈帧的概念
- JVM栈帧创建的详细过程
- slot大小到底是多大
- slot复用
- 操作数栈复用与深度

## entry_point例程生成
entry_point创建链路.png
TemplateInterpreter::initialize()函数：

```
void TemplateInterpreter::initialize() {
  if (_code != NULL) return;
  // assertions
  assert((int)Bytecodes::number_of_codes <= (int)DispatchTable::length,
         "dispatch table too small");

  AbstractInterpreter::initialize();

  TemplateTable::initialize();

  // generate interpreter
  { ResourceMark rm;
    TraceTime timer("Interpreter generation", TraceStartupTime);
    int code_size = InterpreterCodeSize;
    NOT_PRODUCT(code_size *= 4;)  // debug uses extra interpreter code space
    _code = new StubQueue(new InterpreterCodeletInterface, code_size, NULL,
                          "Interpreter");
    InterpreterGenerator g(_code);
    if (PrintInterpreter) print();
  }

  // initialize dispatch table
  _active_table = _normal_table;
}
```
> InterpreterGenerator g(_code);创建解释器生成器的实例，在Hotspot内部有三种解释器，分别是字节码解释器、C++解释器和模板解释器。

- 字节码解释器：逐条解释翻译字节码指令
- 模板解释器：将字节码指令直接翻译成机器指令，比较高效

InterpreterGenerator的构造函数：

```
InterpreterGenerator::InterpreterGenerator(StubQueue* code)
 : TemplateInterpreterGenerator(code) {
   generate_all(); // down here so it can be "virtual"
}
```
> generate_all是产生一块解释器运行时所需要的各种例程及入口，对于模板解释器而言就是生成好的机器指令。generate_all中就包含普通Java函数调用所对应的entry_point的入口：

```
void TemplateInterpreterGenerator::generate_all() {
  //省略部分代码

  { CodeletMark cm(_masm, "return entry points");
    for (int i = 0; i < Interpreter::number_of_return_entries; i++) {
      Interpreter::_return_entry[i] =
        EntryPoint(
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(atos, i),
          generate_return_entry_for(itos, i),
          generate_return_entry_for(ltos, i),
          generate_return_entry_for(ftos, i),
          generate_return_entry_for(dtos, i),
          generate_return_entry_for(vtos, i)
        );
    }
  }

  { CodeletMark cm(_masm, "earlyret entry points");
    Interpreter::_earlyret_entry =
      EntryPoint(
        generate_earlyret_entry_for(btos),
        generate_earlyret_entry_for(ztos),
        generate_earlyret_entry_for(ctos),
        generate_earlyret_entry_for(stos),
        generate_earlyret_entry_for(atos),
        generate_earlyret_entry_for(itos),
        generate_earlyret_entry_for(ltos),
        generate_earlyret_entry_for(ftos),
        generate_earlyret_entry_for(dtos),
        generate_earlyret_entry_for(vtos)
      );
  }

  //省略部分代码

#define method_entry(kind)                                                                    \
  { CodeletMark cm(_masm, "method entry point (kind = " #kind ")");                    \
    Interpreter::_entry_table[Interpreter::kind] = generate_method_entry(Interpreter::kind);  \
  }

  // all non-native method kinds
  method_entry(zerolocals)
  method_entry(zerolocals_synchronized)
  method_entry(empty)
  method_entry(accessor)
  method_entry(abstract)
  method_entry(method_handle)
  method_entry(java_lang_math_sin  )
  method_entry(java_lang_math_cos  )
  method_entry(java_lang_math_tan  )
  method_entry(java_lang_math_abs  )
  method_entry(java_lang_math_sqrt )
  method_entry(java_lang_math_log  )
  method_entry(java_lang_math_log10)
  method_entry(java_lang_ref_reference_get)

  // all native method kinds (must be one contiguous block)
  Interpreter::_native_entry_begin = Interpreter::code()->code_end();
  method_entry(native)
  method_entry(native_synchronized)
  Interpreter::_native_entry_end = Interpreter::code()->code_end();

#undef method_entry

  // Bytecodes
  set_entry_points_for_all_bytes();
  set_safepoints_for_all_bytes();
}
```
> 这个方法将生成模板解释器对应的各种模板例程的机器指令，并保存入口地址，它的上半段定义了一些重要的逻辑入口，同时会生成其对应的机器指令，而从#define method_entry()开始则定义了一系列方法入口。

> 当JVM调用Java函数时，例如Java类的构造函数、类成员方法、静态方法、虚方法等，就会从不同的入口进去，在CallStub例程中进入不同的函数入口。对于正常的java方法调用（包括主函数），其所对应的entry_point一般都是zerolocals或zerolocals_synchronized，如果方法加了同步关键字synchronize，则其entry_point是zerolocals_synchronized。调用method_entry(zerolocals)就相当于执行了一下逻辑：

```
Interpreter::_entry_table[Interpreter::zerolocals] = generate_method_entry(Interpreter::zerolocals);
```
> 这个逻辑执行完后，JVM会为zerolocals生成本地机器指令，同时将这串机器指令的首地址保存到Interpreter::_entry_table数组中。为zerolocals方法入口生成机器指令的函数如下：

```
address AbstractInterpreterGenerator::generate_method_entry(AbstractInterpreter::MethodKind kind) {
  // determine code generation flags
  bool synchronized = false;
  address entry_point = NULL;

  switch (kind) {
    case Interpreter::zerolocals             :                                                                             break;
    case Interpreter::zerolocals_synchronized: synchronized = true;                                                        break;
    case Interpreter::native                 : entry_point = ((InterpreterGenerator*)this)->generate_native_entry(false);  break;
    case Interpreter::native_synchronized    : entry_point = ((InterpreterGenerator*)this)->generate_native_entry(true);   break;
    case Interpreter::empty                  : entry_point = ((InterpreterGenerator*)this)->generate_empty_entry();        break;
    case Interpreter::accessor               : entry_point = ((InterpreterGenerator*)this)->generate_accessor_entry();     break;
    case Interpreter::abstract               : entry_point = ((InterpreterGenerator*)this)->generate_abstract_entry();     break;
    case Interpreter::method_handle          : entry_point = ((InterpreterGenerator*)this)->generate_method_handle_entry(); break;

    case Interpreter::java_lang_math_sin     : // fall thru
    case Interpreter::java_lang_math_cos     : // fall thru
    case Interpreter::java_lang_math_tan     : // fall thru
    case Interpreter::java_lang_math_abs     : // fall thru
    case Interpreter::java_lang_math_log     : // fall thru
    case Interpreter::java_lang_math_log10   : // fall thru
    case Interpreter::java_lang_math_sqrt    : entry_point = ((InterpreterGenerator*)this)->generate_math_entry(kind);     break;
    case Interpreter::java_lang_ref_reference_get
                                             : entry_point = ((InterpreterGenerator*)this)->generate_Reference_get_entry(); break;
    default                                  : ShouldNotReachHere();                                                       break;
  }

  if (entry_point) return entry_point;

  return ((InterpreterGenerator*)this)->generate_normal_entry(synchronized);

}
```
当入参类型是zerolocals时直接执行最后一句，generate_normal_entry就开始为zerolocals生成本地机器指令：

```
address InterpreterGenerator::generate_normal_entry(bool synchronized) {
  // determine code generation flags
  bool inc_counter  = UseCompiler || CountCompiledCalls;

  // rbx,: methodOop
  // rsi: sender sp
  address entry_point = __ pc();

  //定义寄存器变量
  const Address size_of_parameters(rbx, methodOopDesc::size_of_parameters_offset());
  const Address size_of_locals    (rbx, methodOopDesc::size_of_locals_offset());
  const Address invocation_counter(rbx, methodOopDesc::invocation_counter_offset() + InvocationCounter::counter_offset());
  const Address access_flags      (rbx, methodOopDesc::access_flags_offset());

  // get parameter size (always needed)入参数量
  __ load_unsigned_short(rcx, size_of_parameters);

  // rbx,: methodOop
  // rcx: size of parameters

  // rsi: sender_sp (could differ from sp+wordSize if we were called via c2i )
    
  __ load_unsigned_short(rdx, size_of_locals);       // get size of locals in words
  __ subl(rdx, rcx);                                // rdx = no. of additional locals

  // see if we've got enough room on the stack for locals plus overhead.
  generate_stack_overflow_check();

  // get return address获取返回地址
  __ pop(rax);

  // compute beginning of parameters (rdi)计算第一个入参在堆栈中的地址
  __ lea(rdi, Address(rsp, rcx, Interpreter::stackElementScale(), -wordSize));

  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals 为局部变量slot（不包含入参）分配堆栈空间，初始化为0
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }

  if (inc_counter) __ movl(rcx, invocation_counter);  // (pre-)fetch invocation count
  // initialize fixed part of activation frame 创建栈帧
  generate_fixed_frame(false);

  // make sure method is not native & not abstract
#ifdef ASSERT
  __ movl(rax, access_flags);
  {
    Label L;
    __ testl(rax, JVM_ACC_NATIVE);
    __ jcc(Assembler::zero, L);
    __ stop("tried to execute native method as non-native");
    __ bind(L);
  }
  { Label L;
    __ testl(rax, JVM_ACC_ABSTRACT);
    __ jcc(Assembler::zero, L);
    __ stop("tried to execute abstract method in interpreter");
    __ bind(L);
  }
#endif

  // Since at this point in the method invocation the exception handler
  // would try to exit the monitor of synchronized methods which hasn't
  // been entered yet, we set the thread local variable
  // _do_not_unlock_if_synchronized to true. The remove_activation will
  // check this flag.

  __ get_thread(rax);
  const Address do_not_unlock_if_synchronized(rax,
        in_bytes(JavaThread::do_not_unlock_if_synchronized_offset()));
  __ movbool(do_not_unlock_if_synchronized, true);

  // increment invocation count & check for overflow 引用计数
  Label invocation_counter_overflow;
  Label profile_method;
  Label profile_method_continue;
  if (inc_counter) {
    generate_counter_incr(&invocation_counter_overflow, &profile_method, &profile_method_continue);
    if (ProfileInterpreter) {
      __ bind(profile_method_continue);
    }
  }
  Label continue_after_compile;
  __ bind(continue_after_compile);

  bang_stack_shadow_pages(false);

  // reset the _do_not_unlock_if_synchronized flag
  __ get_thread(rax);
  __ movbool(do_not_unlock_if_synchronized, false);

  // check for synchronized methods
  // Must happen AFTER invocation_counter check and stack overflow check,
  // so method is not locked if overflows.
  //
  if (synchronized) {
    // Allocate monitor and lock method
    lock_method();
  } else {
    // no synchronization necessary
#ifdef ASSERT       //开始执行Java方法第一条字节码
      { Label L;
        __ movl(rax, access_flags);
        __ testl(rax, JVM_ACC_SYNCHRONIZED);
        __ jcc(Assembler::zero, L);
        __ stop("method needs synchronization");
        __ bind(L);
      }
#endif
  }

  // start execution
#ifdef ASSERT
  { Label L;
     const Address monitor_block_top (rbp,
                 frame::interpreter_frame_monitor_block_top_offset * wordSize);
    __ movptr(rax, monitor_block_top);
    __ cmpptr(rax, rsp);
    __ jcc(Assembler::equal, L);
    __ stop("broken stack frame setup in interpreter");
    __ bind(L);
  }
#endif

  // jvmti support
  __ notify_method_entry();
    //进入Java方法第一条字节码
  __ dispatch_next(vtos);

  // invocation counter overflow
  if (inc_counter) {
    if (ProfileInterpreter) {
      // We have decided to profile this method in the interpreter
      __ bind(profile_method);
      __ call_VM(noreg, CAST_FROM_FN_PTR(address, InterpreterRuntime::profile_method));
      __ set_method_data_pointer_for_bcp();
      __ get_method(rbx);
      __ jmp(profile_method_continue);
    }
    // Handle overflow of counter and compile method
    __ bind(invocation_counter_overflow);
    generate_counter_overflow(&continue_after_compile);
  }

  return entry_point;
}
```
> 这个函数会在JVM启动过程中调用，执行完成后，会向JVM的代码缓存区写入对应的本地机器指令，当JVM调用一个特定的Java方法时，会根据Java方法所对应的类型找到对应的函数入口，并执行这段预先生成好的机器指令。

## 局部变量表创建
### constMethod的内存布局

对于JDK6，JVM内部通过偏移量为Java函数定义了以下规则：
- method对象的constMethod指针紧跟在methodOop对象头后面，也即constMethod的偏移量固定
- constMethod内部存储Java函数所对应的字节码指令的位置相对于constMethod起始位的偏移量固定
- method对象内部存储Java函数的参数数量、局部变量数量的参数的偏移量是固定的。

JDK6的method机器相关属性的偏移量：
methodOopDesc内存布局.png

> max_locals与size_of_parameters这两个参数相对于methodOop对象头的偏移量分别是38与36，JVM将会基于此计算局部变量表的大小。JDK8中，由于这两个参数恒定不变，因此将其当做只读属性，移到了constMethod对象中。

### 局部变量表空间计算
> 局部变量表包含Java方法的所有入参和方法内部所声明的全部局部变量。
举例说明
```
public class A {
    public void add(int x,int y){
        int z = x + y;
    }
}
```
使用javap -verbose分析：

```
  public void add(int, int);
    descriptor: (II)V
    flags: ACC_PUBLIC
    Code:
      stack=2, locals=4, args_size=3
         0: iload_1
         1: iload_2
         2: iadd
         3: istore_3
         4: return
      LineNumberTable:
        line 5: 0
        line 6: 4
```
> 可以看到add方法的局部变量表的最大容量是4，入参数量是3，这是因为add方法是类的成员方法，因此会有一个隐藏的第一个入参this。3个入参加上内部定义的局部变量z构成了局部变量表。


```
0: iload_1
1: iload_2
这两条字节码指令分别将局部变量表的第一个槽位x和第二个槽位y的数据推送至表达式栈栈顶，
（槽位标号从0开始，很显然第0号槽位是this）
istore_3
这条字节码指令将计算结果板寸到局部变量表的第三个槽位，正是定义的局部变量z
```
> 对于本例，x和y这两个入参在调用方法调用add时便已经分配完毕，因此JVM只需为z分配堆栈空间，而z所需要的堆栈空间大小就是编译期间计算出的局部变量表的大小减去入参数量。entry_point隔出了这种方法：

```
 const Address size_of_parameters(rbx, methodOopDesc::size_of_parameters_offset());
  const Address size_of_locals    (rbx, methodOopDesc::size_of_locals_offset());
  const Address invocation_counter(rbx, methodOopDesc::invocation_counter_offset() + InvocationCounter::counter_offset());
  const Address access_flags      (rbx, methodOopDesc::access_flags_offset());

  // get parameter size (always needed)
  __ load_unsigned_short(rcx, size_of_parameters);

  // rbx,: methodOop
  // rcx: size of parameters

  // rsi: sender_sp (could differ from sp+wordSize if we were called via c2i )

  __ load_unsigned_short(rdx, size_of_locals);       // get size of locals in words
  __ subl(rdx, rcx);                                // rdx = no. of additional locals
```
这段代码最终生成的机器指令是：
```
movl 0x26(%ebx),%ecx
movl 0x24(%ebx),%edx
sub %ecx,%edx
```

### 初始化局部变量区
在CallStub执行call %eax指令前，物理寄存器中所保存的重要信息如下：
寄存器名|指向|
---|---
edx|parameters首地址
ecx|Java函数入参数量
ebx|指向Java函数，即Java函数所对应的的method对象
esi|CallStub栈顶

在entry_point历程中，分配堆栈空间时进行push操作：

```
// get return address获取返回地址
  __ pop(rax);

  // compute beginning of parameters (rdi)计算第一个入参在堆栈中的地址
  __ lea(rdi, Address(rsp, rcx, Interpreter::stackElementScale(), -wordSize)); 
  // rdx - # of additional locals
  // allocate space for locals
  // explicitly initialize locals 为局部变量slot（不包含入参）分配堆栈空间，初始化为0
  {
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }
```
这段逻辑执行了三件事：
- 将栈顶的返回值暂存到rax寄存器
- 获取Java函数第一个入参在堆栈的位置
- 为局部变量表分配堆栈空间

#### 1 暂存返回地址
> JVM为了实现操作数栈与局部变量表的复用，要将接下来被调用的Java方法分配局部变量的堆栈空间与被调用的Java方法的入参区域连城一块，而中间又间隔了return address，所以要将他暂存起来（pop %eax）

#### 2 获取Java函数第一个入参在堆栈的位置
> 这一步将第一个入参的位置保存在edi寄存器中，因为JVM是基于局部变量表的起始位置做偏移的。

#### 3 为局部变量表分配堆栈空间

```
{
    Label exit, loop;
    __ testl(rdx, rdx);
    __ jcc(Assembler::lessEqual, exit);               // do nothing if rdx <= 0
    __ bind(loop);
    __ push((int32_t)NULL_WORD);                      // initialize local variables
    __ decrement(rdx);                                // until everything initialized
    __ jcc(Assembler::greater, loop);
    __ bind(exit);
  }
```
> 这一段指令的逻辑是，先测试edx是否为0，如果为0则没有定义局部变量，直接跳过分配堆栈步骤，否则的话执行循环，通过__ push((int32_t)NULL_WORD);                      // initialize local variables向栈顶压入一个0，然后edx-1，一直进行到edx为0。这么做的原因是可以在分配堆栈空间的同时，将堆栈清零。

## 堆栈与栈帧
> 为局部变量分配完堆栈空间后，接着就要构建Java方法所对应的栈帧中的固定部分（fixed frame）。在JVM内部，每个Java方法对应一个栈帧，这个栈帧包括局部变量表、操作数栈、常量池缓存指针、返回地址等。对于一个Java方法栈帧，其首位分别是局部变量表和操作数栈，而中间的部分则是其他重要数据，正是这部分的数据结构是固定不变的。

### 栈帧是什么
> 一个方法的栈帧就是这个方法对应的堆栈。

- 堆栈是什么
> 函数内不会定义局部变量，这些变量最终要在内存中占用一定的空间。由于这些变量都位于同一个函数，自然地就要将这些变量合起来当做一个整体，将其内存空间分配到一起，这样有利于变量空间的整体内存申请和释放。所以“栈帧”就是一个容器，而堆栈就是栈帧的容器。
- 堆栈有什么用
> 可以节省空间，并有利于CPU的寻址，如果不采用线性布局的堆栈空间，被调用函数的堆栈空间地址保存在内存中，而访问内存的效率要比寄存器直接传递数据的效率低得多。