# Project 1: Analysis & Reporting


### 1. Which program is fastest? Is it always the fastest?

With really low numbers on either `MAX_BYTES` or `NUM_BLOCKS` **alloca.out** is always the fastest. However when larger numbers for either variable come into play, **alloca.cpp** has a `segmentation fault (core dump)` which likely has something to do with its reliance on the stack and its recursive nature. The second fastest is **malloc.cpp** no matter the number of blocks or the number or the number of bytes. 

### 2. Which program is slowest? Is it always the slowest?

**list.cpp** is the slowest program, but **new.cpp** is only marginally faster than **list.cpp**. 

### 3. Was there a trend in program execution time based on the size of data in each Node? If so, what, and why?

The larger the number of bytes stored in each `Node`, the slower the execution time. Since there is more data being put in each node it takes much longer to allocate memory and process/hash.

### 4. Was there a trend in program execution time based on the length of the block chain?

The more `Node`s in the block chain, the slower the execution time. Since there were more nodes to initialize and allocate memory for, the process took longer.

### 5. Consider heap breaks, what's noticeable? Does increasing the stack size affect the heap? Speculate on any similarities and differences in programs?

```
% make clean breaks NUM_BLOCKS=2 MIN_BYTES=6 MAX_BYTES=6
alloca.out:        69  
list.out:          69  
malloc.out:        69  
new.out:           69
```

The minimum number of heap breaks seems to be 69. When I increase the number of blocks and bytes…

```
% make clean breaks NUM_BLOCKS=2000 MIN_BYTES=60 MAX_BYTES=60
alloca.out:        69
list.out:          71
malloc.out:        71
new.out:           71
```

…**alloca.cpp** stays the same while the rest increase the number of heap breaks at an equal pace to a certain point. After increasing the number of blocks even more…

```
% make clean breaks NUM_BLOCKS=2000000 MIN_BYTES=60 MAX_BYTES=60
alloca.out:        66
list.out:          2330
malloc.out:        1979
new.out:           2330
```

…**malloc.cpp** reduces its need for heap breaks compared to **new.cpp** and **list.cpp**. It's likely that **alloca.cpp** has fewer heap breaks than the minimum because the program does not finish.

```
% make clean breaks NUM_BLOCKS=2000 MIN_BYTES=600000 MAX_BYTES=600000
alloca.out:        66
list.out:          4070
malloc.out:        4069
new.out:           4070
```

The idea that **alloca.cpp**’s reduction in heap breaks due to it not finishing is supported by making the number of bytes very large instead of the number of blocks; both show 66 heap breaks, which is 3 less than when the program is able to finish. 

Another thing to mention is that **malloc.out**’s reduction in heap breaks is more significant when the number of blocks is increased compared to increasing the number of bytes. It’s likely that a compiler trick is responsible for the one less heap break in **malloc.cpp** when the number of bytes is so large.

### 6. Considering either the malloc.cpp or alloca.cpp versions of the program, generate a diagram showing two Nodes. Include in the diagram...
  - the relationship of the head, tail, and Node next pointers.
  - show the size (in bytes) and structure of a Node that allocated six bytes of data
  - include the bytes pointer, and indicate using an arrow which byte in the allocated memory it points to.



### 7. There's an overhead to allocating memory, initializing it, and eventually processing (in our case, hashing it). For each program, were any of these tasks the same? Which one(s) were different?

Hashing and initializing (`iota`) are all the same, but the memory allocation for each was different. **alloca.cpp** relied on the stack which is why it didn’t ever have more heap breaks than the minimum. **malloc.cpp** used malloc. **new.cpp** used `operator new` to string nodes into a linked list and `std::vector` to store the data. Finally, **list.cpp** used `std::list` to keep track of nodes and `std::vector` to store data. The reason why **list.cpp** and **new.cpp** were so slow was because they used vectors to store the data which greatly slowed things down.

### 8. As the size of data in a Node increases, does the significance of allocating the node increase or decrease?

The allocation of the overhead decreases in relative significance as the size of data increases per node. This allocation is fixed because there is only one `malloc` or one `new` call, one `next` pointer, one `bytes` pointer, etc. The hashing and initialization both scale linearly with data size. As bytes per node increases, the hashing and initialization cost dominates while the allocation becomes a smaller fraction of the total cost to run the program.
