# Linux-get_number_of_context_switches

## Problem
寫一個system call，讓process可以使用他得到scheduler對其進行context switch的次數。

## 程式作法
在include/linux/sched.h的task_struct中用兩個counter紀錄context switch發生的次數:
1. nvcsw: Number of Voluntary Context Switches
2. nivcsw: Number of InVoluntary Context Switches
```c=
switch_count = &prev->nivcsw;
if (!preempt && prev->state) {
    ...
    // 若非preempt 則為Voluntary Context Switches
    switch_count = &prev->nvcsw;
}
rq->nr_switches++;
rq->curr = next;
++*switch_count;

trace_sched_switch(preempt, prev, next);

rq = context_switch(rq, prev, next, &rf);
```
scheduler會決定目前是要由哪個process是被CPU執行。(在kernel/sched/core.c中的[schedule()](https://elixir.bootlin.com/linux/v4.15.1/source/kernel/sched/core.c#L3427))
trace路徑:schedule()->__schedule()->context_switch()->[switch_to](https://elixir.free-electrons.com/linux/v3.9/source/arch/x86/include/asm/switch_to.h#L31)

switch_to是一個macro，傳入的參數有三個(prev, next, last)，分別是process自己本身、要轉交控制權給下一個的process、最後將控制權還給一開始的process。

當中
```c=
movl %%esp, %[prev_sp]\n\t
```
把esp指到拿到CPU使用權的process的kernel mode stack頂端的位置
```c=
movl %[next_sp],%%esp\n\t
```
這時將控制權交還給原process

### 執行結果
![](https://i.imgur.com/3dGnznq.png)

Process狀態圖:
![](https://i.imgur.com/Laxqhn2.png)

Linux系統中是由fork()建立新的process，process建立後，狀態會是TASK_RUNNING，指的是被放在task queue/run queue（OS叫 ready queue）中的process。
需經過context switch後，才會變成是正在執行中的process(current)。當current正在執行時，可能會被priority更高的process搶奪執行權，此時process會回到ready queue。
