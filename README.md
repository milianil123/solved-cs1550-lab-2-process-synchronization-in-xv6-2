Download Link: https://assignmentchef.com/product/solved-cs1550-lab-2-process-synchronization-in-xv6-2
<br>
In this lab, you will implement a synchronization solution using locks and condition variables to guarantee a specific execution ordering among two processes.  <strong>Make sure that you start from a fresh copy of the Xv6 code. </strong>

1.      PART 1: THREE PROCESSES – A PARENT AND TWO CHILDREN

In the first part you will write a small user-land program that has a parent process forking two child processes, resulting in a total of three processes: the parent process and the two children.

<table width="0">

 <tbody>

  <tr>

   <td width="718">#include “types.h”#include “stat.h”#include “user.h” <em>//We want Child 1 to execute first, then Child 2, and finally Parent.</em> int main() {    int pid = fork(); <em>//fork the first child</em>   <strong>if</strong>(pid &lt; 0) {      printf(1, “Error forking first child.<strong>
</strong>“);} <strong>else</strong> <strong>if</strong> (pid == 0) {     printf(1, “Child 1 Executing<strong>
</strong>“);} <strong>else</strong> {     pid = fork(); <em>//fork the second child</em>     <strong>if</strong>(pid &lt; 0) {       printf(1, “Error forking second child.<strong>
</strong>“);} <strong>else</strong> <strong>if</strong>(pid == 0) {       printf(1, “Child 2 Executing<strong>
</strong>“);} <strong>else</strong> {printf(1, “Parent Waiting<strong>
</strong>“);       int i;       <strong>for</strong>(i=0; i&lt; 2; i++)          wait();       printf(1, “Children completed<strong>
</strong>“);       printf(1, “Parent Executing<strong>
</strong>“);       printf(1, “Parent exiting.<strong>
</strong>“);}   }   exit();}</td>

  </tr>

 </tbody>

</table>




<ul>

 <li>Think of the printf(1, “&lt;X&gt; executing
”) statements as placeholders for the code that each process runs.</li>

 <li>What is the effect of the parent process calling wait() two times?</li>

</ul>

<ol>

 <li>Put the previous code into a file named c inside your Xv6 source code folder. Add _race to the UPROGS variable inside your Makefile. Compile and run Xv6 by typing make qemu-nox. (You may refer to the instructions of setting up your Xv6 environment from Lab 1.)</li>

 <li>Run the user-land program inside Xv6 by typing race at the Xv6 prompt. Notice the order of execution of the three processes. Run the program <strong>multiple times</strong>.

  <ul>

   <li>Do you always get the same order of execution?</li>

   <li>Does Child 1 always execute (print Child 1 Executing) before Child 2?</li>

  </ul></li>

 <li>Add a sleep(5); line before the line where Child 1 prints “Child Executing”.

  <ul>

   <li>What do you notice?</li>

   <li>Can we guarantee that Child 1 always execute before Child 2?</li>

   <li>2.      PART 2: SPIN LOCKS</li>

  </ul></li>

</ol>

We will start by defining a spinlock that we can use in our user-land program. Xv6 already has a spinlock (see spinlock.c) that it uses inside its kernel and is coded in somehow a complex way to handle concurrency caused by interrupts. We don’t need most of the complexity, so will write our own light-weight version of spinlocks. We will put our code inside ulib.c, which includes functions accessible to user-land programs.

<ol start="4">

 <li>Inside ulib.c, add</li>

</ol>

#include “spinlock.h”

after the line:

#include “types.h”




Also, add the following function definitions:

void

init_lock(<strong>struct</strong> spinlock * lk) {   lk-&gt;locked = 0;

}  void lock(<strong>struct</strong> spinlock * lk) {    <strong>while</strong>(xchg(&amp;lk-&gt;locked, 1) != 0)

;

}  void unlock(<strong>struct</strong> spinlock * lk) {   xchg(&amp;lk-&gt;locked, 0); }




We are still using the struct spinlock defined in spinlock.h but we will only use its locked field. Initializing the lock and unlocking it both work by setting locked to 0. Locking uses the atomic xchg instruction, which sets the contents of its first parameter (a memory address) to the second parameter and returns the old value of the contents of the first parameter.

<ol start="5">

 <li>Inside user.h, add the following function prototypes:</li>

</ol>

void init_lock(<strong>struct</strong> spinlock *); void lock(<strong>struct</strong> spinlock *); void unlock(<strong>struct</strong> spinlock *);

to the end of the file and add the following two lines into the beginning of the file:

<strong>struct</strong> condvar; <strong>struct</strong> spinlock;

Now, we have our spinlocks in place. We can use them inside race.c:

<strong>struct</strong> spinlock lk; init_lock(&amp;lk); lock(&amp;lk); <em>//critical section</em> unlock(&amp;lk)

We will use condition variables to be able to make Child 2 sleep (block) until Child 1 finishes execution.

3.      PART 3: CONDITION VARIABLES

We will use condition variables to ensure that Child 1 always executes before Child 2. We will add two system calls to Xv6: cv_wait() and cv_signal() to wait (sleep) on a condition variable and to wakeup (signal) all sleeping processes on a condition variable.

Recall that both waiting and signaling a condition variable has to be called after acquiring a lock (that’s why we defined our spinlock in Part 2). cv_wait releases the lock before sleeping and reacquires it after waking up.

<ol start="6">

 <li>First, define the condition variable structure in condvar.h as follows.</li>

</ol>

#include “spinlock.h” <strong>struct</strong> condvar {    <strong>struct</strong> spinlock lk;

};

A condition variable has a spin lock.

Let’s then add the two system calls.

<ol start="7">

 <li>Inside syscall.h, add the following two lines:</li>

</ol>

#define SYS_cv_signal 22 #define SYS_cv_wait   23

Inside usys.S, add:

SYSCALL(cv_signal)

SYSCALL(cv_wait)

Inside syscall.c, add:

<strong>extern</strong> int sys_cv_signal(void); <strong>extern</strong> int sys_cv_wait(void);

and

[SYS_cv_wait] sys_cv_wait,

[SYS_cv_signal] sys_cv_signal,

Inside user.h, add

<strong>struct</strong> condvar;

to the beginning and

int cv_wait(<strong>struct</strong> condvar *); int cv_signal(<strong>struct</strong> condvar *);

to the end of the system calls section of the file.

Our condition variable implementation depends heavily on the sleep/wakeup mechanism implemented inside Xv6 (Please read <strong>Sleep and Wakeup</strong> on Page 65 of the Xv6 book). We will again define a more light-weight version of the sleep function to use our light-weight spinlocks defined in Part 2 instead of Xv6’s spinlocks.

<ol start="8">

 <li>Inside proc.c, add the following function definition:</li>

</ol>

<table width="0">

 <tbody>

  <tr>

   <td width="580">void sleep1(void *chan, <strong>struct</strong> spinlock *lk){   <strong>struct</strong> proc *p = myproc();<strong>if</strong>(p == 0)     panic(“sleep”);<strong>if</strong>(lk == 0)     panic(“sleep without lk”);acquire(&amp;ptable.lock);   lk-&gt;locked = 0;   <em>// Go to sleep.</em>   p-&gt;chan = chan;   p-&gt;state = SLEEPING;sched(); <em>// Tidy up.</em>   p-&gt;chan = 0; release(&amp;ptable.lock);   <strong>while</strong>(xchg(&amp;lk-&gt;locked, 1) != 0);</td>

  </tr>

 </tbody>

</table>

}

After a couple of sanity checks, the function acquires the process table lock ptable.lock to be able to call sched(), which works on the process table. Then, it releases the spinlock (by setting locked to 0) and goes to sleep by setting the process state to SLEEPING, setting the channel that the process sleeps on, and switching into the scheduler by calling sched(). After the process wakes up, it releases the ptable lock and reacquires the spinlock (using the xchg instruction).

<ol start="9">

 <li>Inside h, add the following function prototype in the //proc.c section:</li>

</ol>

void            sleep1(void*, <strong>struct</strong> spinlock*);

<ol start="10">

 <li>Inside sysproc.c add</li>

</ol>

#include “condvar.h”

after the line:

#include “types.h”

and add the following syscall implementations:

<table width="0">

 <tbody>

  <tr>

   <td width="580">int sys_cv_signal(void){   int i;   <strong>struct</strong> condvar *cv;   argint(0, &amp;i);   cv = (<strong>struct</strong> condvar *) i;   wakeup(cv);   <strong>return</strong> 0;}  int sys_cv_wait(void){   int i;   <strong>struct</strong> condvar *cv;   argint(0, &amp;i);cv = (<strong>struct</strong> condvar *) i;   sleep1(cv, &amp;(cv-&gt;lk));   <strong>return</strong> 0;}</td>

  </tr>

 </tbody>

</table>




to the end. In both functions, the code starts with retrieving the argument (struct condvar *) from the stack:

argint(0, &amp;i);   cv = (<strong>struct</strong> condvar *) i;

The address of the condition variable is used as the channel passed over to the sleep1 function defined in Step 8. The address of the condition variable is unique and this is all what we need: a unique channel number to sleep and to get waked up on.

<strong>After seeing what the two system calls do, why do you think we had to add system calls for the operations on condition variables? Why not just have these operations as functions in ulib.c as we did for the spinlock? </strong>

4.      PART 4: USING THE CONDITION VARIABLES

We can then modify race.c to use a condition variable to guarantee process ordering.

}

Note the highlighted parts. A condition variable is declared. Its spinlock is initialized. Then Child 1 signals the condition variable after acquiring the spinlock. Child 2 sleeps on the condition variable after acquiring the spinlock as well.

Compile and run the modified race program.

<ul>

 <li>Does Child 1 always execute before Child 2?</li>

</ul>

<h1>5.      PART 5: LOST WAKEUPS</h1>

Does it happen that the program gets stuck? This is called a <strong>deadlock</strong>. If Child 2 gets to sleep <strong>after</strong> Child 1 signals, the wakeup signal is lost (i.e., never received by Child 2). In this case, Child 2 has no way of being awaked.

To solve this problem, we need to enclose the cv_wait() call inside a while loop. We need some form of a flag that gets set by Child 1 when it is done executing. Child 2 will then do  while(flag is not set)  cv_wait();

This way, even if Child 1 sets the flag and signals before Child 2 executes the while loop, Child 2 will not avoid going to sleep because the flag will be set.

The flag has to be <strong>shared</strong> between the two processes. We will use a <strong>file</strong> for that. Other methods for sharing are shared memory and pipes.

To create a file,

int fd = open(“flag”, O_RDWR | O_CREATE);

To write into the file,

write(fd, “done”, 4);

Checking the flag has to be non-blocking. The read system call is blocking. Reading the size of the file is not. So, we will check the flag by reading the file size. To read the size of a file,

<strong>struct</strong> stat stats; fstat(fd, &amp;stats);

printf(1, “file size = %d
”, stats.size);




Now, we are ready to write the while loop inside Child 2. It will loop until the file size is greater than zero, which happens when Child 1 writes “done” into the file after it finishes execution.

lock(&amp;cv.lk); <strong>struct</strong> stat stats; fstat(fd, &amp;stats); printf(1, “file size = %d<strong>
</strong>“, stats.size); <strong>while</strong>(stats.size &lt;= 0){    cv_wait(&amp;cv);    fstat(fd, &amp;stats);

<table width="0">

 <tbody>

  <tr>

   <td width="580">   printf(1, “file size = %d<strong>
</strong>“, stats.size);} unlock(&amp;cv.lk);</td>

  </tr>

 </tbody>

</table>

The new race.c is:

<table width="0">

 <tbody>

  <tr>

   <td colspan="4" width="579">      printf(1, “Children completed<strong>
</strong>“);       printf(1, “Parent Executing<strong>
</strong>“);       printf(1, “Parent exiting.<strong>
</strong>“);}}</td>

  </tr>

  <tr>

   <td width="17"> </td>

   <td width="216">


    <table width="0">

     <tbody>

      <tr>

       <td width="80">close(fd);</td>

      </tr>

     </tbody>

    </table></td>

   <td colspan="2" width="346"> </td>

  </tr>

  <tr>

   <td colspan="3" width="267">


    <table width="0">

     <tbody>

      <tr>

       <td width="136">  unlink(“flag”);</td>

      </tr>

     </tbody>

    </table></td>

   <td rowspan="2" width="312"> </td>

  </tr>

  <tr>

   <td colspan="3" width="267">  exit();}</td>

  </tr>

  <tr>

   <td width="17"></td>

   <td width="216"></td>

   <td width="34"></td>

   <td width="312"></td>

  </tr>

 </tbody>

</table>

Note the highlighted parts and also note that we are closing the file and deleting it before the parent exits. This is to start afresh the next time we run the program.

Compile and run race.c many times.

<ul>

 <li>Is it always the case that Child 1 executes before Child 2?</li>

 <li>Do you observe deadlocks?</li>

</ul>

Of course, synchronization bugs cannot be ruled out by running a program many times. Formal proof is typically the preferred way especially for safety- and mission-critical systems. There are tools that help with this kind of formal proofs.