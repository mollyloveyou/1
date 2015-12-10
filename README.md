# pintos1

task：

1.修改相应的.c;.h完成Alarm Clock 部分（具体是：timer.h;timer.c;thread.h）；
2.修改代码完成Priority Scheduling 部分（具体是：synch.h;synch.c;thread.h;thread.c）；
3.修改相应的代码，完成Advanced Scheduler 部分（具体是:timer.c;synch.c;thread.h;thread.c）。

ideas：

Step 1. Timer sleep的方案是读代码理解，并且找到原有代码不足，再修改调试的。

Step 2.Priority Schedule控制新进入就绪队列的线程不破坏原有的优先顺序、解决线程的优先级的变化所导致的顺序变化的解决方案，与Step1中一样。而第三部分，优先捐赠部分，则是通过的对测试样例的注释的阅读与总结实现的。

Step3. MLFQS 主要是在pintos上看对这部分的解释，几个计算公式的定义以及fixed_point.h需要实现的功能的介绍等等，慢慢做出来的。


