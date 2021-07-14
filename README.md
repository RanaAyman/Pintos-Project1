# Pintos - P1: Threads

## About

PintOS is an open source instructional operating system kernel developed by Stanford University. PintOS provides complete documentation & modular projects to introduce students to the major concepts of operating systems development. The components of PintOS project is illustrated in the following figure.

![PintOS Components](resources/pintos_components.png "PintOS Components")

## Installation Guidelines

To install PintOS please use the guidelines listed at the [installation document](Installation.md).

## Project Documents

Source : [ProgrammersSought](https://www.programmersought.com/article/74771135829/).

The following is a list of the documents to explain and document the project and its requirements.

1. [PintOS Official Document](guides/PintOS&#32;Official&#32;Document.pdf): The main official documentation and project requirements compiled from PintOS repository provided by Stanford University.
2. [CSCI 350 - PintOS Guide](guides/CSCI&#32;350&#32;-&#32;Pintos&#32;Guide.pdf): A simple read for PintOS code and its requirement provided from [University of Southern California OS Course](http://bits.usc.edu/cs350/).
3. [The Pintos Instructional Operating System Kernel (PintOS Paper)](guides/The&#32;Pintos&#32;Instructional&#32;Operating&#32;System&#32;Kernel.pdf): A good reading for instructors.
4. [The Pintos Instructional Operating System Kernel (PintOS Presentation)](guides/SIGCSE2009-Pintos.pdf): A good reading for instructors.

## Anatomy of the Pintos kernel
Look for main in /threads/init.c and find the system entry function as follows:

```c
/* Pintos main program. */
int
main (void)
{
  char **argv;

  /* Clear BSS. */  
  bss_init ();

  /* Break command line into arguments and parse options. */
  argv = read_command_line ();
  argv = parse_options (argv);

  /* Initialize ourselves as a thread so we can use locks,
     then enable console locking. */
  thread_init ();
  console_init ();  

  /* Greet user. */
  printf ("Pintos booting with %'"PRIu32" kB RAM...\n",
          init_ram_pages * PGSIZE / 1024);

  /* Initialize memory system. */
  palloc_init (user_page_limit);
  malloc_init ();
  paging_init ();

  /* Segmentation. */
#ifdef USERPROG
  tss_init ();
  gdt_init ();
#endif

  /* Initialize interrupt handlers. */
  intr_init ();
  timer_init ();
  kbd_init ();
  input_init ();
#ifdef USERPROG
  exception_init ();
  syscall_init ();
#endif

  /* Start thread scheduler and enable interrupts. */
  thread_start ();
  serial_init_queue ();
  timer_calibrate ();

#ifdef FILESYS
  /* Initialize file system. */
  ide_init ();
  locate_block_devices ();
  filesys_init (format_filesys);
#endif

  printf ("Boot complete.\n");
  
  /* Run actions specified on kernel command line. */
  run_actions (argv);

  /* Finish up. */
  shutdown ();
  thread_exit ();
}
```

When the system starts, it will initialize the BSS, read and analyze the command line, initialize the main thread, terminal, memory, interrupt and clock, open the thread scheduling, open the interrupt, and finally exit after running the program.

BSS segments are usually used to store uninitialized programs or initialized to 0.Global variablewithStatic variablea piece of memory area. The feature is readable and writable, and the BSS segment is automatically cleared to 0 before the program is executed.

Executable programs include BSS segments,Data segment、Code segment(also known as text segment).

In order to understand the interrupt mechanism, you can first see in /device/timer.h:

```c
#define TIMER_FREQ 100
```
That is, 100 clock interruptions per second, the timing is 0.01 seconds. The operating system kernel obtains CPU time by interrupt.

Then look up the timer_init function in timer.c:
```c
/* Sets up the timer to interrupt TIMER_FREQ times per second,
   and registers the corresponding interrupt. */
void
timer_init (void) 
{
  pit_configure_channel (0, 2, TIMER_FREQ);
  intr_register_ext (0x20, timer_interrupt, "8254 Timer");
}

```
Where 0x20 is the interrupt vector number; timer_interrupt is the entry address of the interrupt handler, which is called whenever the clock is timed out (periodic call);
Timer" is the clock.

Find the timer_interrupt function:
```c
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{

  ticks++;
  thread_tick ();


}

```
The global variable ticks is incremented by 1 each time the clock arrives.

Then enter /threads, first look at interrupt.h:
```c
enum intr_level
{
	INTR_OFF,
	INTR_ON
};
```
Then look at the thread structure in thread.h:

```c
struct thread
  {
    /* Owned by thread.c. */
    tid_t tid;                          /* Thread identifier. */
    enum thread_status status;          /* Thread state. */
    char name[16];                      /* Name (for debugging purposes). */
    uint8_t *stack;                     /* Saved stack pointer. */
    int priority;                       /* Priority. */
    struct list_elem allelem;           /* List element for all threads list. */

    /* Shared between thread.c and synch.c. */
    struct list_elem elem;              /* List element. */

#ifdef USERPROG
    /* Owned by userprog/process.c. */
    uint32_t *pagedir;                  /* Page directory. */
#endif

    /* Owned by thread.c. */
    unsigned magic;                     /* Detects stack overflow. */

    int64_t ticks_blocked;              /* Record the time the thread has been blocked. */
    int base_priority;                  /* Base priority. */
    struct list locks;                  /* Locks that the thread is holding. */
    struct lock *lock_waiting;          /* The lock that the thread is waiting for. */
    int nice;                           /* Niceness. */
    fixed_t recent_cpu;                 /* Recent CPU. */
  };

```
And the state type of thread:
```c
/* States in a thread's life cycle. */
enum thread_status
  {
    THREAD_RUNNING,     /* Running thread. */
    THREAD_READY,       /* Not running but ready to run. */
    THREAD_BLOCKED,     /* Waiting for an event to trigger. */
    THREAD_DYING        /* About to be destroyed. */
  };

```

As seen in the main function, the function that runs the test is run_actions, looking for:



```c
/* Executes all of the actions specified in ARGV[]
   up to the null pointer sentinel. */
static void
run_actions (char **argv) 
{
  /* An action. */
  struct action 
    {
      char *name;                       /* Action name. */
      int argc;                         /* # of args, including action name. */
      void (*function) (char **argv);   /* Function to execute action. */
    };

  /* Table of supported actions. */
  static const struct action actions[] = 
    {
      {"run", 2, run_task},
#ifdef FILESYS
      {"ls", 1, fsutil_ls},
      {"cat", 2, fsutil_cat},
      {"rm", 2, fsutil_rm},
      {"extract", 1, fsutil_extract},
      {"append", 2, fsutil_append},
#endif
      {NULL, 0, NULL},
    };

  while (*argv != NULL)
    {
      const struct action *a;
      int i;

      /* Find action name. */
      for (a = actions; ; a++)
        if (a->name == NULL)
          PANIC ("unknown action `%s' (use -h for help)", *argv);
        else if (!strcmp (*argv, a->name))
          break;

      /* Check for required arguments. */
      for (i = 1; i < a->argc; i++)
        if (argv[i] == NULL)
          PANIC ("action `%s' requires %d argument(s)", *argv, a->argc - 1);

      /* Invoke action and advance. */
      a->function (argv);
      argv += a->argc;
    }
  
}
```
On line 311, you can see that the run command is the first element of the actions structure array, and the a on line 324 is the pointer to the array of structures, the address of the first element.

So, line 340 a->function(argv) runs the third argument of actions[0] with the argv argument, run_task[argv], where argv[0]”run”,argv[1]”alarm-multiple”。

Look for the run_task function:
```c
/* Runs the task specified in ARGV[1]. */
static void
run_task (char **argv)
{
  const char *task = argv[1];
  
  printf ("Executing '%s':\n", task);
#ifdef USERPROG
  process_wait (process_execute (task));
#else
  run_test (task);
#endif
  printf ("Execution of '%s' complete.\n", task);
}

```
Line 290 is equivalent to run_test("alarm-multiple");

Look for the run_test function in /tests/threads/test.c:
```c
/* Runs the test named NAME. */
void
run_test (const char *name) 
{
  const struct test *t;

  for (t = tests; t < tests + sizeof tests / sizeof *tests; t++)
    if (!strcmp (name, t->name))
      {
        test_name = name;
        msg ("begin");
        t->function ();
        msg ("end");
        return;
      }
  PANIC ("no test named \"%s\"", name);
}

```
This loops through the tests array, and the tests array is defined in the same file:
```c
static const struct test tests[] = 
  {
    {"alarm-single", test_alarm_single},
    {"alarm-multiple", test_alarm_multiple},
    {"alarm-simultaneous", test_alarm_simultaneous},
    {"alarm-priority", test_alarm_priority},
    {"alarm-zero", test_alarm_zero},
    {"alarm-negative", test_alarm_negative},
    {"priority-change", test_priority_change},
    {"priority-donate-one", test_priority_donate_one},
    {"priority-donate-multiple", test_priority_donate_multiple},
    {"priority-donate-multiple2", test_priority_donate_multiple2},
    {"priority-donate-nest", test_priority_donate_nest},
    {"priority-donate-sema", test_priority_donate_sema},
    {"priority-donate-lower", test_priority_donate_lower},
    {"priority-donate-chain", test_priority_donate_chain},
    {"priority-fifo", test_priority_fifo},
    {"priority-preempt", test_priority_preempt},
    {"priority-sema", test_priority_sema},
    {"priority-condvar", test_priority_condvar},
    {"mlfqs-load-1", test_mlfqs_load_1},
    {"mlfqs-load-60", test_mlfqs_load_60},
    {"mlfqs-load-avg", test_mlfqs_load_avg},
    {"mlfqs-recent-1", test_mlfqs_recent_1},
    {"mlfqs-fair-2", test_mlfqs_fair_2},
    {"mlfqs-fair-20", test_mlfqs_fair_20},
    {"mlfqs-nice-2", test_mlfqs_nice_2},
    {"mlfqs-nice-10", test_mlfqs_nice_10},
    {"mlfqs-block", test_mlfqs_block},
  };
```
Then line 56 t->function() is equivalent to test_alarm_multiple().

Look at alarm_wait.c in the same directory, you can see that the function calls test_sleep(5,
7), find the test_sleep function, the core code:

```c
static void
test_sleep (int thread_cnt, int iterations) 
{
  struct sleep_test test;
  struct sleep_thread *threads;
  int *output, *op;
  int product;
  int i;

  /* This test does not work with the MLFQS. */
  ASSERT (!thread_mlfqs);

  msg ("Creating %d threads to sleep %d times each.", thread_cnt, iterations);
  msg ("Thread 0 sleeps 10 ticks each time,");
  msg ("thread 1 sleeps 20 ticks each time, and so on.");
  msg ("If successful, product of iteration count and");
  msg ("sleep duration will appear in nondescending order.");

  /* Allocate memory. */
  threads = malloc (sizeof *threads * thread_cnt);
  output = malloc (sizeof *output * iterations * thread_cnt * 2);
  if (threads == NULL || output == NULL)
    PANIC ("couldn't allocate memory for test");

  /* Initialize test. */
  test.start = timer_ticks () + 100;
  test.iterations = iterations;
  lock_init (&test.output_lock);
  test.output_pos = output;

  /* Start threads. */
  ASSERT (output != NULL);
  for (i = 0; i < thread_cnt; i++)
    {
      struct sleep_thread *t = threads + i;
      char name[16];
      
      t->test = &test;
      t->id = i;
      t->duration = (i + 1) * 10;
      t->iterations = 0;

      snprintf (name, sizeof name, "thread %d", i);
      thread_create (name, PRI_DEFAULT, sleeper, t);
    }
  
  /* Wait long enough for all the threads to finish. */
  timer_sleep (100 + thread_cnt * iterations * 10 + 100);

  /* Acquire the output lock in case some rogue thread is still
     running. */
  lock_acquire (&test.output_lock);

  /* Print completion order. */
  product = 0;
  for (op = output; op < test.output_pos; op++) 
    {
      struct sleep_thread *t;
      int new_prod;

      ASSERT (*op >= 0 && *op < thread_cnt);
      t = threads + *op;

      new_prod = ++t->iterations * t->duration;
        
      msg ("thread %d: duration=%d, iteration=%d, product=%d",
           t->id, t->duration, t->iterations, new_prod);
      
      if (new_prod >= product)
        product = new_prod;
      else
        fail ("thread %d woke up out of order (%d > %d)!",
              t->id, product, new_prod);
    }

  /* Verify that we had the proper number of wakeups. */
  for (i = 0; i < thread_cnt; i++)
    if (threads[i].iterations != iterations)
      fail ("thread %d woke up %d times instead of %d",
            i, threads[i].iterations, iterations);
  
  lock_release (&test.output_lock);
  free (output);
  free (threads);
}

```
It can be seen that thread_cnt threads are created in the for loop, and 97th each thread calls timer_sleep for iteration times. So test_sleep(5,
7) Create 5 threads, each call sleep 7 times.

Go back to /device/timer.c and look for the timer_sleep function:
```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks)
{
  if (ticks <= 0)
  {
    return;
  }
  ASSERT (intr_get_level () == INTR_ON);
 while(timer_elapsed(start)<ticks)
 	thread_yield;
}
```
Timer_sleep checks whether the elapsed time reaches the parameter ticks by continuously polling. If it has not been reached, it calls the thread_yield function. When the ticks are reached, it will end hibernation.

Look for the thread_yield function in /threads/thread.c:
```c
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void
thread_yield (void)
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;

  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread)
     list_push_back (&ready_list, &cur->elem); 
   
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}
```
The schedule function is called in thread_yield to reschedule the thread.

Continue to find the schedule function:

```c
/* Schedules a new process.  At entry, interrupts must be off and
   the running process's state must have been changed from
   running to some other state.  This function finds another
   thread to run and switches to it.

   It's not safe to call printf() until thread_schedule_tail()
   has completed. */
static void
schedule (void)
{
  struct thread *cur = running_thread ();
  struct thread *next = next_thread_to_run ();
  struct thread *prev = NULL;

  ASSERT (intr_get_level () == INTR_OFF);
  ASSERT (cur->status != THREAD_RUNNING);
  ASSERT (is_thread (next));

  if (cur != next)
    prev = switch_threads (cur, next);
  thread_schedule_tail (prev);
}
```
After the schedule function is executed, the current thread will be put into the queue and the next thread will be scheduled.

Therefore, we know that when timer_sleep polls,
thread_yield will put the current thread into the ready queue through the schedule function, and schedule the next thread, and ensure that the interrupt is closed when the thread is scheduled.

## Pintos busy and other problem analysis and practice process
To analyze the timer_sleep function, first find the timer_ticks function it calls:

```c
/* Returns the number of timer ticks since the OS booted. */
int64_t
timer_ticks (void)
{
  enum intr_level old_level = intr_disable ();
  int64_t t = ticks;
  intr_set_level (old_level);
  return t;
}
```
Look for intr_disable and try to find out its return value:
```c
/* Disables interrupts and returns the previous interrupt status. */
enum intr_level
intr_disable (void) 
{
  enum intr_level old_level = intr_get_level ();

  /* Disable interrupts by clearing the interrupt flag.
     See [IA32-v2b] "CLI" and [IA32-v3a] 5.8.1 "Masking Maskable
     Hardware Interrupts". */
  asm volatile ("cli" : : : "memory");

  return old_level;
}
```
```c
/* Interrupts on or off? */
enum intr_level 
  {
    INTR_OFF,             /* Interrupts disabled. */
    INTR_ON               /* Interrupts enabled. */
  };
```
The function closes the terminal through assembly and returns the interrupt status before the interrupt is turned off.

In other words, enum intr_level old_level =
intr_disable(); closes the interrupt and saves the previous state, intr_set_level(old_level); restores the previous interrupt state. The middle of these two statements is the atomic operation.

According to the previous analysis of timer_sleep, we know that in thread_yield, if the current thread is not an idle thread, it is inserted into the tail of the queue, and the state becomes THREAD_READY. Therefore, the thread will continue to switch between the ready queue and the run queue, continuously consuming CPU resources, causing problems such as busyness.

So we consider blocking the thread, recording the remaining time ticks_blocked of the thread, and using the clock interrupt to detect the state of all threads, each time the corresponding ticks_blocked is decremented by one, and if it is 0, the corresponding thread is woken up.

Add members to thread.h
Add an initialization statement to thread_create of thread.c
To traverse all threads when the clock is interrupted, we use the thread_foreach function:

```c
/* Invoke function 'func' on all threads, passing along 'aux'.
   This function must be called with interrupts off. */
void
thread_foreach (thread_action_func *func, void *aux)
{
  struct list_elem *e;

  ASSERT (intr_get_level () == INTR_OFF);

  for (e = list_begin (&all_list); e != list_end (&all_list);
       e = list_next (e))
    {
      struct thread *t = list_entry (e, struct thread, allelem);
      func (t, aux);
    }
}
```
Use this function in the timer_interrupt function in timer.c:

```c
/* Timer interrupt handler. */
static void
timer_interrupt (struct intr_frame *args UNUSED)
{

  thread_foreach (blocked_thread_check, NULL);
    ticks++;
  thread_tick ();
}
```
Declare the checkInvoke function in thread.h, implemented in thread.c:


```c
/* Solution Code */
void
checkInvoke(struct thread *t, void *aux UNUSED)
{
  if (t->status == THREAD_BLOCKED && t->ticks_blocked > 0)
  {
    --t->ticks_blocked;
	if (t->ticks_blocked == 0)
	  thread_unblock(t);
  }
}
```

Finally modify the timer_sleep function:

```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks) 
{
  /*
  int64_t start = timer_ticks();
  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed(start) < ticks)
    thread_yield();
  */

  /* Solution Code */

  /* For alarm-negative && alarm-zero */


  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable();

  /* Blocks current thread for ticks */
  thread_current()->ticks_blocked = ticks;
  thread_block();

  intr_set_level(old_level);
}
```
That is to block the current thread ticks duration, this operation is atomic.

Try make
check, found that the alarm-negative and alarm-zero that should have passed did not pass, check the code of the two test programs, found that the parameters of timer_sleep are -100 and 0, respectively, and the requirement is The program does not crash. So we add conditional judgment:

```c
/* Sleeps for approximately TICKS timer ticks.  Interrupts must
   be turned on. */
void
timer_sleep (int64_t ticks) 
{
  /*
  int64_t start = timer_ticks();
  ASSERT (intr_get_level () == INTR_ON);
  while (timer_elapsed(start) < ticks)
    thread_yield();
  */

  /* Solution Code */

  /* For alarm-negative && alarm-zero */
  if (ticks <= 0) return;

  ASSERT (intr_get_level () == INTR_ON);
  enum intr_level old_level = intr_disable();

  /* Blocks current thread for ticks */
  thread_current()->ticks_blocked = ticks;
  thread_block();

  intr_set_level(old_level);
}
```

## Pintos priority scheduling problem analysis and practice process.
First of all, thread.h can see that there is a priority member in the thread structure, and there are:
```c
/* Thread priorities. */
#define PRI_MIN 0                       /* Lowest priority. */
#define PRI_DEFAULT 31                  /* Default priority. */
#define PRI_MAX 63                      /* Highest priority. */

```
In order to pass the alarm-priority, we only need to maintain the ready queue as a priority queue, sorting from high to low according to priority. In other words, you only need to ensure that the thread is inserted into the appropriate location when it is inserted into the ready queue.

After analyzing the pintos source code, we know that only three functions will insert the thread into the ready queue: init_thread,
thread_unblock, thread_yield。

Take thread_yield as an example:
```c

  old_level = intr_disable ();
  if (cur != idle_thread)
     list_push_back (&ready_list, &cur->elem); 
  


```
This is inserted directly into the end of the queue, so we can't use the list_push_back function directly.

Read the linked list implementation source code in /lib/kernel/list.h. We found some interesting functions:

```c
/* Operations on lists with ordered elements. */
void list_sort (struct list *,
                list_less_func *, void *aux);
void list_insert_ordered (struct list *, struct list_elem *,
                          list_less_func *, void *aux);
void list_unique (struct list *, struct list *duplicates,
                  list_less_func *, void *aux);

```
Where list_insert_ordered seems to be able to maintain the order when inserting, see its implementation:

```c
/* Inserts ELEM in the proper position in LIST, which must be
   sorted according to LESS given auxiliary data AUX.
   Runs in O(n) average case in the number of elements in LIST. */
void
list_insert_ordered (struct list *list, struct list_elem *elem,
                     list_less_func *less, void *aux)
{
  struct list_elem *e;

  ASSERT (list != NULL);
  ASSERT (elem != NULL);
  ASSERT (less != NULL);

  for (e = list_begin (list); e != list_end (list); e = list_next (e))
    if (less (elem, e, aux))
      break;
  return list_insert (e, elem);
}
```
As you can guess, we only need to implement a comparison function (declared first in thread.h):

```c
/* Solution Code */
bool thread_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED);
```
The usage of list_entry comes from list.h:

```c
/* Converts pointer to list element LIST_ELEM into a pointer to
   the structure that LIST_ELEM is embedded inside.  Supply the
   name of the outer structure STRUCT and the member name MEMBER
   of the list element.  See the big comment at the top of the
   file for an example. */
#define list_entry(LIST_ELEM, STRUCT, MEMBER)           \
        ((STRUCT *) ((uint8_t *) &(LIST_ELEM)->next     \
                     - offsetof (STRUCT, MEMBER.next)))
```
Then modify the thread_yield function:

```c
/* Yields the CPU.  The current thread is not put to sleep and
   may be scheduled again immediately at the scheduler's whim. */
void
thread_yield (void)
{
  struct thread *cur = thread_current ();
  enum intr_level old_level;

  ASSERT (!intr_context ());

  old_level = intr_disable ();
  if (cur != idle_thread)
    /* list_push_back (&ready_list, &cur->elem); */
    list_insert_ordered (&ready_list, &cur->elem, (list_less_func *) &thread_cmp_priority, NULL);
  cur->status = THREAD_READY;
  schedule ();
  intr_set_level (old_level);
}

```
Finally, replace the list_push_back function in init_thread and thread_unblock with list_insert_ordered and make similar changes.

Next, priority change and preemption scheduling are implemented, namely priority-change and priority-preempt. Looking at the code of the two tests first, I found that when changing the priority of a thread, the execution order of all threads will be affected. So we can only insert the thread priority into the ready queue when it changes, so that all threads can be reordered.

Similarly, when creating a thread, if the priority is higher than the current thread, the current thread will also be inserted into the ready queue.

So we modify the thread_set_priority in thread.c:
```c
void
thread_set_priority (int new_priority) 
{
  
  thread_current ()->priority = new_priority;
  thread_yield();
  ...

}
```
And, at the end of thread_create, it is determined whether the current thread is to be inserted into the ready queue according to the new thread priority.

```c

/*Add to run queue. */
  thread_unblock (t);

  /* Solution Code */
  /* Preempt the current thread */
  if (thread_current()->priority < priority)
    thread_yield();

  return tid;
}
```

Try make check and successfully passed both tests.

Finally, in order to prevent priority inversion, we need to implement priority donations, which will bring a lot of problems, as can be seen from the number of tests.

The tests to be passed this time include: priority-donate-one, priority-donate-multiple,
priority-donate-multiple2, priority-donate-nest, priority-donate-sema,
priority-donate-lower, priority-sema, priority-condvar,
There are a total of nine priority-donate-chain. We analyzed the code of the nine tests separately. Since the code is long, the code part is omitted, and only the results of the analysis are explained.

A new function lock_acquire appears in donate-one.
lock_release, meaning the acquisition and release of the lock. This test shows that if thread A acquires a lock and finds that the thread B with the lock has a lower priority than itself, it raises the priority of thread B; after thread B releases the lock, it also restores the original priority of thread B.

Two locks appear in donate-multiple and donate-multiple2, requiring that other threads consider the priority donation of the thread when restoring priority. This requires us to record all threads that have given priority to the thread.

There are three threads H, M, and L in the donate-nest with priority being high, medium, and low. When M is waiting for the lock of L, the priority of L is raised to M. When H acquires the lock of M, the priority of M and L is raised to H. This is equivalent to asking for priority donations to be recursively donated. So we also need to record which thread is waiting for which thread to release the lock.

Sema_up and sema_down appear in donate-sema, which corresponds to V operation and P operation. The adjustment of the semaphore was added, but it did not make the problem more complicated, because the essence of the lock is also the change of the semaphore.

Donate-lower modifies the priority of a thread that is preferentially donated. The thread priority is still the priority of the donation when it is modified, but the priority of the thread becomes the modified priority after the lock is released.

The two test logics of sema and condvar are very simple. The wait queues that require semaphores and the waiters queues of conditions are priority queues.

Donate-chain requires chained priority donation. If the thread is not donated after the lock is released, the original priority needs to be restored immediately.

After the above analysis, we consider using code to implement these logics. First add the following members to thread.h and initialize them in the init_thread of thread.c Then add members to the lock structure in synch.h.

Modify the lock_acquire function in synch.c:


```c
/* Acquires LOCK, sleeping until it becomes available if
   necessary.  The lock must not already be held by the current
   thread.

   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but interrupts will be turned back on if
   we need to sleep. */
void
lock_acquire (struct lock *lock)
{
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (!lock_held_by_current_thread (lock));
  /*
  sema_down (&lock->semaphore);
  lock->holder = thread_current ();
  */

  /* Solution Code */
  struct thread *cur = thread_current();
  struct lock *tmp;

  /* Lock is currently held by somethread */
  if (!thread_mlfqs && lock->holder != NULL)
  {
    cur->lock_waiting4 = lock;
	tmp = lock;
	while (tmp != NULL && tmp->max_priority < cur->priority)
	{
	  /* Update the max priority */
	  tmp->max_priority = cur->priority;
	  /* Donate priority to its holder thread */
	  thread_donate_priority(tmp->holder);
	  /* Continue donation to threads the holder is waiting for */
	  tmp = tmp->holder->lock_waiting4;
	}
  }
  sema_down(&lock->semaphore);

  enum intr_level old_level = intr_disable();
  cur = thread_current();
  if (!thread_mlfqs)
  {
	/* Now that I've got the lock, I'm not waiting for anylock. */
    cur->lock_waiting4 = NULL;
	/* Besides, the max_priority of this lock must be my priority. */
	lock->max_priority = cur->priority;
	thread_hold_lock(lock);
  }
  lock->holder = cur;

  intr_set_level(old_level);
}
```
Used here

```c
while (tmp != NULL && tmp->max_priority < cur->priority)
	{
	  /* Update the max priority */
	  tmp->max_priority = cur->priority;
	  /* Donate priority to its holder thread */
	  thread_donate_priority(tmp->holder);
	  /* Continue donation to threads the holder is waiting for */
	  tmp = tmp->holder->lock_waiting4;
	}
```
The loop implements the recursive donation and implements the priority donation by modifying the max_priority member of the lock and then updating the priority through the thread_update_priority function.

We declare the following function in thread.h:
```c
void thread_donate_priority(struct thread *t);
void thread_hold_lock(struct lock *lock);
void thread_remove_lock(struct lock *lock);
bool lock_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED);
void thread_update_priority(struct thread *t);

```
First implement thread_donate_priority and thread_hold_lock:

```c
/* Donate the priority of current thread to thread t. */
void
thread_donate_priority(struct thread *t)
{
  enum intr_level old_level = intr_disable();
  thread_update_priority(t);
 
  /* Remove the old t and insert the new one in order */ 
  if (t->status == THREAD_READY)
  {
    list_remove(&t->elem);
	list_insert_ordered(&ready_list, &t->elem, thread_cmp_priority, NULL);
  }

  intr_set_level(old_level);
}

```
By the way, implement the lock_cmp_priority function:

```c
/* Function for lock max priority comparison. */
bool
lock_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)
{
  return list_entry(a, struct lock, elem)->max_priority > list_entry(b, struct lock, elem)->max_priority;
}

```
Then we change the behavior when releasing the lock, that is, modify the lock_release function in synch.c:

```c
/* Releases LOCK, which must be owned by the current thread.

   An interrupt handler cannot acquire a lock, so it does not
   make sense to try to release a lock within an interrupt
   handler. */
void
lock_release (struct lock *lock) 
{
  ASSERT (lock != NULL);
  ASSERT (lock_held_by_current_thread (lock));

  /* Solution Code */
  if (!thread_mlfqs)
    thread_remove_lock(lock);

  lock->holder = NULL;
  sema_up (&lock->semaphore);
}
```
And implement the thread_remove_lock function:

```c
/* Remove the lock for the thread. */
void
thread_remove_lock(struct lock *lock)
{
  enum intr_level old_level = intr_disable();
  list_remove(&lock->elem);
  thread_update_priority(thread_current());

  intr_set_level(old_level);
}
```

Finally, we implement thread_update_priority that is used in many places:

```c
/* Update the thread's priority. */
void
thread_update_priority(struct thread *t)
{
  enum intr_level old_level = intr_disable();
  int max_pri = t->base_priority;
  int lock_pri;

  /* If the thread is holding locks, pick the one with the highest max_priority.
   * And if this priority is greater than the original base priority,
   * the real(donated) priority would be updated.*/
  if (!list_empty(&t->locks_holding))
  {
    list_sort(&t->locks_holding, lock_cmp_priority, NULL);
	lock_pri = list_entry(list_front(&t->locks_holding), struct lock, elem)->max_priority;
    if (max_pri < lock_pri)
	  max_pri = lock_pri;
  }
  t->priority = max_pri;

  intr_set_level(old_level);
}

```
This function handles the change in priority when the lock is released: if the current thread still has a lock, it gets the max_priority of its own lock. Updates the priority of the donation if it is greater than base_priority.

Finally, we modify the thread_set_priority function:

```c
/* Sets the current thread's priority to NEW_PRIORITY. */
void
thread_set_priority (int new_priority) 
{
  /*
  thread_current ()->priority = new_priority;
  thread_yield();
  */

  /* Solution Code */
  if (thread_mlfqs)
    return;
  enum intr_level old_level = intr_disable();
  struct thread *cur = thread_current();
  int old_priority = cur->priority;
  cur->base_priority = new_priority;

  if (list_empty(&cur->locks_holding) || new_priority > old_priority)
  {
    cur->priority = new_priority;
	thread_yield();
  }
  intr_set_level(old_level);
}
```
The test on the priority-donate series should be fine, but there are no two priority queues for sema and condvar, so we implement these two priority queues.

First modify the cond_signal function in synch.c:
```c
/* If any threads are waiting on COND (protected by LOCK), then
   this function signals one of them to wake up from its wait.
   LOCK must be held before calling this function.

   An interrupt handler cannot acquire a lock, so it does not
   make sense to try to signal a condition variable within an
   interrupt handler. */
void
cond_signal (struct condition *cond, struct lock *lock UNUSED) 
{
  ASSERT (cond != NULL);
  ASSERT (lock != NULL);
  ASSERT (!intr_context ());
  ASSERT (lock_held_by_current_thread (lock));

  if (!list_empty (&cond->waiters)) 
  {
	/* Solution Code */
	list_sort(&cond->waiters, cond_cmp_priority, NULL);

    sema_up (&list_entry (list_pop_front (&cond->waiters), struct semaphore_elem, elem)->semaphore);
  }
}
```
Then declare and implement the comparison function cond_cmp_priority:


```c
/* Solution Code */
/* Function for condvar waiters priority comparison. */
bool
cond_cmp_priority(const struct list_elem *a, const struct list_elem *b, void *aux UNUSED)
{
  struct semaphore_elem *sa = list_entry(a, struct semaphore_elem, elem);
  struct semaphore_elem *sb = list_entry(b, struct semaphore_elem, elem);
  return list_entry(list_front(&sa->semaphore.waiters), struct thread, elem)->priority > \ 
		 list_entry(list_front(&sb->semaphore.waiters), struct thread, elem)->priority;
}
```
This makes it possible to change the condition variable wait queue to the priority queue, similarly, modifying sema_up and sema_down respectively:
```c
/* Up or "V" operation on a semaphore.  Increments SEMA's value
   and wakes up one thread of those waiting for SEMA, if any.

   This function may be called from an interrupt handler. */
void
sema_up (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);

  old_level = intr_disable ();
  if (!list_empty (&sema->waiters)) 
  {
    /* Solution Code */
	list_sort(&sema->waiters, thread_cmp_priority, NULL);

	thread_unblock(list_entry(list_pop_front(&sema->waiters), struct thread, elem));
  }
  sema->value++;
  /* Solution Code */
  thread_yield();

  intr_set_level (old_level);
}
```

```c
/* Down or "P" operation on a semaphore.  Waits for SEMA's value
   to become positive and then atomically decrements it.

   This function may sleep, so it must not be called within an
   interrupt handler.  This function may be called with
   interrupts disabled, but if it sleeps then the next scheduled
   thread will probably turn interrupts back on. */
void
sema_down (struct semaphore *sema) 
{
  enum intr_level old_level;

  ASSERT (sema != NULL);
  ASSERT (!intr_context ());

  old_level = intr_disable ();
  while (sema->value == 0) 
    {
	  /* Solution Code */
      list_insert_ordered(&sema->waiters, &thread_current ()->elem, thread_cmp_priority, NULL);

      thread_block ();
    }
  sema->value--;
  intr_set_level (old_level);
}

```
This makes it possible to change the semaphore queue into a priority queue. After the completion, make a check and succeed.

## Pintos advanced scheduling strategy and practice process.

Reference materials:http://www.ccs.neu.edu/home/amislove/teaching/cs5600/fall10/pintos/pintos_7.html

It can be found that the experiment requires a 64-level scheduling queue, each queue has its own priority, ranging from PRI_MIN to PRI_MAX. The operating system then schedules threads from high priority queues and dynamically updates thread priorities over time.

The formula calculation here involves confusing floating-point operations. According to the reference, we may first create a fixed-point.h file in the threads directory to save some macro definitions about floating-point operations. Here we use 16 digits to represent the fractional part, so the operation on integers must start from the 17th digit.

```c
#ifndef __THREAD_FIXED_POINT_H
#define __THREAD_FIXED_POINT_H

/* Basic definitions of fixed point. */
typedef int fixed_t;
/* 16 LSB used for fractional part. */
#define FP_SHIFT_AMOUNT 16
/* Convert a value to a fixed-point value. */
#define FP_CONST(A) ((fixed_t)(A << FP_SHIFT_AMOUNT))
/* Add two fixed-point value. */
#define FP_ADD(A,B) (A + B)
/* Add a fixed-point value A and an int value B. */
#define FP_ADD_MIX(A,B) (A + (B << FP_SHIFT_AMOUNT))
/* Subtract two fixed-point value. */
#define FP_SUB(A,B) (A - B)
/* Subtract an int value B from a fixed-point value A. */
#define FP_SUB_MIX(A,B) (A - (B << FP_SHIFT_AMOUNT))
/* Multiply a fixed-point value A by an int value B. */
#define FP_MULT_MIX(A,B) (A * B)
/* Divide a fixed-point value A by an int value B. */
#define FP_DIV_MIX(A,B) (A / B)
/* Multiply two fixed-point value. */
#define FP_MULT(A,B) ((fixed_t)(((int64_t) A) * B >> FP_SHIFT_AMOUNT))
/* Divide two fixed-point value. */
#define FP_DIV(A,B) ((fixed_t)((((int64_t) A) << FP_SHIFT_AMOUNT) / B))
/* Get the integer part of a fixed-point value. */
#define FP_INT_PART(A) (A >> FP_SHIFT_AMOUNT)
/* Get the rounded integer of a fixed-point value. */
#define FP_ROUND(A) (A >= 0 ? ((A + (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT) \
				: ((A - (1 << (FP_SHIFT_AMOUNT - 1))) >> FP_SHIFT_AMOUNT))
#endif /* threads/fixed-point.h */


```
With this header file, the implementation of the code logic only needs to follow the formula of the reference material, there is no difficulty.

First modify the timer_interrupt function. From the reference we know that each time the clock is interrupted, the running thread's recent_cpu will be incremented by 1, and every TIMER_FREQ ticks will update the system load_avg and all threads' recent_cpu, and the thread priority will be updated every 4 ticks. So add the code.Next, implement the recent_cpu auto-increment function in thread.c (and declare it in thread.h, the same below):

```c
/* Recent CPU++; */
void
mlfqs_inc_recent_cpu()
{
  ASSERT(thread_mlfqs);
  ASSERT(intr_context());

  struct thread *cur = thread_current();
  if (cur == idle_thread)
    return;
  cur->recent_cpu = FP_ADD_MIX(cur->recent_cpu, 1);
}
```
Implement the function to update the system load_avg and all threads recent_cpu:

```c
/* Update load_avg and recent_cpu of all threads every TIMER_FREQ ticks. */
void
mlfqs_update_load_avg_and_recent_cpu()
{
  ASSERT(thread_mlfqs);
  ASSERT(intr_context());

  size_t ready_cnt = list_size(&ready_list);
  if (thread_current() != idle_thread)
    ++ready_cnt;
  load_avg = FP_ADD (FP_DIV_MIX (FP_MULT_MIX (load_avg, 59), 60), FP_DIV_MIX(FP_CONST(ready_cnt), 60));

  struct thread *t;
  struct list_elem *e;
  for (e = list_begin(&all_list); e != list_end(&all_list); e = list_next(e))
  {
    t = list_entry(e, struct thread, allelem);
    if (t != idle_thread)
    {
      t->recent_cpu = FP_ADD_MIX (FP_MULT (FP_DIV (FP_MULT_MIX (load_avg, 2), \ 
					  FP_ADD_MIX (FP_MULT_MIX (load_avg, 2), 1)), t->recent_cpu), t->nice);
	  mlfqs_update_priority(t);
	}
  }
}

```
A function that implements the update priority:
```c
/* Update thread's priority. */
void
mlfqs_update_priority(struct thread *t)
{
  ASSERT(thread_mlfqs);

  if (t == idle_thread)
    return;

  t->priority = FP_INT_PART (FP_SUB_MIX (FP_SUB (FP_CONST (PRI_MAX), \ 
							 FP_DIV_MIX (t->recent_cpu, 4)), 2 * t->nice));
  if (t->priority < PRI_MIN)
    t->priority = PRI_MIN;
  else if (t->priority > PRI_MAX)
    t->priority = PRI_MAX;
}
```
Finally, we add members to the thread structure and initialize them in init_thread:
Then we do the aftermath in thread.c.
Declare the global variable load_avg.
Initialize load_avg in thread_start.
Include floating point header files in thread.h.
Implement four functions that have not yet been completed:
```c
/* Sets the current thread's nice value to NICE. */
void
thread_set_nice (int nice UNUSED) 
{
  /* Solution Code */
  thread_current()->nice = nice;
  mlfqs_update_priority(thread_current());
  thread_yield();
}

/* Returns the current thread's nice value. */
int
thread_get_nice (void) 
{
  /* Solution Code */
  return thread_current()->nice;
}

/* Returns 100 times the system load average. */
int
thread_get_load_avg (void) 
{
  /* Solution Code */
  return FP_ROUND (FP_MULT_MIX (load_avg, 100));
}

/* Returns 100 times the current thread's recent_cpu value. */
int
thread_get_recent_cpu (void) 
{
  /* Solution Code */
  return FP_ROUND (FP_MULT_MIX (thread_current()->recent_cpu, 100));
}

```
At this point, the task should be completed, we get "All 27 tests" after make check
passed.", a complete success.





