---
name: review-mining-ad-angles
description: Mine real customer reviews from an ecommerce product page and turn voice-of-customer language into ad angles, hooks, and messaging. Use this skill whenever the user wants to analyze reviews, extract customer language, find ad angles, understand objections, pull hooks from reviews, or mine voice-of-customer data from a product URL. Also trigger on phrases like "what are people saying about this product", "find ad angles from reviews", "mine the reviews", "read the reviews for me", "voice of customer", "VOC research", "what objections do customers have", or when a user shares a product URL and asks what messaging would work. Always trigger before attempting manual review analysis.
---

# Review Mining → Ad Angle Extractor

Scrapes real customer reviews from an ecommerce product page, then extracts the language, objections, unexpected use cases, and emotional triggers customers actually express — and converts them into ready-to-use ad angles, hooks, and messaging.

The core principle: the highest-converting ad copy is already written by customers. This skill finds it.

---

## Step 1 — Collect Input

Ask the user for:

1. Product page URL (the product whose reviews to mine)
2. Optional: a specific angle focus (e.g. "focus on objections", "focus on gifting use cases") — if none given, run the full analysis

Then confirm before proceeding.

---

## Step 2 — Locate the Review Source

Reviews on ecommerce sites are almost always rendered inside a third-party JavaScript widget that lazy-loads. A plain scrape of the product URL will usually miss them. Before scraping, identify which review platform the store uses, because each behaves differently.

Common review widgets and their signals:

| Platform | How it renders | Notes |
|---|---|---|
| **Judge.me** | `jdgm-` prefixed elements, lazy-loads on scroll | Very common on Shopify |
| **Okendo** | `okeReviews` / `oke-` elements | Loads a widget container, reviews paginate |
| **Loox** | `loox-` elements, photo-review grid | Photo-heavy, reviews behind "load more" |
| **Yotpo** | `yotpo` / `yotpo-main-widget` | Reviews paginate, may need scroll |
| **Stamped.io** | `stamped-` elements | Paginated |
| **Shopify native / product reviews** | inline in page markup | Sometimes in the base HTML |
| **Trustpilot (off-site)** | separate trustpilot.com/review/[domain] page | Scrape the Trustpilot page directly |
| **Amazon** | on the Amazon product page | Reviews often in base HTML, paginate via `?pageNumber=` |

If reviews live on a separate page (Trustpilot, Amazon), scrape that page directly rather than the product page.

---

## Step 3 — Scrape the Reviews via Firecrawl (Tiered, Credit-Aware)

Review widgets lazy-load and paginate behind real interaction, not just time. Below is what actually works with this Firecrawl MCP — verified against the tool's real schema and real behavior, not assumed.

### Hard facts about this MCP (do not deviate from these)

- **There is no `actions` parameter.** Firecrawl scrape does not support a scroll/click actions array. Do not attempt it — it fails validation outright.
- **JSON extraction syntax:** `formats: ["markdown", "json"]` as bare strings, plus a separate top-level `jsonOptions: { prompt: "..." }` object. Do not nest `type`/`prompt` inside `formats`.
- **`waitFor` has a ceiling, not a scaling effect.** Increasing `waitFor` from 2500ms to 10000ms on a paginated review widget returns identical results. Most review widgets render a fixed initial batch (commonly 5) regardless of wait time — the rest is gated behind a real click or dropdown change, not a timer. Do not retry the same static scrape with a longer `waitFor` expecting more content — it will not help and it burns a credit for nothing.
- **`firecrawl_interact` has a concurrency limit (2 jobs) and requires user approval** — it drives a real browser session, which is a permission-gated action. It also frequently returns "maximum concurrent jobs" errors that require waiting and retrying.
- **`firecrawl_agent` (async autonomous agent) is unreliable for this use case.** In testing, a review-sorting-and-pagination task ran 5+ minutes without completing. It is not a dependable default path and should not be reached for automatically.

### Credit budget rule — read before scraping anything

Firecrawl credits are real cost. This skill previously burned 150 credits in a single session by retrying static scrapes with escalating `waitFor` values and then falling back to slow, expensive `interact`/`agent` calls without checking in with the user first. That must never happen again. Follow this budget strictly:

1. **One static scrape, period.** Run exactly ONE `firecrawl_scrape` call with `waitFor: 3000` (no higher — higher does not help, confirmed). Do not repeat this call with a different `waitFor` value. If it returns reviews, move to Step 3B. If it returns zero reviews, try ONE alternate: the review platform's likely widget anchor (e.g. append `#reviews` or `#judgeme_product_reviews` to the URL) — still only one retry, not several.
2. **Stop and ask before any paginated/interactive attempt.** If deeper review coverage (beyond the default ~5) is needed for a stronger analysis, STOP and tell the user plainly: "I can get a fuller picture by driving an interactive browser session to sort by lowest rating and load more reviews, but this is slower, costs more, and needs your approval to run. Want me to do that, or proceed with what's already scraped?" Never call `firecrawl_interact` or `firecrawl_agent` without this explicit check-in.
3. **If approved, use `firecrawl_interact` — not `firecrawl_agent`.** `interact` is the faster, more controllable path. If it hits the concurrency limit, wait ~25 seconds and retry ONCE. If it still fails, tell the user it's not available right now rather than escalating to `firecrawl_agent`, which is slow and has shown unreliable completion.

### Step 3A — The always-free, always-reliable data (get this first, every time)

A single static scrape of the product page, even without any pagination, reliably returns two high-value data points that need zero interaction:

- **The full rating distribution** — e.g. "281 reviews, 4.2★ average: 181×5★, 28×4★, 33×3★, 18×2★, 21×1★." This tells you exactly how skewed the visible reviews will be and how much objection-heavy content (3★ and below) exists that you may not be able to reach.
- **Platform-generated topic tags** (e.g. "Popular topics: results, improvement, quality, issues, feel, benefits, ingredients, price") — these are a free thematic signal even when full review text isn't reachable, and should always be captured and used in the analysis.

Treat these as first-class outputs of Step 3, not an afterthought. They alone are enough to produce a partial, honestly-labeled angle set if deeper pagination isn't available or isn't worth the cost.

### Step 3B — The scrape call

```
formats: ["markdown", "json"]
jsonOptions: {
  prompt: "Extract every customer review visible on the page, in the order they appear, regardless of star rating. For each review return: reviewer name if shown, star rating, review title, full review text verbatim, date if shown. Also extract the overall average rating, total review count, full star-by-star rating distribution, and any 'popular topics' or theme tags shown."
}
waitFor: 3000
```

### Step 3C — Sample honesty

Most product pages default-sort reviews by "Highest rating," so a single static scrape will skew positive (4★–5★ only). This is expected and fine — it is not a failure. Report the skew honestly in the output (Step 8) rather than presenting a 5-review sample as if it represents the full distribution. Do not fabricate or infer what lower-rated reviews might say.

### If reviews are on an off-site platform (Trustpilot, Amazon)

Scrape that URL directly with the same Step 3B call — off-site platforms often render reviews server-side without JS pagination, which avoids this whole problem. Prefer this route when available; it's cheaper and more reliable than fighting an on-site widget.

### Volume expectations — revised

Do not target "40–50 reviews minimum" as a hard requirement — that number assumed pagination would reliably work, which this session proved false. Instead:

- **Baseline (always achievable):** rating distribution + topic tags + whatever reviews the default static scrape returns (typically 3–8)
- **Enhanced (only with explicit user approval for the interactive step):** a larger, ratings-diverse sample via `interact`
- Always tell the user which tier the analysis is based on.

---

## Step 4 — Structure the Raw Reviews

Once scraped, organize the reviews before analysis:

- Total count collected
- Rating distribution (how many 5, 4, 3, 2, 1 star)
- Separate the reviews into buckets: glowing (5★), qualified positive (4★), mixed (3★), negative (1–2★)

Keep the raw verbatim text — the exact wording is the asset. Do not paraphrase at this stage.

---

## Step 5 — Extract the Voice-of-Customer Intelligence

Analyze the review corpus across these dimensions. This is the core of the skill.

**Work with whatever tier was actually reached (see Step 3).** If only the baseline tier (rating distribution + topic tags + a handful of top-sorted reviews) is available, extract everything possible from that — the topic tags and rating distribution are real, useful signal even without full review text. Do not wait for a "complete" dataset that pagination may never deliver. Label every angle with which tier it came from.

### 5A — Repeated Language Patterns
Identify the exact words and phrases customers use over and over. These are the words that should appear in ad copy because they match how the customer already thinks. Pull direct quotes. Note which phrases recur across multiple reviews.

### 5B — Objections and Hesitations
From the 3★ and 4★ reviews especially, extract what customers were skeptical about, what nearly stopped them buying, and what they were surprised didn't happen. Objections that get defeated in the same review are the highest-value ad material.

### 5C — Emotional Triggers
Identify the emotional payoff customers describe — the before-state (frustration, embarrassment, fatigue) and the after-state (relief, confidence, delight). Capture the emotional arc, not just the functional benefit.

### 5D — Unexpected Use Cases
Find uses the brand may not be marketing: gifting, travel, a specific occasion, an unintended audience, a problem the product solved that isn't on the product page. These are often untapped ad angles.

### 5E — Objection → Resolution Pairs
The highest-converting ad structure. Find reviews that follow "I was worried about X, but Y happened." These map directly to ads that pre-empt the exact hesitation a cold prospect has.

### 5F — Standout Verbatim Quotes
Pull 8–12 of the single most usable direct quotes — specific, vivid, emotionally charged, and short enough to work as a hook or headline. Keep them verbatim.

---

## Step 6 — Convert Intelligence into Ad Angles

Turn the extracted intelligence into concrete, usable ad angles. Produce at least 5 distinct angles. Each angle must include:

1. **Angle name** — a short label for the strategic direction
2. **The insight it's built on** — which review pattern or objection it comes from
3. **The hook line** — a scroll-stopping opening line built from customer language (this is the actual copy)
4. **Supporting proof** — which verbatim quote(s) back it up
5. **Best format** — which ad archetype or format suits this angle (static, UGC, before/after, etc.)

### Angle selection heuristic
Do not build every angle around the most flattering review. Prioritize angles built on:
- An objection that gets defeated (converts skeptics)
- A specific, vivid transformation (concrete beats vague)
- An unexpected use case (opens new audiences)
- A repeated phrase that reveals how customers actually describe the product (matches their internal language)

---

## Step 7 — Copy Quality Gate

Every hook line and piece of ad copy produced must pass the banned-pattern checklist before being presented. Never skip this.

### Banned patterns — flag and remove all of these:

1. **Anaphora** — stacked sentences sharing the same opening structure ("Higher X. Higher Y.")
2. **False negatives** — "[Thing] isn't X, it's Y" / "It's not about X, it's about Y"
3. **Reversal framing** — "Most people think X. The truth is Y." / "Everyone focuses on A. The real leverage is B."
4. **Asyndeton** — fragmented one-word lists separated by periods. "No edits. No switches. No tweaks."
5. **Mechanism contrast** — abstract superiority framing ("The math compounds in ways that acquisition can't")
6. **Rhetorical questions** — "The reality?" / "Why does this matter?" / "And honestly?"
7. **Clever comparative endings** — "The worst X do Y, the best X do Z."
8. **Observational authority claims** — "I've seen this play out" / "I see this constantly."
9. **Banned phrases** — "Here's the thing" / "But here's" / "It's not X, it's Y" / the word "shift" / "What if I told you" / "Wonderful" / any sentence starting with "Most."
10. **Formatting violations** — no bold, italic, or underlined text in the copy. No excessive em dashes. No unnecessary numbered lists inside prose.

### Important exception for verbatim customer quotes
Customer quotes are reproduced exactly as written, even if they happen to contain a banned pattern — they are evidence, not brand copy. The banned-pattern rules apply to the hook lines and ad copy YOU write, not to the customer's own words. Always mark which is which: a verbatim quote is labelled as a customer quote; a hook line is your copy and must pass the checklist.

---

## Step 8 — Present the Output

Deliver in this structure:

1. **Scrape summary** — how many reviews were analyzed, rating distribution, review platform detected, and **which tier the analysis is based on** (baseline static scrape only, or enhanced with an approved interactive pass). State plainly if the sample skews toward high ratings because pagination wasn't used or wasn't available — never present a small, positive-skewed sample as if it's representative of the full review base.
2. **Voice-of-customer findings** — the repeated language, objections, emotional triggers, and unexpected use cases (from Step 5)
3. **The ad angles** — at least 5, each fully specified (from Step 6)
4. **Verbatim quote bank** — the standout quotes actually scraped, ready to drop into UGC ads or as social proof
5. **Handoff note** — which angles feed naturally into the meta-static-ad-generator skill or a UGC ad skill

If the user has the meta-static-ad-generator skill, offer to pass the strongest angle straight into it to generate the actual ad.

---

## Edge Cases

**Static scrape returns very few reviews (e.g. 3–8) despite the product having hundreds:** This is expected default behavior, not a bug — most review widgets render a fixed initial batch. Report the baseline tier honestly (Step 8) rather than retrying the scrape. Only attempt deeper pagination if the user explicitly approves the interactive step (Step 3, budget rule 2).

**`firecrawl_interact` returns "maximum number of concurrent jobs" error:** Wait about 25 seconds and retry once. If it fails again, stop — do not escalate to `firecrawl_agent` as a workaround, and do not keep retrying `interact`. Tell the user the interactive pass isn't available right now and proceed with the baseline tier.

**`firecrawl_agent` doesn't complete within a few minutes:** Stop polling. This tool has shown unreliable completion times for review-pagination tasks specifically. Do not use it as the default or fallback path for this skill — `firecrawl_interact` (with explicit approval) is the only interactive method this skill should attempt.

**Few or no reviews on site at all:** Tell the user plainly. Offer to check Trustpilot, Amazon, or other off-site sources — these are often easier to scrape since they render server-side. Do not fabricate reviews or invent customer language under any circumstance.

**All reviews are 5-star / clearly filtered, or the sample is small and skewed positive:** State this openly in the output. The angles will skew positive and objection data will be thin. Recommend checking a third-party source for balance, or getting user approval for the interactive pass to reach lower-rated reviews.

**Fabrication guardrail:** Every quote presented must come from the actual scrape. Never invent, embellish, or "clean up" a customer quote beyond fixing obvious typos. If a claim isn't traceable to a real scraped review, it does not go in the output.
