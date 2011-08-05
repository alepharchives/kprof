# kprof

kprof allows you to see in which layer of your Erlang application a
request is spending time.

Requests are traced from entry point all the way through your code
until the request completes by returning to the entry point, it is
even traced across processes.

The trace is broken down into "tiers", so you can see exactly how much
time is spent in the different layers of your system. Only these tiers
are traced by the VM, which makes the performance impact much lower. A
full trace is also possible for exploring a request.

kprof is lightweight and allows tracing thousands of requests per
second.

The results from each requests are aggregated on the fly and written
to a file or to couchdb once a minute. A provided CouchApp
(kprof_couchapp) has views and a simple GUI for viewing the data.

kprof is licensed under the MIT License. If you include it in your
product or in general find it useful in any way, please let me know.

## Project state

There are very few features apart from the most basic so I would love
to get feedback on what could make the "product" more usable! Any form
of ideas/suggestions/musings are very much welcome. I also welcome
patches and pull requests.

kprof is used by wooga (github.com/wooga, wooga.com), the second
largest social games developer in the world.

## Usage

The entry point you want traced need to be patched to set and remove a
token that is used to identify the request. kprof provides helpers to
do this:

 * kprof:do_apply(M, F, A) which manages the token and runs
   erlang:apply/3 with the supplied arguments

 * kprof:misultin_req_loop(M, F) which returns a fun suitable for use
   with misultins request loop. The fun will invoke M:F(Request).

kprof:set_token/0 and kprof:clear_token/0 are the true workhorses and
might be used inside your own code if you do not want to use the
provided helpers.

To start tracing, start kprof (kprof:start()) and call
kprof:start_trace/1 like this:

    kprof:start_trace([{tier_config, TierConfig},
                       {identity_f, IdentityF},
                       {print_calls, false},
                       {stats_dumper, {file, "stats.log"}}]).


## Tiers

The trace is broken down into "tiers", which allows for much smaller
impact due to tracing and simpler aggregation. You can think of a tier
as the function to trace and aggregate. There is no need for any
hierarchy, so you can trace "sibling" function calls.

When you start tracing, you must supply the tier configuration on the
following format:

    [{TierName::atom(), {M::atom(), F:atom(), A::integer()}}]

For example, a simple key-value system might be broken down like this:

    [{parsing, {network_server, parse_request, 1}}, % decoding network message
     {handle, {requests, handle_request, 2}},
     {formatter, {formatter, format_response, 1}},
     {db, {db, network_request, 1}}, % to measure network latency
     {accounting, {accounting, incr_reqs, 1}}]


In addition, there is a special tier "_total" that is the aggregated
data across all tiers.

## Identity

A request may be identified by a supplied function. The identity
serves as a grouper across tiers, in the case of the key-value server
above the identity might be the type of request, for example "create",
"read", "update", "delete". In other words, you will be able to know
how much time is spent in each layer for each type of request.

The identity function should be a fun of arity 1. It will receive the module name, function name, and actual Example:

    fun ({requests, handle_request, [create, _]}) ->
            create;
        ({requests, handle_request, [update, _]}) ->
            update
        end

## Full trace

It is possible to trace all functions invoked during a request which
is very helpful in exploration. However, a full trace has a much
higher performance impact than tiered tracing.

Aggregation for full tracing is not possible, as it is currently done
per {Module, Function, Arity} not on the position of the call within
the call trace. This might change in the future.

## Code-reload

If a module with a traced function is reloaded, the trace BIFs no
longer traces that function. In this case, the trace has to be stopped
and restarted.

## Overload protection

kprof currently has no overload protection. An idea is to pause
tracing if any received trace message is older than N microseconds and
resume some time later.

## TODO
 * overload protection in kprof_server, pause tracing, look at timestamp of trace messages, may be no longer than N microseconds in the past, configurable, resume tracing N seconds later
 * sampling based on frequency, sample 10% of all calls, somehow include this info in the couchapp
 * full tracing of all calls in a seq trace (fabian), "slow calls" in rpm
 * run the aggregation externally to the node using {tracer, Port}
 * trace garbage collection more meaningfully, send as separate call to aggregator?
 * trace gc after a request completed, shows gc triggered by a particular request
 * use basho_bench to run the sample app, while profiling, make sure concurrency doesn't break the profiler
 * configurable reporting interval
 * option to disable tracing of arguments to lower trace impact
 * plain text aggregated output, show tier totals, list most time consuming calls with tier breakdown:


Sample output:
                                http    database        templating
-------------------------------------------------------------------
Totals across all calls:        50 ms   20 ms           10 ms
-------------------------------------------------------------------
foobar                          70 ms   30 ms           10 ms
quux                            120 ms  70 ms           10 ms

## Couchapp TODO
 * select specific node or all nodes
 * select time period, last 30 minutise, 60 minutes, 3 hours, etc, use for all charts
