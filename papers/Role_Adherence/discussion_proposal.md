# Role Adherence Metric

## Problem Statement

AI assistants deployed in production are almost always role-constrained — a customer support bot, a medical information assistant, a coding tutor. Despite this, models frequently deviate from their assigned roles: answering out-of-scope questions, abandoning their persona, or failing to exhibit the behavior the role demands.

Current evaluation frameworks either measure adherence per turn in isolation, or rely entirely on LLM judges with no deterministic fallback. Neither approach captures the full picture: a turn that looks off-role in isolation may be appropriate in context, and judge-only evaluation introduces reproducibility and cost concerns.

## Solution

We propose a **Role Adherence** metric that evaluates how consistently an assistant maintains its assigned role across a full multi-turn conversation.

**Key design decisions:**

- **Constructive adherence** — the metric penalises both scope violations (the assistant answers something it shouldn't) and positive failures (the assistant fails to exhibit what its role requires). A support agent that responds "I don't know" to a question within its domain fails constructive adherence even if it never goes out of scope.

- **`chatbot_role` as a `Dataset` field** — the role is a property of the session, not of the metric. It supports two formats: a free-form string for quick evaluation, or a structured `RoleDefinition` object (`description`, `allowed_topics`, `tone`, `restrictions`) for auditable, reproducible evaluation.

- **Full conversational context** — each turn is evaluated against the full conversation history up to that point, not in isolation.

- **Dual scoring path** — when no ground truth is available, an LLM-as-a-judge evaluates each turn against the role definition. When `ground_truth_assistant` is present, deterministic metrics are used instead: BERTScore as the primary metric, cosine similarity as a lightweight alternative, and NLI for explicit violation detection. Lexical metrics (ROUGE, BLEU) were considered and discarded — role-adherent responses are not expected to reproduce exact phrasing.

- **Configurable LLM judge output** — the judge can return binary scores (default, more consistent) or continuous scores (more expressive, but LLMs are not well-calibrated for numeric outputs).

**Score:** session-level scalar in `[0, 1]`, computed as the mean of per-turn adherence scores. Supports `threshold`, `strict_mode` (binary session verdict), `include_reason`, and `model` parameters.

## Which SDKs Would Benefit

- Python (Gaussia) — primary implementation
- TypeScript — future port

## Next Steps

The current proposal scopes the metric to text-only conversations. The primary direction for future work is extending it to **multimodal settings**: vision-capable judge models for image content, CLIP-based similarity for deterministic image scoring, and a `modalities` field in `RoleDefinition` to make modality compliance an explicit dimension of adherence.

A full design paper is available in `papers/Role_Adherence/`.
