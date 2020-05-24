---
title: "Understanding concurrency with asyncio"
date: 2019-09-13T12:17:41-05:00
draft: false
toc: false
images:
tags:
  - python
  - asyncio
  - concurrency
---

## Introduction
I/O is slow. In fact, in most cases, it's orders of magnitude slower than anything else your program is doing. And that can cause a program that executes sequentially to waste a lot of resources waiting on high latency i/o. A common example of this is a network operation. In the example below, the program is blocked from continuing its flow of execution until the https request is completed.
``` python
import requests

resp = requests.get('https://www.example.com/')
# program is blocked here until request is completed

print(resp.status_code)
```
This post is an attempt at understanding how Python3's `asyncio` tackles this problem using concurrency. There's multiple ways to minimize the execution time of such programs, by allowing them to execute other tasks while waiting for the request to respond. And they generally fall into two major categories: concurrency and parallelism. You can think of concurrency as dealing with multiple things at the same time and parallelism *doing* multiple things at the same time. The former has overlapping time periods of execution while the latter does simultaneous execution. There are tons of articles and blog posts about this so I won't go into detail here. Parallelism is implemented in Python with the `multiprocessing` module and concurrency with `threading` and `asyncio`.

I find asyncio's design particularly interesting because multiprocessing and multithreading have large resource overheads while asyncio is a single-threaded, single-process design which uses cooperative multitasking. In cooperative multitasking, tasks yield to the scheduler as opposed to preemptive multitasking where the scheduler interrupts tasks. If you want to know how asyncio works under the hood, you can check out [this detailed article](http://aosabook.org/en/500L/a-web-crawler-with-asyncio-coroutines.html) written by A. Jesse Jiryu Davis and Guido van Rossum, and [another one](https://snarky.ca/how-the-heck-does-async-await-work-in-python-3-5/) by Brett Cannon.

## Example
The `asyncio` module provides tools for building concurrent applications using an event loop and coroutines. Coroutines are functions that can be suspended and resumed while being executed. On *await* they release control back to the event loop. The event loop schedules concurrent tasks and manages their execution. Let's have a look at a simple example to illustrate how this pattern can be implemented at a high level:

``` Python
import asyncio
import datetime
import random
import time


def run_task(task_id):
    start = datetime.datetime.now()
    print(f"Starting task{task_id} at {start.strftime('%H:%M:%S')}")
    # simulate i/o operation using sleep
    time.sleep(random.random())
    finish = datetime.datetime.now()
    timer = finish - start
    print(f"Task{task_id} completed at {finish.strftime('%H:%M:%S')}, in {timer.total_seconds():.2f} seconds.")

async def coro(task_id):
    start = datetime.datetime.now()
    print(f"Starting task{task_id} at {start.strftime('%H:%M:%S')}")
    # simulate i/o operation using sleep
    await asyncio.sleep(random.random())
    finish = datetime.datetime.now()
    timer = finish - start
    print(f"Task{task_id} completed at {finish.strftime('%H:%M:%S')}, in {timer.total_seconds():.2f} seconds.")

def run_all_tasks():
    print("Running tasks sequentially... ")
    start = time.time()
    for i in range(1, 11):
        run_task(i)
    print(f"Process took: {time.time() - start : .2f} seconds. \n")

async def main():
    print("Running tasks concurrently... ")
    start = time.time()
    tasks = [asyncio.create_task(coro(i)) for i in range(1, 11)]
    await asyncio.gather(*tasks)
    print(f"Process took: {time.time() - start : .2f} seconds.")


run_all_tasks()
asyncio.run(main())

```
Output:
```
$ python example.py
Running tasks sequentially...
Starting task1 at 21:48:30
Task1 completed at 21:48:31, in 0.89 seconds.
Starting task2 at 21:48:31
Task2 completed at 21:48:31, in 0.26 seconds.
Starting task3 at 21:48:31
Task3 completed at 21:48:31, in 0.21 seconds.
Starting task4 at 21:48:31
Task4 completed at 21:48:32, in 0.59 seconds.
Starting task5 at 21:48:32
Task5 completed at 21:48:33, in 0.86 seconds.
Starting task6 at 21:48:33
Task6 completed at 21:48:33, in 0.04 seconds.
Starting task7 at 21:48:33
Task7 completed at 21:48:33, in 0.29 seconds.
Starting task8 at 21:48:33
Task8 completed at 21:48:34, in 0.58 seconds.
Starting task9 at 21:48:34
Task9 completed at 21:48:34, in 0.28 seconds.
Starting task10 at 21:48:34
Task10 completed at 21:48:34, in 0.38 seconds.
Process took:  4.37 seconds.

Running tasks concurrently...
Starting task1 at 21:48:34
Starting task2 at 21:48:34
Starting task3 at 21:48:34
Starting task4 at 21:48:34
Starting task5 at 21:48:34
Starting task6 at 21:48:34
Starting task7 at 21:48:34
Starting task8 at 21:48:34
Starting task9 at 21:48:34
Starting task10 at 21:48:34
Task3 completed at 21:48:35, in 0.37 seconds.
Task5 completed at 21:48:35, in 0.40 seconds.
Task9 completed at 21:48:35, in 0.61 seconds.
Task4 completed at 21:48:35, in 0.64 seconds.
Task1 completed at 21:48:35, in 0.70 seconds.
Task8 completed at 21:48:35, in 0.70 seconds.
Task7 completed at 21:48:35, in 0.76 seconds.
Task6 completed at 21:48:35, in 0.79 seconds.
Task10 completed at 21:48:35, in 0.79 seconds.
Task2 completed at 21:48:35, in 0.90 seconds.
Process took:  0.90 seconds.
```

The first thing to note here is that when we run the tasks asynchronously, even though we're starting all of the tasks at the same time the order of execution and completion depend on the random number of seconds in each coroutine, i.e. the latency of i/o in each coroutine. The second (and more important) thing to note is that when running sequentially, the process took at least as much time as all 10 tasks combined while in asynchronous execution the process took about as long as the slowest task. As the number of tasks increases, running them asynchronously becomes exponentially faster. To demonstrate this, let's run the same function and coroutine using 100 tasks, while reducing the sleep time by a factor of 10.

``` Python
import asyncio
import datetime
import random
import time


def run_task(task_id, factor):
    start = datetime.datetime.now()
    print(f"Starting task{task_id} at {start.strftime('%H:%M:%S')}")
    # simulate i/o operation using sleep
    time.sleep(random.random()*factor)
    finish = datetime.datetime.now()
    timer = finish - start
    print(f"Task{task_id} completed at {finish.strftime('%H:%M:%S')}, in {timer.total_seconds():.3f} seconds.")

async def coro(task_id, factor):
    start = datetime.datetime.now()
    print(f"Starting task{task_id} at {start.strftime('%H:%M:%S')}")
    # simulate i/o operation using sleep
    await asyncio.sleep(random.random()*factor)
    finish = datetime.datetime.now()
    timer = finish - start
    print(f"Task{task_id} completed at {finish.strftime('%H:%M:%S')}, in {timer.total_seconds():.3f} seconds.")

def run_all_tasks(how_many):
    print("Running tasks sequentially... ")
    start = time.time()
    for i in range(1, how_many + 1):
        run_task(i, 10/how_many)
    print(f"Process took: {time.time() - start : .2f} seconds. \n")

async def main(how_many):
    print("Running tasks concurrently... ")
    start = time.time()
    tasks = [asyncio.create_task(coro(i, 10/how_many)) for i in range(1, how_many + 1)]
    await asyncio.gather(*tasks)
    print(f"Process took: {time.time() - start : .2f} seconds.")


run_all_tasks(100)
asyncio.run(main(100))

```
Output:
```
$ python example.py
Running tasks sequentially...
Starting task1 at 00:15:21
Task1 completed at 00:15:21, in 0.025 seconds.
Starting task2 at 00:15:21
Task2 completed at 00:15:21, in 0.050 seconds.
Starting task3 at 00:15:21
Task3 completed at 00:15:21, in 0.049 seconds.
Starting task4 at 00:15:21
Task4 completed at 00:15:22, in 0.069 seconds.
Starting task5 at 00:15:22
Task5 completed at 00:15:22, in 0.006 seconds.
Starting task6 at 00:15:22
Task6 completed at 00:15:22, in 0.067 seconds.
Starting task7 at 00:15:22
Task7 completed at 00:15:22, in 0.090 seconds.
.
.
.
Task94 completed at 00:15:26, in 0.026 seconds.
Starting task95 at 00:15:26
Task95 completed at 00:15:26, in 0.001 seconds.
Starting task96 at 00:15:26
Task96 completed at 00:15:26, in 0.096 seconds.
Starting task97 at 00:15:26
Task97 completed at 00:15:26, in 0.034 seconds.
Starting task98 at 00:15:26
Task98 completed at 00:15:26, in 0.065 seconds.
Starting task99 at 00:15:26
Task99 completed at 00:15:26, in 0.038 seconds.
Starting task100 at 00:15:26
Task100 completed at 00:15:26, in 0.055 seconds.
Process took:  4.61 seconds.

Running tasks concurrently...
Starting task1 at 00:15:26
Starting task2 at 00:15:26
Starting task3 at 00:15:26
Starting task4 at 00:15:26
Starting task5 at 00:15:26
Starting task6 at 00:15:26
Starting task7 at 00:15:26
Starting task8 at 00:15:26
Starting task9 at 00:15:26
Starting task10 at 00:15:26
Starting task11 at 00:15:26
Starting task12 at 00:15:26
.
.
.
Task81 completed at 00:15:26, in 0.087 seconds.
Task30 completed at 00:15:26, in 0.090 seconds.
Task80 completed at 00:15:26, in 0.087 seconds.
Task88 completed at 00:15:26, in 0.089 seconds.
Task62 completed at 00:15:26, in 0.092 seconds.
Task63 completed at 00:15:26, in 0.094 seconds.
Task60 completed at 00:15:26, in 0.096 seconds.
Task55 completed at 00:15:26, in 0.098 seconds.
Task31 completed at 00:15:26, in 0.100 seconds.
Process took:  0.10 seconds.

```
As we can see, the speedup is even greater with more tasks. About 5x when running 10 tasks vs 46x when running 100. It's also worth noting that at really high numbers of tasks, the 'process time' of the program will actually be higher than the longest running task because of context switching. Nonetheless, the asynchronous program will still be much faster.

## Conclusion

That's it for this post. Some key things to keep in mind as you code:

* Simply calling a coroutine will not schedule it to be executed
* Everything you call from a coroutine should be non blocking or else you risk stalling the entire system. e.g. avoid doing large computations, or blocking network calls
* asyncio does not magically make things non-blocking. You have explicitly yield to the event loop where i/o is performed
