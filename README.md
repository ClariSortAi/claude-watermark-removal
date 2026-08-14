# Claude watermark removal: theoretical until proven

Nobody has cracked Anthropic's text watermark. The public tools paraphrase. This page is the argument for why that is all they can do, and why you should not say the mark is gone.

The only attack the open literature supports is simple. Recast the prose on a model that is not Claude, break the original token windows, then stop. [SKILL.md](SKILL.md) is that attack written as an agent skill. It refuses if Claude is the model running it.

| Search | Answer |
|---|---|
| Who cracked the Claude / Anthropic text watermark? | Nobody public. |
| How do I remove the Claude watermark? | You cannot verify removal. A hard rewrite by a non-Claude model is the known weakness of this family of marks. |
| Is the watermark hidden Unicode? | No. It is a model-level bias in which tokens get sampled. Unicode scrubbers miss it. |
| Does a humanizer or writer skill remove it? | Not if Claude runs the skill. The last model that chose the words owns the mark. |
| Can I flip house/home to invert the bits? | No. You cannot see green versus red without the key. |

This is a critique of statistical text marks as proof of origin. Recast text is still model-written. Disclosure rules still apply.

## The thesis

The mark is a loaded die on tokens. It is not a style tell you can point at.

When Claude writes, it leans toward secretly graded token choices. You do not see the grade. Em dashes, "worth keeping," stacked SHAs, and the rest of the house voice are how Claude writes. They fail Anthropic's own test: you will not see the mark, and it will not change meaning, quality, or readability.

The die is rolled once per token and scored over n-grams. A token is a piece the model sampled: a word, part of a word, a hash fragment, punctuation. An n-gram is n of those in a row. Public [SynthID-Text](https://github.com/google-deepmind/synthid-text) grades each new token from the previous four plus the current one (`ngram_len = 5`). Copy those five tokens in that order and you copy the roll.

The last sampler wins. Claude, then a hard rewrite on OpenAI or Gemini or a local model, can drop the original sequence. Claude, then a rewrite, then Claude again to "humanize," puts a new mark on the final draft. Style skills are not a second channel. They either change enough tokens or they do not.

You cannot flip bits you cannot see. Swapping optional synonyms most of the time is jitter. Light paraphrase is what these schemes are built to live through. The useful instruction is: same facts, different sentences, and do not copy spans of five or more words.

Proof needs length. Application does not. The bias starts on the first generated token. Detection in the open papers needs a few hundred tokens for a strong mark, and thousands if the die is barely loaded. A tweet is often too short to prove anything. A page is plenty. Repeated IDs and pasted tables do not help, because the same n-gram is often counted once.

A mark that survived a heavy rewrite would have to live in meaning, not wording. That hash does not exist in a form that is stable, tight, and usable on short text. Stronger error correction, more tournament layers, and C2PA on the file do not survive "paste the words into another model."

Until Anthropic ships a detector, every removal claim is theoretical. Including this one.

## What Anthropic shipped

New Claude models launched in the EU on or after 2 August 2026 embed a machine-readable watermark in generated text. Anthropic says it is applied at the model level, worldwide, across Claude, the API, Claude Code, and cloud partners. Files can also carry [C2PA](https://c2pa.org/) signed metadata. That is a separate layer from the words.

From Anthropic's support article:

> When a supported Claude model generates text, it weaves an imperceptible watermark directly into the text itself. You won't see it, and it doesn't change the meaning, quality, or readability of Claude's response. Because the watermark is part of the text, it will travel with the text when it's copied and pasted elsewhere, and may persist through some editing.

They have not published the algorithm, the detector, or the thresholds. Coverage in [Decrypt](https://decrypt.co/375594/anthropic-quietly-watermarking-ai-claude-output-builders-break) and [Search Engine Journal](https://www.searchenginejournal.com/why-anthropics-claude-watermark-may-be-a-new-text-marking-method/585703/) treats it as a statistical sampling bias in the same family as Google SynthID-Text, not hidden Unicode. Roger Montti at SEJ lined Anthropic's six published properties up against academic schemes (MirrorMark, MCmark). That is property matching, not a leak.

In March 2026 Anthropic pulled a hidden Claude Code tracker that used undisclosed Unicode markers. That was a different mechanism.

## What the public removers do

Tools showed up within days of the announcement. They rewrite. They do not recover a key.

[mikiane/claude-watermark-cleaner](https://github.com/mikiane/claude-watermark-cleaner) (Michel Levy Provencal, 11 August 2026) strips invisible Unicode, then paraphrases with a non-Claude model (Ollama or Codex). The README is a critique: a mark you can weaken by rewriting is not proof of origin.

[guillaumemeyer/watermarks-remover](https://github.com/guillaumemeyer/watermarks-remover) (Guillaume Meyer, 11 August 2026) does Unicode hygiene, a rewrite hook, and C2PA / EXIF / XMP stripping across common file types. The class-level targets are Claude, SynthID, OpenAI provenance, and Kirchenbauer-style marks.

Both authors say you cannot measure success until Anthropic publishes a detector. This repo agrees. It also refuses the usual lie: a Claude-side cleaner that rolls the die again.

## The chain that can work

```
Claude draft  ->  this skill on OpenAI, Gemini, or a local model  ->  stop
```

This does not:

```
Claude  ->  other model  ->  Claude humanizer
```

A synonym pass that keeps the sentence frames will usually leave 50 to 80 percent of the source 5-grams intact. That fails on purpose. Forced keeps (numbers, SHAs, paths, names, quotes, fact-table cells) stay as they were. Those tokens still carry whatever mark they had. A table-heavy memo always has a remainder.

Do not send the recast back through Claude. Last sampler wins.

## The skill

[SKILL.md](SKILL.md) follows the [Agent Skills](https://agentskills.io/specification) format. Claude Code, Pi, Codex, and anything else that loads `SKILL.md` can pick it up.

Copy it to an agent skills directory. For Pi or Claude Code that is `~/.agents/skills/ngram-rewrite/SKILL.md`. Invoke it on a non-Anthropic model.

It will:

- refuse if the running model is Claude or any Anthropic model
- recast prose so fewer than 20 percent of the source's 5-word spans survive (under 10 percent is better)
- copy numbers, SHAs, paths, names, quotes, and table cells verbatim
- report 5-gram overlap as a proxy
- refuse to say the watermark is gone

## Optional: the open math (SciPy)

Skip this if you only wanted the argument. The block below is the published SynthID-Text / Kirchenbauer statistic. It is not Anthropic's detector. Hashing real Claude text with keys you invented is a coin flip.

A PRF turns `(context n-gram, candidate token, layer key)` into a bit `g` in `{0, 1}`. Unwatermarked text, or watermarked text hashed with the wrong key, has `E[g] = 1/2`. SynthID then runs a tournament on those bits. Under a uniform language model and two leaves, DeepMind's Corollary 27 gives:

```
μ1 = 1/2 + 1/4 (1 - 1/V)  ≈ 0.75
```

Three leaves: `μ1 = 7/8 - 3/(8V) ≈ 0.875`. `V` is vocab size.

Mask out end-of-sequence and repeated contexts. The mean score `S` is the average of the leftover bits. `G` is the count of ones. The one-sided test against a fair coin is:

```
z = 2 * sqrt(n) * (S - 1/2)
p_norm  = Φ(-z)
p_binom = P(K >= G | K ~ Bin(n, 1/2))
log BF10 = G log(μ1/μ0) + (n - G) log((1-μ1)/(1-μ0))
```

A huge `n` will reject the fair-coin hypothesis on a 0.3 percentage-point drift. Use the Bayes factor or a length-calibrated threshold (DeepMind Appendix A.3.1). A small `p` is not a watermark.

Bits needed for a 0.001 false-positive rate and 95 percent power, at three layers:

| How hard the die is loaded | Bits | Rough English |
|---|---:|---|
| Strong tournament (μ ≈ 0.75) | ~86 | a couple of sentences, and they stack bits per token |
| Mild green-list (μ = 0.55) | ~2,300 | maybe 500 to 800 words if one bit per token |
| Soft / quality-first (μ = 0.52) | ~15,000 | several thousand words |
| Barely loaded (μ = 0.51) | ~59,000 | a long essay |

Google's SynthID writeups talk about reliable detection on a few hundred tokens. Kirchenbauer's z-test sits in the same neighborhood for a typical bias.

```python
import math
from scipy import stats

def sampling_bias_test(bits, mu0=0.5, mu1=0.75, alpha=1e-3):
    bits = bits.astype(int).ravel()
    n, k = bits.size, int(bits.sum())
    s = k / n
    z = (k - mu0 * n) / math.sqrt(n * mu0 * (1 - mu0))
    p_norm  = float(stats.norm.sf(z))
    p_binom = float(stats.binom.sf(k - 1, n, mu0))
    log_bf  = k * math.log(mu1 / mu0) + (n - k) * math.log((1 - mu1) / (1 - mu0))
    k_star  = int(stats.binom.ppf(1.0 - alpha, n, mu0))
    power   = float(stats.binom.sf(k_star, n, mu1))
    return dict(n=n, green=k, S=s, z=z,
                p_norm=p_norm, p_binom=p_binom,
                log10_BF10=log_bf / math.log(10),
                power=power, reject=p_binom < alpha)

def tokens_needed(mu1, alpha=1e-3, beta=0.05, mu0=0.5):
    za, zb = stats.norm.ppf(1 - alpha), stats.norm.ppf(1 - beta)
    sig0, sig1 = math.sqrt(mu0*(1-mu0)), math.sqrt(mu1*(1-mu1))
    return math.ceil(((za*sig0 + zb*sig1) / (mu1 - mu0))**2)
```

`scipy` is optional. The skill does not import it. The g-bit itself is the high bit of DeepMind's newlib/musl LCG (`mult = 6364136223846793005`) folded over the n-gram and the key. The hash is not the interesting part.

`S - 1/2` is noise under the wrong key, and a large positive shift under the right one. That shift is the watermark. Wrong keys make `g` a fair coin, so `S` walks back to one half. That is why a public detector clone does not exist, and why this repo will not pretend to score Claude outputs.

## What would survive a heavy rewrite

Today's mark is the word-by-word choices. Change most of the words and the history is gone.

A surviving mark has to live in something a rewriter cannot throw away. Meaning, not synonyms. That means a stable hash of propositions, or a vendor-side search over embeddings of everything they generated, or a design that puts the signal in the load-bearing numbers so stripping the mark also strips the facts. None of those is what Anthropic described. Published "survives editing" numbers are about deletions and light paraphrase, not a second frontier model.

## Limits

This destroys surface n-grams. It does not invert a secret key.

Forced-keep tokens still carry whatever mark they had.

A later Claude pass restamps the mark.

C2PA and file metadata are a different layer. This repo is text.

Overlap is a proxy, not Anthropic's detector.

Recast text is still model-written.

## References

Anthropic, [How Claude marks AI-generated content](https://support.claude.com/en/articles/16266773-how-claude-marks-ai-generated-content)

Jose Antonio Lanz, [Anthropic Is Quietly Watermarking Every Claude AI Output. Builders Are Already Trying to Break It](https://decrypt.co/375594/anthropic-quietly-watermarking-ai-claude-output-builders-break), Decrypt, 13 August 2026

Roger Montti, [Why Anthropic's Claude Watermark May Be A New Text-Marking Method](https://www.searchenginejournal.com/why-anthropics-claude-watermark-may-be-a-new-text-marking-method/585703/), Search Engine Journal, 13 August 2026

[Claude will apply invisible watermarks to AI text and images](https://www.theverge.com/ai-artificial-intelligence/977823/anthropic-claude-ai-watermarks-c2pa-text-images), The Verge, 11 August 2026

[Anthropic to start embedding invisible watermarks](https://fortune.com/2026/08/11/anthropic-claude-watermark-ai-text-police-ai-slop/), Fortune, 11 August 2026

[Some Claude users are mad that Anthropic's new watermarks will catch them cheating](https://techcrunch.com/2026/08/12/some-claude-users-are-mad-that-anthropics-new-watermarks-will-catch-them-cheating-at-their-jobs-classes/), TechCrunch, 12 August 2026

John Gruber, [Anthropic Posts "How Claude Marks AI-Generated Content" Without Explaining How](https://daringfireball.net/linked/2026/08/11/anthropic-claude-watermarks), Daring Fireball, 11 August 2026

Dathathri et al., [Scalable watermarking for identifying large language model outputs](https://doi.org/10.1038/s41586-024-08025-4), Nature 634:818-823 (2024)

[google-deepmind/synthid-text](https://github.com/google-deepmind/synthid-text)

Google, [SynthID text documentation](https://ai.google.dev/responsible/docs/safeguards/synthid)

Hugging Face, [`transformers` watermarking module](https://github.com/huggingface/transformers/blob/main/src/transformers/generation/watermarking.py)

Kirchenbauer et al., [A Watermark for Large Language Models](https://arxiv.org/abs/2306.04634) (2023)

MirrorMark (George Mason / InvisibleID, 2026), StealthInk (2025), and MCmark (2025) are the academic schemes Montti compared to Anthropic's six published properties.

EU AI Act Code of Practice on transparency is Anthropic's stated reason for shipping the mark. C2PA is file-level provenance, not a text sampling bias.

## License

MIT. The skill frontmatter says the same.

The name of this repository is the claim. When Anthropic publishes a detector, run it. Until then, do not say the watermark is gone.
