# Wave 8 — state check: Dify #41854 (2026-09-15)

Purpose: state check at master for whoever takes the suggested-questions parse leg after #41881 closed unmerged. Single comment; no external cold read (state-verified only).

Posted: https://github.com/langgenius/dify/issues/41854#issuecomment-5672846442

## Claims (verbatim against main@6afe07f94475, read 2026-09-15 NZ)
- W1: the extract runs without stripping reasoning blocks: `api/core/llm_generator/output_parser/suggested_questions_after_answer.py:27` matches the first `\[.*?\]` right after `.strip()`. A numeric array inside `<think>` matches first, parses, then filters as non-strings to `[]`; an invalid one like `[CLS]` logs and returns `[]`. Either way, empty.
- W2: partial mitigation, not a fix for the parse leg: #41705 (Sep 3) added `_default_suggested_questions_model_parameters` (`api/core/llm_generator/llm_generator.py:82`), which requests `thinking=false` / lowest `reasoning_effort` where the model schema exposes those rules. Completions that still carry a reasoning block break the same way.
- W3: empty seats: no test file for this parser on master (`api/tests/unit_tests/core/llm_generator/output_parser/` holds only rule-config and structured-output tests); #41881's diff never landed.

Warranty: a miss here gets a public correction on the comment.
