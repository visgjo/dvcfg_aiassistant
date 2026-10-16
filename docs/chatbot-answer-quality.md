# Chatbot Answer Quality: What to Measure

Oct 15, 2026 · @Daniel Gamiz Antunez

This checklist defines what we measure about the chatbot's answers, what each result means, and which questions it helps us answer. How each check is implemented is up to the implementing team; tools named here are suggestions.

| Criterion | Needs | Covered by the current setup? |
| --- | --- | --- |
| 1. Correctness and completeness | Reference facts | Partly: Factual Correctness in recall mode covers completeness. Correctness is not covered; it needs a separate contradiction judge |
| 2. Retrieval | Expected references, and the references the answer used | No |

The current setup uses Factual Correctness, Answer Accuracy and Semantic Similarity. Answer Accuracy gives only a rough right/wrong signal, and Semantic Similarity covers no criterion; we keep it only to flag answers that changed a lot between versions.

Some criteria from earlier versions of this checklist were dropped; see [Criteria we do not measure](#criteria-we-do-not-measure).

## 1. Correctness and completeness

Is the answer true, and does it contain what it should, compared with the reference?

**Works with:** test questions that have reference facts, either generated (from the documentation or the project configuration) or written by experts. Real user questions can only be used once someone adds their reference facts.

- **Correctness:** share of answer claims that do not contradict the reference. Each claim gets one of three labels: *matches the reference*, *contradicts the reference*, or *not in the reference*. Only contradictions count as errors; "not in the reference" claims are reported separately and spot-checked.
- **Completeness:** share of the reference's required facts that the answer contains.

> **Use Ragas Factual Correctness in "recall mode" for completeness only, not for "correctness".** For correctness it counts every claim missing from the reference as an error, so correct extra information is penalized like a hallucination, and it treats the reference as always right.
>
> **Correctness needs a separate contradiction judge that:**
>
> 1. Labels each answer claim as matches, contradicts, or not in the reference, and counts only contradictions as errors.
> 2. When a claim contradicts the reference, checks both against the source document, and flags the reference for review if the source supports the answer.
>
> Correctness and completeness are always reported as two separate scores.

**What the result means:** high correctness means what we tell users is true. High completeness means users get everything they need from one answer. High correctness with low completeness means answers are safe but users still have to look elsewhere. Low correctness means users receive wrong information.

**Based on:**

- **Completeness:** Ragas [Factual Correctness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/factual_correctness/) in recall mode.
- **Correctness:** the [DeepEval Hallucination metric](https://deepeval.com/docs/metrics-hallucination), which scores an answer by its contradictions with ground-truth reference material.

## 2. Retrieval

Did the chatbot use the right references as sources for its answer?

**Why we measure it:** it tells us how good our retrieval method is (for example RAG), and whether the agent actually uses the information it is given.

**Works with:** test questions with expected references.

- **Expected reference:** a reference an expert has already judged relevant to the question, and expects the answer to use as a source.
- **Relevant reference:** any reference that provides relevant information for the question. This includes the expected references, and any other reference that turns out to be relevant.

**Scores:**

1. **Expected references used** = expected references used ÷ expected references
2. **Relevant references used** = relevant references used (expected + other relevant) ÷ expected references

Both scores share the same denominator, so the second can exceed 100% when the answer uses relevant references beyond the expected ones.

**What the result means:** the first score tells us whether the answer used the references we expected. The difference between the two tells us whether we are also finding relevant references that the expert did not list. This can be good or bad, so those extra references should be reviewed, and added to the expected references if they belong there.

**Questions it answers:** run the same test set before and after each change, and compare both scores.

- **Does making our documentation "AI Ready" improve retrieval?**
- **If we improve chunking, do we retrieve information better?**
- **Does llms.txt work better than RAG as a retrieval method?**

**Based on:** [Ragas ID Based Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/), which compares references instead of retrieved text. At the very least, we can indicate the document each reference was found in.

## Criteria we do not measure

These criteria were part of earlier versions of this checklist and were dropped.

| Criterion | Why we dropped it |
| --- | --- |
| Faithfulness and citations | Completeness covers it better. |
| Abstention (hallucination check) | Not needed. Correctness and completeness serve the same goal: getting trustworthy answers. |
| Overall usefulness | Meant to measure how well we deliver our answers, regardless of whether they are correct: mostly their format. But a high usefulness score is most likely high because the question was answered correctly, so it does not really measure the quality of the format, as originally intended. |
| Error severity | Meant as a separate quality measure, since a small error with high consequences can make an answer much worse. But it calls for the same action as correctness and completeness together: it only means those two scores should be even higher. |

## Worked example

One question followed through both criteria, with Claude answering through our MCP tools. Parameter names and values are invented for illustration.

**Test question:** "What is the default bus-off recovery time, and what is it set to in my project?"

**Stored in the test set:** required facts (default 100 ms; project value 50 ms) and the expected references `D-42` and `CanChannel1`.

**During the run:**

1. Claude calls the documentation search tool and receives 8 large chunks. Our MCP server logs only which references they come from: `D-17, D-42, D-55, …`
2. Claude calls the project tool. The server logs `CanChannel1`.
3. Claude reads everything, selects what is relevant, and answers with a reference for each statement.

> The default bus-off recovery time is 100 ms \[D-42\]. In your project it is set to 50 ms \[CanChannel1\]. Shorter times recover faster but increase bus load \[D-55\].

D-55, a section on bus load, was not among the expected references.

**Grading:**

| Criterion | What is checked | Result |
| --- | --- | --- |
| 1. Correctness and completeness | Answer against the required facts; the project value is checked by a script against the configuration | Correct, complete 2 of 2; the bus-load claim is reported as "not in reference" |
| 2. Retrieval | Which references did the answer use, and are they expected or other relevant references? | Expected references used: 2 of 2 (100%). Relevant references used: 3 of 2 (150%); D-55 is relevant but was not expected, so it is reviewed and may be added to the expected references |

The evaluation never reads the 8 retrieved chunks. It only needs MCP tools that return a reference with each chunk, and Claude instructed to cite those references in its answer.

## Reporting and reliability

- Scores are for comparing versions and solutions on the same test set, not absolute measures of quality.

### Example report

| Criterion | v2.3 | v2.4 | Change |
| --- | --- | --- | --- |
| 1. Correctness: no contradictions with the reference answer | 86% | 87% | +1 (within noise) |
| 1. Completeness: required facts present | 74% | 81% | +7 |
| 2. Retrieval: expected references used | 78% | 85% | +7 |
| 2. Retrieval: relevant references used | 91% | 104% | +13 |

## Practical guides

Step-by-step guides for implementing the checklist, written for practitioners rather than researchers.

| Guide | Helps with |
| --- | --- |
| [Hamel Husain: Using LLM-as-a-Judge for Evaluation](https://hamel.dev/blog/posts/llm-judge/) | Building and calibrating the LLM judge: one domain expert grades pass/fail with a short critique, the critiques become examples in the judge prompt, and the judge is iterated until it agrees with the expert |
| [Ragas: Align an LLM as a Judge](https://docs.ragas.io/en/stable/howtos/applications/align-llm-as-judge/) | The same calibration, as runnable code in Ragas |
| [Ragas: Evaluate and Improve a RAG App](https://docs.ragas.io/en/stable/howtos/applications/evaluate-and-improve-rag/) | A hands-on walkthrough of evaluating a RAG application end to end |
| [Evidently AI: A complete guide to RAG evaluation](https://www.evidentlyai.com/llm-guide/rag-evaluation) | Overview of retrieval and answer evaluation, with and without reference answers, and generating synthetic test data |
| [Evidently AI: RAG evaluation tutorial](https://docs.evidentlyai.com/examples/LLM_rag_evals) | A notebook that scores retrieval and answers and produces a report |
| [DeepEval: Hallucination metric](https://deepeval.com/docs/metrics-hallucination) | A ready-made contradiction check against reference material (criterion 1) |
