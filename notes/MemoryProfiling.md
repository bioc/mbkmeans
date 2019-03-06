# Summary of Memory profiling options

Note for memory profiling, we have to remember that the memory may not scale naturally with the size of the data due to caching. Small datasets may be able to cache large portions of the data for small datasets, even if they are in hd5 files and not loaded into memory in R ((http://htmlpreview.github.io/?https://github.com/berkeley-scf/tutorial-efficient-R/blob/master/efficient-R.html).

>> In addition to main memory (what we usually mean when we talk about RAM), computers also have memory caches, which are small amounts of fast memory that can be accessed very quickly by the processor. For example your computer might have L1, L2, and L3 caches, with L1 the smallest and fastest and L3 the largest and slowest. The idea is to try to have the data that is most used by the processor in the cache.
>> If the next piece of data needed for computation is available in the cache, this is a cache hit and the data can be accessed very quickly. However, if the data is not available in the cache, this is a cache miss and the speed of access will be a lot slower. Cache-aware programming involves writing your code to minimize cache misses. Generally when data is read from memory it will be read in chunks, so values that are contiguous will be read together.

## Memory usage Outside of R (useful for Parallel computing).

From Chris (email): 
>> One thing that might complicate matters is that mclapply forks the master process, and it turns out that unless an object is changed by the worker process, the worker process 'copy' of the object is just a pointer to the original object. This can confuse things  because although top and gc() will probably report memory being used by the worker, it's not really extra memory above-and-beyond that used by the master process. 

Chris recommended adding a one-line entry to my shell script using `free` to record the memory usage of the machine every x minutes. I wrote a little helper function (`readMemoryLog.R`) that is part of the gitrepos for `clusterExperiment` to read that log and give a summary. Here's an example of shell script where I did that every 15 seconds (this was a long-running code).

```
#!/bin/bash
#SBATCH --job-name=oeData
#SBATCH --cpus-per-task 1
CURRDATE="$(date +'%Y_%m_%d_%R')"
FILE="OEAnalysisV3"
TAG="OnlyPlots"
BASEFILE="${FILE}_${TAG}_${CURRDATE}"
MEMORYFILE="${BASEFILE}_memoryLogger.txt"
MEMORYSUMMARY="${BASEFILE}_memorySummary.txt"
RFILE="${FILE}.R"
ROUT="${BASEFILE}.Rout"

while true; do free -h >> $MEMORYFILE; sleep 15; done &
R-3.5.0 CMD BATCH --vanilla $RFILE $ROUT
Rscript ../../clusterExperiment/tests/checkClusterMany/readMemoryLog.R $MEMORYFILE > $MEMORYSUMMARY
```

## Built-in R (base/utils package)

- `print(gc())` at the end of the R code, you should be able to see the maximum memory use by the R session (and the current memory use at the top gc() is called).
- Object size: `object.size` tells you the size of a single object. 
- `Rprof(memory.profiling = TRUE)` captures memory usage frequently (Hadley: "profiling can, at best, capture memory usage every 1 ms"). Running `gctorture` forces gc after every memory allocation, which can help to slow down a program. It can capture different aspects (you have to turn on the memory profiling). From help of `summaryRprof`: 

>> When called with memory.profiling = TRUE, the profiler writes information on three aspects of memory use: vector memory in small blocks on the R heap, vector memory in large blocks (from malloc), memory in nodes on the R heap. It also records the number of calls to the internal function duplicate in the time interval. duplicate is called by C code when arguments need to be copied. Note that the profiler does not track which function actually allocated the memory.
>> If the code being run has source reference information retained (via keep.source = TRUE in source or KeepSource = TRUE in a package ‘DESCRIPTION’ file or some other way), then information about the origin of lines is recorded during profiling. By default this is not displayed, but the lines parameter can enable the display.

`Rprof`, you need to set up a file for saving the output of the profile, start it before running commands, then end it and read the output. Example of usage of `Rprof` (from `proftools`)
```
profout <- tempfile()
Rprof(file = profout, gc.profiling = TRUE, line.profiling = TRUE)
source(srcfile)
Rprof(NULL)
pd <- readProfileData(profout)
unlink(profout)
```

From Chris' tutorials (http://htmlpreview.github.io/?https://github.com/berkeley-scf/tutorial-efficient-R/blob/master/efficient-R.html): 
>> Warning: Rprof conflicts with threaded linear algebra, so you may need to set OMP_NUM_THREADS to 1 to disable threaded linear algebra if you profile code that involves linear algebra

- `summaryRprof` Summarise the output of the `Rprof` function to show the amount of time used by different R functions.

- `tracemem` This function marks an object so that a message is printed whenever the internal code copies the object (interacts poorly with knitr). For interactive use (difficult to program with it). `untracemem` undoes it.
- `Rprofmem` Profiling writes the call stack to the specified file every time malloc is called to allocate a large vector object or to allocate a page of memory for small objects. The size of a page of memory and the size above which malloc is used for vectors are compile-time constants, by default 2000 and 128 bytes respectively. Similar usage as `Rprof`. 

Comparison to `Rprof`  (from package `profmem` man pages):

>> In addition to utils::Rprofmem(), R also provides utils::Rprof(memory.profiling = TRUE). Despite the close similarity of their names, they use completely different approaches for profiling the memory usage. As explained above, the former logs all individual (allocVector3()) memory allocation whereas the latter probes the total memory usage of R at regular time intervals. If memory is allocated and deallocated between two such probing time points, utils::Rprof(memory.profiling = TRUE) will not log that memory whereas utils::Rprofmem() will pick it up. On the other hand, with utils::Rprofmem() it is not possible to quantify the total memory usage at a given time because it only logs allocations and does therefore not reflect deallocations done by the garbage collector.




## pryr

- `mem_used()` tells total size of all objects in memory. From Advanced R: 

>> This number won’t agree with the amount of memory reported by your operating system for a number of reasons:
>>
>> * It only includes objects created by R, not the R interpreter itself.
>> * Both R and the operating system are lazy: they won’t reclaim memory until it’s actually needed. R might be holding on to memory because the OS hasn’t yet asked for it back.
>> * R counts the memory occupied by objects but there may be gaps due to deleted objects. This problem is known as memory fragmentation.


- `mem_changed` tells you how memory changes during code execution, so that the input to the function would be an excecution.
- `address` returns the memory location of the object, and `refs` returns the number of references pointing to the underlying object, alternative to `tracemem` for when you copied versus modifying in place. But seems rather limited usage based on Hadley's description of it's quirks:

>> refs() is only an estimate. It can only distinguish between one and more than one reference (future versions of R might do better) [i.e. returns either 1 or 2]...Note that if you’re using RStudio, refs() will always return 2: the environment browser makes a reference to every object you create on the command line")...Non-primitive functions that touch the object always increment the ref count

Seems like in many cases ref could be 2 even without copying, since non-primitive functions with increment count.


## lineprof (Hadley)

This uses `Rprof`, but nicer feed back and pairs it up with your code. It requires that you use source to load the code, however. (http://adv-r.had.co.nz/memory.html#memory-profiling). 

>> Using lineprof is straightforward. source() the code, apply lineprof() to an expression, then use shine() to view the results. Note that you must use source() to load the code. This is because lineprof uses srcrefs to match up the code and run times. The needed srcrefs are only created when you load code from disk.

>> Next to the source code, four columns provide details about the performance of the code:
>>  * t, the time (in seconds) spent on that line of code (explained in measuring performance).
>>  * a, the memory (in megabytes) allocated by that line of code.
>>  * r, the memory (in megabytes) released by that line of code. While memory allocation is deterministic, memory release is stochastic: it depends on when the GC was run. This means that memory release only tells you that the memory released was no longer needed before this line.
>>  * d, the number of vector duplications that occurred. A vector duplication occurs when R copies a vector as a result of its copy on modify semantics.
>> You can hover over any of the bars to get the exact numbers. In this example, looking at the allocations tells us most of the story:

## proftools (Tierney)

Tools for having easier interface with `Rprof`, including a lot of graphical displays. Seems more advanced version of `lineprof`. 

## profmem (Henrick)

(From the package) The profmem() function uses the utils::Rprofmem() function for logging memory allocation events to a temporary file. The logged events are parsed and returned as an in-memory R object in a format that is convenient to work with. All memory allocations that are done via the native allocVector3() part of R's native API are logged, which means that nearly all memory allocations are logged. Any objects allocated this way are automatically deallocated by R's garbage collector at some point. Garbage collection events are not logged by profmem(). Allocations not logged are those done by non-R native libraries or R packages that use native code Calloc() / Free() for internal objects. Such objects are not handled by the R garbage collector.

In order for profmem() to work, R must have been built with memory profiling enabled. If not, profmem() will produce an error with an informative message.

`profmem` takes as an argument an expression. Seems a lighter version of `lineprof`, except it uses `Rprofmem` and not `Rprof`; seems many others use `Rprof` (by looking at differences in time); this seems to argue `Rprofmem` better for profiling, but if so not clear why others don't also use it. It may be because `Rprof` also profiles time usage, so that you can get it all in one profiling even if not as precise on the memory. (for our purpose we want total memory anyway, so `Rprofmem` isn't relevant). It does not line up the results with the code (so presumably not require source the expression), but does give the calls that correspond to the memory usage. 

## profvis
