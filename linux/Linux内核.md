## 汇编指令

通用寄存器：（E代表32位，R代表64位）

EAX 累加器（Accumulator）

EBX 基地址寄存器（Base Register）

ECX 计数寄存器（Count Register）

EDX 数据寄存器（Data Register）



ESP 堆栈顶指针（Stack Pointer）

EBP 堆栈基（底）址针（Bse Pointer）

内存执行：MOV，PUSH，POP

b、w、l、q分别代表8位、16位、32位、64位

%eax ，%开头是一个寄存器



call 0x12345 （函数调用）相当于两条汇编指令

- pushl %eip(*)
- movl $0x12345, %eip(*)

是指把eip寄存器上的内容保存到堆栈中，然后把$0x12345放入当前来执行，随后使用ret指令

ret 相当于

- popl %eip(*)

注意：(*)代表伪指令，不能被直接修改



计算器三个法宝：存储程序计算机、函数调用堆栈、中断；
操作系统两把宝剑：中断上下文切换、进程上下文切换