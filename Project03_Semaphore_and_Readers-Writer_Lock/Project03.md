# Semaphore / Readers-writer Lock
+ Semaphore
  + An object with an integer value
  + Prevents multiple threads from accessing data or critical section of shared resources by providing mutual exclusion
  + If the value is 0, access is not granted
  + Else, the value represents the number of threads that can access the critical section concurrently

+ Readers-writer Lock
  + Only a single writer can acquire the lock
  + Readers can acquire the lock simultaneously as long as there are no writers
  + Locking method to supplement the weakness of using Mutex
  + Weakness mentioned above: Limiting the number of accessible thread to resources to 1

# POSIX semaphore
+ `sem_t`
  + `typedef int sem_t`
  + An object with an integer value

+ `int sem_init(sem_t *sem, int pshared, unsigned int value)`
  + `sem_t *sem`: the pointer of sem_t to initialize
  + `int pshared`: indicates whether given semaphore is to be shared between the threads of a process, or between processes.
    + If pshared == 0, the given semaphore is shared between threads in the same process
    + Else, the semaphore is shared between processes
  + `unsigned int value`: specifies the initial value of the given semaphore
  + Return value: 0 on success, -1 on error

+ `int sem_wait(sem_t *sem)`
  + Decrements the value of the given semaphore
  + If the value of the semaphore was zero when called `sem_wait()`, return right away
  + When negative, wait for the value to be changed to 0
  + When negative, the value of the semaphore is equal to the number of waiting threads

+ `int sem_post(sem_t *sem)`
  + Increments the value of the given semaphore
  + If the value is not positive, wakes one of the waiting threads who is sleeping

# POSIX reader-writer lock
+ `pthread_rwlock_t`
  + A fair reader writer lock solving the starving problem with priority queue
  + `int readers`: The number of readers inside the critical section
  + `priority_queue_t queue`: queue of waiting threads
  + `mutex_t mutex`: Provides mutual exclusion on reading and writing

+ `int pthread_rwlock_init(pthread_rwlock_t* lock, const pthread_rwlockattr_t* attr)`
  + `pthread_rwlock_t* lock`: the pointer of pthread_rwlock_t to initialize
  + `const pthread_rwlockattr_t* attr`
    + If NULL, the default read-write lock attributes are used
  + Return value: 0 on success, otherwise error number

+ `int pthread_rwlock_rdlock(pthread_rwlock_t* lock)`
  + The calling thread acquires a readlock if there is no writer for the given lock
  + If there is a writer for the given lock, wait until the writer release the lock
  + Return value: 0 on success, otherwise error number

+ `int pthread_rwlock_wrlock(pthread_rwlock_t* lock)`
  + The calling thread acquires a writelock if there is no thread holding the given lock
  + If there is any threads for the given lock, wait until the threads release the lock
  + Return value: 0 on success, otherwise error number

+ `int pthread_rwlock_unlock(pthread_rwlock_t* lock)`
  + If there are other readers for the given lock, the read-write lock object remains in the read locked state.
  + If calling thread is the last reader for the given lock, release the lock
  + If calling thread is a writer for the given lock, release the lock
  + Return value: 0 on success, otherwise error number

# Implement the Basic Mutex
```c
typedef struct thread_mutex
{
  uint locked;
  void * owner;
} thread_mutex_t;
```
+ `thread_mutex_t`
  + Defined in `types.h`
  + `uint locked`: 0 for unlocked, 1 for locked
  + `void * owner`: a thread who has the lock


```c
int
mutex_init(thread_mutex_t* lock)
{
  lock->locked = 0;
  lock->owner = 0;
  
  if(!(lock->locked == 0 && lock->owner == 0)) {
    return -1;
  }

  return 0;
}
```
+ `int mutex_init(thread_mutex_t* lock)`
  + Defined in `thread.c`
  + Initial value of `locked` is 0
  + Initial value of `owner` is 0


```c
void
mutex_lock(thread_mutex_t* lock)
{
  pushcli();
  struct thread* curthd = mythd();
  
  while(xchg(&lock->locked, 1) != 0) {
    ;
  }
  
  __sync_synchronize();

  if(lock->locked == 0) {
    panic("mutex_lock");
  }

  lock->owner = curthd;
}
```
+ `void mutex_lock(thread_mutex_t* lock)`
  + Defined in `thread.c`
  + Disable interrupts to avoid deadlock by using `pushcli()`
  + Locks (i.e. spinlock) in xv6 are implemented using the `xchg` atomic instruction
  + Change the value of `locked` to 1 by using `xchg`
  + `__sync_synchronize()` ensures that the critical section's memory references happen after the lock is acquired
  + If locked == 0 even though we changed it to 1, call `panic`
  + `owner` is the calling thread

```c
void
mutex_unlock(thread_mutex_t* lock)
{
  lock->owner = 0;

  __sync_synchronize();

  asm volatile("movl $0, %0" : "+m" (lock->locked) : );

  if(lock->locked == 1) {
    panic("mutex_unlock");
  }

  popcli();
}
```
+ `void mutex_unlock(thread_mutex_t* lock)`
  + Defined in `thread.c`
  + Change `owner` to 0
  + `__sync_synchronize()` ensures that the critical section's memory references happen before the lock is released
  + By using the `asm` atomic instruction, change the value of `locked` to 0
  + If locked == 1 even though we changed it to 0, call `panic`
  + Allow interrupts by using `popcli()`

# Implement the Basic Condition Variable
```c
typedef struct thread_attr
{
  int waiters;
  thread_mutex_t *mutex;
} thread_cond_t;
```
+ `thread_cond_t`
  + Defined in `types.h`
  + `int waiters`: the number of waiters who are waiting for 
  + `thread mutex_t *mutex`: mutex of condition variable

```c
int
cond_init(thread_cond_t* cond)
{
  cond->waiters = 0;
  
  if(mutex_init(cond->mutex) == -1) {
    return -1;
  }

  return 0;
}
```
+ `in cond_init(thread_cond_t* cond)`
  + Defined in `thread.c`
  + Initial value of `waiters` is 0
  + Initialize the `mutex` of condition variable

```c
void
cond_wait(thread_cond_t* cond, thread_mutex_t* lock)
{
  mutex_lock(cond->mutex);
  cond->waiters++;
  mutex_unlock(cond->mutex);

  mutex_unlock(lock);
  
  cond_sleep("condition");

  mutex_lock(lock);
}
```
+ `void cond_wait(thread_cond_t* cond, thread_mutex_t* lock)`
  + Defined in `thread.c`
  + Increase `waiters` by 1
  + Protect the above increment by using `cond->mutex` to make it atomic
  + Unlock the given lock temporarily
    + If not, `cond_sleep()` can get multiple interrupts which result in panic
  + Calling thread go to sleep by `cond_sleep()` with "condition" chan and lock the temporarily unlocked lock

```c
void
cond_sleep(void *chan)
{
  struct proc *curproc = myproc();
  struct thread *curthd = mythd();

  if(curproc == 0 || curthd == 0)
    panic("cond_sleep");


  acquire(&ptable.lock);  //DOC: sleeplock1
  // Go to sleep.
  curthd->chan = chan;
  setstate(curproc, curthd, SLEEPING);

  sched();

  // Tidy up.
  curthd->chan = 0;

  // Reacquire original lock.
  release(&ptable.lock);
}
```
+ `void cond_sleep(void *chan)`
  + Defined in `thread.c`
  + Does the same thing as the original `sleep()` in xv6 does, but doesn't receive the locking variable as a parameter

```c
void
cond_signal(thread_cond_t* cond)
{
  if(cond->waiters == 0) {
    return;
  }
  mutex_lock(cond->mutex);
  cond->waiters--;
  mutex_unlock(cond->mutex);
  
  cond_wakeup("condition");
}
```
+ `void cond_signal(thread_cond_t* cond)`
  + Defined in `thread.c`
  + If the number of waiter for the given condition variable is 0, do nothing
  + Decrease `waiters` by 1
  + Protect the above decrement by using `cond->mutex` to make it atomic
  + Wakeup one of the sleeping thread with "condition" chan by `cond_wakeup()`

```c
void
cond_wakeup(void *chan)
{
  acquire(&ptable.lock);

  struct thread *t;
  for(t = ptable.thread; t < &ptable.thread[NTHREAD]; t++) {
    if(t->state == SLEEPING && t->chan == chan) {
      setstate(t->mp, t, RUNNABLE);
      break;
    }
  }

  release(&ptable.lock);
}
```
+ `void cond_wakeup(void *chan)`
  + Defined in `thread.c`
  + The original `wakeup()` in xv6 wakeups all the sleeping thread with the given chan
  + But, `cond_wakeup()` just wakeups 1 sleeping thread and then break the for loop


# Implement the Basic Semaphore
```c
typedef struct __xem_t {
  int value;
  thread_cond_t cond;
  thread_mutex_t lock;
} xem_t;
```
+ `xem_t`
  + Defined in `types.h`
  + `int value`: an integer value of semaphore
  + `thread_cond_t cond`: condition variable of semaphore
  + `thread_mutex_t`: mutex of semaphore

```c
int
xem_init(xem_t *semaphore)
{
  semaphore->value = 1;

  if(mutex_init(&semaphore->lock) == -1) {
    return -1;
  }

  if(cond_init(&semaphore->cond) == -1) {
    return -1;
  }
  
  return 0;
}
```
+ `int xem_init(xem_t *semaphore)`
  + Defined in `thread.c`
  + Initial value of `value` is 1
  + Initialize `lock` and `cond`

```c
int
xem_wait(xem_t *semaphore)
{
  mutex_lock(&semaphore->lock);
  while(semaphore->value <= 0) {
    cond_wait(&semaphore->cond, &semaphore->lock);
  }
  semaphore->value--;
  mutex_unlock(&semaphore->lock);
  
  return 0;
}
```
+ `int xem_wait(xem_t *semaphore)`
  + Defined in `thread.c`
  + Protect the whole section by using `semaphore->lock` to make it atomic
  + Wait until `value` becomes 1 by using `cond_wait()`
  + It will cause the caller to suspend execution waiting for a subsequent unlock
  + If `value` becomes 1, decrease `value` by 1 and acquire the semaphore

```c
int
xem_unlock(xem_t *semaphore)
{
  mutex_lock(&semaphore->lock);
  semaphore->value++;
  cond_signal(&semaphore->cond);
  mutex_unlock(&semaphore->lock);

  return 0;
}
```
+ `int xem_unlock(xem_t *semaphore)`
  + Defined in `thread.c`
  + Protect the whole section by using `semaphore->lock` to make it atomic
  + Increase `value` by 1
  + If there is a thread waiting to be woken, wakes one of them up by using `cond_signal()`

# Implement the Readers-writer Lock using Semaphores

```c
typedef struct __rwlock_t {
  xem_t lock;
  xem_t writelock;
  int readers;
} rwlock_t;
```
+ `rwlock_t`
  + Defined in `types.h`
  + `xem_t lock`: normal semaphore of read-write lock
  + `xem_t writelock`: semaphore for writer, but reader also can acquire this
  + `int readers`: the number of readers

```c
int
rwlock_init(rwlock_t *rwlock)
{
  rwlock->readers = 0;
  if(xem_init(&rwlock->lock) == -1) {
    return -1;
  }
  if(xem_init(&rwlock->writelock) == -1) {
    return -1;
  }

  return 0;
}
```
+ `int rwlock_init(rwlock_t *rwlock)`
  + Defined in `thread.c`
  + Initial value of `readers` is 0
  + Initialize `lock` and `writelock`

```c
int
rwlock_acquire_readlock(rwlock_t *rwlock)
{
  xem_wait(&rwlock->lock);
  rwlock->readers++;
  if(rwlock->readers == 1) {
    xem_wait(&rwlock->writelock);
  }
  xem_unlock(&rwlock->lock);

  return 0;
}
```
+ `int rwlock_acquire_readlock(rwlock_t *rwlock)`
  + Defined in `thread.c`
  + Protect the whole section by using `rwlock->lock` to make it atomic
  + Increase the value of `readers`
  + If increased `readers` == 1, it means the calling thread is the first reader of the given rwlock
    + It means, the calling thread also has to acquire `writelock`
    + It can protect all group of readers from a writer

```c
int
rwlock_acquire_writelock(rwlock_t *rwlock)
{
  xem_wait(&rwlock->writelock);

  return 0;
}
```
+ `int rwlock_acquire_writelock(rwlock_t *rwlock)`
  + Defined in `thread.c`
  + Simply wait for the `writelock` to be released and acquire it

```c
int
rwlock_release_readlock(rwlock_t *rwlock)
{
  xem_wait(&rwlock->lock);
  rwlock->readers--;
  if(rwlock->readers == 0) {
    xem_unlock(&rwlock->writelock);
  }
  xem_unlock(&rwlock->lock);

  return 0;
}
```
+ `int rwlock_release_readlock(rwlock_t *rwlock)`
  + Defined in `thread.c`
  + Protect the whole section by using `rwlock->lock` to make it atomic
  + Decrease the value of `readers`
  + If decreased `readers` == 0, it means the calling thread is the last reader of the given rwlock
    + It means, the calling thread also has to release `writelock`
    + So that a waiting writer can acquire the `writelock`

```c
int
rwlock_release_writelock(rwlock_t *rwlock)
{
  xem_unlock(&rwlock->writelock);

  return 0;
}
```
+ `int rwlock_release_writelock(rwlock_t *rwlock)`
  + Defined in `thread.c`
  + Simply release the `writelock`