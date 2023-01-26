# Parallelization in R


# 1 Parallelization in R

**TL;DR**

```R
fun <- function(...) {
    # define your function
}

# init
library(future.apply)
plan(multicore, 32)  # in Mac, choose multicore

# run!
future_lapply(iterable, fun)
```

> `future` needs a little bit time to pick. I've seen cases where processing 1000 items used 20 s, and processing 5000 still uses 20 s!


