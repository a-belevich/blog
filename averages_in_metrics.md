Recently I found myself trying to debug an SLI dashboard. This dashboard is based on Grafana/Prometheus/Thanos,
and it is displaying things like service's availability (% of successfully handled requests) or p99 latency over 
periods of time from 1 day to 1 month. The whole text below represents my own understanding and may be incorrect in
some important aspects.

What drove my attention to this dashboard was the counter-intuitive data displayed there. Time after time, the 
availability exceeded 100%. I still don't know why it's happening by the way. The dashboard literally queries the same
rate counter twice: once with filter for `grpc_success="true"`, once without; and the filtered is bigger. 
But, it's not the topic anyway. The topic of this article is based on another finding. While trying to debug it,
I found something very counter-intuitive (for myself).

First, set up. The service publishes a Prometheus counter of requests with `grpc_success` label. 
Let's call it `requests`. Due to the large number of requests and slowness of calculating the rate every time,
it applies a recording rule that aggregates requests' rates over 1 minute interval. Let's call the derivative metric
`requests:rate1m`. Technically, in Prometheus terms, it's a counter, but it contains the average per-second rates 
over a minute-long interval; not the monotonously incremented number of requests served. You can consider it 
a shortcut for `rate(requests{}[1m])`; it doesn't change anything in the logic below.

In our codebase I saw 2 different ways to calculate availability. It could either 
```
avg_over_time(
  sum(
    requests:rate1m{grpc_success="true"}
  )[7d:]
)
/
avg_over_time(
  sum(
    requests:rate1m{}
  )[7d:]
)
* 100
```
or
```
sum(
  avg_over_time(
    requests:rate1m{grpc_success="true"}[7d]
  )
)
/
sum(
  avg_over_time(
    requests:rate1m{grpc_success="true"}[7d]
  )
)
* 100
```

I.e., to calculate the average request rate of the total service, we can count average rate per server and then 
sum them up. Or we can sum the rates from all servers into a single time series; and then count its average. 
Same thing different angle? Not exactly.

Trying to debug the issue and looking for clues, I checked the graph of daily average rate of requests. Something like
```
sum(
  avg_over_time(
    requests:rate1m{grpc_success="true"}[1d]
  )
)
```
over 7 days. I was shocked. 
![](/images/averages_in_metrics/graph2.png)
I definitely didn't expect the request rate to go from 75k requests/sec to 6k and back. What could possibly go wrong?

For the comparison, here's what the graph of `avg_over_time(sum(...)[1d:])` looks like.
![](/images/averages_in_metrics/graph1.png)
Very reasonable to my taste.

What's the cause of the sudden drops in volumes? [This guy](https://en.wikipedia.org/wiki/The_Weeknd). We don't deploy during weekends (and during Fridays). 
And we do actively deploy during the weekdays.

Apparently, I had a wrong mental model. Those 2 ways to calculate the average request rate should produce similar 
results over time. But only if you deal with a static set of servers. If your service gets re-deployed 10 times a day,
only one of those models works.

Imagine 5 servers, each handling 100 requests/sec. `sum(avg_over_time(requests:rate1m{}[1d]))` results in `sum(100)` 
that results in constant horizontal line of 500 requests/sec (correct). `avg_over_time(sum(requests:rate1m{})[1d:])` results
a total counter of 500 requests/sec and is averaged to the same constant graph of 500 requests/sec.

Now imagine that we deployed 6 versions of a service that day. 
Which means, there are 30 time series in Prometheus instead of 5; going for shorter periods of time.
The `avg_over_time(sum(requests:rate1m{})[1d:])` doesn't care. It still produces a single series 
worth 500 requests/sec; 
just does it by summing up bigger number of source time series; and averages them to the total of 500. 
The story of `sum(avg_over_time(requests{}[1d]))` becomes trickier though. 
What does `avg_over_time(requests:rate1m{}[1d])` even mean if the time series was only published for an hour or two? 
Apparently, it means that we'll count the average rate over the period the series exists. So, suddenly we have 30
time series, each averaging to 100. But then, it affects the totals for the next 24 hours 
(since its average of 100 is still visible in the lookback window).
`sum` starts at 500 at the beginning of Monday (since we only saw 5 series over last 24 hours),
but then starts to rise. By the end of Monday it rises up to 3000 requests per second.

The same logic applies to calculating average latencies: `sum by (le) (avg_over_time(...[1d]))` vs `avg_over_time(sum by (le) (...)[1d:])`.
Or to derivatives (the abovementioned availability). Availability masks the problem, since both numerator and denominator
got affected by the same re-deployments. 
But, it still means that elevated error rate during the weekdays affect overall availability compared to the same 
error rate during a weekend.

Don't even know whether my understanding is correct; and don't know whether it's only me who finds this behavior 
counter-intuitive. But, for now and for myself, I made a single rule: never average over time the set of a variable size.
First aggregate away the confusing part (set of servers participating in the calculation), and only then aggregate over time.
