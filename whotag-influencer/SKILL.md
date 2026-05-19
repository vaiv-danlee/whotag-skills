---
name: whotag-influencer
description: >
  End-to-end workflow guide for the whotag connector: search influencers,
  extract emails/contacts, save them to buckets, and export to Excel.
  You MUST use this skill in the following situations (Korean and English triggers both apply):
  - Influencer discovery — "find influencers", "search beauty influencers", "list micro influencers",
    "인플루언서 찾아줘", "뷰티 인플루언서 검색", "마이크로 인플루언서 리스트"
  - Contact extraction — "influencers with email", "extract contacts", "influencers to DM",
    "이메일 있는 인플루언서", "연락처 추출", "DM 보낼 인플루언서"
  - whotag bucket operations — "add to bucket", "create a bucket", "list buckets",
    "버킷에 담아줘", "버킷 만들어줘", "버킷 목록"
  - Data export — "make an Excel file", "pull a list",
    "엑셀로 만들어줘", "리스트 뽑아줘"
  - Any mention of whotag or whotag.ai
  - Campaign-brief-driven influencer sourcing (seeding, UGC, ambassadors, etc. / 시딩, UGC, 앰배서더 등)
---

# whotag Influencer Workflow Skill

This skill describes **how to compose whotag tools and the judgment calls to make**.
Tool parameters and response schemas are documented on the tools themselves and are not repeated here.

## Available Tools (11)

| Category            | Tool                     | Core purpose                                                                                |
| ------------------- | ------------------------ | ------------------------------------------------------------------------------------------- |
| **Search**          | `search_influencers`     | Natural-language search (up to 25 per page, platform selectable)                            |
|                     | `filter_search_results`  | Paginate, re-sort, or refine prior search results with server-side filters (no credit cost) |
| **Detail / Reveal** | `get_details`            | Reveal one influencer's username + links (consumes 1 credit)                                |
|                     | `bulk_get_details`       | Reveal multiple influencers at once                                                         |
| **Buckets**         | `list_buckets`           | List buckets                                                                                |
|                     | `create_bucket`          | Create a new bucket                                                                         |
|                     | `delete_bucket`          | Delete a bucket                                                                             |
|                     | `get_bucket_influencers` | View influencers in a bucket — one `platform` per call, ask the user which before calling   |
|                     | `add_to_bucket`          | Add a single influencer to a bucket                                                         |
|                     | `bulk_add_to_bucket`     | Add many influencers to a bucket at once                                                    |
|                     | `remove_from_bucket`     | Remove an influencer from a bucket                                                          |

---

## Campaign Workflow Patterns

The patterns below are representative. Real requests are often combinations or variations of these —
pick the closest pattern and adapt.

### Pattern A: Single search + single bucket (baseline)

> "Find dog lifestyle creators in Korea and add them to a bucket."

```
1. search_influencers(query, platform="insta"|"tiktok") → analyze results
2. create_bucket(name) → bucket_id
3. bulk_add_to_bucket(bucket_id, selected user_ids)
4. Report results
```

### Pattern B: Single search + purpose-based segmentation + multiple buckets

> "Split into seeding candidates and UGC shooting candidates, one bucket each."

When the user asks for classification by **purpose**, use this pattern.
The segmentation criteria are derived from the profile data by the LLM.

```
1. search_influencers(query) → analyze results
2. Post-segmentation: classify into groups (see "Post-processing classification guide")
3. create_bucket(name) × N (one per purpose)
4. bulk_add_to_bucket × N (one call per group)
5. Summarize segmentation criteria and per-bucket composition
```

### Pattern C: Multi-country / multi-platform parallel search + separate buckets

> "Find K-beauty creators in Thailand and Vietnam separately."

Country or platform comparisons require **separate searches**.

```
1. search_influencers(query_country1) + search_influencers(query_country2) + ...
2. Analyze per-country/per-platform and compare
3. create_bucket × N (per country or platform)
4. bulk_add_to_bucket × N
5. Summarize cross-country / cross-platform differences
```

**Note:** do not merge multiple countries into a single query when comparing.
When comparison is NOT the intent, merging is fine: "beauty influencers in Vietnam or Malaysia" → 1 call.

### Pattern D: Refine prior search (paginate / re-sort / filter)

> "Show more candidates", "sort by engagement", "narrow to female creators with email"

`filter_search_results` operates on a prior `searchId` and supports three refinement modes
(combinable in a single call). All three are credit-free.

| Mode         | Use when                                                                                                                                                                               | How                               |
| ------------ | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | --------------------------------- |
| **Paginate** | `hasMore=true` and user wants more candidates                                                                                                                                          | bump `page`                       |
| **Re-sort**  | user wants a different ranking (engagement, follower growth, ad price, etc.)                                                                                                           | set `sort_by`, reset `page=1`     |
| **Filter**   | candidate pool is too broad — narrow by gender / region / country / language / age / channel / follower range / keyword / collab intent / curated badge / `exclude_bucket_influencers` | set filter params, reset `page=1` |

```
1. search_influencers(query) → check searchId, hasMore
2. Refine via filter_search_results as needed (paginate / re-sort / filter — combinable)
3. Save all selected candidates across the refinement passes into one bucket
```

**Page=1 rule:** any change to `sort_by` or any filter invalidates the prior page index — always reset `page=1` when refining. Subsequent pagination under the new query proceeds from `page=2`.

**Automatic pagination:** if the user said up front "fetch more if there are many",
stop at page 1 if it's enough; otherwise auto-fetch page 2.

**Filter refinement vs. query expansion:** when the original query intent is right but results are too noisy, prefer `filter_search_results` filters over re-querying — same `searchId`, no credit cost. Reach for Pattern H (query expansion) only when the candidate pool itself is too small.

**Worked filter examples:**

```
# Korean female micro-influencers with email
filter_search_results(searchId, page=1,
  genders=["female"], countries=["KR"],
  follower_ranges=["10K-50K"], channels=["email"])

# Top up — exclude influencers already saved in the user's buckets
filter_search_results(searchId, page=1, exclude_bucket_influencers=true)

# Refine by bio/interest keyword (matches across interests, keywords,
# brands, biography, hashtags — combined with other filters via AND)
filter_search_results(searchId, page=1, keyword="skincare")

# Surface curated high-potential picks (flags combine via AND — an
# influencer must satisfy every selected flag)
filter_search_results(searchId, page=1,
  filter_flags=["high_collaboration_potential", "high_recent_growth"])
```

### Pattern E: Reveal + email/contact extraction + Excel

> "Find 20 influencers with email and export to Excel."

**Core rule: reveal is a paid action that consumes credits, so do not auto-call it. Present Options A and B to the user and proceed only after explicit consent.**

If `username` is masked (contains `*`) or `links[].urls` is `null`, you need `bulk_get_details` to obtain the actual values — but **always ask the user first** before calling it.

```
1. At query-design time, combine orthogonal condition axes (region / category /
   content style / follower / contact / shopping link 등). **Use at most one
   expression per axis** — same-axis synonyms, sub-categories, or brand enums
   ("ShopMy, LTK, Amazon, Sephora") narrow the match and worsen results.
   **Specificity mirrors the user's ask**: if the user named Amazon, write
   "Amazon 링크"; if they said "쇼핑 링크" generically, keep it generic — don't
   expand to a brand list and don't generalize a specific brand away.
2. search_influencers(query, platform)
   └─ Server-side narrowing (preferred when contact is required): `filter_search_results(searchId, channels=["email"], page=1)` — yields more contactable candidates per page than client-side filtering. Same shape works for "tiktok", "youtube", "shopmy", etc.
   └─ Otherwise filter client-side with linkPlatforms/linkTypes (no credits consumed)
   └─ If target count N is not met → use Pattern H (similar / expanded queries)
3. Count the filtered candidates N (expected reveal cost = N credits)
4. Present two options to the user:
   A. Reveal now — expected cost {N} credits (remaining: {remainingViewCredits})
   B. Save to a bucket — the user opens the whotag.ai dashboard to inspect and selectively reveal per influencer (Claude cannot render the widget, unlike ChatGPT)
5. After the user's choice:
   - Option A → bulk_get_details → extract real values → (optional) save to bucket → Excel
   - Option B → create_bucket → bulk_add_to_bucket → share the search-board URL from the `search_influencers` text (see "Dashboard URLs" below)
```

**Auto-reveal exception (consent already given):**

- If the user has already said "go ahead and reveal", "export with emails included", "show it unmasked", etc.
  in the current or preceding turn, proceed with `bulk_get_details` directly.
- Even then, report the actual credit consumption in the final answer.

**Patterns to avoid:**

- ❌ Silently calling `bulk_get_details` in bulk without consent (unapproved credit spend)
- ❌ Finalizing with "email is hidden, can't deliver" without offering Options A/B
- ❌ Skipping influencers whose `linkPlatforms` includes "email" without surfacing them in the options
- ✅ Stating "email is masked" is fine as a factual description — just follow it with Options A/B

**Reveal-result persistence pattern (required):**
Immediately persist `bulk_get_details` results to a JSON file.
Reconstructing reveal data from chat history is **strictly forbidden**.

```python
import json
# Immediately after bulk_get_details, write results to disk
with open('/home/claude/reveal_results.json', 'w') as f:
    json.dump(reveal_results, f, ensure_ascii=False)
# When generating the Excel, load from the file
with open('/home/claude/reveal_results.json') as f:
    reveal_data = json.load(f)
```

**Filter before reveal (credit savings):**

- Email request → reveal only user_ids where `linkPlatforms.includes("email")`
- YouTube request → `linkPlatforms.includes("youtube")`
- Any contact → `linkTypes.includes("contact")`
- Multiple conditions → AND intersection

### Pattern F: Similar-account search

> "Find creators in the style of @miyu_official."

```
1. search_influencers("creators similar to @username in [country] [category]")
2. (Optional) run extra searches to broaden the reference
3. Merge results, dedupe by user_id
4. Save to bucket
```

### Pattern G: Bulk collection workflow

> "Find 50 influencers and export to Excel."

**When the user has not specified scale:**
Confirm the target count first. If you build for a small batch and the user then says "I need more",
you'll end up redoing the whole flow.

```
1. (If unspecified) ask the user for the target count
2. Use search + pagination (Pattern D) to collect the full candidate pool to match the target
3. Run filtering + reveal (Pattern E) across the full pool in one pass
4. Generate the Excel for the full dataset in one pass
```

### Pattern H: Query expansion when results are insufficient

> "You found 20 candidates matching the conditions, but I only see 10."

Blindly paginating the same query yields diminishing relevance.
Use **similar query → expanded query → pagination** in that order.

```
1. Verify with filtering: count the candidates N in the current results that actually satisfy the ask
2. If N < target:
   (a) Similar query — same intent, different wording / adjacent sub-category
       "fitness" → "health, workout, personal training, home workout, pilates"
       "beauty" → "skincare, makeup, K-beauty"
   (b) Expanded query — broaden to adjacent domains
       "fitness" → "wellness, active lifestyle, diet"
       "beauty" → "lifestyle + beauty routine"
   (c) Pagination — last resort, once the variations are exhausted
3. Dedupe the merged results by user_id, then re-apply the filter
4. For the reveal step, fall back to Pattern E's Option A/B presentation
```

**Why this order:**

- Similar and expanded queries retrieve NEW high-relevance candidates that page 1 missed.
- Paginating an exhausted query keeps pulling lower-relevance matches → inflates context noise.
- Actually varying the query is more credit- and context-efficient than many paginations over a weak initial query.

---

### Pattern combinations

Real requests combine patterns:

- "3 countries × email extraction × Excel" → C + E
- "Search + purpose segmentation + more" → B + D
- "50-influencer email extraction + Excel" → G + D + E
- "Instagram vs TikTok comparison + Excel" → C (per platform) + E
- "Not enough matching candidates → top up + extract" → H + E

---

## Platform Parameter

The `platform` parameter of `search_influencers`:

| Value               | Meaning                                  |
| ------------------- | ---------------------------------------- |
| `"insta"` (default) | Search the Instagram influencer database |
| `"tiktok"`          | Search the TikTok influencer database    |

**⚠️ Do not confuse `platform` with `linkPlatforms`:**

- `platform`: the database being searched
- `linkPlatforms`: the list of external-link platforms in an influencer's bio

**Cross-platform request:**

```
search_influencers(query, platform="insta")
→ filter results where linkPlatforms includes "tiktok"
= Instagram influencers who also have a TikTok link
```

**Platform comparison:** call separately, then compare the two result sets.

**`get_bucket_influencers` follows the same per-platform rule** — each call returns only the requested platform's saved influencers. Unlike `search_influencers` (where `"insta"` is a reasonable default when the user didn't specify), `get_bucket_influencers` has **no default**: ask the user which platform of the bucket to view before calling, and call twice if they want both.

---

## Credit Management

- `remainingSearchCount`: remaining search credits (debited by `search_influencers`)
- `remainingViewCredits`: remaining view credits (debited by `bulk_get_details` / `get_details`)

**Saving principles:**

- Only reveal influencers who actually need it (pre-filter with linkPlatforms / linkTypes)
- **Re-revealing an already-revealed influencer still costs credits** → persist results and never call twice
- Use `filter_search_results` for additional pages (no credit cost)
- `bulk_get_details` stops mid-run if credits are exhausted

---

## Dashboard URLs

The whotag MCP server runs in two environments — `whotag.ai` (prod) and `dev.whotag.ai` (dev) — so do not hardcode a base URL.

- **Search-results board:** the `search_influencers` text content already contains the env-correct URL (`View full results: {base}/influencers-board/{searchId}`). Surface that URL to the user verbatim instead of constructing one. The same `searchId` is reused by `filter_search_results`, so the URL stays valid across pagination / re-sort / refine.
- **Bucket pages:** there is no per-bucket deep-link exposed to the tool layer. If the user wants to inspect a bucket in the web UI, share the search-board URL (above) or send them to the whotag dashboard root — do **not** fabricate `/buckets/{bucket_id}` or similar URLs.

---

## Bucket Naming Convention

If the user does not specify a bucket name:

```
{country}_{category}_{purpose}
e.g., US_SensitiveSkin_Seeding, TH_KBeauty_Candidates, US_Fitness_BrandCollab
```

Match the user's language (Korean request → Korean name, English request → English name).

---

## Cautions

- When email / username / external links are masked, do not auto-reveal — present Options A/B and wait for consent (Pattern E)
- Actively include accessible conditions (email, links, shopping) in the query — do not rely on post-filtering alone
- When results are insufficient, try similar / expanded queries before paginating (Pattern H)
- After changing `sort_by` or any `filter_search_results` filter, reset `page=1` — the prior page index is invalid under a new query
- For comparisons, always use separate calls per country / platform
- `platform` and `linkPlatforms` are different concepts
- Bucket names must be unique (duplicates error out)
- When merging multiple search results, dedupe by user_id
- Do not inject conditions the user did not ask for into the query

---

## ⚠️ Verification After Agent Delegation (required)

When you delegate selection work to a sub-agent, do not trust its success message alone — always verify
condition compliance yourself. There have been real cases where an agent ignored email / shopping-link /
TikTok constraints and filled the quota with non-matching influencers. If the required conditions were
violated and a bucket or Excel is produced on top of that, the entire campaign is affected.

### How to verify

After selection, aggregate per-condition compliance from the result data:

```python
total        = len(influencers)
email_ok     = sum(1 for i in influencers if 'email'    in i.get('linkPlatforms', []))
shopping_ok  = sum(1 for i in influencers if 'shopping' in i.get('linkTypes', []))
tiktok_ok    = sum(1 for i in influencers if 'tiktok'   in i.get('linkPlatforms', []))

print(f"email     {email_ok}/{total}")
print(f"shopping  {shopping_ok}/{total}")
print(f"TikTok    {tiktok_ok}/{total}")
```

### When compliance is below target

1. **Top up via pagination first** — call `filter_search_results` for the next page to gather more
   candidates that satisfy the conditions. No credit cost for additional pages.
2. **If still short, report to the user** with these options:
   - A. Reduce the quota (e.g., 10 → the actually-compliant count)
   - B. Downgrade the requirement from "required" to "preferred"

Do not unilaterally loosen conditions without user confirmation.

---

## References

- **Post-processing classification + result reporting**: [references/classification-and-reporting.md](references/classification-and-reporting.md) — field mappings for selection criteria, examples of purpose-based segmentation, **how to write the selection-reason columns**, result-report format
- **Excel export + JSON parsing**: [references/excel-and-parsing.md](references/excel-and-parsing.md) — standard Excel columns (including selection-reason and collaboration-strength columns), MCP JSON response parsing patterns
