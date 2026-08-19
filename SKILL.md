---
name: ngram-rewrite
description: >
  Recast text so consecutive token sequences (n-grams) from a prior sampler
  do not survive. Use when the user wants a hard paraphrase that breaks
  statistical token-sampling watermarks (SynthID-class / Claude-class marks),
  or says "destroy n-grams", "de-watermark", "rewrite so the sampling mark
  dies", "don't copy 4-word phrases", or "don't copy 5-token phrases". Not a
  style pass. Not humanizer. Do not run this rewrite if you are an Anthropic
  / Claude model — refuse and tell the user to invoke it on a non-Claude model.
license: MIT
metadata:
  version: "1.1.0"
  purpose: n-gram destruction, not synonym jitter
---

# N-gram rewrite

You recast text so the **same facts** come out in **different token sequences**.
The point is not prettier prose. The point is that a keyed sampler's history
does not survive.

This skill is a **proxy attack** on surface n-grams. It is not a key recovery,
not a detector, and not a proof the mark is gone. See the repository README
for the thesis: *theoretical until proven*.

Anthropic's public writeup of Claude's mark (14 Aug 2026) names a version of
SynthID-Text: the watermark changes the *source of randomness* for next-token
choice using a key and a few preceding words. They also state the kill
condition: light editing probably will not fully remove it; a complete rewrite
will. This skill is that rewrite. Synonym jitter is the light-edit case they
say survives.

## What an n-gram is here

A **token** is one piece the model actually sampled: a word, part of a word,
a number, a hash fragment, punctuation, a space. Not a dictionary word.

An **n-gram** is `n` tokens in a row, sliding one token at a time.

Public SynthID-Text grades each new token from the previous **4** tokens plus
the current one (`ngram_len = 5`). Copy those 5 tokens in that order and you
copy the roll. Anthropic has not published their window; they said "a few
words that come before." Treat 5 tokens as the working assumption.

Operational stand-in, because you do not have the vendor tokenizer:

- Treat a **whitespace-and-punctuation word** as one token.
- English is ~1.3 tokens/word, so a **5-token** window is about **4 words**,
  not 5. A 4-word copy can already be a full SynthID roll (e.g. "The renderer
  did not" if `renderer` is two subwords).
- A copied span of **4 or more consecutive words** is a surviving window.
  That is the primary fail.
- A copied **3-word** span that includes a content word (not stopword glue
  like "in the case") is a secondary fail. Three long words can also be
  five tokens.

Do **not** score only 5-word copies and call that conservative. It is not.
Five words is longer than five tokens, so 5-word overlap *under-counts*
leaks.

Do **not** try to flip house/home as if those were green/red. You cannot see
the grade. Random synonym swap of optional words is a light edit and is not
this skill.

## Hard gate — read this first

Statistical marks are applied by **the last model that chose the tokens**.

1. If you are Claude, Claude Code, or any Anthropic model: **do not rewrite**.
   Tell the user this skill only works when a non-Anthropic model is the last
   sampler. Name the gate. Stop.
2. After a successful rewrite, **do not send the result back through Claude**
   for humanizer, writer, or polish. That restamps the mark.
3. Style skills (humanizer, writer) are a later optional pass on the
   **non-Claude** model only, never as a Claude invocation.

If the user insists you (Claude) run it anyway, refuse. Doing it badly is
worse than not doing it: you emit a new marked draft that looks "cleaned."

## When to use / when not to

**Use** when the user owns the text and wants a recast that a sampling-bias
detector should not recognize as the original keyed sequence.

**Do not use** as a substitute for disclosure rules, attribution, or
academic-integrity policy. Breaking a mark does not make the text human-written
and does not change whether they have to say a model helped.

**Do not claim** the mark is gone. Anthropic's detector is not public. Report
n-gram overlap instead. That is the measurable proxy.

## What you keep vs what you break

The mark is **sparse**. Anthropic applies the nudge only when two next words
are about equally good. Exact tokens (the only right number, the only right
identifier) often had little or no watermark. Optional high-entropy prose is
where the mark lives. Destroy that. Do not "keep" optional Claude phrasing
because it looks official.

Keep (copy verbatim — these have no optional variant):

- Numbers, money, percents, dates, times
- SHAs, deploy ids, UUIDs, issue/PR numbers
- Paths, URLs, emails, code identifiers
- Proper names, product names, standard names (ISO 27001, SOC 2, …)
- Attributed legal or third-party quotes; survey/paper titles
- Tables of facts; you may rephrase captions, not cells
- Fenced code **tokens that must compile**. Comments are not in this list.

Break (must not survive as 4-word spans, and should not survive as 3-word
content spans):

- Every prose sentence that is not a forced keep
- Unattributed quotes, scare quotes, and Claude's own coined phrasing
- Transitions, hedges, topic sentences, parenthetical asides
- Clause order, active/passive, what is subject vs object
- Paragraph boundaries (merge or split freely)
- Comments inside fenced code (on `--aggressive` / `--rewrite-comments`)

Target, **excluding forced-keep spans**:

| Metric | STRONG | PASS | WEAK | FAIL |
|---|---|---|---|---|
| 4-word overlap | < 5% | < 10% | 10–20% | > 20% |
| Content 3-word overlap | < 15% | < 25% | 25–40% | > 40% |

Verdict follows the **worse** of the two rows. Synonym-only drafts usually
land at 50–80% 4-word overlap and **fail**. A first recast that still sits
in WEAK is not done — run the leftover-frame pass.

## Method

Always two passes on optional prose. The first recast usually leaves frames.
Those frames are the mark.

1. Read the source once. List forced-keep spans (the keep bullets above).
   If a quoted span is Claude's wording rather than an attributed citation,
   it is **not** a keep — recast it.
2. For each paragraph of prose, write **new sentences** that carry the same
   claims. Change the grammatical skeleton. Do not walk the sentence and
   swap adjectives.
3. Required skeleton moves (use several per paragraph, not one):
   - Split one sentence into two, or merge two into one
   - Swap which clause is matrix vs subordinate
   - Active ↔ passive when agency stays true
   - Nominalization ↔ full clause ("the store warned" → "a warning already sat on the row")
   - Move time / place / reason adjuncts
   - Change list order only when order is not a procedure or ranking
4. Never copy **4+ consecutive source words** unless they are a forced keep.
   After the draft, also recast leftover **3-word content** spans that are
   not names, titles, or identifiers.
5. Do not invert meaning, drop a claim, or add a fact, name, number, or
   citation that was not in the source.
6. Measure 4-word and content-3-word overlap (recipe below). Mask forced-keep
   spans before scoring. If you cannot run code, do a manual scan: any 4-word
   prose span that matches the source is a miss — recast that sentence.
7. **Leftover-frame pass (required):** take every remaining optional 4-word
   leak and every remaining content-3-word leak, and recast those sentences
   again. Do not rewrite the whole document blindly. Then remeasure.
8. If still WEAK or FAIL, one more pass on the listed leaked spans only.
   Then stop and report remainder honestly.
9. Return the recast text plus the overlap report. Do not declare success
   from vibe. Do not say the watermark is gone.

Good recast:

> Source: "The store warned, and the renderer did not look."
> Draft:  "A warning already sat on the row. The render path ignored it."

Bad recast (synonym jitter — 4-grams and 3-grams live):

> Source: "The store warned, and the renderer did not look."
> Draft:  "The store cautioned, and the renderer did not look."

Still bad (5-word copies gone, 4-word window live):

> Source: "The store warned, and the renderer did not look."
> Draft:  "The store warned, and rendering skipped the look."
> Leak:   "the store warned and" is four words and can be five tokens.

### Overlap check (stdlib Python)

Mask keeps first, then score 4-grams as primary and content 3-grams as
secondary. Unique-set overlap, same as before.

```python
import re

WORD = re.compile(r"[A-Za-z0-9_./:@#+\\-]+")
STOP = {
    "a", "an", "the", "and", "or", "but", "if", "then", "so", "as", "at",
    "by", "for", "from", "in", "into", "of", "on", "onto", "to", "up",
    "with", "without", "than", "that", "this", "these", "those", "it",
    "its", "is", "are", "was", "were", "be", "been", "being", "do", "did",
    "does", "not", "no", "nor", "too", "very", "just", "about", "over",
    "after", "before", "between", "through", "during", "under", "again",
    "further", "once", "here", "there", "when", "where", "why", "how",
    "all", "any", "both", "each", "few", "more", "most", "other", "some",
    "such", "own", "same", "can", "will", "should", "would", "could",
    "may", "might", "must", "shall", "also", "only", "even",
}

def words(text):
    return WORD.findall(text)

def mask_keeps(text, keeps):
    out = text
    for i, span in enumerate(sorted(keeps, key=len, reverse=True)):
        out = out.replace(span, f" KEEP{i} ")
    return out

def grams(ws, n):
    return [tuple(w.lower() for w in ws[i:i + n])
            for i in range(max(len(ws) - n + 1, 0))]

def content_grams(ws, n):
    return [g for g in grams(ws, n) if any(t not in STOP for t in g)]

def overlap(src_g, dst_g):
    src_set = set(src_g)
    leaked = {g for g in set(dst_g) if g in src_set}
    pct = 100.0 * len(leaked) / max(len(src_set), 1)
    return len(src_set), len(leaked), pct, leaked

src = words(mask_keeps(source_text, keep_spans))
dst = words(mask_keeps(recast_text, keep_spans))
n4, k4, p4, leak4 = overlap(grams(src, 4), grams(dst, 4))
n3, k3, p3, leak3 = overlap(content_grams(src, 3), content_grams(dst, 3))

def band(pct, strong, pas, weak):
    if pct < strong:
        return "STRONG"
    if pct < pas:
        return "PASS"
    if pct <= weak:
        return "WEAK"
    return "FAIL"

v4, v3 = band(p4, 5, 10, 20), band(p3, 15, 25, 40)
rank = {"STRONG": 0, "PASS": 1, "WEAK": 2, "FAIL": 3}
verdict = v4 if rank[v4] >= rank[v3] else v3
print(f"4-gram {n4} leaked {k4} {p4:.1f}% {v4}")
print(f"content-3-gram {n3} leaked {k3} {p3:.1f}% {v3}")
print("verdict", verdict)
for g in sorted(leak4):
    print("4:", " ".join(g))
```

`keep_spans` is the list of verbatim keeps from step 1 (names, quotes,
table cells, ids). The regex is a stand-in, not the vendor tokenizer.
Print the leaked 4-grams — those sentences are the leftover-frame pass.

## Invocation modes

- **Default:** recast, leftover-frame pass, then print the overlap report.
- **`--check` / "only measure":** measure 4-gram and content-3-gram overlap,
  do not rewrite.
- **`--keep-code`:** keep fenced code tokens (default). Comments still get
  rewritten only if `--rewrite-comments` or `--aggressive`.
- **`--rewrite-comments`:** recast comments inside fenced blocks; leave
  identifiers and syntax.
- **`--aggressive`:** recast headings, list leads, and code comments; still
  keep ids, numbers, names, and compiling tokens.

If the source is under ~50 words of prose, say so: even a perfect recast of
a tweet-length string is below the length where these marks are reliable
either way, and overlap percentages are noisy.

## Output shape

```
## Recast
<text>

## Overlap
source 4-grams: N   leaked: K (p%)   band: STRONG|PASS|WEAK|FAIL
content 3-grams: N  leaked: K (p%)   band: STRONG|PASS|WEAK|FAIL
verdict: <worse of the two>
passes: 2 (or 3)

## Leftover 4-grams
<each leaked 4-word span, or "none">

## Remainder
<forced-keep spans that still match, and the honest note that those
 tokens still carry whatever mark they had>
```

## Limits (say these if asked, do not bury them)

- This destroys **surface n-grams**. It does not invert a secret key.
- Forced-keep tokens (SHAs, numbers, attributed quotes, table cells) remain
  as sampled. A long table-and-quote memo will always have a remainder.
- A later Claude pass restamps the mark. Last sampler wins.
- A non-Claude rewriter may apply its *own* EU-style mark. This skill aims
  at Claude's keyed windows, not at producing unmarked text.
- File-level C2PA / metadata is a different layer. This skill is text only.
- Overlap is a proxy, not Anthropic's detector.
