# Common Tools Time Budget

## Definition

### Creation

The timeBudget is defined as follows.  


![image.png](../.gitbook/assets/1%20%2811%29.jpeg)

  
 The creation code is as follows.

![image.png](../.gitbook/assets/2%20%287%29.jpeg)

## Process

### Bonus

After the timeBudget starts running, a work collaboration is created. In this coroutine, the budget increase operation will be triggered once a second. If the budget is greater than the upper limit, the upper limit is taken.  
 

![image.png](../.gitbook/assets/3%20%286%29.jpeg)

### Take & Return

The takeAvailable acquires all budgets at a time and resets the budget. The returnUnused returns the remaining budget.  


![image.png](../.gitbook/assets/4%20%286%29.jpeg)

