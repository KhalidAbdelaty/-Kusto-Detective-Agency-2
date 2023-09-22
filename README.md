# Case Two - Election Fraud?

## Table of Contents
- The Case
- Proving
- Solving
- Solution

## The Case
The second case revolves around suspicious elections and a dubious goldfish. The task is twofold:
1. Prove the Fish fixed it.
2. Work out the correct totals.

## Proving
To investigate, we apply Benford's Law to the voting results:

\
kql
Votes
| summarize Count=count() by vote, via_ip
| extend num = substring(tostring(Count),0,1)
| summarize Count=count() by num, vote
| sort by num asc
| render columnchart with (series=vote)
\

The results reveal something fishy. It seems Gaul might also be involved in vote rigging.

## Solving
The query that tallies the votes is provided:

\
kql
// Query that counts the votes:
Votes
| summarize Count=count() by vote
| as hint.materialized=true T
| extend Total = toscalar(T | summarize sum(Count))
| project vote, Percentage = round(Count*100.0 / Total, 1), Count
| order by Count
\

Running this query gives us the initial voting results.

## Solution
After some investigation, we focus on the time intervals between votes. With a small enough bin, other candidates receive a single vote per time slice, whereas Poppy nearly always exceeds this.

By removing any votes where the count per bin > 1 and summing using the given logic, we find the answer:

\
kql
Votes
| summarize Count = count() by vote, via_ip, bin(Timestamp, 500ms)
| extend Count = iff(Count > 1, 0, Count)
| summarize Count=sum(Count) by vote
| as hint.materialized=true T
| extend Total = toscalar(T | summarize sum(Count))
| project vote, Percentage = round(Count*100.0 / Total, 1), Count
| order by Count
\

The final results show Kastor leading with 50.8% of the votes. It seems there might be a better way using advanced ML in Kusto but this approach provides a satisfactory solution for now.
