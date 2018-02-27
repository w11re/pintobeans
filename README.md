			+--------------------+
			|      CS 320CA      |
			| PROJECT 1: THREADS |
			|   DESIGN DOCUMENT  |
			+--------------------+
				   
---- GROUP ----

>> Fill in the names and email addresses of your group members.

Hieu Tran <htran@cogswell.edu>

Duy Nguyen <dcnguyen@cogswell.edu>


---- PRELIMINARIES ----

>> If you have any preliminary comments on your submission, notes for the
>> TAs, or extra credit, please give them here.

Notes taken during class, assisstance through emails from the professor, Pintos documentation.

>> Please cite any offline or online sources you consulted while
>> preparing your submission, other than the Pintos documentation, course
>> text, lecture notes, and course staff.


			 PRIORITY SCHEDULING
			 ===================

---- DATA STRUCTURES ----

>> B1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.


thread.c:
  struct thread:
struct list_elem donatelem - This is the list elements for donator’s list.
struct list donators_list - This is the list of all the threads (X) waiting on thread Y, used for priority donation bookkeeping.
struct lock *wanted_lock - A mutex that a thread needs to acquire; used to identify the holder of the lock the thread is waiting for.

>> B2: Explain the data structure used to track priority donation.
>> Use ASCII art to diagram a nested donation.  (Alternately, submit a
>> .png file.)

In the same /pintos directory as README.md, we have a png file called Diagram.png showing our data structure. Alternatively, it can also be accessed through: https://drive.google.com/file/d/17l_ezvC2HK-wwEb_X34BrWkRsUgoYr-4/view

A linked list is used to track the priority donation. In Struct thread, a donators_list is used to keep track of all the threads waiting on “this” thread for its mutex. This is used for bookkeeping, keeping track of which thread donated to the thread for nested donation. 

---- ALGORITHMS ----

>> B3: How do you ensure that the highest priority thread waiting for
>> a lock, semaphore, or condition variable wakes up first?


We ensure that the highest priority thread wakes up through bookkeeping and priority donation. The list that we use to keep track of a thread’s donation (donators_list) is sorted through every entry. 

Sema_down implementation was also changed to keep the "waiters" list sorted by priority, such that the thread with the highest priority will always be given the chance to run next instead of waking up a random thread. We've also done the same with sema_up.

>> B4: Describe the sequence of events when a call to lock_acquire()
>> causes a priority donation.  How is nested donation handled?

If a lock/mutex’s holder is not NULL, meaning that there is a thread holding it, thread_current->wanted_lock is set to that thread. Thread_current is inserted into the lock->holder’s donators_list. Donate function is then called to perform donation priority if the lock->holder’s priority is less than thread_current. 
Nested donation is handled by bookkeeping through the donators_list, as it keeps track of all the threads waiting for “that” thread.

>> B5: Describe the sequence of events when lock_release() is called
>> on a lock that a higher-priority thread is waiting for.

When lock_release() is called, iterate through lock->holder’s donators_list to find the lock that is being released to remove it from the list.
The lock->holder’s priority is also updated with it’s old_priority so that the higher-priority thread can run its process.
The lock->holder is set to NULL since the thread has released it.

---- SYNCHRONIZATION ----

>> B6: Describe a potential race in thread_set_priority() and explain
>> how your implementation avoids it.  Can you use a lock to avoid
>> this race?


A potential race in thread_set_priority() is when a thread is in the process of changing its priority, while another thread is in the process of "donating" its priority to the original priority.
To avoid this race condition, locks can be used to prevent another thread from donating while the original thread is setting its priority. 

---- RATIONALE ----

>> B7: Why did you choose this design?  In what ways is it superior to
>> another design you considered?


We chose this design because it was the the simplest and most efficient way we could think of to implement priority scheduling to Pintos. This design which has changes to Struct Thread to contain elements to perform bookkeeping, and other functions to handle priority sorting, makes the project efficient. This is far superior to our original design which featured no donator’s list to keep track of all the threads that is waiting on a thread’s mutex.



			  ADVANCED SCHEDULER
			  ==================

---- DATA STRUCTURES ----

>> C1: Copy here the declaration of each new or changed `struct' or
>> `struct' member, global or static variable, `typedef', or
>> enumeration.  Identify the purpose of each in 25 words or less.

Thread.h:
	int nice - Nice value of thread, determines how "nice" the thread should be to other threads. */
	int recent_cpu - Measures how much CPU time each process has received "recently."
	#define NICE_MIN -20 - Lowest nice value is -20. 
	#define NICE_DEFAULT 0 - Default nice is 0. 
	#define NICE_MAX 20	- Max nice is 20.
	#define P 17 - 17.14 fixed point representation, 17 bits before decimal.
	#define Q 14 - 17.14 representation, 14 bits after decimal.
	#define F 1<<(Q) - fraction used to convert FP/INT.
	#define FP_REP(x) (x) * (F) - convert INT to FP.
	#define ADD(x, n) (x) + (n) * (F) - FP and INT addition.
	#define SUB(x, n) (x) - (n) * (F) - FP and INT subtraction.
	#define INT_NEAR(x) ((x) >= 0 ? ((x) + (F) / 2) / (F) : ((x) - (F) / 2) / (F)) - converts FP to int closer to next int.
	#define INT_ZERO(x) (x) / (F) - converts FP to INT closer to zero.
	#define FP_PRODUCT(x, y) (((int64_t)(x)) * (y) / (F)) 
	#define FP_DIV(x, y) ((int64_t)(x)) * (F) / (y)

---- ALGORITHMS ----

>> C2: Suppose threads A, B, and C have nice values 0, 1, and 2.  Each
>> has a recent_cpu value of 0.  Fill in the table below showing the
>> scheduling decision and the priority and recent_cpu values for each
>> thread after each given number of timer ticks:

timer  recent_cpu    priority   thread
ticks   A   B   C   A   B   C   to run
-----  --  --  --  --  --  --   ------
0      0   0    0   63.00 61.00 59.00   A
4      4   0    0   62.00 61.00 59.00   A
8      8   0    0   61.00 61.00 59.00   A
12     12  0    0   60.00 61.00 59.00   B
16     12  4    0   60.00 60.00 59.00   B
20     12  8    0   60.00 59.00 59.00   A
24     16  8    0   59.00 59.00 59.00   A
28     20  8    0   58.00 59.00 59.00   C
32     20  8    4   58.00 59.00 58.00   B
36     20  12   4   58.00 58.00 58.00   B

>> C3: Did any ambiguities in the scheduler specification make values
>> in the table uncertain?  If so, what rule did you use to resolve
>> them?  Does this match the behavior of your scheduler?


A specific ambiguity is the recent_cpu as we cannot define how much time it spends running as the thread(s) needs to yield during calculations for recent_cpu, load_avg, and priority. Every so often amount of ticks we predicted is not the actual amount.

>> C4: How is the way you divided the cost of scheduling between code
>> inside and outside interrupt context likely to affect performance?

The cost of scheduling outside of the interrupt will hinder the performance of the CPU less than if the cost of scheduling inside goes up. The CPU could potentially take too much time on calculations for recent_cpu and load_avg and lower the thread priority.

---- RATIONALE ----

>> C5: Briefly critique your design, pointing out advantages and
>> disadvantages in your design choices.  If you were to have extra
>> time to work on this part of the project, how might you choose to
>> refine or improve your design?

Our implementation did not work and more time and guidance would have helped. 



>> C6: The assignment explains arithmetic for fixed-point math in
>> detail, but it leaves it open to you to implement it.  Why did you
>> decide to implement it the way you did?  If you created an
>> abstraction layer for fixed-point math, that is, an abstract data
>> type and/or a set of functions or macros to manipulate fixed-point
>> numbers, why did you do so?  If not, why not?

We use fixed-point numbers to represent recent_cpu and load_avg because pintos does not allow you to use float numbers.

Within the assignment/pintos's documentation, there was a mention of a separate 
header file that would be used to for fixed point arithmetics, called fixed_point.h. 
However, we decided to create a set of macros to manipulate and representation
fixed point numbers and arithmetics. This was done for simplicity as we did not
have to create a separate header file for fixed point arithmetics. 

			   SURVEY QUESTIONS
			   ================

Answering these questions is optional, but it will help us improve the
course in future quarters.  Feel free to tell us anything you
want--these questions are just to spur your thoughts.  You may also
choose to respond anonymously in the course evaluations at the end of
the quarter.

>> In your opinion, was this assignment, or any one of the three problems
>> in it, too easy or too hard?  Did it take too long or too little time?

>> Did you find that working on a particular part of the assignment gave
>> you greater insight into some aspect of OS design?

>> Is there some particular fact or hint we should give students in
>> future quarters to help them solve the problems?  Conversely, did you
>> find any of our guidance to be misleading?

>> Do you have any suggestions for the TAs to more effectively assist
>> students, either for future quarters or the remaining projects?

>> Any other comments?


