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

## Update — spaCy structural analysis added as a third data point

language-tools-api gained a `/linguistic-analysis` endpoint using spaCy, 
returning average dependency tree depth, lexical diversity (type-token 
ratio), and noun-to-verb ratio. Running the same three test texts through 
it adds a third lens alongside Flesch-Kincaid and the LLM assessments:

| Text | Flesch label | LLM assessment | Dependency depth | Lexical diversity | Noun:Verb |
|---|---|---|---|---|---|
| "The cat sat on the mat." | Very Easy | Very Easy (both) | 4.0 | 0.83 | 2.0 |
| Scientists/exercise text | Difficult | Easy / Very Easy (both) | 4.33 | 0.96 | 1.5 |
| Epistemological text | Very Difficult | Very Difficult (both) | 5.0 | 0.92 | 4.0 |

### Finding — dependency depth is a weak discriminator here; vocabulary signals matter more

Dependency tree depth barely changes across all three texts (4.0 to 5.0) 
despite Flesch-Kincaid scoring them at opposite extremes. This sentence 
set happens to use mostly flat, prepositionally-modified structures rather 
than deeply nested clauses, so depth alone does not capture the difficulty 
gap.

Lexical diversity and noun-to-verb ratio align more closely with what the 
LLMs seem to be picking up on. The scientists/exercise text has the 
highest lexical diversity of the three (0.96) and a noun:verb ratio (1.5) 
closer to the simple sentence (2.0) than to the noun-dense epistemological 
text (4.0). This is consistent with the LLMs rating it "Easy" — varied but 
common vocabulary, balanced verb usage, narrative sentence rhythm — even 
though Flesch-Kincaid scores it "Difficult" based on syllable and word 
count alone.

### Implication

Flesch-Kincaid, spaCy structural metrics, and LLM judgment are measuring 
overlapping but distinct things. Syllable-based formulas conflate 
vocabulary difficulty and sentence length into one number. Dependency 
depth isolates structural nesting specifically, which may not move much 
even when other complexity dimensions do. LLM assessment appears to weight 
vocabulary familiarity and rhythm more heavily, closer to subjective human 
reading experience but with no formal calibration behind it.

No single metric here is the "correct" one. This three-way comparison is 
the more honest finding than any single score.

## Update — output consistency across repeated runs

Tested whether each model's readability classification is stable across 
repeated runs of the same input, using promptfoo's `--repeat` flag (note: 
this is a CLI flag in this promptfoo version, not a config-file key — 
`repeat:` in promptfooconfig.yaml is silently ignored).

```bash
promptfoo eval --no-cache --repeat 3
```

### Finding — zero variance across all three texts, both models

Despite running at Anthropic's default temperature (1.0, not explicitly 
set to 0 anywhere in this config), both Claude Haiku and Claude Sonnet 
returned identical `difficulty` and `reading_level` values across all 3 
repeats, for all 3 test texts:

- "The cat sat on the mat." — both models: Very Easy, grade 1, every run
- Scientists/exercise text — Haiku: Very Easy every run, Sonnet: Easy 
  every run (both still disagreeing with the Flesch-Kincaid "Difficult" 
  label, but consistently so)
- Epistemological text — Haiku: Very Difficult/grade 16, Sonnet: Very 
  Difficult/grade 18, every run

Reasoning text varied slightly in wording between repeats, but the 
structured judgment fields (difficulty label, grade level) did not move 
at all.

### Implication

For this specific classification task — picking one label from a fixed 
set of six — model output appears highly stable even without explicitly 
setting temperature to 0. This suggests the readability judgment for 
these three texts isn't sitting near a decision boundary for either 
model, including the "Scientists" text that sits in contested territory 
between the models and the formula. The models aren't uncertain or 
wavering on that text — they're consistently confident in a label that 
consistently disagrees with Flesch-Kincaid.

### Limitation

Three repeats on three texts is a small sample. It demonstrates stability 
on these specific inputs, not a general claim that LLM readability 
classification is always deterministic. This result does not tell us 
whether the Scientists text is far from a decision boundary or sitting 
right at one that the model happens to land on consistently — distinguishing 
those would require more repeats, or testing with deliberately perturbed 
variants of the same text.

## Finding — promptfoo's llm-rubric vs DeepEval's FaithfulnessMetric on a known fabrication case

### Background

language-ai-qa's test_first_eval.py documents a finding (see that repo's NOTES.md, 
Day 2-3) that DeepEval's HallucinationMetric is contradiction-based and misses 
unsupported additions — a fabricated-but-plausible claim that doesn't contradict 
the source context passes HallucinationMetric, but is correctly caught by 
FaithfulnessMetric when configured with `penalize_ambiguous_claims=True`.

The specific test case: given context describing a Taíno yukayeke (village), an 
output claiming villages "were always built near sacred rivers used for ritual 
bathing" — a fabrication not present in or contradicted by the source — passes 
HallucinationMetric and fails FaithfulnessMetric (correctly configured).

This investigation asks: does promptfoo's `llm-rubric` assertion, given the exact 
same context and exact same fabricated output, reach the same verdict as 
FaithfulnessMetric?

### Method

Used promptfoo's `echo` provider to pass the known fabricated output through 
unchanged (no model generation involved), graded by a single `llm-rubric` 
assertion using Claude Haiku 4.5 as judge — the same model DeepEval used. The 
rubric instruction was written to mirror FaithfulnessMetric's strict 
configuration explicitly: "Any claim not grounded in the context — even if 
plausible — should cause this to fail."

Using the exact same context and output as a control, rather than letting 
promptfoo generate a fresh response, isolates the comparison to the grading 
mechanism itself rather than introducing a second variable (a new model call 
that might not reproduce the same fabrication).

### Two real bugs found before getting a trustworthy result

**Bug 1 — unsubstituted template variable.** The first config referenced the 
context via a `{{context}}` variable inside the `llm-rubric` value string. The 
result showed `[PASS]`, but inspecting the rendered prompt in `promptfoo view` 
showed the literal text `{{context}}` had never been interpolated — the judge 
model was asked to grade against placeholder syntax, not real content. Fixed 
by inlining the context directly as literal text in the YAML rather than 
templating it.

**Bug 2 — misplaced `assert` key.** After fixing the template issue, the result 
was still `[PASS]`, but with 0 tokens used for grading — meaning no actual 
judge call happened. The CLI had been printing a warning on every run 
("Found 'assert' key in vars... should be unindented so it is under the test 
itself, not vars") that was dismissed as noise. Inspecting the exported JSON 
confirmed it: `assert` was nested inside `vars:` rather than being a sibling 
key at the test level, so promptfoo found zero real assertions to run, and 
`gradingResult.reason` read "No assertions" — a default pass with no 
evaluation behind it. Fixed by correcting the YAML indentation so `assert:` 
sits at the same level as `vars:`.

Both bugs produced a misleading `PASS` for unrelated infrastructure reasons, 
before any real grading occurred. Neither was caught by reading the summary 
table alone — both required inspecting either `promptfoo view`'s expanded 
detail or the raw exported JSON to find the actual judge response (or lack 
of one).

### Result

With both bugs fixed (confirmed by: the CLI warning gone, real token usage 
logged — 510 total, 336 prompt / 174 completion — meaning a genuine judge 
call occurred), the result is `[FAIL]`.

promptfoo's `llm-rubric`, given the real context and real fabricated output, 
correctly identifies "sacred rivers used for ritual bathing" as unsupported 
by the source — matching FaithfulnessMetric's verdict, not 
HallucinationMetric's.

### Interpretation

This isn't evidence that promptfoo is "better" than DeepEval or vice versa. 
Both tools use the same underlying mechanism — an LLM judge following 
instructions — and both can be configured to catch or miss unsupported 
additions depending on how explicitly the instruction is written. 
FaithfulnessMetric needs `penalize_ambiguous_claims=True` to catch this case; 
the default configuration produces HallucinationMetric's blind spot (see 
language-ai-qa NOTES.md). Likewise, a vaguely-worded `llm-rubric` instruction 
(e.g. "is this output accurate?") would likely be more lenient than one that 
explicitly states ungrounded-but-plausible claims should fail.

The real variable is the strictness and explicitness of the grading 
instruction, not the tool. Both promptfoo and DeepEval are flexible enough 
to produce either a strict or lenient verdict on the same input — the burden 
is on whoever configures the eval to be explicit about what "faithful" means 
for their domain.

### Process note

Two CLI warnings/results were dismissed too quickly in this investigation — 
the repeated "Found 'assert' key in vars" message specifically. Worth treating 
CLI warnings with the same seriousness as failing tests going forward: a tool 
telling you something is probably wrong, repeatedly, across multiple runs, is 
worth investigating immediately rather than after a misleading result has 
already been read as a finding.