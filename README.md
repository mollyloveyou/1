# pintos1

task：

1.修改相应的.c;.h完成Alarm Clock 部分（具体是：timer.h;timer.c;thread.h）；
2.修改代码完成Priority Scheduling 部分（具体是：synch.h;synch.c;thread.h;thread.c）；
3.修改相应的代码，完成Advanced Scheduler 部分（具体是:timer.c;synch.c;thread.h;thread.c）。

ideas：

Step 1. Timer sleep的方案是读代码理解，并且找到原有代码不足，再修改调试的。

Step 2.Priority Schedule控制新进入就绪队列的线程不破坏原有的优先顺序、解决线程的优先级的变化所导致的顺序变化的解决方案，与Step1中一样。而第三部分，优先捐赠部分，则是通过的对测试样例的注释的阅读与总结实现的。

Step3. MLFQS 主要是在pintos上看对这部分的解释，几个计算公式的定义以及fixed_point.h需要实现的功能的介绍等等，慢慢做出来的。

file：
     								threads/thread.h
Line7-9
//ADDED for mlfqs
#include "fixed_point.h"
//END ADDED
Line107-109
    //ADDED for timer_sleep
    int64_t ticks_blocked;              /* To store how long it should be blocked. */
    //END ADDED
Line111-115
    //ADDED for donate
    int base_priority;                  /* To store the base priority. */
    struct list locks;                  /* Locks the thread is holding. */
    struct lock *lock_waiting;          /* The lock that the thread is waiting for. */
    //END ADDED
Line117-120
    //ADDED for mlfqs. */
    int nice;                           /* Niceness. */
    fixed_t recent_cpu;                 /* Recent cpu. */
    //END ADDED
Line159-161
//ADDED for timer_sleep
void thread_blocked_check (struct thread *t, void *aux UNUSED);
//END ADDED
Line163-165
//ADDED for priority
bool thread_priority_cmp (const struct list_elem *a, const struct list_elem *b, void *aux UNUSED);
//END ADDED
Line167-172
//ADDED for donate
void thread_hold_lock (struct lock *);
void thread_remove_lock (struct lock *);
void thread_priority_update (struct thread *);
void thread_priority_donate (struct thread *);
//END ADDED
Line174-179
//ADDED for mlfqs
void thread_mlfqs_increase_one_recent_cpu (void);
void thread_mlfqs_update_recent_cpu_and_load_avg (void);
void thread_mlfqs_update_priority (void);
void thread_mlfqs_update_one_priority (struct thread *);
//END ADDED
      														threads/thread.c
Line62-64
//ADDED for mlfqs
fixed_t load_avg;
//END ADDED
Line122-124
  //ADDED for mlfqs
  load_avg = FP_CONVERT(0);
  //END ADDED
Line214-216
  //ADDED for timer_sleep
  t->ticks_blocked = 0;
  //END ADDED
Line222-225
  //ADDED for priority
  if ( thread_current ()->priority < priority )
    thread_yield();
  //END ADDED
Line262-265
  //MODIFIED for priority
  /* list_push_back (&ready_list, &t->elem); */
  list_insert_ordered (&ready_list, &t->elem, (list_less_func *) &thread_priority_cmp, NULL);
  //END MODIFIED
Line336-339
    //MODIFIED for priority
    /* list_push_back (&ready_list, &cur->elem); */
    list_insert_ordered (&ready_list, &cur->elem, (list_less_func *) &thread_priority_cmp, NULL);
    //END MODIFIED
Line366-390
  //MODIFIED for donate
  if ( thread_mlfqs )   return;
  enum intr_level old_level = intr_disable ();
  struct thread *cur = thread_current ();
  int old_priority = cur->priority;
  cur->base_priority = new_priority;
  if ( list_empty(&cur->locks) || new_priority > old_priority )
  {
    cur->priority = new_priority;
    //ADDED for priority
    thread_yield();
    //END ADDED
  }
  intr_set_level (old_level);
  //END MODIFIED
Line395-399
  //ADDED for mlfqs
  thread_current ()->nice = nice;
  thread_mlfqs_update_one_priority (thread_current ());
  thread_yield ();
  //END ADDED
Line406-410
  /* Not yet implemented. */
  //MODIFIED for mlfqs
  /* return 0; */
  return thread_current ()->nice;
  //END MODIFIED
Line417-421
  /* Not yet implemented. */
  //MODIFIED for mlfqs
  /* return 0; */
  return FP_CONVERT_ROUND (FP_MUL_INT (load_avg, 100));
  //END MODIFIED
Line428-432
  /* Not yet implemented. */
  //MODIFIED for mlfqs
  /* return 0; */
  return FP_CONVERT_ROUND (FP_MUL_INT (thread_current ()->recent_cpu, 100));
  //END MODIFIED
Line520-523
  //MODIFIED for priority schedule
  /* list_push_back (&all_list, &t->allelem); */
  list_insert_ordered (&all_list, &t->allelem, (list_less_func *) &thread_priority_cmp, NULL);
  //END MODIFIED
Line525-529
  //ADDED for donate
  t->base_priority = priority;
  list_init (&t->locks);
  t->lock_waiting = NULL;
  //END ADDED
Line531-534
  //ADDED for mlfqs
  t->nice = 0;
  t->recent_cpu = FP_CONVERT(0);
  //END ADDED
Line651-663
//ADDED for timer_sleep
/* Check the blocked thread */
void
thread_blocked_check (struct thread *t, void *aux UNUSED)
{
  if ( t->status == THREAD_BLOCKED && t->ticks_blocked > 0 )
  {
    t->ticks_blocked--;
    if ( t->ticks_blocked == 0 )
      thread_unblock(t);
  }
}
//END ADDED
Line665-672
//ADDED for priority
/* To compare the priority of two threads */
bool
thread_priority_cmp (const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)
{
  return list_entry(a, struct thread, elem)->priority > list_entry(b, struct thread, elem)->priority;
}
//END ADDED
Line674-792
//ADDED for donate
/* Let the current thread held the lock. */
void
thread_hold_lock (struct lock *lock)
{
  enum intr_level old_level = intr_disable ();
  list_insert_ordered(&thread_current ()->locks, &lock->elem, lock_priority_cmp, NULL);
  if ( lock->max_priority > thread_current ()->priority )
  {
    thread_current ()->priority = lock->max_priority;
    thread_yield ();
  }
  intr_set_level (old_level);
}

/* Remove a lock of a thread. */
void
thread_remove_lock (struct lock *lock)
{
    enum intr_level old_level = intr_disable ();
    list_remove (&lock->elem);
    thread_priority_update (thread_current ());
    intr_set_level (old_level);
}

/* Updating. */
void
thread_priority_update (struct thread *t)
{
  enum intr_level old_level = intr_disable ();
  int max_priority = t->base_priority, lock_priority;
  if ( !list_empty (&t->locks) )
  {
    list_sort (&t->locks, lock_priority_cmp, NULL);
    lock_priority = list_entry (list_front (&t->locks), struct lock, elem)->max_priority;
    if ( lock_priority > max_priority )
        max_priority = lock_priority;
  }
  t->priority = max_priority;
  intr_set_level (old_level);
}

/* To donate a priority to thread t, and rearrange the list. */
void
thread_priority_donate (struct thread *t)
{
  enum intr_level old_level = intr_disable ();
  thread_priority_update (t);
  if ( t->status == THREAD_READY )
  {
    list_remove (&t->elem);
    list_insert_ordered (&ready_list, &t->elem, thread_priority_cmp, NULL);
  }
  intr_set_level (old_level);
}
//END ADDED

//ADDED for mlfqs
void
thread_mlfqs_increase_one_recent_cpu ()
{
  ASSERT ( thread_mlfqs );
  ASSERT ( intr_context () );

  struct thread *cur = thread_current ();
  if ( cur == idle_thread ) return;
  cur->recent_cpu = FP_ADD_INT (cur->recent_cpu, 1);
}

void
thread_mlfqs_update_recent_cpu_and_load_avg ()
{
  ASSERT ( thread_mlfqs );
  ASSERT ( intr_context () );

  /* To update load_avg. */
  size_t ready_threads = list_size (&ready_list);
  if ( thread_current () != idle_thread )   ready_threads++;
  load_avg = FP_ADD (FP_DIV_INT (FP_MUL_INT (load_avg, 59), 60), FP_DIV_INT (FP_CONVERT (ready_threads), 60));

  /* To update recent_cpu. */
  struct thread *t;
  struct list_elem *e = list_begin (&all_list);
  for ( ; e != list_end (&all_list); e = list_next (e) )
  {
    t = list_entry (e, struct thread, allelem);
    if ( t != idle_thread )
      t->recent_cpu = FP_ADD_INT (FP_MUL (FP_DIV (FP_MUL_INT (load_avg, 2), FP_ADD_INT (FP_MUL_INT (load_avg, 2), 1)), t->recent_cpu), t->nice);
  }
}

void
thread_mlfqs_update_priority ()
{
  ASSERT ( thread_mlfqs );
  ASSERT ( intr_context() );

  struct thread *t;
  struct list_elem *e = list_begin(&all_list);
  for ( ; e != list_end (&all_list); e = list_next (e) )
  {
    t = list_entry (e, struct thread, allelem);
    thread_mlfqs_update_one_priority (t);
  }
}

void
thread_mlfqs_update_one_priority (struct thread *t)
{
  if ( t == idle_thread )   return;

  ASSERT ( thread_mlfqs );
  ASSERT ( t != idle_thread );

  t->priority = FP_CONVERT_INT (FP_SUB_INT (FP_SUB (FP_CONVERT (PRI_MAX), FP_DIV_INT (t->recent_cpu, 4)), 2 * t->nice));
  t->priority = t->priority < PRI_MIN ? PRI_MIN : t->priority;
  t->priority = t->priority > PRI_MAX ? PRI_MAX : t->priority;
}
//END ADDED
      														threads/synch.h
Line26-29
    //ADDED for donate
    struct list_elem elem;      /* List element for priority donation. */
    int max_priority;           /* Max priority among the acquiring threads. */
    //END ADDED
Line56-59
//ADDED for donate
bool lock_priority_cmp (const struct list_elem *a, const struct list_elem *b, void *aux);
bool cond_sema_priority_cmp (const struct list_elem *, const struct list_elem *, void *aux UNUSED);
//END ADDED
      														threads/synch.c
Line71-74
    //MODIFIED for donate
    /* list_push_back (&sema->waiters, &thread_current ()->elem); */
    list_insert_ordered(&sema->waiters, &thread_current ()->elem, thread_priority_cmp, NULL);
    //END MODIFIED
Line121-123
    //ADDED for donate
    list_sort (&sema->waiters, thread_priority_cmp, NULL);
    //END ADDED
Line128-130
  //ADDED for donate
  thread_yield();
  //END ADDED
Line206-209
  //ADDED for donate
  struct thread *cur = thread_current();
  struct lock *l;
  //END ADDED
Line215-228
  //ADDED for donate
  //To recursively doing priority donate.
  if ( lock->holder != NULL && !thread_mlfqs )
  {
    cur->lock_waiting = lock;
    l = lock;
    while ( l && cur->priority > l->max_priority )
    {
      l->max_priority = cur->priority;
      thread_priority_donate (l->holder);
      l = l->holder->lock_waiting;
    }
  }
  //END ADDED
Line230-240
  //ADDED for donate
  enum intr_level old_level;
  old_level = intr_disable();
  cur = thread_current ();
  if ( !thread_mlfqs )
  {
    cur->lock_waiting = NULL;
    lock->max_priority = cur->priority;
    thread_hold_lock(lock);
  }
  //END ADDED
Line242-244
  //ADDED for donate
  intr_set_level (old_level);
  //END ADDED
      													threads/fixed_point.h
Line1-31
#ifndef THREADS_FIXED_POINT_H
#define THREADS_FIXED_POINT_H

/* Basic definition. */
typedef int fixed_t;
/* Used for fractional part. */
#define FP_SHIFT_AMOUNT 16
/* Convert to fixed-point value. */
#define FP_CONVERT(A) ((fixed_t)(A << FP_SHIFT_AMOUNT))
/* Get the integer part of a fixed-point value. */
#define FP_CONVERT_INT(A) (A >> FP_SHIFT_AMOUNT)
/* Get the rounded integer part. */
#define FP_CONVERT_ROUND(A) (A >= 0 ? ((A + (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT) :\
 ((A - (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT))
/* Two fixed-point addition. */
#define FP_ADD(A, B) (A + B)
/* Two fixed-point subtraction. */
#define FP_SUB(A, B) (A - B)
/* A fixed-point and an int addition. */
#define FP_ADD_INT(A, B) (A + (B << FP_SHIFT_AMOUNT))
/* A fixed-point and an int subtraction. */
#define FP_SUB_INT(A, B) (A - (B << FP_SHIFT_AMOUNT))
/* Two fixed-point multiplication. */
#define FP_MUL(A, B) ((fixed_t)((((int64_t) A) * B) >> FP_SHIFT_AMOUNT))
/* Two fixed-point division. */
#define FP_DIV(A, B) ((fixed_t)((((int64_t) A) << FP_SHIFT_AMOUNT) / B))
/* A fixed-point and an int multiplication. */
#define FP_MUL_INT(A, B) (A * B)
/* A fixed-point and an int division. */
#define FP_DIV_INT(A, B) (A / B)
#endif // THREADS_FIXED_POINT_H
      														devices/timer.h
Line101-114
  //MODIFIED 
  /* int64_t start = timer_ticks (); */
  if ( ticks <= 0 ) return;

  ASSERT (intr_get_level () == INTR_ON);
  /* while (timer_elapsed (start) < ticks)
    thread_yield (); */

  enum intr_level old_level = intr_disable();
  struct thread *cur = thread_current();
  cur->ticks_blocked = ticks;
  thread_block();
  intr_set_level(old_level);
  //END MODIFIED
Line193-195
  //ADDED for timer_sleep
  thread_foreach (thread_blocked_check, NULL);
  //END ADDED
Line197-210
  //ADDED for mlfqs
  if ( thread_mlfqs )
  {
    thread_mlfqs_increase_one_recent_cpu ();
    if ( ticks % TIMER_FREQ == 0 )
    {
      thread_mlfqs_update_recent_cpu_and_load_avg ();
    }
    if ( ticks % 4 == 0 )
    {
thread_mlfqs_update_priority ();
    }
  }
  //END ADDED



