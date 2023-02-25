## UID: 305787875
(IMPORTANT: Only replace the above numbers with your true UID, do not modify spacing and newlines, otherwise your tarfile might not be created correctly)

# Hash Hash Hash

A hash table that implements mutual exclusions (mutex) known as locks to refraim critical sections
taking an effect on the program when multiple threads are running asynchronously.

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

This line executes the executable file created from the **MAKE** Section and changes
default values given 2 flags:

`-s` changes how many hash table entries to add per thread (default 25,000)
`-t` changes the number of threads to use (default 4)

Here is an example after running the following command:

```sh
$ ./hash-table-tester -t 5 -s 50000
Generation: 51,764 usec
Hash table base: 1,449,545 usec
  - 0 missing
Hash table v1: 4,588,561 usec
  - 0 missing
Hash table v2: 521,157 usec
  - 0 missing
```

Notice that flags were passed effected both `-s` and `-t` default values to conform to the past in values. In this example, *Hash table base* runs at about 1.45 secs, while *Hash table v1* is noticeably slower in comparison at about 4.6 secs. On the otherhand, *Hash table v2* runs at about 0.52 secs, faster than both previous implementations.

Note: missing represents the amount of entries missing in the hash table, which by implementation should return 0 for each, as it was the main objective of this lab.


## First Implementation

In the first implementation, the only task was to handle locking with a single mutex, disregarding performance. As a result, my strategy was to ensure complete lock on the critical section, which occurs in the function *hash_table_v1_add_entry*. Since it only requires one single mutex for the entire hash table, I declared a global variable mutex:

```sh
static pthread_mutex_t mutex = PTHREAD_MUTEX_INITIALIZER;
```


Then to handle the locking feature, within the function *hash_table_v1_add_entry* I encompassed the entire function definition with the following code segment:

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
This approach works because the critical section is argueably the entire function *hash_table_v1_add_entry* as it allows multiple threads have access to the shared data segment. Therefore, creating a lock on this part of the implementation disables this type of undefined behavior and hence, O missing hash table entries from occurring.

To avoid unnecessary memory leaks, I added the following line in the function *hash_table_v1_destroy*:

```sh
	int destroy_res = pthread_mutex_destroy(&mutex);
```

which ensures the mutex created earlier is destroyed appropriately.

### Performance

Consider the test case when the flags are `-t 2` and `-s 140000`:

```sh
$ ./hash-table-tester -t 2 -s 140000
Generation: 58,446 usec
Hash table base: 1,842,972 usec
  - 0 missing
Hash table v1: 5,704,068 usec
  - 0 missing
```

If we decrease the amount of threads to `-t 1`:

```sh
$ ./hash-table-tester -t 1 -s 140000
Generation: 29,898 usec
Hash table base: 408,169 usec
  - 0 missing
Hash table v1: 586,110 usec
  - 0 missing
```
Which shows that less threads decreases the rate of both the *Hash table base* and the *Hash table v1* implementation.

Then notice that if we increase the amount of threads to `-t 4`:

```sh
$ ./hash-table-tester -t 4 -s 140000
Generation: 115,296 usec
Hash table base: 7,993,502 usec
  - 0 missing
Hash table v1: 17,421,749 usec
  - 0 missing
```

Which show that as more threads are introduced, the rate of both *Hash table base* and *Hash table v1* will increase, however Hash table v1 increase at a faster rate, which is a result of implementing a single mutex, effecting the performance directly as shown.


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

Then within the previous mentioned critical section, *hash_table_v2_add_entry*,
after the line of getting the hash table entry, I create a lock which locks that specific hash table entry, but not others.

Finally to avoid memory leaks, I implemented a call to the 
`pthread_mutex_destroy()` function to loop through and destroy the mutex for each hash table entry.

This is the correct strategy to implement as it doesn't monopolize the critical section just for one hash table entry, but instead locks the corresponding hash table entry and enables other threads to work on other hash table entries.


### Performance

Consider the test with the corresponding flags:

```sh
$ ./hash-table-tester -t 2 -s 140000
Generation: 58,649 usec
Hash table base: 1,923,465 usec
  - 0 missing
Hash table v2: 1,389,126 usec
  - 0 missing
```
which notice that *Hash table v2* is faster than the base case.

Then lets increase the threads to `-t 4`:
```sh
$ ./hash-table-tester -t 4 -s 140000
Generation: 115,052 usec
Hash table base: 8,099,767 usec
  - 0 missing
Hash table v2: 3,145,016 usec
  - 0 missing
```

Which *Hash table v2* is increasingly getting faster than *Hash table base*.

Notice that the second implementation utilizes *multiple* mutexes. While the first implementation resorts to a single mutex. This effects the performance tremondously as the single mutex locks the entire hash table from other threads, but the second implementation only locks a single hash table *entry*, allowing other threads to access other hash table entries. As such, it is quite intuitive as to why the second implementation increase the performance wrt the first implementation.


## Cleaning up

To clean up the binaries, simply run the following command on the command line:
```sh
make clean
```

This would remove all of these unnecessary binaries after use, and cleans up your work space.
