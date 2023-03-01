## UID: 305787875
(IMPORTANT: Only replace the above numbers with your true UID, do not modify spacing and newlines, otherwise your tarfile might not be created correctly)

# Hash Hash Hash

A hash table that implements mutual exclusions (mutex) locks to refrain multiple threads from accessing a shared resource.


## Building

First and foremost make sure the repository is downloaded and you are within the respected
repository.

Then run the command on the command line:

```sh
make
```

This will create a `hash-table-tester` executable file.


## Running

To run the program, simply type into the command line:

```sh
./hash-table-tester -t 4 -s 50000
```

This line executes the executable file created from the **BUILDING** Section and changes
default values given 2 flags:

`-s` changes how many hash table entries to add per thread (default 25,000)
`-t` changes the number of threads to use (default 4)

Here is an example after running the following command:

```sh
$ ./hash-table-tester -t 4 -s 100000
Generation: 69,673 usec
Hash table base: 1,035,521 usec
  - 0 missing
Hash table v1: 2,028,074 usec
  - 0 missing
Hash table v2: 344,466 usec
  - 0 missing
```

 In this example, *Hash table base* runs at about 1.03 secs, while *Hash table v1* is noticeably slower in comparison at about 2.03 secs. On the otherhand, *Hash table v2* runs at about 0.344 secs, faster than both previous implementations.

Note: missing represents the amount of entries missing in the hash table, which by the objective of this code should return 0 for each.


## First Implementation

In the first implementation, the only task was to handle locking with a single mutex, disregarding performance. As a result, my strategy was to ensure complete lock on the critical section, which occurs in the function *hash_table_v1_add_entry*. Since it only requires one single mutex for the entire hash table, I declared a global variable mutex:

```sh
static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```


Then to handle the locking feature, within the function *hash_table_v1_add_entry* I encompassed the function definition with the following code segment:

```sh
	int lockres = pthread_mutex_lock(mutex);
	if (lockres != 0)
	  {
	    exit(lockres);
	  }
	  
	...
	
	int unlockres = pthread_mutex_unlock(mutex);
	if (unlockres != 0)
	  {
	    exit(unlockres);
	  }

```
which ensures that the critical section isn't accessed unless the thread has the lock.

*Why the strategy is correct*
This approach works because the critical section is argueably the entire function *hash_table_v1_add_entry* definition as it allows multiple threads have access to the shared data segment. Therefore, creating a lock on this part of the implementation disables this type of undefined behavior and hence, O missing hash table entries from occurring.

To avoid unnecessary memory leaks, I added the following line in the function *hash_table_v1_destroy*:

```sh
	int destroy_res = pthread_mutex_destroy(&mutex);
```

which ensures the mutex created earlier is destroyed appropriately.

### Performance

*Test Case that completes in 1-2 seconds*
Consider the test case when the flags are `-t 2` and `-s 240000`:

```sh
$ ./hash-table-tester -t 2 -s 240000
Generation: 85,648 usec
Hash table base: 1,870,425 usec
  - 0 missing
Hash table v1: 2,990,122 usec
  - 0 missing
```

If we decrease the amount of threads to `-t 1` and `-s 480000`:

```sh
$ ./hash-table-tester -t 1 -s 480000
Generation: 83,522 usec
Hash table base: 1,724,289 usec
  - 0 missing
Hash table v1: 2,078,140 usec
  - 0 missing
```
Which shows that less threads decreases the time of both the *Hash table base* and the *Hash table v1* implementation.

Then notice that if we increase the amount of threads to `-t 4`:

```sh
$ ./hash-table-tester -t 4 -s 120000
Generation: 83,030 usec
Hash table base: 2,128,762 usec
  - 0 missing
Hash table v1: 3,914,614 usec
  - 0 missing
```
*Relative Speedup or Slow Down wrt Threads*

As more threads are introduced, the time of both *Hash table base* and *Hash table v1* will increase, however Hash table v1 increase at a faster rate, which is a result of implementing a single mutex, effecting the performance directly as shown.

This is mainly because a singly mutex locks up the critical section for 1 process, the one holding the lock. As a result, other threads are forced to wait until the lock is given up to then obtain the lock and do its work, hence introducing extra overhead.
Hence, the v1 implementation with a single mutex proves to be inefficient with respect to time as it slows down the more threads introduced to it. This is primarily because the single mutex adds overhead when extra threads are introduced, since they would be forced to wait until given access to the shared resource.

The main difference between low threads and high threads for the 1st implementation is that increasing the threads in fact adds extra overhead, which implies slower performance. Nonetheless, the 1st implementation is slower than the base implementation because even at 1 thread, the 1st implementation adds extra overhead for the mutex.


## Second Implementation

For the *Hash table v2* we were tasked with not only addressing the missing hash table entries, but also improving performance. As a result, a single mutex will not suffice.

We must result to multiple mutex in order to incentivize parallelism of other threads, while some threads block their current list.

Hence the approach I took was to create a mutex for each list within the hashtable, since the critical section occurs when multiple threads access the same *list*. Hence, creating a lock for that list would prevent other threads from accessing that list, but enable other threads to run on different lists concurrently.

Therefore, the implementation approach I took was first:

I placed `pthread_mutex_t mutex` within the `hash_table_entry` struct and
had it initialize each one within the `hash_table_v2_create` function via 
```sh
		int init_res = pthread_mutex_init(&entry->mutex, NULL);
		if (init_res != 0)
		  {
		    exit(init_res);
		  }
```

which if an error occurs, exits with the corresponding error code.

Then within  *hash_table_v2_add_entry*, I create a lock which locks that specific hash table entry list.
Therefore, multiple threads can work on different buckets, given that they have the lock for the corresponding list.

Finally to avoid memory leaks, I implemented a call to the `pthread_mutex_destroy()` function to loop through and destroy the mutex for each hash table entry.
*Why strategy is correct*

This is the correct strategy to implement as it doesn't monopolize the critical section just for one hash table entry, but instead locks the corresponding hash table entry list and enables other threads to work on other hash table entries.
As a result, this 2nd implementation sees most benefit as threads are increased, since the more threads allows concurrency to neglect the extra overhead provided by the locks. This is seen in the following section when analyzing the performance of the second implementation.


### Performance

Consider the test with the corresponding flags:

```sh
$ ./hash-table-tester -t 2 -s 200000
Generation: 68,805 usec
Hash table base: 1,118,084 usec
  - 0 missing
Hash table v2: 672,656 usec
  - 0 missing
```
Which notice that *Hash table v2* is faster than the base case.

Then lets increase the threads to `-t 4`:
```sh
$ ./hash-table-tester -t 4 -s 100000
Generation: 68,649 usec
Hash table base: 1,099,480 usec
  - 0 missing
Hash table v2: 341,071 usec
  - 0 missing
```

Which *Hash table v2* is increasingly getting faster than *Hash table base*.

*Speed up relative to base hash table and thread increase*

This speed up is happening as threads increase primarily because the *Hash table v2* utilizes multiple mutexes to take advantage of concurrency. As before, locks provide extra overhead, which
ends up slowing down procedures. However notice that it is not in the case of *Hash table v2*. This is because as more threads are introduced, concurrency can outweigh the extra overhead. As a result, the 2nd implementation can finish faster than the base case because threads can work independently from each other since multiple mutexes are used to lock corresponding lists of Hash Entries, rather than locking the entire list like *Hash table v1* did.

*Difference between 1st and 2nd implementation*

 The first implementation resorts to a single mutex. This effects the performance a lot as the single mutex locks the entire hash table from other threads, but the second implementation only locks a single hash table *entry*, in other words a list, allowing other threads to access other hash table entries. As such, the 2nd implementation is shown to perform faster since it allows concurrency. 
Hence, the 2nd implementation actually speeds up the process a lot as more threads are introduced, while the 1st implementation actually slows down the process because more threads end up adding to the extra overhead of a single mutex. 

## Cleaning up

To clean up the binaries, simply run the following command on the command line:
```sh
make clean
```

This would remove the executable created earlier from the *MAKING* section and cleans up your work space.
