# Post-Processing Classification + Result Reporting Guide

## Table of Contents

- [Server Filter vs. Client-Side Post-Processing](#server-filter-vs-client-side-post-processing)
- [Field Mapping by Selection Criterion](#field-mapping-by-selection-criterion)
- [Purpose-Based Segmentation Examples](#purpose-based-segmentation-examples)
- [Follower-Range Filtering](#follower-range-filtering)
- [Using searchSummary](#using-searchsummary)
- [Selection-Reason Columns (Excel)](#selection-reason-columns-excel)
- [Result Reporting Format](#result-reporting-format)

---

## Server Filter vs. Client-Side Post-Processing

`filter_search_results` and this guide are complementary, not redundant. Pick the right side based on the task type:

| Task type | Where it belongs | Why |
|-----------|------------------|-----|
| **Hard gating** — in/out filter, every match must satisfy ("must be Korean female with email, 10K–50K") | `filter_search_results` server-side | Saves credits, returns more matches per page, no post-process noise |
| **Soft segmentation / scoring** — group by purpose (Seeding / UGC / Ambassador), rank by collab fit | This guide, client-side | Requires AI judgment combining multiple fields; no direct enum |
| **Selection-reason composition** — Excel "selection conditions" + "collaboration strength" cells, narration | This guide, client-side | Synthesis from multiple fields into prose |
| **Reporting** — final summary, bucket composition, comparative analysis | This guide, client-side | Output composition |

When a criterion has BOTH a server filter and a soft signal (e.g., follower range exists as `follower_ranges` enum and as raw `followed_by`), use the server filter for **gating** and reserve the returned field for **scoring / segmentation / reasoning**.

---

## Field Mapping by Selection Criterion

The "Server filter?" column shows the equivalent `filter_search_results` parameter when one exists. Prefer the server filter for hard gating; use the returned fields for soft segmentation and reporting.

| Criterion | Returned fields (post-processing) | Server filter? (`filter_search_results`) |
|-----------|-----------------------------------|------------------------------------------|
| Brand collaboration readiness | `collaborate_brand` (proven sponsored history), `note_for_brand_collaborate_point`, `note_for_brand_weak_point` | `collab_experience=["yes" \| "not_sure"]`, `filter_flags=["high_collaboration_potential"]` |
| Past collaboration history | `collaborate_brand` (sponsored), `tag_brand` (mentioned) | — (post-process only) |
| Content style | `description`, `field_of_creator` | `keyword` (substring across interests/keywords/brands/biography/hashtags) |
| Collaboration strengths / weaknesses | `note_for_brand_collaborate_point`, `note_for_brand_weak_point` | — (post-process only) |
| Follower scale (numeric) | `followed_by` → nano (<10K), micro (10K–50K), mid (50K–500K), macro (500K+) | `follower_ranges`: `<1K`, `1K-10K`, `10K-50K`, `50K-100K`, `100K-500K`, `500K-1M`, `1M+`, `unknown` |
| Follower scale (label) | `tier`: `Mega` \| `Macro` \| `Mid-tier` \| `Micro` \| `Nano` | (gate via `follower_ranges`; `tier` is a returned label, not a filter input) |
| Engagement | `engagement_rate` (+ `engagement_rate_tag` UI badge) | — (rank in post-processing) |
| Virality | `viral_reach_rate` (+ `viral_reach_rate_tag` UI badge) | `filter_flags=["high_viral_reach"]` |
| Growth | `follower_growth_rate` (+ `follower_growth_rate_tag` UI badge) | `filter_flags=["high_recent_growth"]` |
| Reel performance | `avg_reel_views` (fit for UGC / video) | — (post-process only) |
| K-culture interest | Infer from `description` / `field_of_creator` (e.g., "K-beauty", "K-pop") | `countries=["KR"]`, `languages=["ko"]`, `keyword="K-pop"` |
| Demographics | `age_range`, `gender`, `country`, `language` | `genders`, `age_ranges`, `countries`, `languages`, `regions` (continent-level) |
| External links present | `linkPlatforms[]`, `linkTypes[]` | `channels` (instagram, youtube, email, shopmy, …) — preferred when contact / specific platform is required |
| Unrevealed links exist | `hasUnrevealedLinks` (true → reveal needed) | — (post-process only) |
| Already-saved candidates | `is_in_my_bucket` | `exclude_bucket_influencers=true` (preferred for "show me more new picks") |

---

## Purpose-Based Segmentation Examples

- **Seeding fit**: non-empty `collaborate_brand` (proven sponsored history), `note_for_brand_collaborate_point` signaling collaboration readiness, similar-category brands in `collaborate_brand` / `tag_brand`
- **UGC fit**: `description` mentions tutorials / before-after / demos, high `avg_reel_views`
- **Ambassador fit**: `tier` in `Macro` / `Mega`, premium brands in `collaborate_brand` (e.g., luxury houses), `description` describing upscale lifestyle, brand-tone alignment
- **Long-term partnership fit**: positive `follower_growth_rate` (or `follower_growth_rate_tag` "Above Avg"+) + high `engagement_rate` + informational content mix

---

## Follower-Range Filtering

**Default — use the server filter.** `filter_search_results(searchId, follower_ranges=["10K-50K"], page=1)` is the canonical path:

- Bucketed enum: `<1K`, `1K-10K`, `10K-50K`, `50K-100K`, `100K-500K`, `500K-1M`, `1M+`, `unknown`
- Multiple buckets combine via OR — e.g., `["10K-50K", "50K-100K"]` for "10K to 100K"
- No credit cost, more matches per page than embedding the range in the search query and post-filtering

**Post-processing fallback** is only needed when the user names a custom range that doesn't align with the bucket boundaries (e.g., "23K to 47K"):

1. Pick the smallest enum that fully contains the user's range (here: `["10K-50K"]`)
2. Verify `followed_by` in the results and remove out-of-range accounts client-side
3. Report which accounts were removed and why

Do not embed follower-range expressions in the natural-language query — the server filter is precise and credit-free, while query embedding is a soft signal that the search ranker may or may not honor.

---

## Using searchSummary

`summary.searchSummary` in the search result is an AI-generated summary (in Korean).
You can reference or quote it when reporting the search result to the user.

The `search_influencers` text content already includes the env-correct dashboard URL (`{base}/influencers-board/{searchId}`). Share that URL verbatim when relevant — do not hardcode the base. See the "Dashboard URLs" section in `SKILL.md`.

---

## Selection-Reason Columns (Excel)

When exporting a selected set to Excel, always include the **two columns** below.
Keep them as **separate cells** — do not merge into one.

### Selection conditions column

List the campaign-required filter conditions that this influencer actually satisfies.

```
email✓, ShopMy✓, followers 10K–100K✓
```

- The conditions vary per campaign (email, shopping link, TikTok link, follower range, etc.)
- Build this string dynamically from `linkPlatforms`, `linkTypes`, `followed_by`
- List only satisfied conditions; do not include unmet ones

### Collaboration strength column

Synthesize **both** `note_for_brand_collaborate_point` (AI strength analysis) and `collaborate_brand` (actual collab history) into a **single sentence (20–40 characters)** tailored to this campaign's context.

Role of each field:
- `note_for_brand_collaborate_point` → practical strengths (content expertise, conversion-trackable, agency-represented, etc.)
- `collaborate_brand` → actual brand collaboration history. If similar-category brands exist, highlight them.

Key judgment principles:
- Combine both fields and extract only the 1–2 highest-impact points
- Weigh how well the content expertise matches this campaign's category
- Note whether purchase tracking is feasible (shopping link, shopltk, shopmy, etc.)
- If there is similar-brand collab history, mention it

**Good examples:**
```
Top-tier skincare content, ShopMy purchase-tracking available
Multiple K-beauty collabs (IUNIK, Anua), trusted ingredient analysis
Agency-represented (komi.group), real purchases via shopltk
Shiseido & L'Oréal collab history, beauty/skincare review specialist
```

**Bad example (do NOT do this):**
```
Consistently produces high-quality product photography for various categories
← Never copy-paste the note_for_brand_collaborate_point verbatim. Too long and generic.
```

If both fields are empty, author the sentence from `tier` plus performance tags (`engagement_rate_tag` / `viral_reach_rate_tag`) — e.g., "Mid-tier creator with above-avg engagement, no public collab history yet."

### Selection-reason narration in chat

Only expand the selection reasoning into prose in the chat when **the user explicitly asks**.
Examples: "Why did you pick this one?", "Explain your selection rationale."
Do not include it in the default workflow-completion report.

---

## Result Reporting Format

After finishing any workflow, include the following in your report:

1. **Search summary**: the query, returned count, reference to `searchSummary`
2. **Selection criteria**: which fields / rules were used to pick or segment candidates
3. **Bucket composition**: who went into each bucket (username, followed_by)
4. **Comparative analysis** (for multi-country flows): candidate counts, follower distributions, relevance differences per country
5. **Next-step suggestions**: reveal via `bulk_get_details`, pruning unfit candidates, Excel export, etc.
