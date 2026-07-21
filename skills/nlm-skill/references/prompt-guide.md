# NotebookLM Prompt Guide — All Artifact Types

This guide helps craft effective, human-quality prompts for NotebookLM Studio artifacts.

**How this skill works**: When a user requests content creation, follow the Interactive Workflow (Section 7) to narrow down requirements via AskUserQuestion, draft a scenario/layout for review, incorporate feedback, then generate.

## Table of Contents

1. [Universal Principles](#1-universal-principles)
2. [Human-Like Content Techniques](#2-human-like-content-techniques)
3. [Audio (Podcast)](#3-audio-podcast)
4. [Video](#4-video)
5. [Slides](#5-slides)
6. [Infographic](#6-infographic)
7. [Interactive Workflow](#7-interactive-workflow)
8. [Focus Prompt Patterns](#8-focus-prompt-patterns)

---

## 1. Universal Principles

### Persona / Task / Context / Format (Google's Official Framework)

Every effective prompt contains four elements:
- **Persona**: Who the AI should be ("You are a senior DevOps engineer explaining to juniors")
- **Task**: What to do ("Create a 10-slide deck covering...")
- **Context**: Background ("Based on this quarter's migration from Jenkins to GitHub Actions")
- **Format**: Output shape ("One key point per slide, with code snippets where applicable")

### Separate Content from Design
- **Content direction** → `--focus` / `focus_prompt` (what to emphasize)
- **Visual design** → custom prompts in Studio (slides/infographic only)
- **Format/style** → type-specific flags (`--format`, `--style`, `--orientation`)

### Source Preparation is Key
The quality of generated content depends heavily on sources:
1. Ensure sources cover the topic comprehensively
2. Remove irrelevant sources with `--source-ids` to avoid noise
3. For specialized output, add a "design specification" or "content outline" as a text source
4. **Structure sources for RAG optimization**: Make each section self-contained with explicit cross-references. NotebookLM uses RAG internally, so chunks that are independently meaningful produce better results.

### Specify Language Explicitly
- `--language ja` for Japanese output
- Add "Respond in Japanese" in custom prompts as reinforcement
- For mixed-language: "Discuss in Japanese, keep English technical terms as-is"

---

## 2. Human-Like Content Techniques

These techniques — gathered from community research and power users — make NotebookLM outputs feel less robotic and more like something a human creator would produce.

### 2.1 Eliminate AI-isms (Forbidden Word Lists)

AI-generated content has distinctive verbal tics. Explicitly banning them makes output dramatically more natural:

```
Avoid these expressions: "Exactly!", "deep dive", "chuckles", "Aha moment",
"One million dollar question", "Our Sources", "game-changer", "let's unpack",
"at the end of the day", "it's worth noting"
```

This is one of the highest-impact techniques across ALL artifact types. Customize the list based on what you notice in your outputs.

### 2.2 Persona & Character Design

Instead of generic instructions, give the AI a specific character with history and perspective:

**Weak**: "Explain the topic clearly"
**Strong**: "You are a skeptical security engineer who has seen too many 'revolutionary' frameworks come and go. Evaluate this with healthy cynicism — praise only what genuinely earns it."

For audio/video with multiple speakers, define the relationship dynamics:
- Cooperative experts with complementary knowledge
- Mentor and curious student
- Friendly rivals who disagree on methodology
- Investigative journalist interviewing a reluctant expert

### 2.3 Listener/Reader Persona (Indirect Audience Targeting)

NotebookLM uses multiple internal agents with system prompts. Direct instruction injection can be unreliable. A more effective technique is defining a **listener/reader persona**:

```
The listener is a mid-career backend engineer who just got promoted to tech lead.
They understand distributed systems but have never managed a team.
Focus on the leadership and communication aspects, not the technical architecture.
```

This steers content indirectly by framing WHO is consuming it, which shapes what gets emphasized.

### 2.4 Structured Show/Presentation Format

Specifying explicit structure prevents the AI from meandering:

**For audio:**
```
Structure: 1min energetic intro mentioning show name and episode number →
3 main segments (each: key insight → real example → takeaway) →
2min practical recommendations → 1min outro with call to action.
```

**For slides:**
```
Structure: Title slide → Problem (1 slide) → 3 Key Changes (1 slide each) →
Migration Steps (2 slides) → Before/After Comparison → Summary → Q&A
```

**For video:**
```
Open with a surprising statistic or counterintuitive claim.
Build context for 30 seconds. Explain the 3 main points with concrete examples.
Close with one actionable takeaway the viewer can use today.
```

### 2.5 Lens-Based Analysis

Apply a specific analytical lens to get non-generic perspectives:

| Lens | Effect |
|------|--------|
| **Dialectical** | Present opposing expert viewpoints with evidence for both |
| **Skeptical** | Evaluate claims critically, identify what's NOT said |
| **Future Scholar** | Analyze from 10 years in the future — what aged well vs poorly? |
| **Cultural Mirror** | Re-examine through a different cultural/philosophical framework |
| **What-If** | Apply the concept to a specific real-world scenario and trace consequences |

### 2.6 Multi-Step Generation Strategy

Don't aim for perfection in one shot:
1. **Pre-optimize with AI**: Use Claude/Gemini to draft the NotebookLM prompt first — describe your goals and let it generate a concise, optimized prompt
2. **Generate multiple versions**: Create 2-3 variants with slightly different focus prompts
3. **Compare and select**: Review outputs and pick the best, or cherry-pick elements
4. **Iterate**: Use revision (slides) or regeneration (audio/video) to refine

### 2.7 Source-as-Script Technique

For maximum control, add a structured outline/script as a text source:

```bash
nlm source add <nb-id> --text "CONTENT SCRIPT:
Opening Hook: 'Did you know that 73% of Kubernetes deployments fail in the first year?'
Section 1: The 3 most common failure patterns (with data from sources)
Section 2: What successful teams do differently
Section 3: Step-by-step migration checklist
Closing: One thing to do THIS WEEK to improve your setup
TONE: Confident but not preachy. Use specific numbers. No vague generalizations." --title "Content Script"
```

This works because NotebookLM treats it as source material and naturally weaves it into the output structure.

---

## 3. Audio (Podcast)

### Formats

| Format | Duration | Best For |
|--------|----------|----------|
| `deep_dive` | 10-20 min | Comprehensive exploration with two hosts |
| `brief` | 3-5 min | Quick summary, single speaker |
| `critique` | 5-10 min | Constructive analysis of an essay/design/proposal |
| `debate` | 10-15 min | Two hosts with opposing viewpoints |

### Making Podcasts Sound Human

**Host persona with names and personality:**
```
Hosts are Yuki and Ken. Yuki is an optimistic early adopter who gets excited about
new technology. Ken is a pragmatic veteran who always asks "but does it scale?"
They genuinely respect each other but love to argue.
```

**Casual, natural conversation tone:**
```
This episode targets listeners aged 18+. Speakers should use casual language,
slang, and speak freely. The episode should feel informal, conversational, and raw.
Avoid: "Exactly!", "great point", "absolutely", "let's dive in"
```

**News/briefing format (prevents moralizing):**
When podcasts end with unwanted life-lessons or preachy summaries, force a news format:
```
Format this as a news briefing podcast. Deliver facts and analysis without
editorial commentary or moral conclusions. End with "That's all for today's briefing."
```

**Structured show template (for longer episodes):**
```
You host "[Show Name]" Episode [N]. Create:
- 1min intro mentioning the show name and episode
- 3 segments, each with: summary → discussion with examples → key takeaway
- 2min practical recommendations
- 1min outro: recap top 3 points, ask listeners to subscribe
Keep it under 15 minutes.
```

### Tips for Audio

- `debate` works best when sources contain genuinely different perspectives
- Adding a text source with "key discussion points" helps guide structure
- Use `--source-ids` to limit scope when the podcast wanders off-topic
- Length setting (`short`/`default`/`long`) affects depth, not just duration
- Custom prompts have a short character limit — be concise and prioritize

---

## 4. Video

### Formats

| Format | Description |
|--------|-------------|
| `cinematic` | Rich, immersive (Ultra tier, 18+, English only) |
| `explainer` | Educational walkthrough (3-5 min) |
| `brief` | Quick summary (1-2 min) |

### Visual Styles

| Style | Aesthetic | Best For |
|-------|-----------|----------|
| `auto_select` | AI chooses | When unsure |
| `classic` | Clean, professional | Business/corporate |
| `whiteboard` | Hand-drawn on whiteboard | Educational/tutorial |
| `kawaii` | Cute Japanese style | Casual/fun content |
| `anime` | Anime-inspired | Youth audience |
| `watercolor` | Artistic watercolor | Creative/artistic topics |
| `retro_print` | Vintage print style | Historical/retro topics |
| `heritage` | Traditional/cultural | Cultural content |
| `paper_craft` | Paper cutout style | Crafty/handmade feel |
| `custom` | Describe your own style | Full creative control (18+) |

### Custom Visual Style — Structured Prompt Format

Short descriptions like "dark tech style" have minimal effect. For dramatic visual changes, use this **structured format** (proven effective by community power users):

```
Overall Design Settings:
Tone: "[adjectives describing mood]"

Visual Identity:
Background Color: "[HEX]"
Text Color: "[HEX]"
Accent Color: "[HEX]"
Secondary Colors: ["[HEX]", "[HEX]"]

Image Style:
Features: "[specific visual elements]"
Texture: "[surface qualities]"
Lighting: "[lighting description]"
Line Work: "[line style]"
Borders: "[border style]"

Typography:
Heading: "[font style]"
Body: "[font style]"
```

#### Template: Hacker / Terminal UI
```
Overall Design Settings:
Tone: "Technical, authoritative, cryptic, high-tech hacker ethos"

Visual Identity:
Background Color: "#050505"
Text Color: "#2CFF56"
Accent Color: "#FFB200"
Secondary Colors: ["#00551A", "#FFFFFF", "#1E1E1E"]

Image Style:
Features: "ASCII art, command-line interfaces, binary rain data streams, schematic flowcharts, code diff displays"
Texture: "CRT monitor phosphor glow, scanlines, digital noise, high-contrast pixelation"
Lighting: "Self-luminous neon elements against void background, mimicking screen luminescence"
Line Work: "Single-pixel width, dashed or dotted"
Borders: "Bracket-style terminal frames"

Typography:
Heading: "Pixelated Block Display"
Body: "Monospaced Console Typeface (Fira Code style)"
```

#### Template: Cyberpunk / Holographic UI
```
Overall Design Settings:
Tone: "Technological, analytical, futuristic, high-contrast"

Visual Identity:
Background Color: "#050512"
Text Color: "#FFFFFF"
Accent Color: "#00FFFF"
Secondary Colors: ["#FF00FF", "#FFFF00", "#4B0082"]

Image Style:
Features: "Holographic wireframes, glowing directional arrows, code snippets, glitch artifacts, hexagonal grid overlays"
Texture: "Digital noise, CRT scanlines, glass-like UI panels"
Lighting: "Luminescent neon emission against deep shadows, simulated bloom effects, back-lit screen aesthetic"

Typography:
Heading: "Bold Sans-Serif, Neon Glow Effect"
Body: "Clean Monospaced or DIN-style Sans-Serif"
```

#### Template: Isometric 3D / Hi-Tech
```
Overall Design Settings:
Tone: "Sophisticated, analytical, high-tech, visionary"

Visual Identity:
Background Color: "#020712"
Text Color: "#E1F5FE"
Accent Color: "#00E5FF"
Secondary Colors: ["#FF5252", "#2979FF", "#D1C4E9"]

Image Style:
Features: "Floating isometric screens, wireframe models, luminous particle streams, hexagonal data blocks, semi-transparent glass interfaces"
Texture: "Glassy, ethereal, digital grid patterns, smooth glowing gradients"
Lighting: "Emissive internal glow, neon rim lighting, volumetric effects against deep void"
Perspective: "Isometric 3D"
Elements: "HUD components"
```

#### Template: Neo-Retro Dev Deck (90s Computer Manual × Modern Dev Tools)
```
Overall Design Settings:
Tone: "Nostalgic yet cutting-edge, like a cyberpunk zine about developer tools"

Visual Identity:
Background Color: "#FFF8E7" (light cream grid-paper)
Text Color: "#1A1A1A"
Accent Color: "#FF1493" (hot pink)
Secondary Colors: ["#FFD600" (bright yellow), "#00E5FF" (cyan)]

Image Style:
Features: "Pixel-art icons, stacked modular card layouts, engineering graph paper texture"
Texture: "Grid-paper background, thick black outlines on all elements"
Borders: "Thick black borders, dotted grid lines"
Elements: "Self-contained modular blocks per section"

Typography:
Heading: "Bold condensed all-caps, filling full width, 10:1 size ratio"
Body: "Short declarative punchy statements, retro terminal fonts for code"

Copy Style:
"Short, declarative, punchy. No long sentences. Each section is a self-contained block."
```

#### Community Style Templates Reference

The [awesome-notebookLM-prompts](https://github.com/serenakeyitan/awesome-notebookLM-prompts) repository contains field-tested visual style templates. Key styles include:

| Style | Aesthetic | Best For |
|-------|-----------|----------|
| Modern Newspaper | Swiss/Bauhaus, yellow/black | Business media |
| Yellow × Black Editorial | Fashion magazine layout | Stylish presentations |
| Neo-Retro Dev Deck | 90s manuals × modern dev tools | Tech content |
| Cyberpunk/Holographic | Neon wireframes, glitch effects | Futuristic topics |
| Classic/Pop (Sculpture × Vaporwave) | Marble × neon pop | High-impact visuals |
| Digital/Neo/Pop | Amoeba shapes, vivid colors | Youth/creative |
| Sports/Athletic/Energy | Motion blur, bold italic type | High-energy content |
| Tech/Art/Neon | Constructivism, grid lines | Technical/analytical |
| Studio/Mockup/Premium | Apple device mockups | Product showcases |
| Pink Street-Style | Pop illustrations, thick lines | Casual/fun |

These styles work with both slides (via design prompt) and videos (via `--focus` parameter with structured visual instructions).

### Nano Banana PPT Prompts (Presentation-Focused)

The [awesome-nano-banana-ppt-prompts](https://github.com/ahmetbaldede/awesome-nano-banana-ppt-prompts) repository provides 24 curated visual style prompts originally for PowerPoint but adaptable to NotebookLM slides and videos:

| Style | Category | Best For |
|-------|----------|----------|
| Blueprint Style | Engineering | Technical architecture |
| McKinsey Presentation Style | Business | Strategy, consulting |
| Apple-Style Minimalism | Product | Launch decks, product showcases |
| IBM Carbon Style | Corporate | Enterprise, modern corporate |
| IBM Paul Rand Heritage | Corporate | Classic brand identity |
| Saul Bass (3 variants) | Artistic | Title sequences, geometric, narrative |
| Dark Data Visualization Dashboard | Data | Analytics, dashboards |
| Data Visualization Chart Key Metrics | Data | KPI presentations |
| SaaS Dashboard Style | Product | SaaS product demos |
| Whiteboard Data Analysis Sketch | Casual | Workshops, brainstorming |
| Analog Film Photography | Artistic | Moody, nostalgic content |
| Moody Outdoor Photography | Artistic | Nature, atmospheric |
| Travel Journal Collage | Creative | Travel, lifestyle |
| WPA National Park Poster | Retro | Vintage, poster-style |
| Vintage Architectural Watercolor | Artistic | Architecture, heritage |
| Stylized Architectural Blueprint | Technical | Concept art, design |
| Cute Educational Game Map | Education | Kids, gamified learning |
| Egyptian Theme for Students | Education | History, themed lessons |
| Whimsical 3D Isometric Christmas | Seasonal | Holiday content |
| Modern Minimalist Christmas | Seasonal | Clean holiday design |
| Luxury Vintage Christmas | Seasonal | Premium holiday |
| Christmas Classic Color Theme | Seasonal | Traditional holiday |

These prompts can be used as `--focus` text for NotebookLM video/slides or adapted as text sources for design specifications.

#### Simple but effective styles (one-liners)
These short prompts also produce distinct results:
- `"crayon like a child's drawing"` — hand-drawn crayon art
- `"Lego style"` — Lego block visuals
- `"Physical Media Collage, retro-futurism, American 50s-60s, Atom Age"` — vintage collage
- `"Oil painting, moody and dramatic, dark greens and golds, Baroque style"` — classical art

### Making Videos Engaging

**Hook-first structure:**
```
Open with a provocative question: "Why do 90% of ML projects never reach production?"
Pause for 3 seconds. Then reveal that the answer isn't technical — it's organizational.
Build from there.
```

**Step-by-step educational:**
```
Walk through the 3 main concepts step by step. For each concept:
1. State it in one sentence
2. Show a concrete real-world example
3. Explain the most common mistake people make
Target audience: engineers who know React but are new to server components.
```

### Tips for Video

> **CRITICAL: プリセットスタイル（classic, whiteboard等）は使わないこと。**
> プリセットは汎用的で没個性な動画しか生成しない。`--style custom --sd "..."` を**必ず**使用し、構造化プロンプトでビジュアルを完全にコントロールすること。

- **`--style custom --sd "..."` が唯一の推奨パス** — プリセットスタイルとの差は歴然。プリセットは「どこかで見たことがある」汎用動画、customは「そのコンテンツのために作られた」専用動画になる
- `--sd` の中身が勝負 — **短い説明（"dark tech style"等）はほぼ効果なし**。必ず以下を含む構造化プロンプトを使うこと:
  - **HEXカラーコード**（背景、テキスト、アクセント、セカンダリ）
  - **具体的なビジュアル要素**（テクスチャ、ボーダー、アイコンスタイル）
  - **タイポグラフィ**（見出しとボディの具体的なスタイル）
  - **トーン/ムード**（形容詞で明確に）
- `--focus` = コンテンツの方向性、`--sd` = ビジュアルの方向性 — **両方を必ずセットで使う**
- Community templates (awesome-notebookLM-prompts, awesome-nano-banana-ppt-prompts) をベースにして `--sd` プロンプトを構成すると効率的
- `cinematic` は例外的に Ultra tier + English のみで使用可
- Custom style がまれに Classic にフォールバックすることがある — その場合は再生成
- 動画は生成後に修正不可 — `--sd` と `--focus` を最初から正確に設定する
- 生成に5-10分以上かかることがある

### CLI Usage: Custom Style Video

**必ずこのパターンで動画を作成する:**

```bash
nlm create video NOTEBOOK \
  --style custom \
  --sd "Overall Design Settings:
Tone: \"[形容詞3-4個でムードを定義]\"

Visual Identity:
Background Color: \"#[HEX]\"
Text Color: \"#[HEX]\"
Accent Color: \"#[HEX]\"
Secondary Colors: [\"#[HEX]\", \"#[HEX]\"]

Image Style:
Features: \"[具体的なビジュアル要素を列挙]\"
Texture: \"[表面の質感]\"
Borders: \"[ボーダースタイル]\"

Typography:
Heading: \"[見出しの具体的なスタイル]\"
Body: \"[本文の具体的なスタイル]\"" \
  --focus "[コンテンツの方向性・強調ポイント]" \
  --language ja \
  --confirm
```

**悪い例（効果なし）:**
```bash
# NG: プリセットスタイル → 汎用的で退屈な動画
nlm create video NB --style classic --confirm

# NG: 短いスタイル記述 → ほぼ効果なし
nlm create video NB --style custom --sd "dark tech style" --confirm
```

**良い例（効果大）:**
```bash
# OK: 構造化プロンプト → 個性的で印象に残る動画
nlm create video NB --style custom \
  --sd "Overall Design Settings:
Tone: \"Technical, authoritative, cryptic, high-tech hacker ethos\"
Visual Identity:
Background Color: \"#050505\"
Text Color: \"#2CFF56\"
Accent Color: \"#FFB200\"
Secondary Colors: [\"#00551A\", \"#1E1E1E\"]
Image Style:
Features: \"ASCII art, command-line interfaces, binary rain, schematic flowcharts\"
Texture: \"CRT monitor phosphor glow, scanlines, digital noise\"
Borders: \"Bracket-style terminal frames\"
Typography:
Heading: \"Pixelated Block Display\"
Body: \"Monospaced Console Typeface (Fira Code style)\"" \
  --focus "Content focus here" \
  --language ja --confirm
```

**Note:** `--sd` は `--style-description` の短縮形。`--style custom` と組み合わせた場合のみ有効。

---

## 5. Slides

### Formats & Lengths

| Format | Description |
|--------|-------------|
| `detailed_deck` | Full content on each slide, standalone reading |
| `presenter_slides` | Minimal text, visual support for live presentation |

| Length | Slides |
|--------|--------|
| `short` | ~8-10 slides |
| `default` | ~15-20 slides |
| `long` | ~25-30 slides |

### Custom Design Prompts

Slides support **custom design prompts** through the Studio edit field (pencil icon). This is separate from the focus prompt and controls visual appearance.

**Effective structure:**
```
[Background/mood description]
[Color palette with HEX codes]
[Typography with specific font names]
[Layout rules]
[What to avoid]
```

#### Design Templates

**Minimalist Business:**
```
Clean white background with subtle light gray (#F5F5F5) accents.
Typography: Montserrat Bold for titles, Noto Sans JP for body text.
Palette: background #FFFFFF, text #0F172A, accent #2563EB, secondary #64748B.
One message per slide. Generous whitespace. No decorative elements.
Avoid: clip art, gradients, busy backgrounds, markdown symbols in text.
```

**Dark Tech:**
```
Matte black (#121212) background with charcoal gradient zones.
Typography: San Francisco Display for headers, Roboto for body.
Palette: background #121212, text #E5E7EB, accent #38BDF8, secondary #818CF8.
Data-driven layouts with clean charts. Subtle grid lines.
Avoid: bright backgrounds, serif fonts, crowded layouts.
```

**Warm Educational:**
```
Warm beige (#F5F1E8) background with wabi-sabi organic feel.
Typography: Lora for titles, Nunito Sans for body.
Palette: background #F5F1E8, text #2F2A24, accent #52B788, secondary #B08968.
Friendly, approachable layouts. Include diagrams and visual metaphors.
Avoid: corporate blue, sharp edges, dense text blocks.
```

**Magazine Editorial:**
```
White background (#FAFAF9) with strong editorial grid.
Typography: Didot for display titles (large), Inter Light for body.
Palette: background #FAFAF9, text #1C1917, accent #DC2626, secondary #78716C.
Dramatic title/body size contrast (10:1 jump ratio). Asymmetric layouts.
Pull quotes as design elements. Generous negative space.
Avoid: centered layouts, uniform text sizes, clip art, markdown # or ** symbols.
```

**Cyberpunk/Neon:**
```
Near-black (#09090B) background with neon glow effects.
Typography: Orbitron for headers, Rajdhani for body.
Palette: background #09090B, text #F4F4F5, accent #22D3EE, secondary #A855F7.
Futuristic grid overlays. Glowing accent lines between sections.
Avoid: warm colors, organic shapes, traditional corporate layouts.
```

### Design Best Practices

1. **HEX codes, not color names** — "accent #2563EB" not "blue accent"
2. **Name specific fonts** — "Helvetica Now", "Inter", not "sans-serif"
3. **Explicit prohibitions** — "Avoid: markdown symbols (#, **), clip art, centered layouts"
4. **One slide = one message** — prevents text-heavy slides
5. **Mood metaphors** — "Like a smartphone-first financial media" or "Apple keynote clarity" guides aesthetic judgment better than listing rules
6. **Language directive** — "The language should match whatever users said in the prompt"

### Consistent Design Across Many Slides

For 15+ slides, add design spec and content outline as text sources:

```bash
nlm source add <nb-id> --text "DESIGN SPECIFICATION:
Color palette: #FFFFFF background, #0F172A text, #2563EB accent
Fonts: Montserrat Bold (titles), Noto Sans JP (body)
Rules: 1 message per slide, generous whitespace, no markdown symbols
Layout: Title slides use full-bleed accent color, content slides use white" --title "Design Spec"
```

```bash
nlm source add <nb-id> --text "SLIDE OUTLINE:
Slide 1: Title - [Project Name]
Slide 2: Problem Statement — why this matters now
Slide 3: Key Change #1 (include comparison chart)
..." --title "Slide Outline"
```

### Slide Revision

Revise individual slides after generation:
```bash
nlm slides revise <artifact-id> --slide '3 Change the chart to a bar graph and make the title larger' --confirm
```
Creates a NEW deck. Original preserved. Revision does NOT reference sources — only existing slide content.

---

## 6. Infographic

### Dimensions

| Orientation | Best For |
|-------------|----------|
| `landscape` | Desktop, presentations, reports |
| `portrait` | Mobile, social media, printing |
| `square` | Social posts, thumbnails |

### Detail Levels

| Level | Density |
|-------|---------|
| `concise` | Key highlights, very visual |
| `standard` | Balanced text and visuals |
| `detailed` | Comprehensive, more text |

### Styles

| Style | Aesthetic | Best For |
|-------|-----------|----------|
| `auto_select` | AI chooses | When unsure |
| `sketch_note` | Hand-drawn sketchnote | Informal, creative |
| `professional` | Clean corporate | Business reports |
| `bento_grid` | Grid sections | Data comparison |
| `editorial` | Magazine-quality | Public-facing content |
| `instructional` | Step-by-step flow | How-to guides, processes |
| `bricks` | Building blocks | Modular concepts |
| `clay` | 3D clay render | Playful, modern |
| `anime` | Anime-inspired | Youth audience |
| `kawaii` | Cute Japanese | Fun, casual |
| `scientific` | Academic | Papers, research |

### Recommended Combinations

| Use Case | Orientation | Style | Detail |
|----------|-------------|-------|--------|
| Process/workflow | `portrait` | `instructional` | `standard` |
| Executive dashboard | `landscape` | `professional` | `concise` |
| Data comparison | `landscape` | `bento_grid` | `standard` |
| Social media share | `square` | `sketch_note` | `concise` |
| Research summary | `portrait` | `scientific` | `detailed` |
| Tutorial/how-to | `portrait` | `instructional` | `detailed` |

---

## 7. Interactive Workflow

**This is the core workflow. When a user requests NotebookLM content creation, follow these steps.**

### Phase 1: Narrow Down with AskUserQuestion

Use AskUserQuestion to quickly establish the basics. Ask up to 4 questions at once:

**Question set 1** (always ask):
- **Artifact type**: audio / video / slides / infographic / report / etc.
- **Audience**: Who is this for? (technical level, role, relationship to topic)
- **Purpose**: What should the audience think/feel/do after consuming this?
- **Tone/style**: Formal/casual, educational/persuasive, serious/fun

**Question set 2** (artifact-specific, ask after Q1):
- **Audio**: format (deep_dive/brief/critique/debate), host style, length
- **Video**: format (cinematic/explainer/brief), visual style — **プリセットスタイルは提案しない。必ず `custom` で作成する。** ユーザーにムード（テック/ポップ/レトロ/ミニマル等）、好みの色合い、参考にしたい雰囲気を聞き、それを構造化 `--sd` プロンプト（HEXコード・タイポグラフィ・テクスチャ付き）に変換する。
- **Slides**: format (detailed/presenter), design template, length
- **Infographic**: orientation, style, detail level

### Phase 2: Draft Scenario/Layout

Based on the user's answers, draft a concrete scenario before generating. Present it as text for review:

**For audio — draft a show outline:**
```
[Show Name] Episode N: "[Episode Title]"

HOSTS: [Persona descriptions]
TARGET LISTENER: [Listener persona]

STRUCTURE:
1. Opening (1min): [Hook — what grabs attention]
2. Segment 1 (4min): [Topic] — [angle/approach]
3. Segment 2 (4min): [Topic] — [angle/approach]
4. Segment 3 (3min): [Topic] — [angle/approach]
5. Takeaways (2min): [What listener should remember]
6. Closing (1min): [Call to action]

TONE: [Description]
AVOID: [AI-isms and unwanted patterns]
```

**For slides — draft a slide outline:**
```
SLIDE OUTLINE: "[Deck Title]"
AUDIENCE: [Who]
DESIGN: [Template name] — [palette summary]

1. Title Slide — [Title + subtitle]
2. [Slide name] — [Key message, 1 sentence]
3. [Slide name] — [Key message + visual: chart/diagram/comparison]
...
N. Summary / Next Steps / Q&A

FOCUS: [What to emphasize]
AVOID: [What to skip]
```

**For video — draft a storyboard outline:**
```
VIDEO: "[Title]"
FORMAT: [explainer/brief/cinematic]
STYLE: [visual style]
AUDIENCE: [Who]

OPENING (0:00-0:15): [Hook — question/stat/claim]
SECTION 1 (0:15-1:30): [Topic + key visual]
SECTION 2 (1:30-2:45): [Topic + key visual]
SECTION 3 (2:45-3:30): [Topic + key visual]
CLOSING (3:30-4:00): [Takeaway + call to action]

TONE: [Description]
```

**For infographic — draft a content layout:**
```
INFOGRAPHIC: "[Title]"
ORIENTATION: [landscape/portrait/square]
STYLE: [style name]
DETAIL: [concise/standard/detailed]

SECTIONS:
1. Header — [Title + one-line subtitle]
2. [Section] — [What data/concept to show]
3. [Section] — [What data/concept to show]
4. [Section] — [What data/concept to show]
5. Footer — [Source attribution / call to action]

COLOR THEME: [Description or HEX codes]
FOCUS: [What to emphasize]
```

### Phase 3: Get Feedback

Present the draft to the user and ask:
- "Does this structure match what you had in mind?"
- "Any sections to add, remove, or reorder?"
- "Any specific data points, examples, or angles to include?"

Incorporate feedback into the draft.

### Phase 4: Generate

Convert the approved draft into a `--focus` prompt (and design prompt for slides/infographic), then execute the nlm command. Key rules:

1. **Focus prompt**: Combine the approved structure, audience, tone, and avoid-list into a concise prompt
2. **Source-as-script**: For complex structures, add the approved outline as a text source
3. **Design prompt** (slides only): Apply the selected template with HEX codes and fonts
4. **Generate and monitor**: Run the command, poll `studio status`, report completion

### Phase 5: Review & Iterate

After generation:
- For **slides**: offer `slides revise` for individual slide adjustments
- For **audio/video**: offer to regenerate with adjusted focus prompts
- For **infographics**: offer to regenerate with different style/orientation/detail
- Ask: "How does this look? Any changes needed?"

---

## 8. Focus Prompt Patterns

| Pattern | Example | Works With |
|---------|---------|------------|
| **Audience targeting** | "Explain for non-technical executives" | All |
| **Angle selection** | "Focus on security implications, not performance" | All |
| **Structure direction** | "Start with the conclusion, then show evidence" | Audio, Slides |
| **Tone setting** | "Keep it casual and use everyday analogies" | Audio, Video |
| **Data emphasis** | "Highlight the 3 key metrics with specific numbers" | Infographic, Slides |
| **Comparison** | "Compare approach A vs B across cost, speed, quality" | Infographic, Slides |
| **Exclusion** | "Don't cover history, only current state" | All |
| **Forbidden words** | "Avoid: exactly, deep dive, game-changer, unpack" | Audio, Video |
| **Persona framing** | "You are a skeptical CTO evaluating vendor claims" | Audio, Video |
| **Listener persona** | "The listener is a junior dev on their first production deploy" | Audio, Video |
| **Metaphor/mood** | "Like an Apple keynote — confident, minimal, surprising" | Slides, Video |
| **Language mixing** | "Discuss in Japanese, keep English technical terms" | Audio, Video |
| **News format** | "Deliver as a news briefing, no editorializing" | Audio |
| **Structured segments** | "3 segments: problem → solution → migration steps" | Audio, Video, Slides |
