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

