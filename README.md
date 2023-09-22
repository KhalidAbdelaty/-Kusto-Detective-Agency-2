# Case Two - Election Fraud?

![Insert Picture Here]

## Table of Contents
1. [The Case](#the-case)
2. [Proving](#proving)
3. [Solving](#solving)
4. [Solution](#solution)

## The Case
In our second case, we delve into the world of fishy elections and a rather suspicious goldfish.

## Prove the Fish fixed it.
Our mission begins with the daunting task of proving that the fish had a hand in this election mischief. We'll also need to crunch the numbers and calculate the correct totals.

## Proving
Now, I might not be a data wizard like Matt Parker, but let's take a rather amateurish attempt at scrutinizing the results using Benford's Law:


```kql
Votes
| summarize Count=count() by vote, via_ip
| extend num = substring(tostring(Count), 0, 1)
| summarize Count=count() by num, vote
| sort by num asc
| render columnchart with (series=vote)
```

There's something rather fishy going on here. And perhaps, Gaul may have had some involvement in vote rigging as well?

## Solving
We've got our hands on the query that tallies the votes:


```kql
// Query that counts the votes:
Votes
| summarize Count=count() by vote
| as hint.materialized=true T
| extend Total = toscalar(T | summarize sum(Count))
| project vote, Percentage = round(Count * 100.0 / Total, 1), Count
| order by Count
```

So, let's put it to the test:


markdown
| Vote   | Percentage | Count    |
|--------|------------|----------|
| Poppy  | 51.7       | 2,601,570  |
| Kastor | 25.6       | 1,285,782  |
| Gaul   | 19.4       | 976,570   |
| Willie | 3.3        | 166,499   |


## Solution
After some meticulous investigation (I really should start keeping better notes), I began exploring time intervals between votes. With a sufficiently small bin size, it became evident that the other candidates were consistently receiving only one vote per time slice, whereas Poppy nearly always exceeded this.

By eliminating votes where the count per bin was greater than 1 and using the provided logic, I arrived at the answer:


```kql
Votes
| summarize Count = count() by vote, via_ip, bin(Timestamp, 500ms)
| extend Count = iff(Count > 1, 0, Count)
| summarize Count=sum(Count) by vote
| as hint.materialized=true T
| extend Total = toscalar(T | summarize sum(Count))
| project vote, Percentage = round(Count * 100.0 / Total, 1), Count
| order by Count

```

markdown
| Vote   | Percentage | Count    |
|--------|------------|----------|
| Kastor | 50.8       | 1,284,188  |
| Gaul   | 38.6       | 975,554   |
| Willie | 6.6        | 166,479   |
| Poppy  | 4.0        | 102,278   |


There might be a more sophisticated approach involving fancy machine learning in Kusto, but as I mentioned earlier, I'm no Matt Parker.
