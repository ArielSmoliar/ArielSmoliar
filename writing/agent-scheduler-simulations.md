# Watching an Agent Scheduler Decide Who Goes Next

I turned LOCO-Agent's scheduling knob all the way toward “serve the busiest agent first.” A third of the agents starved.

This is a 20-seed simulation, not a production benchmark. That distinction is the point: the experiment validates the behavior of the scheduling function under controlled contention. It does not measure live model throughput, network latency, or provider performance.

The code and implementation are available in the [LOCO-Agent repository](https://github.com/ArielSmoliar/loco-agent).

## The scheduling function

At each tick, every waiting agent receives a score based on two signals:

1. Weighted queue depth: how much work the agent has accumulated
2. Maximum wait: how long its oldest task has been waiting

The scheduler combines them through one parameter, alpha:

```text
L(i) = alpha × normalized_queue_depth
     + (1 - alpha) × normalized_max_wait
```

- At **alpha = 1**, scheduling uses queue depth only.
- At **alpha = 0**, scheduling uses wait time only.
- The highest-scoring agent receives the next available slot.
- Ties are broken randomly.

There are no manual priority flags or per-agent priority rules in this experiment.

Trust scoring and adaptive alpha are intentionally excluded. The experiment isolates queue depth, wait time, and contention.

## Experimental scope

The simulation is discrete:

- At most one task is served per tick.
- There are no live LLM calls.
- There is no network or real-world latency.
- Scenarios 2 and 3 report means across 20 random seeds with 95% confidence intervals.
- Scenario 1 is an invariant that held on every seed.

The three scenarios test work conservation, fairness under sustained pressure, and automatic urgency escalation.

## Scenario 1: work conservation

Eight idle agents receive a burst at tick zero. Agent `i` receives `i + 1` tasks—from one task for the first agent to eight for the last.

That produces 36 tasks in total.

Across all 20 seeds:

- All 36 tasks clear in exactly 36 ticks.
- No task is dropped.
- No task is served twice.
- Each agent is served exactly as many times as it has tasks.
- The busiest queues drain first.

This is the minimum bar. Before optimizing any tradeoff, a scheduler must conserve work and surface the largest backlogs.

## Scenario 2: fairness under sustained pressure

Ten agents run for 500 ticks:

- Five receive work at a high arrival rate.
- Five receive work at a lower arrival rate.
- Alpha is swept from 0 to 1.

I ran the setup under two capacity regimes.

### Overloaded system

Expected arrivals are 2.5 tasks per tick while the scheduler can serve at most one. The system cannot keep up by construction.

In this regime, the scheduling policy determines who suffers.

As alpha increases from 0 to 1:

- Starved agents rise from 0 to approximately 3.7 of 10.
- Jain's fairness index falls from 0.72 to 0.49.
- Queue-depth-only scheduling concentrates service on the busiest agents.
- Roughly one-third of the fleet receives zero completions during the window.

The wait-time component is what prevents starvation. Remove it and quiet agents can disappear from the schedule.

One misleading metric deserves mention: at high alpha, average wait for quiet agents can appear low because completely starved agents have no completed tasks to include in the average. Starvation count and fairness are the honest headline—not average wait.

### Sustainable system

Expected arrivals fall to 0.7 tasks per tick against capacity of one task per tick.

Now the scheduler can keep up:

- Mean wait stays near one tick across alpha values.
- No agent starves.
- Approximately 99.8% of work clears.
- Fairness remains near 0.82.

Under spare capacity, the scheduling knob largely stops mattering.

That is the core product lesson:

> A scheduler does not create capacity. It decides who waits when capacity is scarce.

Showing both regimes prevents the overloaded result from becoming a rigged demonstration. The dramatic tradeoff is a property of contention.

## Scenario 3: urgency without priority flags

Ten background agents run steadily. At tick 30, five urgent webhook agents arrive.

They receive:

- No priority flag
- No special queue
- No manually assigned preference

All five urgent agents are eventually served at every alpha setting. But low-alpha, latency-oriented scheduling serves them roughly three times faster:

- Approximately 33 ticks after the spike at low alpha
- Approximately 103 ticks at high alpha

Their wait score grows every tick until they outrank the deeper background queues.

Urgency emerges from waiting rather than from a human-applied label.

The same limitation remains: if arrivals exceed capacity indefinitely, no scheduling rule can rescue every workload. This experiment demonstrates responsiveness under pressure, not additional capacity.

## What the experiment shows

The scheduling function:

- Conserves work
- Surfaces deep backlogs
- Trades throughput concentration against fairness
- Can starve quieter agents when tuned too heavily toward queue depth
- Escalates newly urgent work through accumulated wait time
- Becomes largely irrelevant when capacity exceeds demand

## What it does not show

This experiment does not establish:

- Production LLM throughput
- Provider or network latency
- Model-quality effects
- Trust-aware scheduling
- Adaptive alpha behavior
- Performance under every arrival distribution

Those require separate evaluation.

The credibility of a systems experiment depends partly on stating what it cannot prove.

## Product implications

### Expose the tradeoff

A scheduling control should not be presented as a generic “optimization level.” Operators should understand whether it favors latency, fairness, or backlog throughput.

### Measure starvation directly

Average latency can hide workloads that never complete. Starvation count and per-agent completion distributions belong in the operational dashboard.

### Make scarcity visible

When demand exceeds capacity, the product should say so. Scheduling can allocate scarcity but cannot eliminate it.

### Prefer guardrails over unrestricted tuning

The simulation supports a practical constraint: do not allow configurations that turn off the wait-time contribution without clearly warning that starvation becomes likely.

### Adapt only after measurement

An adaptive scheduler should respond to observed contention, wait variance, and starvation—not tune itself merely because it can.

## Takeaway

One equation and one parameter produced three distinct behaviors: backlog draining, fairness loss, and urgency escalation.

The surprising result was not that the knob changed performance. It was that queue-depth-only scheduling could make approximately one-third of agents disappear from the completion record under overload—and that the problem vanished when the system had enough capacity.

Scheduling is a scarcity policy. Evaluate it as one.
