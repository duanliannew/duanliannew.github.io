```
// clang++ -std=c++20 thread_context_switch_cost.cpp -o thread_context_switch_cost
// ./thread_context_switch_cost

#include <stdio.h>  
#include <stdlib.h>  
#include <sys/time.h>  
#include <time.h>  
#include <sched.h>  
#include <sys/types.h>  
#include <unistd.h>      //pipe()  
  
int main() {
    // we will schedule two processes to run on the same processor
    int cpu_id = 1;
    cpu_set_t cpuset;
    CPU_ZERO(&cpuset);
    CPU_SET(cpu_id, &cpuset);

    // set the highest scheduling priority
    struct sched_param param;
    param.sched_priority = 0;

    // construct two pipes that will be shared by two processes
    int p1[2], p2[2];    
    pipe(p1);
    pipe(p2);

    const int ping_pong_times = 10000;
    char send = 's';  
    char receive;

    pid_t pid;
    while ((pid = fork()) == -1); 
    if (pid == 0) {
        sched_setaffinity(getpid(), sizeof(cpuset), &cpuset);
        sched_setscheduler(getpid(), SCHED_FIFO, &param); 
        for (int i = 0; i < ping_pong_times; i++) {  
            // read(p1[0], &receive, 1);  
            // write(p2[1], &send, 1);
            sched_yield();
        }  
        exit(0);  
    } else {
        sched_setaffinity(getpid(), sizeof(cpuset), &cpuset);
        sched_setscheduler(getpid(), SCHED_FIFO, &param);

        struct timeval start;
        gettimeofday(&start, NULL);  
        // printf("Before Context Switch Time: %ld s, %ld us\n", start.tv_sec, start.tv_usec); 
        for (int i = 0; i < ping_pong_times; i++) {  
            // write(p1[1], &send, 1);
            // read(p2[0], &receive, 1);
            sched_yield();
        }

        struct timeval end;
        gettimeofday(&end, NULL);
        // printf("After Context SWitch Time:  %ld s, %ld us\n", end.tv_sec, end.tv_usec);

        long time_us = (end.tv_sec - start.tv_sec)*1000000 + (end.tv_usec - start.tv_usec);
        printf("Average Context Switch time:%lf us\n", time_us*1.0/(2*ping_pong_times));
    }
    return 0;
}
```

```
// go run ./goroutine_context_switch_cost.go
package main

import (
	"fmt"
	"runtime"
	"sync"
	"time"
)

func main() {
	runtime.GOMAXPROCS(1)
	wg := sync.WaitGroup{}

	loop := 10000
	run := func() {
		for i := 0; i < loop; i++ {
			runtime.Gosched()
		}
		wg.Done()
	}

	start := time.Now()
	wg.Add(2)
	go run()
	run()
	wg.Wait()
	elapsed := time.Since(start).Nanoseconds()
	fmt.Println("Average goroutine context switch time:", elapsed/int64(2*loop))
}
```