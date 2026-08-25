# Steam Publishing Research & Plan

This document captures research findings on publishing games to Steam and a
proposed process for repeatedly creating small games, producing promotional
media, and submitting them to Steam. It is a planning reference, not an
implementation — no automation described here exists yet in this repository.

Last researched: August 2026.

## 1. Steam's fixed timelines ("how fast can this go")

Steam publishing involves several **hard, non-negotiable clocks set by
Valve**. No amount of tooling or automation compresses them.

| Gate | Duration | Starts when | Compressible? |
|---|---|---|---|
| Identity/tax verification | ~2–7 business days | You submit Steamworks paperwork | Slightly (clean docs submitted fast) |
| Mandatory new-partner wait | **30 days** | Steam Direct fee paid on first app | **No** |
| Store page review | 3–5 business days (budget 7) | You click "Mark as ready for review" | Slightly |
| **"Coming Soon" minimum** | **14 days** | Store page approved & posted live | **No** |
| Build (technical) review | 3–5 business days (budget 7) | Store page approved, build submitted | Slightly |
| Final release | Manual click | "Release App" button | Never automatic |

Sources: [Steamworks release process](https://partner.steamgames.com/doc/store/releasing),
[Steamworks review process](https://partner.steamgames.com/doc/store/review_process),
[Steamworks Coming Soon docs](https://partner.steamgames.com/doc/store/coming_soon),
[Immutable 2026 publishing guide](https://www.immutable.com/guides/how-to-publish-a-game-on-steam).

**Key insight for publishing many games:** the 30-day identity/tax wait is
**per Steamworks partner account, one-time** — not per game. Once the account
is verified:

- **First game:** realistic floor is ~30 days if everything else is prepped
  in parallel; realistically 6–10 weeks once review queues and asset
  preparation are counted.
- **Every subsequent game:** floor drops to roughly the 14-day Coming Soon
  window plus ~1 week of review cycles, and these windows can run **in
  parallel** across multiple titles/apps if submissions are staggered.

## 2. Requirements to publish on Steam

### Account / legal / financial (one-time, per Steamworks partner account)

- Steamworks Partner account; signed NDA + Steam Distribution Agreement
- Tax form (W-9 / W-8BEN or local equivalent) + banking details for payouts
- **$100 USD Steam Direct Fee, per game** (not per account) — recoupable
  automatically once that title reaches $1,000 in Adjusted Gross Revenue
  ([Steamworks fee docs](https://partner.steamgames.com/doc/gettingstarted/appfee))

### Technical (per game)

- Standalone desktop build — Steam does **not** accept browser/web-only
  games. Windows `.exe` is the minimum; macOS/Linux are optional.
- The build must launch cleanly on every OS listed on the store page (this
  is explicitly checked during build review).
- Builds are uploaded via **SteamPipe** (`steamcmd`), which is fully
  scriptable from the command line — this is what makes an automated,
  repeatable pipeline realistic.

### Store page assets (per game)

All of the following are required before store-page review can pass:

| Asset | Spec |
|---|---|
| Header Capsule | 920×430 px |
| Small Capsule | 462×174 px |
| Main Capsule | 1232×706 px |
| Vertical Capsule | 748×896 px |
| Library Capsule | 600×900 px |
| Library Hero | 3840×1240 px PNG, artwork only, no text |
| Library Logo | Transparent PNG, up to 1280×720 |
| Shortcut / App icon | 256×256 `.ico`/`.png`, 184×184 `.jpg` |
| Screenshots | ≥5, minimum 1920×1080, 16:9, **real gameplay only** (no concept art), ≥4 marked "all ages" |
| Trailer | H.264 `.mp4`/`.mov`/`.wmv`, 1920×1080, 30 or 60 fps, 5,000+ Kbps, 60–90s recommended |
| Tags / description / surveys | Store tags (up to 20), short + long description, Mature Content Survey, Generative AI Content Survey |

Capsule art must contain only the game's title/logo and key visual — no
review scores, award logos, or discount marketing copy.

Sources: [Steamworks trailer docs](https://partner.steamgames.com/doc/store/trailer),
[Steam capsule asset requirements](https://www.steampageanalyzer.com/blog/steam-page-asset-requirements).

### Policy compliance (critical for a multi-game pipeline)

Valve does not ban low-quality games outright, but it aggressively bans
**developer accounts** that mass-produce near-identical, asset-flipped
games, farm trading cards/keys, or manipulate reviews/pricing. Entire
studios have had 173–248 titles purged at once for this
([Polygon](https://www.polygon.com/2017/9/26/16368178/steam-shovelware-removed-asset-flipping/),
[PCGamesN](https://www.pcgamesn.com/steam/valve-bans-games)).

**This is the single biggest risk to a "many games" strategy.** A
template-driven pipeline must produce genuinely differentiated games — not
recolors or reskins of the same asset pack — or the entire account (and
every title under it) risks termination.

### AI-generated content disclosure

Steam's Content Survey requires disclosing AI-generated content that
**ships to players** (art, audio, dialogue, code). Internal dev-tool use
(coding assistants, etc.) is explicitly exempt. This policy was rewritten in
January 2026, and sources still disagree on whether AI-made *marketing/store
assets* need disclosure — treat them as disclosable to be safe, and verify
against the live Steamworks form before each submission
([details](https://legalmoveslawfirm.com/steam-ai-policy/)).

## 3. Proposed pipeline

### Stage A — Reusable game template(s), not one-off builds

- Recommended engine: **Godot 4** — free, MIT-licensed, no revenue share,
  small install size, and fully **headless/CLI exportable**
  (`godot --headless --export-release`), which is what makes a repeatable
  pipeline realistic. GameMaker is faster for pure 2D but is a paid license
  and less automation-friendly; Unity is heavier and has revenue-threshold
  licensing.
- Build a small number of genuinely distinct template genres (e.g., one
  arcade-action template, one puzzle template) with swappable art/level
  config, rather than one template reskinned repeatedly — needed to avoid
  asset-flip flags.

### Stage B — Promotional asset generation

- Screenshots/capsules: captured or rendered directly from the actual build
  (Steam requires real gameplay, not concept art).
- Trailer: automated in-engine or scripted playthrough capture (e.g.,
  headless capture via OBS/ffmpeg) → edited via a scripted `ffmpeg` cut to
  spec (1920×1080, H.264, 30–90s).
- Key art/capsules: AI-assisted generation is viable but must respect IP
  rules and likely needs disclosure.

### Stage C — Steamworks onboarding (once) → per-game submission (repeatable)

1. Complete account paperwork + the 30-day wait **once**, as early as
   possible, decoupled from any specific game.
2. Per game: pay the $100 fee → build the store page → submit for review →
   upload the build via scripted SteamPipe → post "Coming Soon" immediately
   once approved (start the 14-day clock ASAP and accumulate wishlists) →
   submit the build for review → manually click "Release".

### Stage D — Scaling safely

- Stagger multiple titles' review/Coming Soon windows in parallel rather
  than serially.
- Track per-title state (fee paid, page/build review status, Coming Soon
  date, release date).
- Deliberately pace releases and ensure real gameplay differences between
  titles to stay inside Valve's tolerance for quantity.

## 4. Open questions to resolve before implementation

1. Is there an existing Steamworks Partner account, or does the 30-day/tax
   step need to start from zero?
2. What genre/complexity of "small game" is intended (arcade, puzzle,
   platformer, etc.), and roughly how many titles should the pipeline
   target?
3. Is AI-assisted art/trailer generation acceptable (with required
   disclosure), or should assets be human-made only?
4. Should the actual Godot template + automation scripts (asset capture,
   `ffmpeg` trailer assembly, SteamPipe upload scripts) be built next in
   this repository, or should the plan be discussed/adjusted further first?
