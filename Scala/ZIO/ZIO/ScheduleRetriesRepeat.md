## Schedules

- Schedules are data structures that describes how effect should be timed.

```
def httpGet(url: String): Task[String] = ???

val retriedZIO = httpGet(url).retry(Schedule.recurs(10))
```

- This will retry `httpGet` 10 times, therefore, total number of tries - 1 (original call) + 10 (retries) = 11

- Schedules -
  - `Schedule.once` - Retries exactly once
  - `Schedule.recurs` - Recurrent retries
  - `Schedule.spaced(1.second)` - retries every 1 second infinitely until a success is returned
  - `Schedule.exponential(1.second, 2.0)` - `1.second` is the base / starting value and `2.0` is the doubling factor. This will retry at 1s, 2s, 4s, 8s, 16s etc. 
  - `Schedule.fibonacci(1.second)` - `1.second` is the first two values of fibonacci sequence. This will retry at 1s, 1s, 2s, 3s, 5s, 8s, 13s etc.


- Combining Schedules - repeating 3 times every second - `Schedule.recurs(3) && Schedule.spaced(1.second)`
- Sequencing schedules - 3 retries instantly and then retry after 1 second - `Schedule.retries(3) ++ Schedule.spaced(1.second)`

- Schedule returns - `Schedule.WithState[R, I, O, Duration]` -
  - `R` - environment
  - `I` - Input - errors in case of `.retry` and values in case of `.repeat`
  - `O` - Output - values for the next schedule
  
  - To measure the total time taken by the each retries - 
  ```
  Schedule.spaced(1.second) >>> Schedule.elapsed.map(time => println(s"Total time elapsed: $time"))
  ```

  > [!TIP]
  > If the total time elapsed is N seconds, it will print `PTNS`.
