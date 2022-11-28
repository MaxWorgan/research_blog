---
aliases:
- /Julia/Memory/OpenFrameworks/POSIX/2022/03/15/shared-data
categories:
- Julia
- OpenFrameworks
- Memory
- POSIX
date: '2022-03-15'
author: Max Worgan
description: Using the POSIX shared memory API to allow shared access to swarm data
layout: post
title: Sharing matrix data between OpenFrameworks and Julia
toc: true

---

# The problem

My current project incorporates a swarming simulation written in C++ using the [OpenFrameworks](http://openframeworks.cc) toolkit. All my machine learning research has been written in [Julia](https://julialang.org/). Since I require real-time analysis of the swarm data, I need to send data from my simulation into julia for processing. There are a number of ways to go about this:

1. Network based data exchange options:
    - Something like [Open Sound Control](https://en.wikipedia.org/wiki/Open_Sound_Control) which is commonly used for musical data exchange although not typically intended for large quantities of data. Also it is UDP based, so depending on network configuration/stability data could be lost.
    - A custom data exchange protocol - quite a lot of implementation/testing testing time involved.
    - Any network based approach would require serialization/de-serialization which is likely to make it prohibitively costly given the quantity of data we are attempting to transmit.
2. A shared data file
    - Would require the simulation to write to a file, while julia reads from the file. Synchronization mechanism would be needed. Seems wasteful.
3. IPC (Inter Process Communication) e.g. Shared memory
    - If the raw data from the simulation could be shared with the Julia process via a shared memory address then it would be a very efficient, since no serialization, disk IO, or network IO, would be required. Synchronization would still need to be managed to avoid threading issues.


# Shared Memory
There are two commonly used methods for directly sharing memory between two processes: [System V's IPC mechanisms](https://man7.org/linux/man-pages/man7/sysvipc.7.html), which includes shared memory, and [POSIX shared memory](https://man7.org/linux/man-pages/man7/shm_overview.7.html). 

System V's implementation is more fully featured but deprecated, so we will consider the POSIX implementation only.

## POSIX shared memory

The primary function for creating shared memory is via the [shm_open](https://man7.org/linux/man-pages/man3/shm_open.3.html) call:

```c
int shm_open(const char* name, int flags,  mode_t mode);
```
This creates a new (or opens and existing) shared memory object and returns a file descriptor. In the case of of it creating a new shared memory object, it will have a default size of 0 bytes. The call [ftruncate](https://man7.org/linux/man-pages/man3/ftruncate.3p.html) can be used to set the appropriate size of the shared memory object:

```c
int ftruncate(int file_descriptor, off_t memory_size);
```
This simply returns 0 on success and -1 otherwise.

Once the shared memory object has been setup and initialized then the data needs to be mapped into this address space using the call [mmap](https://man7.org/linux/man-pages/man2/mmap.2.html)

```
void* mmap(void *addr, size_t memory_size, int prot, int flags, int fd, off_t offset);
```
In our case the `address` parameter is simply set to `NULL` since we don't care where the kernel uses memory from. The `prot` argument describes the desired memory protection of the mapping, whether it can be read, written executed. The `flags` argument specifies (among other things) if this mapping is shared across processes, or is private. In our case we set it to `MAP_SHARED`. `fd` is the file descriptor of the shared memory object, and `offset` is unused in our case.

There are also the calls [munmap](https://man7.org/linux/man-pages/man2/mmap.2.html) and [shm_unlink](https://man7.org/linux/man-pages/man3/shm_unlink.3p.html) used to clean up allocated resources.

## POSIX Mutex
Since the simulation will be writing to the shared memory and the julia process will be reading concurrently, we still need to synchronise their access to the shared memory object. For this the POSIX thread library provides an implementation of a mutex.

For a general overview I found this website helpful: 
https://riptutorial.com/posix/example/15910/simple-mutex-usage

One additional implementation detail is that we're going to use shared memory again to store this mutex, meaning that both the simulation and
 the julia process can use it. This means more calls to `shm_open`, `ftruncate` and `mmap`.

 This is in addition to the usual POSIX mutex initialisation, including:

- [pthread_mutex_init](https://man7.org/linux/man-pages/man3/pthread_mutex_init.3p.html)

 - [pthread_mutexattr_setpshared](https://man7.org/linux/man-pages/man3/pthread_mutexattr_getpshared.3.html)


# OpenFrameworks
Since OpenFrameworks is written in C++, the POSIX shared memory API and POSIX threads interfaces are easily accessible, at least on Linux/OSX.

To make matters even more straightforward I found an OpenFrameworks addon [ofxSharedMemory](https://github.com/trentbrooks/ofxSharedMemory) which implements the POSIX shared memory part. I adapted this implementation for my use by including the pthreads logic to syncronise the access to the shared memory. 

# Julia
Julia has easy C/C++ interop through the [`ccall`](https://docs.julialang.org/en/v1/base/c/#ccall) function making the implementation fairly potentially straightforward. 

Helpfully I found the [InterProcessCommunicaltion.jl](https://github.com/emmt/InterProcessCommunication.jl) package which not only provided the shared memory implementation, but also a mutex implementation. There were however, issues on my OSX system with calls to deprecated and missing libraries. However, fairly minor changes allowed me to use this package, including their [WrappedArrays](https://emmt.github.io/InterProcessCommunication.jl/dev/reference/#Wrapped-arrays-1) datatype, which made coercing the shared data into a Matrix structure utterly painless.

Full code will be available  on [my Github](http://github.com/MaxWorgan) soon!
