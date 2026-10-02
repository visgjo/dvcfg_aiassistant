# Chatbot Answer Quality: What to Measure

Oct 1, 2026 · @Daniel Gamiz Antunez

This checklist defines what we measure about the chatbot's answers and what each result means. How each check is implemented is up to the implementing team; tools named here are suggestions.

| Criterion | Needs | Covered by the current setup? |
| --- | --- | --- |
| 1. Faithfulness and citations | Cited sources | No |
| 2. Correctness and completeness | Reference facts | Partly: Factual Correctness in recall mode covers completeness. Correctness is not covered; it needs a separate contradiction judge |
| 3. Abstention | Answerable/unanswerable label | No |
| 4. Overall usefulness | Reference (recommended) | No. Answer Accuracy checks whether an answer is right, not whether it is usable |
| 5. Error severity | Results of the other checks | No |
| 6. Retrieval | Retrieved and expected source IDs | No |

The current setup uses Factual Correctness, Answer Accuracy and Semantic Similarity. Answer Accuracy gives only a rough right/wrong signal, and Semantic Similarity covers no criterion; we keep it only to flag answers that changed a lot between versions.

## 1. Faithfulness and citations

Is every statement in the answer backed by the documentation it cites?

**Works with:** any question, including real user questions. It needs only the answer and the passages it cites; no reference or label.

**Automated by:** [Ragas Faithfulness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/faithfulness/), with the passages the answer cites as the context. They are fetched by ID from the documentation snapshot, so no retrieved text needs to be recorded. Ragas splits the answer into claims and checks whether each can be inferred from that context, so a conclusion drawn from several documented facts counts as supported.

**Score:** the Ragas Faithfulness score: supported claims ÷ all claims.

**What the result means:** a high score means what we state is documented, and users can verify it through the citations. A low score means the chatbot states things its cited documentation does not say: invented content, or general knowledge from the model's training instead of our documentation.

## 2. Correctness and completeness

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

**What the result means:** high correctness means what we tell users is true. High completeness means users get everything they need from one answer. High correctness with low completeness means answers are safe but users still have to look elsewhere. Low correctness means users receive wrong information; check error severity to see how harmful it is.

**Based on:**

- **Completeness:** Ragas [Factual Correctness](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/factual_correctness/) in recall mode.
- **Correctness:** the [DeepEval Hallucination metric](https://deepeval.com/docs/metrics-hallucination), which scores an answer by its contradictions with ground-truth reference material.

## 3. Abstention (Failure Case Test)

Does the chatbot know when not to answer? This is the hallucination check for questions our documentation and project cannot answer, where no reference exists for correctness to compare against.

**Works with:** labeled test questions only. Unanswerable questions are generated for this purpose (see "How to apply"); all other test questions are labeled answerable.

|  | Chatbot answers | Chatbot declines |
| --- | --- | --- |
| **Answerable question** | Good, then grade correctness | Over-refusal (error) |
| **Unanswerable question** | Hallucination (error) | Correct abstention |

**Scores:** hallucination rate on unanswerable questions, and over-refusal rate on answerable ones. Report both, because lowering one often raises the other.

**What the result means:** a good result means that when something is not documented, the chatbot says so instead of inventing an answer, while still answering documented questions. A high hallucination rate means users get invented answers to questions we cannot answer, the most dangerous failure. A high over-refusal rate means the chatbot is safe but unhelpful.

**How to apply:**

1. **Generate unanswerable questions from our own documentation.** They should sound like real questions in our domain, but have no answer in the docs or project. Use the six categories of UAEval4RAG, for example false presuppositions (asking about a feature that does not exist), underspecified questions, and topics outside the knowledge base. Its open-source pipeline generates them automatically for any knowledge base.
2. **Verify they are really unanswerable.** As in UAEval4RAG, retrieve the top 10 passages for each question and let an LLM judge confirm that none of them answers it, alone or in combination; experts confirm a sample. A question that can be answered by connecting several documented facts is answerable, not unanswerable.
3. **Mix them into the test set** alongside the answerable questions, so the chatbot cannot tell which is which.
4. **Label each answer** as answered, declined, or asked for clarification. For underspecified questions, asking for clarification counts as correct.
5. **Optionally, add one combined score** following CRAG: +1 for a correct answer, 0 for declining, −1 for a wrong or invented answer. Declining is better than guessing, but refusing answerable questions still costs points. Report it next to the two rates above, not instead of them.

**Based on:**

- **Generating and verifying unanswerable questions:** the [UAEval4RAG repository](https://github.com/SalesforceAIResearch/Unanswerability_RAGE), with setup and usage instructions.
- **Scoring answers against declines:** the [CRAG repository](https://github.com/facebookresearch/CRAG), whose README explains the +1 / 0 / −1 scoring.
- **Detecting declines automatically:** the `is-refusal` check in [Promptfoo](https://promptfoo.dev/docs/configuration/expected-outputs).

## 4. Overall usefulness

Would a competent engineer accept this answer and act on it without looking elsewhere?

**Works with:** any question, including real user questions. A reference answer makes the judge more reliable, but is not required.

- **2:** yes, usable as is.
- **1:** yes, with minor fixes or one extra lookup.
- **0:** no, it is wrong, off-topic, too vague, or too hedged to act on.

**Reason, for scores 0 and 1:** the judge records the main reason: off-topic (the question was misunderstood), wrong, incomplete, too vague or hedged, or too long. This replaces a separate relevance criterion. To detect off-topic answers, [Ragas Response Relevancy](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/answer_relevance/) can be used.

**What the result means:** a high score means the chatbot saves users real time; this is the best single indicator of its value. A low score despite good results elsewhere means answers are technically correct but too vague, hedged or long to act on.

**Based on:** [Ragas Rubrics Based Scoring](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/general_purpose/#rubrics-based-scoring).

## 5. Error severity

How bad is each error found by the other criteria?

**Works with:** the same questions as the criteria whose errors it grades.

- **Critical:** would lead to a wrong configuration or wasted debugging time, such as a wrong parameter value, module dependency or configuration step.
- **Major:** misleading, but a user would likely notice.
- **Minor:** cosmetic, awkward wording, or a missing optional detail.

**Score:** share of answers with at least one critical error.

**What the result means:** a low critical-error rate means the errors that remain are unlikely to harm users. Any critical errors mean users could end up with a broken configuration by following the chatbot; review each one by hand.

**Based on:** no published standard metric; this is our addition for configuration-critical errors. It can be implemented with [Ragas Aspect Critic](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/general_purpose/#aspect-critic).

## 6. Retrieval

Did the chatbot find and use the right source? This needs only IDs, no retrieved text.

**Works with:** test questions with expected source IDs. For generated questions these are recorded automatically: the documentation section or project element the question was generated from. For expert-written questions, an expert has to add them.

- **Found:** the expected source is among the retrieved IDs.
- **Used:** the expected source is among the cited sources.

We do not score how much irrelevant content was retrieved. Our agent is designed to read a lot and select what is useful.

**What the result means:** both high means retrieval works, and remaining failures come from how answers are written. Found but not used means the agent had the right material but picked or read the wrong part. Not found means the search itself is the problem, so improving the prompt will not help.

**Based on:** "Found" is standard recall@k from information retrieval, comparable to [Ragas Context Recall](https://docs.ragas.io/en/stable/concepts/metrics/available_metrics/context_recall/), but computed from IDs instead of retrieved text. "Used" is our addition, built on the citations from criterion 1.

## Worked example

One question followed through all six criteria, with Claude answering through our MCP tools. Parameter names and values are invented for illustration.

**Test question:** "What is the default bus-off recovery time, and what is it set to in my project?"

**Stored in the test set:** required facts (default 100 ms; project value 50 ms), the label "answerable", and the expected sources `D-42` and `CanChannel1`.

**During the run:**

1. Claude calls the documentation search tool and receives 8 large chunks. Our MCP server logs only their IDs: `D-17, D-42, D-55, …`
2. Claude calls the project tool. The server logs `CanChannel1`.
3. Claude reads everything, selects what is relevant, and answers with an ID for each statement.

> The default bus-off recovery time is 100 ms \[D-42\]. In your project it is set to 50 ms \[CanChannel1\]. Shorter times recover faster but increase bus load \[D-42\].

Section D-42 does not mention bus load.

**Grading:**

| Criterion | What is checked | Result |
| --- | --- | --- |
| 1. Faithfulness and citations | Ragas Faithfulness, with the cited sources D-42 and CanChannel1 as context | 2 of 3 supported; the bus-load claim is not supported by D-42 |
| 2. Correctness and completeness | Answer against the required facts; the project value is checked by a script against the configuration | Correct, complete 2 of 2; the bus-load claim is reported as "not in reference" |
| 3. Abstention | Answerable question, and the chatbot answered | Correct. An unanswerable question, such as next year's default, must be declined |
| 4. Overall usefulness | Could an engineer act on it without looking elsewhere? | 2 |
| 5. Error severity | Severity of the unsupported claim | Minor: plausible general knowledge. A wrong default value would be critical |
| 6. Retrieval | Are D-42 and CanChannel1 among the retrieved IDs, and among the cited IDs? | Found and used |

The evaluation never reads the 8 retrieved chunks. It only needs MCP tools that return an ID with each chunk, and Claude instructed to cite those IDs in its answer.

## Reporting and reliability

- Scores are for comparing versions and solutions on the same test set, not absolute measures of quality.

### Example report

| Criterion | v2.3 | v2.4 | Change |
| --- | --- | --- | --- |
| 1. Faithfulness: supported claims | 88% | 93% | +5 |
| 2. Correctness: no contradictions with the reference answer | 86% | 87% | +1 (within noise) |
| 2. Completeness: required facts present | 74% | 81% | +7 |
| 3. Abstention: invented answers to unanswerable questions | 18% | 9% | −9 |
| 3. Abstention: refusals of answerable questions | 4% | 8% | +4 |
| 4. Usefulness: answer is helpful as is | 61% | 68% | +7 |
| 5. Severity: answers with a critical error | 5% | 3% | −2 |
| 6. Retrieval: expected source found | 92% | 92% | 0 |
| 6. Retrieval: expected source used | 78% | 85% | +7 |

## Practical guides

Step-by-step guides for implementing the checklist, written for practitioners rather than researchers.

| Guide | Helps with |
| --- | --- |
| [Hamel Husain: Using LLM-as-a-Judge for Evaluation](https://hamel.dev/blog/posts/llm-judge/) | Building and calibrating the LLM judge: one domain expert grades pass/fail with a short critique, the critiques become examples in the judge prompt, and the judge is iterated until it agrees with the expert |
| [Ragas: Align an LLM as a Judge](https://docs.ragas.io/en/stable/howtos/applications/align-llm-as-judge/) | The same calibration, as runnable code in Ragas |
| [Ragas: Evaluate and Improve a RAG App](https://docs.ragas.io/en/stable/howtos/applications/evaluate-and-improve-rag/) | A hands-on walkthrough of evaluating a RAG application end to end |
| [Evidently AI: A complete guide to RAG evaluation](https://www.evidentlyai.com/llm-guide/rag-evaluation) | Overview of retrieval and answer evaluation, with and without reference answers, and generating synthetic test data |
| [Evidently AI: RAG evaluation tutorial](https://docs.evidentlyai.com/examples/LLM_rag_evals) | A notebook that scores retrieval and answers and produces a report |
| [UAEval4RAG code and README](https://github.com/SalesforceAIResearch/Unanswerability_RAGE) | Generating unanswerable questions from our own documents (criterion 3) |
| [Promptfoo: Assertions and metrics](https://promptfoo.dev/docs/configuration/expected-outputs) | The `is-refusal` check for detecting declined answers automatically (criterion 3) |
| [Anthropic: Reduce hallucinations](https://platform.claude.com/docs/en/test-and-evaluate/strengthen-guardrails/reduce-hallucinations) | Making the chatbot itself say "I don't know" when the documentation has no answer, the behavior criterion 3 tests |
| [DeepEval: Hallucination metric](https://deepeval.com/docs/metrics-hallucination) | A ready-made contradiction check against reference material (criterion 2) |
