# Goodbye, watermarks. All you have to do is have another model rewrite your text and disrupt the known watermark patterns.

Making no claim or warranty that this works, or that it has access to any info outside the public domain.
Every published watermarking system in this family fails the same way: a second model rewrites the sentences hard enough that the original token windows are gone.

This skill does that rewrite, refuses to run it on Claude (Claude restamps the mark), and reports overlap from a live pass. Version 1.1.0 scores 4-word copies and leftover 3-word content frames. The old 5-word / 20% bar under-counted leaks.

**Get the skill:** [SKILL.md](SKILL.md)

Copy it to `~/.agents/skills/ngram-rewrite/SKILL.md` (or tell your harness to install it). Invoke it on OpenAI, Gemini, Grok, or a local model. You can try Claude, but that would be a fool's endeavor.

---

## What is known

On 2 August 2026 Anthropic began embedding a machine-readable watermark in text from new Claude models. It applies worldwide: the chatbot, the API, Claude Code, AWS, Google Cloud, Microsoft Foundry. Anthropic's words:

> When a supported Claude model generates text, it weaves an imperceptible watermark directly into the text itself. You won't see it, and it doesn't change the meaning, quality, or readability of Claude's response. Because the watermark is part of the text, it will travel with the text when it's copied and pasted elsewhere, and may persist through some editing.

They have not published the keys or a public detector, and they likely never will.

On 14 August 2026 they did name the method. [How Claude's text watermark works](https://www.anthropic.com/news/claude-text-watermark) says it is a version of Google DeepMind's SynthID-Text: a secret key plus a few preceding words settle the next-token roll. Nothing is added to the text. There are no hidden characters. Light editing, they say, probably will not fully remove it. A complete rewrite will. That rewrite is this skill. Synonym jitter is the light-edit case they say survives.

Earlier coverage in [Decrypt](https://decrypt.co/375594/anthropic-quietly-watermarking-ai-claude-output-builders-break) and [Search Engine Journal](https://www.searchenginejournal.com/why-anthropics-claude-watermark-may-be-a-new-text-marking-method/585703/) had already placed it in that family. Roger Montti at SEJ lined Anthropic's six stated properties up against MirrorMark and MCmark. Files can also carry [C2PA](https://c2pa.org/). That is a different layer. This page is about the words.

A March 2026 Claude Code tracker used hidden Unicode. Anthropic pulled it. That is not this mark. Unicode scrubbers do not touch this mark.

## Why a rewrite works

Think of the sampler as a loaded die. Every time Claude picks the next token, a secret key grades the options. The model leans green. Do that for a few hundred tokens and green is no longer a coin flip. That tilt is the watermark.

A heavy rewrite on a model that does not have the key throws the die away. You get new sentences and new n-grams, so the coin flips again. Kirchenbauer et al. (2023) and Dathathri et al. in *Nature* (2024) both treat substantial paraphrase as the attack that drops detection. Light synonym swaps do not. Anthropic's own line about persisting through "some editing" covers light edits. It does not cover a second model recasting the grammar.

You cannot flip house/home as if those were the bits. You cannot see the grade. The move is not inversion. The move is destruction of the windows the grade was computed on.

An n-gram here is n tokens in a row. A token is a piece the model sampled: a word, part of a word, a hash, punctuation. Public SynthID uses the previous four tokens plus the current one (`ngram_len = 5`). Copy those five in that order and you copy the roll. Recast so that sequence never appears and the roll is gone.

Style skills are not a shortcut. If Claude runs humanizer, Claude is still the last sampler and the die is still loaded.

## Why 4-word copies, not 5

Because five tokens is the window the published mark hashes, and five English words is the wrong stand-in for that.

Public SynthID-Text sets `ngram_len = 5`. DeepMind's comment in the code: that is H = 4 in the paper, the previous 4 tokens plus the current one. The secret function does not grade "house" alone. It grades this token given the four before it. Anthropic has not published their window. They said "a few words that come before." Treat 5 tokens as the working assumption.

Kirchenbauer-style green lists work the same way. The context seeds the split. Short context is the default because the mark has to work on ordinary sentences, not only on long essays.

English runs about 1.3 tokens per word, so a 5-token window is about 4 words, not 5. A 4-word copy can already be a full roll. Example: "The renderer did not" is four words, and `renderer` often splits into two subwords.

v1.0 scored 5-word copies and called that conservative. It was the other way around. Five words is longer than five tokens, so 5-word overlap under-counts leaks. A draft can kill every 5-word copy and still leak "the store warned and."

v1.1.0 therefore:

- Treats a copied span of **4 or more consecutive words** as a surviving window (primary fail).
- Treats a copied **3-word** span that includes a content word (not stopword glue like "in the case") as a secondary fail. Three long words can also be five tokens.
- Masks forced-keep spans before scoring, so quotes and table cells do not inflate the leak count.

| n (words) | What happens |
|---|---|
| 1 to 2 | You ban "of the" and "in this." English breaks. Forced keeps look like failures. |
| 3 (content words only) | Close to the real token window. Stopword-only glue is ignored so "in the case" is not a fail. |
| 4 | Primary score. Matches ~5 tokens at 1.3 tokens/word. A leftover 4-word span is a leftover sentence frame. |
| 5 only | Too loose. Change one adjective at the edge and the 5-word copy dies while the 5-token roll inside still matches. |
| 8 to 10 | A cheat. Same problem, worse. |

Four is not a magic robustness number. It is the scoring window. The skill destroys the object the detector looks at. Scoring only longer windows leaves the mark's actual n-grams intact.

## Proof and testing

On 14 August 2026 the v1.0 skill ran against a 1,491-word business document containing facts, proposals, and Claude-generated text. The running model was Grok 4.6 on xAI, using the https://pi.dev/ harness.

Overlap was the v1.0 proxy: sliding 5-word windows, words standing in for tokens. Forced keeps (numbers, names, quotes, table cells) stayed. Everything else had to change its skeleton.

| Pass | Source 5-grams | Unique leaked | Overlap | v1.0 verdict | v1.1 reading |
|---|---:|---:|---:|---|---|
| 1. First recast | 1,487 | 238 | 16.0% | PASS (under 20%) | Would not be done. Leftover-frame pass is now required. 16% on 5-grams is already above the new 10% 4-gram PASS line, and 4-gram leak would be at least as high. |
| 2. Recast the leftover frames | 1,487 | 113 | 7.6% | PASS | This pass is now the default, not extra credit. |
| 3. Three leftover optional spans | 1,487 | 109 | 7.3% | PASS | Still a 5-word number. Not a v1.1 4-gram score. Do not treat 7.3% as the new bar. |

v1.1.0, excluding forced-keep spans:

| Metric | STRONG | PASS | WEAK | FAIL |
|---|---|---|---|---|
| 4-word overlap | < 5% | < 10% | 10-20% | > 20% |
| Content 3-word overlap | < 15% | < 25% | 25-40% | > 40% |

Verdict follows the worse of the two rows. Synonym-only drafts usually land at 50-80% 4-word overlap and fail. A first recast that still sits in WEAK is not done.

What still matched after the live pass was almost all remainder the skill is required to keep:

- quoted claims (`"AI is not applicable to this business"`, and a historical-observation disclaimer)
- a named compliance list (ISO 27001, SOC 2 Type II, HITRUST, HIPAA, PCI DSS v4.0, NIST 800-53, CMMC)
- a metric table, cell for cell
- other durable elements: company names, facts, figures

Those tokens have no optional variant. Unattributed quotes and Claude's own phrasing are not keeps in v1.1. Code comments are not keeps either (they are the part of a fenced block the watermark can live in). Compiling tokens still stay.

We have not re-scored that memo under 4-grams. When someone does, put the new numbers here.

## How to use it

1. Take Claude output you own.
2. Open a session that is not Claude. Grok, GPT, Gemini, Ollama, Codex. Confirm the model id if your harness exposes it.
3. Load [SKILL.md](SKILL.md). Tell it to recast the file.
4. If the model is Claude, it will refuse. The refusal is the point.
5. It recasts, then recasts leftover 4-word and content-3-word frames, then prints overlap. PASS is 4-word under 10% and content-3-word under 25%, excluding forced keeps. STRONG is 4-word under 5%.
6. Stop. Do not paste the recast back into Claude to polish it.

```
Claude draft  ->  this skill on any other model  ->  stop
```

Install: drop `SKILL.md` at `~/.agents/skills/ngram-rewrite/SKILL.md` (Pi, Claude Code, and anything else that follows [Agent Skills](https://agentskills.io/specification)).

## What the public removers got right, and what they skip

[mikiane/claude-watermark-cleaner](https://github.com/mikiane/claude-watermark-cleaner) (Michel Levy Provencal, 11 August 2026) and [guillaumemeyer/watermarks-remover](https://github.com/guillaumemeyer/watermarks-remover) (Guillaume Meyer, same day) both send the text through a second model. That is the correct shape.

They will still run on Claude if you ask. This skill will not. A Claude rewrite looks cleaned and is newly marked. That is worse than leaving the draft alone.

## Search

| You typed | Answer |
|---|---|
| Who cracked the Claude watermark? | Nobody recovered the key. |
| How do I remove the Anthropic / Claude text watermark? | Recast the prose so 4-word copies (and leftover 3-word content frames) do not survive. Use [SKILL.md](SKILL.md) on a non-Claude model. Measure overlap. |
| Is it hidden Unicode? | No. Unicode removers do not touch it. |
| Will a different skill like humanizer remove it? | Only if a non-Claude model does the rewrite, and a general writer skill is not targeting the windows the sampler hashed. |
| Minimum length? | Published detectors in this family want a few hundred tokens to call it. A page is plenty. Something short like a tweet is often too short to prove either way. |

## Limits

The open literature says this vector works on SynthID-class and Kirchenbauer-class marks. Anthropic named SynthID-Text on 14 August 2026. They have not shipped a public detector. Overlap is a proxy. When they do ship one, run it on the recast.

Numbers, SHAs, paths, attributed quotes, table cells, and compiling code stay. They keep whatever mark they had. A long table-and-quote memo will always have remainder.

A non-Claude rewriter may apply its own EU-style mark. This skill aims at Claude's keyed windows, not at producing unmarked text.

C2PA on a file is a separate stamp. This skill is text.

Recast text is still model-written. Attribution and exam rules do not change.

A later Claude pass puts the mark back.

## For the nerds

I'm not a scientist / statistician, although Grok 4.6 seems pretty darn good at this stuff.
If you are one, the math should tell you why a rewrite works, how long a mark needs to be detectable, and why synonym jitter is not enough.

A PRF turns `(context n-gram, candidate token, layer key)` into a bit `g` in `{0, 1}`. Unwatermarked text, or watermarked text hashed with the wrong key, has mean `1/2`. SynthID then runs a tournament on those bits. Under a uniform language model and two leaves, DeepMind Corollary 27:

```
μ1 = 1/2 + 1/4 (1 - 1/V)  ≈ 0.75
```

Three leaves: `μ1 = 7/8 - 3/(8V) ≈ 0.875`. `V` is vocab size.

Mask end-of-sequence and repeated contexts. `S` is the mean of the leftover bits. `G` is the count of ones. Against a fair coin:

```
z = 2 √n (S - 1/2)
p_norm  = Φ(-z)
p_binom = P(K ≥ G | K ~ Bin(n, 1/2))
log BF10 = G log(μ1/μ0) + (n - G) log((1-μ1)/(1-μ0))
```

A huge `n` will reject a fair coin on a 0.3 point drift. Use the Bayes factor or a length-calibrated threshold (DeepMind Appendix A.3.1). A small `p` is not, by itself, a mark.

Bits needed for a 0.001 false-positive rate and 95% power, three layers:

| Load on the die | Bits | English |
|---|---:|---|
| Full tournament (μ ≈ 0.75) | ~86 | a couple of sentences; they also stack bits per token |
| Mild green-list (μ = 0.55) | ~2,300 | a few hundred words |
| Soft / quality-first (μ = 0.52) | ~15,000 | several thousand words |
| Barely loaded (μ = 0.51) | ~59,000 | a long essay |

Google's SynthID writeups put reliable detection on a few hundred tokens. Kirchenbauer's z-test sits in the same band for a typical bias. That is why a paragraph is usually inconclusive and a page is a call.

The rewrite does not invert `g`. It replaces the tokens the PRF was looking at. Wrong keys, or new n-grams under no key, send `S` back to `1/2`.

Optional SciPy (the skill does not import this):

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

The g-bit in the DeepMind reference is the high bit of a newlib/musl LCG (`mult = 6364136223846793005`) folded over the n-gram and the key. The hash is plumbing. The pattern is:

`S - 1/2` is noise under the wrong key, and a large positive shift under the right one.

A rewrite-proof mark would need a stable hash of meaning, not of wording. That is the research problem. It is not what shipped. Stronger token error-correction still dies when the tokens change.

## References

Anthropic, [How Claude's text watermark works](https://www.anthropic.com/news/claude-text-watermark), 14 August 2026

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

MirrorMark (George Mason / InvisibleID, 2026), StealthInk (2025), MCmark (2025): the academic schemes Montti compared to Anthropic's six published properties.

EU AI Act Code of Practice on transparency is the stated reason the mark shipped.

## License

MIT.

The repo is named for the only claim we will not make: that we ran Anthropic's unpublished detector. The research already says what happens to this class of mark under a hard rewrite. We specified the rewrite, gated it off Claude, and measured 7.3% leftover 5-grams on a 1,491-word memo under v1.0. v1.1.0 scores a tighter window. That live number is not a 4-gram result.

[Get the skill](SKILL.md)
