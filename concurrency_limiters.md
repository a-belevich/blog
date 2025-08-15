# TLDR

Concurrency limiters, while providing a valuable stability improvement for the service itself and for its upstreams, can cause some less expected effects when being combined with other popular configuration options (least requests or least CPU load balancing; HPA auto-scaling; other concurrency limiters).

# Speculations

Important note: I may lack some important knowledge about internal details of all the components mentioned below.

## Concurrency limiters with zero error tolerance

In-process concurrency limiters discussed here are used to protect services during the period of fragility, either caused by excessive load or by fragile upstream dependencies. Under the hood, the limiter is a library (in cases I observed - Go library, usually based on [github.com/platinummonkey/go-concurrency-limits](https://github.com/platinummonkey/go-concurrency-limits)) that collects local instance-level statistics and defines its internal limit of parallelism. When the limit is reached, it can either put the excessive requests into an internal queue, or quickly drop them (return an error like *HTTP 429* or *gRPC ResourceExhausted*).

Concurrency limiters can be configured in multiple ways. In particular, they can have:

* Fixed hardcoded limit.
* Limit based on latency (limit decreases as the latency grows, and increases as the latency falls).
* Limit based on errors (limit decreases as the errors happen, and increases as the old limit is reached and no errors happened).
   * The ultimate example is the zero-error-tolerant limiters: even a single error per observation time frame (like, a second) decreases the limit.

A popular strategy to modify the limit values is AIMD (additive increase multiplicative decrease). After each time frame (e.g., a second):

* If the success condition was not satisfied (e.g., there was an error, or latency exceeded a threshold), the limit is multiplied by some value (e.g., by 0.9);
* If the success condition was satisfied **and** the ongoing limit was reached, the limit is incremented by some value (e.g., by 1).

The idea of protecting a team's precious services with back-pressure that discards the workloads above manageable ones looks intuitive and attractive. When a resource experiences degradation and starts to return errors - quickly decrease the load down to an error-free level, and then gradually increase the load - sounds reasonable? Yes. But in some circumstances I expect this policy to backfire.

## Envoys

One more popular configuration option in service oriented architecture is Envoy. Often envoys are configured to use `LeastRequest` or `LeastCPU` load balancing strategy. Unfortunately, from the point of view of envoy, the service that immediately discards lots of requests, and tries to concurrently process less requests than others, looks like a very performant service. In other words, a great candidate for getting more traffic.

So: back-pressure strategies resulting in quick drop, when combined with `LeastRequest` or `LeastCPU` balancing, can work as an error multiplicator. Degraded instances receive (and discard) more traffic than healthy ones. The upstream services still remain protected and have a chance to recover, but the downstream services experience higher than needed rate of errors and retries (compared to, for example, simple *RoundRobin*). Note that back-pressure strategies resulting in throttling (or queueing), i.e. limiting internal processing concurrency while still keeping the RPC requests open, don't cause such problems for `LeastRequest` (though still can affect `LeastCPU`).

## Auto-scaling

Back-pressure strategies causing quick drops are not spending a lot of local CPU time. So, from the point of view of HPA auto-scaler, the deployments that are actively applying back-pressure at the moment don't look like they need extra pods. Quite the opposite, they are processing their workload with lower CPU utilization than usual, and can even trigger scaling in.

So, my assumption is that auto-scaling for deployments with back-pressure should be based on number of downstream requests or queue lengths or something similar; not on CPU utilization.

## Chained zero error tolerance

Any quick-discard back-pressure combined with request-level load balancing (unlike ClusterIP-like connection-level balancing) leads to the errors from even a single degraded pod (say, bad canary or hardware problem) being distributed across multiple downstream callers. Of course, if the whole deployment is struggling, the downstream services suffer too. What happens, though, when the callers, in turn, also have zero error tolerance? And also start decreasing their own concurrency limits? And their own envoys are also configured to send more traffic to the instances that degraded first?

Also, any limit growth of the AIMD concurrency limiters with quick discard is (almost) always associated with drops. Since the growth requires reaching the limit during a given time frame (say, second), it takes a lot of luck to reach the limit multiple times (to increment it by 1 every time) without over-reaching it time after time. So, while the services in a deployment are growing their limits, their downstreams experience drops (and, if they happen to be zero-error-tolerant, decrease their own limits).

In theory, the multi-level zero-error-tolerant chain looks like a way to quickly degrade multiple levels of dependencies, and let them slowly ramp up after the source of errors is fixed.

# Modeling

To understand how the zero-error-tolerant AIMD quick-discard concurrency limiters behave when facing errors, I wrote a small simulator.

## Approach

In current version, the simulated behavior includes:

1. A test driver (emits instructions to clients and collects statistics).
2. Clients: multiple actors sending the requests downstream, waiting for results, and retrying with backoff if needed.
3. Services: multiple actors emulating a service behavior:
   1. Configurable concurrency limiter:
      1. Discarding or throttling.
      2. Static or error-based or latency-based limits (or no limits).
   2. If the service has downstream dependencies, it sends the request downstream, waits and retries if needed.
   3. Once received a successful response, it runs a calculation (increasing the calculation time if the number of parallel calculations exceeds a threshold).
   4. Returns a success or an error (frequency of errors is configurable).
4. Service groups: emulating deployments with load balancing. Right now, they support envoy-like behaviors (request level balancing) with either round-robin or least-busy strategies, or ClusterIP-like behaviors (connection level balancing).

The source code is published at [https://github.com/a-belevich/workloads](https://github.com/a-belevich/workloads). The Main class contains the definition of the simulated environment (all services and their configurations). When executed, it displays per-second statistics.

## Configuration

The existing codebase contains a config of:

1. A test driver initiating 10,000 requests/sec from
2. 10,000 client actors sending requests to
3. An envoy with `LeastRequest` load balancing, sending requests to
4. One out of a 100 client-facing services with zero-error-tolerance quick-discard concurrency limiter, sending requests to:
5. An upstream envoy with `LeastRequest` load balancing, sending requests to
6. One of the upstream services with zero-error-tolerance quick-discard concurrency limiter.
7. There are 99 upstream services that always return a success, and 1 that returns one error in 2 seconds.

## Results

In the configured environment, both levels of services quickly (within a minute) degrade. The manner of degradation is non-linear.

* First, the single bad upstream service starts to decrease its concurrency limit due to regular errors.
* Then, it reaches the point where it starts to discard messages because of it.
* Then, it starts to receive significantly more requests compared to other workers in its deployment.
* Downstream, many or all of the client-facing services start to receive their share of upstream errors (drops from the bad actor).
* After the upstream errors reach a certain percentage, the downstream service's retry policy cannot catch all errors, and lets some of the errors through. Which, in turn, causes a decrease in their own concurrency limits.
* Eventually, even the client’s retry policy becomes not strong enough, and the driver starts to receive a significant amount of errors, despite the retries on 2 levels.

I.e.: one bad (not even super bad: just one error in 2 seconds) actor pushes the system from the stable state successfully handling 10k requests/sec into a state where 1k-1.5k requests per second fail; all due to an interaction between AIMD limiters, `LeastRequest` load balancers, and 2 levels of zero error tolerance.

## Caveats

The model can be not completely accurate. For example, the `LeastRequest` balancing mode always chooses the least busy worker globally; in real envoys it’s a 2-step process. How accurate is the model’s representation of reality - difficult to answer without real-world tests.

# Real life

So far, I have multiple observations of concurrency limiters eagerly protecting their own services at the expense of downstream ones. And I observed one occurrence of a fix: after reconfiguring the system (increasing retries even more, and replacing load balancing strategy from `LeastRequest` to `RoundRobin`) the massive losses of messages stopped.

I don’t have a complete understanding of all the possible effects yet, and my testing capacity is limited due not owning any concurrency limited services; I only happen to call one of them. So, maybe some of my modeling assumptions are wrong, and the services in the wild don’t behave as predicted.

However, in case you and your team own some of the concurrency limited services, or are calling them as a dependency - please take a look at their behaviors during the incidents (or try to find/recall their behaviors during past incidents):

1. If the services don’t degrade uniformly, but there are outliers (bad canaries during deployment can be a good example) - are the degraded outliers receiving (and discarding) a significantly larger percentage of traffic, compared to healthy pods?
2. Does HPA auto-scale the degraded deployment down during the incidents?
3. In case you have a chain of calls from one concurrency-limited service into another, do the rare errors upstream grow and cause bigger and bigger problems downstream?

If some of those predictions happen to be correct, it can mean that the concurrency limiters can require an additional support from the infrastructure around them; and that some of the popular configuration options are not playing well with the concurrency-limited services.

# Possible workarounds (proposed by colleagues)

## Readiness probes

In theory, the problem with `LeastRequest` sending most of traffic to concurrently limited pods could be solved by readiness probes failing when the concurrency limiter trips. The result is the pod stays up, is balanced out for traffic, and can recover. It comes back alive when it's in a healthy state. However, it comes back with low concurrency limit set, so the whole time the limits are growing back to healthy values the pod still will demonstrate problematic behavior?

## Retries

\* Retries should redispatch to a different upstream. See [https://www.envoyproxy.io/docs/envoy/latest/faq/load\_balancing/transient\_failures#outlier-detection](https://www.envoyproxy.io/docs/envoy/latest/faq/load_balancing/transient_failures#outlier-detection)

[https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/retry/host/previous\_hosts/v3/previous\_hosts.proto#envoy-v3-api-file-envoy-extensions-retry-host-previous-hosts-v3-previous-hosts-proto](https://www.envoyproxy.io/docs/envoy/latest/api-v3/extensions/retry/host/previous_hosts/v3/previous_hosts.proto#envoy-v3-api-file-envoy-extensions-retry-host-previous-hosts-v3-previous-hosts-proto)

\* The server should communicate with the client that the retry is safe always due to the rate limit. In HTTP terms it is not safe to retry a POST request on a 5xx response. But in that case the client can retry because the service discarded the request.

# Per host thresholds on Envoy

If your service is accepting traffic only via Envoy you can use \`per\_host\_thresholds\` in [https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit\_breaker.proto#config-cluster-v3-circuitbreakers](https://www.envoyproxy.io/docs/envoy/latest/api-v3/config/cluster/v3/circuit_breaker.proto#config-cluster-v3-circuitbreakers) to setup a concurrency rate limiter per endpoint.

# Queueing vs Dropping

Requests being queued/throttled inside a service could be a great alternative to quick dropping. This approach doesn't conflict with `LeastRequest` balancing: the pod that fails to process messages quickly receives less messages than healthy ones, not more.

# Back-pressure and SLA

In general case it's not clear who's responsible for the dropped requests (is the downstream service DDoSing upstream? or is the upstream service unhealthy? how much of retries are too much?). So, when using techniques like quick-drop concurrency limiting, it's especially important to negotiate a clear SLA. Otherwise it can grow into a big source of friction and disappointment for all involved teams.

# Async messaging

In many cases (especially for the latency-insensitive workflows) the architecture with async message passing looks like a better option compared to RPC. Back-pressure in the form of loading less messages for processing and leaving unprocessed messages in some a queue/topic makes things more predictable and manageable.

* It does not introduce artificial errors which are different from real errors and should be handled and counted separately.
* It does not skew traffic towards failing pods.
* It does not completely exclude slower/concurrency-limited pods from serving traffic (especially when **all** pods get slow for some reason during an incident).
* Backups are easily observable.
* It does not require a lot of coordination between clients and servers (clients can send requests at their own rate, and servers can pull them for processing at their own rate, assuming the queueing solution is robust enough to avoid crashes when backed up).
* It simplifies SLAs. All back-pressure aspects, retry policy etc are defined by the service owners; the downstream teams should only agree on overall throughput/latency.

One more intriguing opportunity that I haven't seen implemented anywhere yet would be an ability to calculate "message processing capacity surplus". When message processors are trying to preload more messages than the queue has buffered, the difference can play a role of a "negative queue length". This surplus could make scaling down more efficient: we know how many message processors are idling, not only that the queue length is zero.

# Credits

In addition to my colleagues who discussed this topic with me, I also want to thank David Dunning and Justing Kruger for the confidence to model behaviors of tools that I don’t know well.
