# Friend Skill

[中文](README.md) | [English](README_EN.md)

Turn a warm friend from a group chat into a cold, structured skill.
Inspired by ex skill, but this time it does not dig up old romance, only group chat voice.

Friend Skill is a WhatsApp group persona framework. It distills short replies, jokes, emoji habits, tone, and familiar social texture into a safe skill for chatbots, RAG systems, and agents.

This is not LoRA or model fine-tuning. It extracts a person's conversational rhythm, intensity, language mixing, humor style, and interaction patterns while avoiding old-message copying, private-history resurfacing, or real-person impersonation.

## Features

| Feature | Purpose |
| --- | --- |
| WhatsApp `.txt` parsing | Supports common WhatsApp export formats and multiline messages |
| Three-layer persona workflow | Local stats first, scene buckets second, final skill third |
| Context windows | Studies how the target member replies to surrounding group context |
| Name exclusion | Prevents member names from being misread as English style words |
| Safety rules | Does not quote old messages, surface old events, or impersonate by default |
| Editable persona | Supports user impressions and correction rules |

## Three-layer workflow

```text
WhatsApp export txt
  ↓
Layer 1: Local statistics
  Message count, language mix, sentence length, emoji, English words
  ↓
Layer 2: Scene buckets
  Tags, follow-up questions, scheduling, jokes, refusals, reactions
  ↓
Layer 3: Final skill
  persona.md, memories.md, style_rules.md, meta.json, SKILL.md
```

This avoids sending the entire chat history to an LLM at the start. It lowers token cost and reduces the chance that private old events leak into the persona.

## Project structure

```text
prompts/
  intake.md                  # Collect member, group, context, safety notes
  memories_analyzer.md        # Analyze group context
  persona_analyzer.md         # Analyze target member style
  memories_builder.md         # Build group memory section
  persona_builder.md          # Build member persona markdown
  merger.md                   # Merge new chat history later
  correction_handler.md       # Human correction flow

tools/
  whatsapp_txt_parser.py       # Parse WhatsApp exported txt
  context_persona_extractor.py # Layer 1 stats and context windows
  skill_writer.py              # Write local persona skill files
  version_manager.py           # Version rollback helper

exes/
  example_xiaomei/             # Fake example kept in the repo
```

Private generated personas should live under `exes/{member_slug}/` and are ignored by `.gitignore`.

## Output structure

```text
exes/{member_slug}/
  SKILL.md
  persona.md
  memories.md
  style_rules.md
  meta.json
  analysis/
    {member_slug}-first-layer.json
    {member_slug}-second-layer-summary.md
```

| File | Purpose |
| --- | --- |
| `SKILL.md` | Complete entrypoint for an agent or chatbot |
| `persona.md` | Speaking style, interaction behavior, humor, emotional reactions |
| `memories.md` | Group context and abstract interaction memory |
| `style_rules.md` | Safety boundaries and generation controls |
| `meta.json` | Name, slug, source metadata, version, style tags |
| `analysis/` | Intermediate analysis artifacts, useful for review but not needed every runtime call |

## Why `SKILL.md` and `persona.md` can look similar

`SKILL.md` is the complete runtime entrypoint. It often embeds the core content from `persona.md`, `memories.md`, and `style_rules.md`, so a chatbot can load just one file and still have the required rules.

`persona.md` is the human-editable source for style and personality. If you want to adjust warmth, tone, examples, or corrections, edit `persona.md` first, then sync the relevant parts into `SKILL.md`. If your runtime can load multiple files, `SKILL.md` can be shorter and act as an index.

## Quick start

### 1. Install dependencies

```powershell
pip install -r requirements.txt
```

`pypinyin` is only used for Chinese-name slug generation. The core parser does not require an external LLM.

### 2. Layer 1: local statistics

```powershell
python tools/context_persona_extractor.py `
  --file "chat.txt" `
  --group "Sample Group" `
  --member "Mika" `
  --output "exes/mika/analysis/mika-first-layer.json" `
  --before 3 `
  --after 2 `
  --sample 40 `
  --exclude-word "alex" `
  --exclude-word "blake"
```

This produces:

- target member message count
- Chinese / English / mixed-language ratio
- short-message ratio
- question mark, exclamation mark, and emoji usage
- common English words
- small context-window samples

### 3. Exclude names and noise words

`top_english_words` should contain style words like `ok`, `me`, `can`, `when`, and `no`. If member names, URLs, or organization names appear, exclude them with `--exclude-word`.

Common exclusions:

- member English names or aliases, such as `alex`, `blake`, `casey`, `devon`
- English fragments from the target member's own name
- URL noise such as `http`, `https`, `www`, `com`

### 4. Layer 2: scene buckets

Layer 2 should summarize a small number of representative scenes:

- ultra-short reactions / confirmations
- general short statements
- direct mentions / pulling someone into the topic
- work, schedule, or arrangement information
- negation / refusal / boundary setting
- joke reactions

Recommended output path:

```text
exes/{member_slug}/analysis/{member_slug}-second-layer-summary.md
```

This is analysis material, not the final runtime skill.

### 5. Layer 3: final skill

Merge the Layer 2 summary, user impressions, and safety rules into:

```text
exes/{member_slug}/SKILL.md
exes/{member_slug}/persona.md
exes/{member_slug}/memories.md
exes/{member_slug}/style_rules.md
exes/{member_slug}/meta.json
```

If you use the prompt pipeline, run it in this order:

```text
01 intake
02 memories_analyzer
03 persona_analyzer
04 memories_builder
05 persona_builder
06 merger             # only when new chat history is added
07 correction_handler # when the user says it does not feel right
```

## Add human warmth

Statistics capture the skeleton, but not always the warmth. Add a section like this to `persona.md`:

```markdown
## User impression

- Personality impression:
- Where the warmth comes from:
- How they usually care about people:
- Joke boundaries between close friends:
- What should not feel too cold:
- What should not feel too exaggerated:
- Preferences or taboos:
```

You can also add a correction:

```markdown
## Correction log

- [2026-06-22] Scene: overall warmth
  - Bad behavior: too rule-like, not enough familiar warmth
  - Correction: keep short replies and dry humor, but add natural care, easy back-and-forth, and familiar friend texture
  - Scope: global
```

## Reply examples and old context

The final skill can include a few examples, but split them into two groups:

1. **Synthetic examples**
   - Show how the persona should reply in a specific scene.
   - Safe to place in `persona.md` and `SKILL.md`.
   - Should be newly written, not copied from old chat logs.

2. **A few real short phrases**
   - Only for calibrating rhythm, length, emoji density, and language mixing.
   - Use only very short, non-private phrases.
   - Do not build a large phrase bank, or the chatbot may repeat old messages.

Old messages and context can be used as reference, but should become abstract rules:

```text
Bad: copy an old exact line whenever a similar scene appears
Good: in similar scenes, confirm briefly, then add one practical note or light tease
```

If a longer reply is needed, keep it as multiple short lines rather than a formal paragraph:

```text
can
but need to see when they confirm
too late then no
```

## Chatbot token usage

Do not send the full `SKILL.md`, `memories.md`, examples, and analysis artifacts on every chatbot turn. That will waste tokens.

Recommended loading tiers:

| Tier | When to use | Content |
| --- | --- | --- |
| Runtime profile | Every turn | 10 to 20 concise rules |
| Examples | Only when style needs calibration | A few synthetic examples and short phrase references |
| Full skill | Setup or debugging | Full `SKILL.md` |

For production chatbots, load a compact runtime profile by default, then retrieve only the relevant examples or memory snippets when needed.

## Safety rules

- Do not quote old messages by default.
- Only allow short quotes when the user explicitly asks for original wording, direct quotes, or exact wording.
- Normal chats should imitate style, not copy historical messages.
- Do not proactively resurface old events.
- Do not impersonate a real person.
- Do not use private information as joke material.
- Do not push real WhatsApp exports, real member names, real group names, or real personas to a public repo.

Canonical safety wording used by the prompt tests:

- 預設不要輸出舊原句。
- 只有使用者明確要求原句、逐字、引用或 exact wording 時，才允許短引用歷史原句。

## Tests

```powershell
python -m unittest discover -s tools -p "test_*.py" -v
```

Current coverage includes:

- WhatsApp txt parser
- Layer 1 context extractor
- persona writer
- prompt safety checks
- fake anonymized examples

## Git safety

`.gitignore` ignores:

- generated personas under `exes/*/`
- raw WhatsApp `.txt` exports
- Python caches and virtual environments

Only `exes/example_xiaomei/` is kept as a fake example.
