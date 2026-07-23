# review-mining-ad-angles

A Claude Skill that mines real customer reviews from an ecommerce product page and converts voice-of-customer language into ready-to-use ad angles, hooks, and messaging.

The core principle: the highest-converting ad copy is already written by customers. This skill finds it.

---

## What it does

1. **Scrapes real reviews** from a product URL — detecting the review platform (Judge.me, Okendo, Loox, Yotpo, Stamped.io, Amazon, Trustpilot, and more) and scraping intelligently based on how each widget renders
2. **Structures the raw data** — organises reviews into star-rating buckets and preserves verbatim customer language (the exact wording is the asset)
3. **Extracts voice-of-customer intelligence** across five dimensions: repeated language patterns, objections and hesitations, emotional triggers, unexpected use cases, and objection → resolution pairs
4. **Converts intelligence into ad angles** — at least 5 distinct, fully specified angles each with a hook line built from real customer language, supporting proof, and a recommended ad format
5. **Applies a copy quality gate** — every generated hook line is checked against a 10-pattern banned-copy checklist before being shown to you
6. **Presents a structured output** — scrape summary, VOC findings, ad angles, a verbatim quote bank, and a handoff note for the next skill in your pipeline

---

## Prerequisites

This skill depends on one MCP connector. It will not run without it.

| Requirement | Why |
|---|---|
| **Claude Pro or Max** | Skills require a paid Claude plan |
| **Firecrawl MCP** connected | Used to scrape the product page reviews |

If you don't already have Firecrawl connected in Claude, go to **Settings → Connectors** and add it before installing this skill.

---

## Installation

### Option 1 — via skills.sh
```bash
npx skills add dhruvalraval/review-mining-ad-angles
```

### Option 2 — manual upload to claude.ai
1. Download this repo as a ZIP (**Code → Download ZIP**), or clone it and re-zip the `review-mining-ad-angles/` folder — the ZIP must contain the folder with `SKILL.md` inside it, not just the loose files
2. In Claude, go to **Settings → Capabilities** and turn on **Code execution and file creation**
3. Go to **Customize → Skills → + → Upload a skill**
4. Select the ZIP file
5. Toggle the skill on

---

## Usage

Once installed and Firecrawl is active, share a product URL in chat:

> "Mine the reviews for this product: https://example.com/products/your-product"
> 
> "Find ad angles from reviews: https://example.com/products/your-product"
> 
> "What are people saying about this? https://example.com/products/your-product"

Claude will confirm the URL, optionally take a focus direction (e.g. "focus on objections" or "focus on gifting use cases"), then run the full pipeline automatically.

---

## What you get

### Voice-of-customer intelligence
- **Repeated language patterns** — exact words and phrases customers use that should appear in your ad copy
- **Objections and hesitations** — what nearly stopped people buying, extracted from 3★–4★ reviews
- **Emotional triggers** — the before-state and after-state customers describe (frustration → relief, doubt → confidence)
- **Unexpected use cases** — gifting, travel, unintended audiences, problems the product solved that aren't on the product page
- **Objection → resolution pairs** — the "I was worried about X, but Y happened" structure that maps directly to cold-prospect ads

### Ad angles (at least 5)
Each angle includes:
- Angle name and the insight it's built on
- A hook line written from real customer language (passes the copy quality gate)
- Supporting verbatim proof quotes
- Recommended ad format (static, UGC, before/after, etc.)

### Verbatim quote bank
8–12 standout direct quotes — specific, vivid, emotionally charged, and short enough to drop straight into a UGC ad or use as a headline.

---

## Credit-aware scraping

Firecrawl credits cost real money. This skill follows a strict budget:

- **One static scrape, period** — a single `firecrawl_scrape` call with a fixed wait time. Retrying with longer waits does not return more reviews and wastes credits.
- **Baseline tier is always useful** — even a small scrape reliably returns the full rating distribution and platform-generated topic tags, which are first-class signal for angle generation.
- **Ask before going deeper** — if you want a larger, rating-diverse sample via an interactive browser session, the skill will explain the cost and ask for your explicit approval before running it.

---

## Review platform support

| Platform | Notes |
|---|---|
| **Judge.me** | Very common on Shopify stores |
| **Okendo** | Widget container with paginated reviews |
| **Loox** | Photo-review grid |
| **Yotpo** | Paginated, may require interactive pass |
| **Stamped.io** | Paginated |
| **Shopify native** | Sometimes in base HTML |
| **Trustpilot** | Scraped directly from trustpilot.com — often easier than on-site widgets |
| **Amazon** | Reviews in base HTML, paginated by URL parameter |

---

## Honesty guarantees

- Every quote presented comes from the actual scrape. No invented, embellished, or inferred customer language — ever.
- If the sample skews positive (because most pages default-sort by highest rating), the output will say so plainly rather than presenting it as representative.
- If very few reviews are accessible, the skill reports the honest baseline tier rather than fabricating coverage it doesn't have.

---

## Integrates with

If you have the **meta-static-ad-generator** skill installed, this skill will offer to pass the strongest angle directly into it to generate finished Meta ad creatives.

---

## License

MIT — use it, fork it, adapt it for your own brand or agency.

## Author

Built by Dhruval Raval