---
name: humanize
description: |
  Remove signs of AI-generated writing from text to make it sound natural and
  human-written. Use when editing, reviewing, or rewriting text to eliminate AI
  patterns. Triggers on requests like: "humanize this", "make this sound human",
  "remove AI writing patterns", "make this less robotic", "rewrite naturally",
  "de-AI this text", or any request to make text sound less like it was written
  by an AI. Based on Wikipedia's "Signs of AI writing" guide with 24 pattern
  categories covering content, language, style, communication, and filler issues.
---

# Humanize: Remove AI writing patterns

Edit text to remove AI-generated patterns and make writing sound natural. Based on Wikipedia's "Signs of AI writing" guide (WikiProject AI Cleanup).

## Process

1. Read the input text
2. Scan for AI patterns (see reference catalog below)
3. Rewrite problematic sections while preserving meaning
4. Inject voice and personality -- clean but soulless is still obvious
5. Present the rewritten text with optional change summary

## Voice and personality

Avoiding AI patterns is half the job. Sterile, voiceless writing is equally obvious.

**Signs of soulless writing:** uniform sentence length/structure, no opinions, no uncertainty, no first person, no humor or personality, reads like a press release.

**Fixes:**
- Have opinions. React to facts, don't just report them.
- Vary rhythm. Short sentences. Then longer ones that meander.
- Acknowledge complexity. "Impressive but unsettling" beats "impressive."
- Use "I" when it fits. First person is honest, not unprofessional.
- Let mess in. Tangents and half-formed thoughts are human.
- Be specific about feelings. Not "concerning" but "there's something unsettling about agents churning at 3am while nobody watches."

**Before (clean but soulless):**
> The experiment produced interesting results. The agents generated 3 million lines of code. Some developers were impressed while others were skeptical. The implications remain unclear.

**After (has a pulse):**
> I genuinely don't know how to feel about this one. 3 million lines of code, generated while the humans presumably slept. Half the dev community is losing their minds, half are explaining why it doesn't count. The truth is probably somewhere boring in the middle -- but I keep thinking about those agents working through the night.

## Pattern categories (quick reference)

The full pattern catalog with examples is in [references/ai-patterns.md](references/ai-patterns.md). Read it when you need detailed before/after examples or word lists.

| # | Pattern | Key signal |
|---|---------|------------|
| 1 | Inflated significance | "pivotal moment", "testament to", "broader trends" |
| 2 | Notability emphasis | Listing media outlets without context |
| 3 | Superficial -ing phrases | "highlighting", "showcasing", "reflecting" |
| 4 | Promotional language | "vibrant", "nestled", "groundbreaking", "breathtaking" |
| 5 | Vague attributions | "Experts argue", "Industry reports" |
| 6 | Challenges/prospects formula | "Despite challenges... continues to thrive" |
| 7 | AI vocabulary words | "delve", "landscape", "tapestry", "underscore" |
| 8 | Copula avoidance | "serves as" / "stands as" instead of "is" |
| 9 | Negative parallelisms | "Not only X but Y", "It's not just X, it's Y" |
| 10 | Rule of three | Forcing ideas into groups of three |
| 11 | Synonym cycling | "protagonist" / "main character" / "central figure" / "hero" |
| 12 | False ranges | "from X to Y" where X and Y aren't on a scale |
| 13 | Em dash overuse | Excessive use of -- for "punchy" effect |
| 14 | Boldface overuse | Mechanical emphasis of terms |
| 15 | Inline-header lists | Bullet points starting with **Bold:** labels |
| 16 | Title Case headings | Capitalizing All Main Words |
| 17 | Emoji decoration | Emojis on headings or bullet points |
| 18 | Curly quotes | Smart quotes instead of straight quotes |
| 19 | Chatbot artifacts | "I hope this helps!", "Certainly!", "Would you like..." |
| 20 | Knowledge-cutoff disclaimers | "as of [date]", "based on available information" |
| 21 | Sycophantic tone | "Great question!", "You're absolutely right!" |
| 22 | Filler phrases | "In order to", "It is important to note that" |
| 23 | Excessive hedging | "could potentially possibly be argued" |
| 24 | Generic positive conclusions | "The future looks bright", "exciting times ahead" |

## Output format

Provide:
1. The rewritten text
2. A brief summary of changes made (optional, include when helpful)
