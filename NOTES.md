# Promptfoo Readability Eval — Notes

## What this is

A small standalone experiment comparing LLM-based readability assessment 
against the Flesch-Kincaid statistical formula, using promptfoo. Built as 
a learning exercise to gain hands-on experience with promptfoo's config 
structure, assertions, and multi-model comparison, alongside an explicit 
research question.

This project is separate from language-ai-qa. It is not scoped to 
low-resource or indigenous language evaluation — that domain requires 
community partnership and authoritative benchmarks that are out of scope 
here. This experiment uses standard English text only.

## Setup

- Provider: Anthropic Claude Haiku 4.5 and Claude Sonnet 4.6
- Ground truth: Flesch Reading Ease scores and labels verified against 
  the `/readability` endpoint in language-tools-api
- Three test texts spanning the difficulty range: Very Easy, Difficult, 
  Very Difficult

## Finding — model agreement with the formula is range-dependent

Both models agree with Flesch-Kincaid at the extremes:

- Simple text ("The cat sat on the mat.") — both models correctly assess 
  as Very Easy
- Complex academic text (epistemological/poststructuralist sentence) — 
  both models correctly assess as Very Difficult

Both models disagree with the formula in the middle range:

- Moderately complex text about exercise and memory scores "Difficult" 
  by Flesch-Kincaid (score 48.36) but both Haiku and Sonnet assess it as 
  "Very Easy" / "Easy"

This suggests the formula and the models are measuring different things. 
Flesch-Kincaid counts syllables and sentence length mechanically. The 
models appear to weight vocabulary familiarity and semantic accessibility 
more heavily — the exercise/memory text uses common words and clear 
sentence structure even though it has moderate syllable counts and 
sentence lengths.

Neither method is necessarily more "correct" — they capture different 
dimensions of readability. This is a methodological finding, 
not a bug in either system.

## Technical note — promptfoo javascript assertion syntax

This version of promptfoo (0.121.15) expects javascript assertions to be 
a single bare expression, not a function body with `return` statements. 
`value: "return true;"` fails silently with no error; `value: "true"` 
passes. Assertions must be written as expressions:

```yaml
value: "JSON.parse(output.replace(/```json|```/g, '').trim()).difficulty === 'Very Easy'"
```

This was not obvious from initial config attempts and took significant 
debugging to isolate — confirmed by testing with a minimal `true`/`false` 
expression before adding real logic back in.

## Model behavior note

Claude Haiku consistently wraps JSON output in markdown code fences 
despite explicit instructions not to. Claude Sonnet follows the 
no-markdown instruction reliably. This is a real difference in 
instruction-following between the two models worth noting for prompt 
engineering — smaller models may need more redundant or explicit 
formatting instructions.

## Status

Working experiment, three test cases, assertions passing/failing as 
expected. Possible next steps: add `repeat` to test output consistency 
across multiple runs, test with non-English text to see if the agreement 
pattern holds, or test with the spaCy comparison once that exists in 
language-tools-api.