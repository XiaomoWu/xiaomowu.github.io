# How to query an interval? Understand `foverlap` and `roll` in data.table


# Query the interval: `foverlap`

We often need to query overlaps. For example, given a table of daily trading volume, we want to find for each day `t`, the trading volume of its *surrounding days*, i.e., `[t-N,t+N]`.

**Example:**

Suppose we have an `event` table. The `event` table contains the event date. Events are *infrequent*. In our example, events only take place on day 1, 3, 5:
```R
# event: infrequent events of interests
event = data.table(date=c(1,3,5))
event

#>    date
#> 1:    1
#> 2:    3
#> 3:    5
```

There's another table, called `history`. The `history` table records values of interest *on each day.* In the example, the `history` table has a `value` for each day from 1 to 5:
```R
# history: chronical records *for each date*.
history = data.table(date=1:5, value=sample(letters, 5, replace=TRUE))
history

#>    date value
#> 1:    1     v
#> 2:    2     s
#> 3:    3     j
#> 4:    4     p
#> 5:    5     p
```

**Our goal: For each date `t`, find the `value` in the range of `[t-1,t+1]`:**
```R
# our goal:
#>    date value
#> 1:    1   v,s
#> 2:    3 s,j,p
#> 3:    5   p,p
```

To achieve our goal, follow these steps:

```R
# Step 1: add start/end to event

event[, ':='(start=date-1, end=date+1)]
setkey(event, start, end)  # event *must* be keyed!

event
#>    date start end
#> 1:    1     0   2
#> 2:    3     2   4
#> 3:    5     4   6


# Step 2: add start/end to history
history[, ':='(start=date, end=date)]
setkey(history, start, end)  # history is *recommended* to be keyed

history
#>    date value start end
#> 1:    1     v     1   1
#> 2:    2     s     2   2
#> 3:    3     j     3   3
#> 4:    4     p     4   4
#> 5:    5     p     5   5


# Step 3: run foverlaps!
out = foverlaps(history, event, nomatch=NULL)

out[, .(value=list(value)), keyby=.(date)]
#>    date value
#> 1:    1   v,s
#> 2:    3 s,j,p
#> 3:    5   p,p
```

# Query the fuzzy point: `roll`

Sometimes we don't need to find *all* surrounding days for the event date. Instead, we only need to find the "nearest" day for the event date. This is when `roll` comes in.

**Example:**

Let's say this time we only have one event at day 2:
```R
event = data.table(date=c(2))

#>    date
#> 1:    2
```

The history table, contains 5 obs:
```R
history = data.table(date=c(1, 3, 5, 8, 9), value=sample(letters, 5, replace=TRUE))

#>    date value
#> 1:    1     r
#> 2:    3     b
#> 3:    5     g
#> 4:    8     e
#> 5:    9     g
```

**Our goal: find the matched obs 4 days after the event (i.e., day 6)**

Step 1: Add the `join_date` to `event` and `history`
```R
# Add join_date to the event table
event[, ':='(join_date=date+4)]

#>    date join_date
#> 1:    2         6


# Add join_date to the history table
history[, ':='(join_date=date)]  # the join_date is the same as date!

#>    date value join_date
#> 1:    1     r         1
#> 2:    3     b         3
#> 3:    5     g         5
#> 4:    8     e         8
#> 5:    9     g         9
```

Step 2: Use `event` to "query" `history`. 

The results *vary* depending on the `roll` options
```R
# Case 1: closest: the closest to "6" is "5"
history[event, on=.(join_date), roll='nearest'][, .(date, value)]

#>    date value
#> 1:    5     g

# Case 2: Inf: match the *previling* value (LOCF). The previling value of "6" is "5"
history[event, on=.(join_date), roll=Inf][, .(date, value)]

#>    date value
#> 1:    5     g

# Case 3: -Inf: match the *next* value (NOCB). The next value of "6" is "8"
history[event, on=.(join_date), roll=-Inf][, .(date, value)]

#>    date value
#> 1:    8     e

# Case 4: Positive: match the previling at no more than "roll" days
# The previling value of "6" is "5", which is within 1 day
history[event, on=.(join_date), roll=1][, .(date, value)]

#>    date value
#> 1:    5     g

# Case 5: Negative: match the next at no more than "roll" days
# The next no-more-than-one value of "6" is "7", match *failed*
history[event, on=.(join_date), roll=-1][, .(date, value)]

#>    date value
#> 1:   NA  <NA>
```

{{< admonition type=tip title="The unit of `roll`" open=true >}}
The unit of `roll` (when specified as a number) *depends on the unit of `join_date`*. 

- If `join_date` is `Date`, the unit is *day*, i.e.,  `roll=5` means LOCF no more than 5 *days*.
- If `join_date` is `POSIXct`, the unit is *second*, i.e., `roll=5` means LOCF no more than 5 *seconds.* 
{{< /admonition >}}

{{< admonition type=tip title="`foverlap` vs. `roll`: When to use which?" open=true >}}
Both methods are fast!

`foverlaps`
- Pros: 
	- allows for *precise* matching
	- could match to *multiple* values
- Cons:
	- can't do *fuzzy* matching
	- more complicated
	  
`roll`
- Pros:
	- could do *fuzzy* matching
	- less code
- Cons:
	- couldn't do precise matching (can only specify a range)
	- only matches to *one* value
{{< /admonition >}}
