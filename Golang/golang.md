- [GoLang](#golang)
  - [GMP模型](#gmp模型)

# GoLang

## GMP模型

G是GoRoutine的缩写，包含函数执行指令和一些参数，比如任务对象、线程上下文切换、必须的寄存器的字段保护和恢复（field protection and field recovery required registers）等。

M是一个线程(thread)或者机器(machine)的缩写，所有的线程共享一个线程栈，如果你没有给线程栈分配内存，系统会自动分配内存。一个M中，包括一个指向g的指针，一个sp指针和一个pc指针，分别用户现场保护和现场恢复。当线程栈被指定后，G.stack = M.stack，且M的PC寄存器会指向G提供的函数，然后执行该函数。

P是一个抽象的概念，不代表一个真正的CPU核心。P会创建或者唤醒一个系统线程去执行队列中的任务。？？？P和M需要绑定一个计算单元？？？