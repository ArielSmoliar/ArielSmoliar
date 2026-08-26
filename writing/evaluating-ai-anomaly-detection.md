# How I Evaluate AI Anomaly Detection

Most anomaly-detection demos answer the easiest question:

> Can the model produce a plausible explanation?

A production system has to answer harder questions:

- Did it identify the anomalous evidence?
- Is every claim grounded in the source logs?
- Did it distinguish unusual behavior from dangerous behavior?
- Does the result remain consistent across model and prompt changes?
- Does the explanation help an operator decide what to do next?

This is the evaluation approach I developed while building [Flare AI](https://www.tryflare.ai/), an LLM-first investigation product for AWS CloudTrail and GCP Audit Logs.

## The system boundary

Flare separates deterministic analysis from model-generated interpretation.

The deterministic layer:

1. Parses and normalizes cloud audit logs.
2. Builds historical baselines.
3. Ranks statistically unusual fields and events.
4. Selects the evidence presented for investigation.

The model layer:

1. Interprets the ranked evidence.
2. Explains why the behavior may matter.
3. Connects related activity across events.
4. Recommends investigation steps.

This boundary is deliberate. The model should not invent the evidence used to justify its conclusion.

## What I evaluate

I evaluate the system across four dimensions.

### 1. Detection quality

Does the system surface the events and fields that distinguish the incident from normal activity?

Example checks:

- The anomalous principal is identified.
- The unusual API action is surfaced.
- The relevant region or resource is included.
- Common but irrelevant fields do not dominate the result.

### 2. Evidence grounding

Every material claim should be traceable to the supplied logs or a clearly identified baseline.

A response fails grounding when it:

- Introduces an event that is absent from the source data.
- Assigns intent that the evidence does not support.
- Confuses statistical rarity with maliciousness.
- Omits evidence that contradicts its conclusion.

### 3. Explanation usefulness

A technically accurate result can still be unhelpful.

I ask whether a security operator can determine:

- What happened?
- Why is it unusual?
- What evidence supports the conclusion?
- What should be investigated next?
- How confident should the operator be?

### 4. Regression stability

A prompt, model, or pipeline change should not silently repair one scenario while breaking another.

I maintain golden fixtures representing distinct incident and non-incident patterns. Each fixture contains:

- Source logs
- Historical context
- Expected relevant evidence
- Acceptable conclusions
- Prohibited unsupported claims
- Expected investigation steps

## Failure taxonomy

I group failures by the layer responsible for them.

| Failure type | Example | Likely intervention |
| --- | --- | --- |
| Signal-selection failure | Important event never reaches the model | Ranking or baseline logic |
| Grounding failure | Explanation cites nonexistent activity | Prompt, evidence format, or validation |
| Interpretation failure | Rare behavior is labeled malicious | Reasoning rubric or examples |
| Coverage failure | One stage of the incident is omitted | Context selection or aggregation |
| Actionability failure | Explanation offers no useful next step | Output contract or product design |
| Regression | A model change breaks a previously passing fixture | Evaluation gate or model selection |

This matters because not every failure should be addressed by changing the prompt. Some belong in deterministic software, retrieval, evidence selection, or the product interface.

## A representative evaluation

Suppose the source contains:

- A principal operating from an unfamiliar region
- An unusual permission change
- A later access attempt against a sensitive resource
- A large volume of routine activity from the same period

A useful result should connect the unusual events without allowing the routine volume to dominate the explanation.

The evaluation checks whether the system:

1. Ranks the permission change and sensitive-resource access highly.
2. Grounds the unfamiliar-region observation in the source.
3. Avoids declaring the activity malicious without sufficient evidence.
4. Recommends specific identity and resource checks.
5. Preserves the event sequence.

## What I learned

### Statistical relevance and semantic importance are different

A field can be statistically unusual without being operationally important. The model helps interpret relevance, but it needs explicit access to the underlying evidence and baseline.

### Plausibility is a dangerous evaluation standard

A polished explanation can conceal unsupported claims. Grounding must be evaluated independently from writing quality.

### Full-dataset context can outperform arbitrary chunking

Conservative chunk sizes can create cross-call blindness and unnecessary latency. Measuring actual context utilization revealed that some workloads could be analyzed together rather than split prematurely.

### Failure ownership should be explicit

When an output is wrong, the team should be able to identify whether the fault belongs to ingestion, normalization, ranking, context construction, model reasoning, or presentation.

## Limitations

This evaluation system does not prove that an anomaly is malicious. It measures whether the product identifies relevant evidence, explains it faithfully, and supports a defensible investigation.

Production usefulness also requires feedback from security operators and measurement of downstream actions. That remains distinct from offline fixture performance.

## Why this matters

Reliable AI products require more than selecting a capable model. They require an operating system for:

- Capturing failures
- Turning failures into reusable evaluation cases
- Separating model and system responsibilities
- Comparing changes against stable expectations
- Connecting offline quality to real user outcomes

That evaluation loop is ultimately what makes a probabilistic capability shippable.

---

Originally published as an [X thread](https://x.com/ariel_smoliar/status/2053200148814024838).
