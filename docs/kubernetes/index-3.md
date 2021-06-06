# Common Tools Go Routine Tools

## Parallelize

![common-tools-parallelize-until.svg](../.gitbook/assets/1%20%288%29.jpeg)

## Stop Channel Mode

### Group

Group enhances the ability to sync.Group in the Go standard library. It's used to execute the method to control the termination condition through Context or channel and execute the method in a  stand-alone Go Routine. 

![image.png](../.gitbook/assets/2%20%284%29.jpeg)

### Forever & Until

![image.png](../.gitbook/assets/3%20%284%29.jpeg)

### JitterUntil

From the comments in the code, we need to solve two key problems, how the sliding works and how the jitterFactor works. Once both of them are solved, everything will be clear.

![image.png](../.gitbook/assets/4%20%284%29.jpeg)

#### Sliding

From the code, it's not difficult to see that the sliding is whether the time interval contains execution time. Looking at the 170 lines of code, it's not difficult to guess that Backoff\(\) returns a Timer type, then the Timer start time is the key.  
One detail needs to be noted in this code, the select starting at line 167 doesn't guarantee order, in other words, if the timer has been triggered and the stopCh has been closed, it isn't necessary to ensure exit. But after entering the lower wheel cycle, due to the 144th line code, it must ensure that the program is normal to exit.

![image.png](../.gitbook/assets/5.jpeg)

#### JitterFactor

The jitterFactor works rely on the BackoffManager, let's look at the creation process, the configuration parameters and other associated objects are simply preserved.

![image.png](../.gitbook/assets/6.jpeg)

Let's look at the implementation of its backoff method again. Pay attention to lines 379-383 to ensure that only one timer is working.

![image.png](../.gitbook/assets/7.jpeg)

Continue to see the Jitter implementation, add a dynamic value on a fixed duration, and the two problems are solved.

![image.png](../.gitbook/assets/8.jpeg)

