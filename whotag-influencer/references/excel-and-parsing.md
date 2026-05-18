# Excel Export + JSON Parsing

## Table of Contents

- [JSON Parsing Pattern](#json-parsing-pattern)
- [Standard Excel Column Layout](#standard-excel-column-layout)
- [Column Selection Principles](#column-selection-principles)

---

## JSON Parsing Pattern

In Claude, MCP responses are delivered as a `content` array. The first item is human-readable summary text; the second is the structured JSON payload.

If results are saved to a file under `/mnt/user-data/tool_results/`:

```python
import json

with open('/mnt/user-data/tool_results/<filename>.json') as f:
    data = json.load(f)

# Shape: data = [{"type": "text", "text": "summary..."}, {"type": "text", "text": "{json...}"}]
sc = json.loads(data[1]['text'])

# search_influencers result
influencers = sc['influencers']          # array of influencers
search_id = sc['summary']['searchId']    # id for additional pages
total = sc['summary']['totalCount']      # total result count

# Keep only influencers with an email link
email_list = [i for i in influencers if 'email' in (i.get('linkPlatforms') or [])]
user_ids = [i['user_id'] for i in email_list]
```

---

## Standard Excel Column Layout

### Basic information

| Column | Field | Notes |
|--------|-------|-------|
| No. | — | Row number |
| username | `username` | Instagram handle when platform="insta"; TikTok handle when platform="tiktok" (after reveal) |
| Followers | `followed_by` | |
| Following | `follows` | |
| Media count | `media_count` | |
| Engagement rate (%) | `engagement_rate` | 0–100 |
| Viral reach rate (%) | `viral_reach_rate` | 0–100 |
| Follower growth rate (%) | `follower_growth_rate` | −100–100 |
| Avg comments | `avg_comments` | |
| Avg likes | `avg_likes` | |
| Avg reel views | `avg_reel_views` | |
| Age range | `age_range` | Estimated, in 5-year bins |
| Gender | `gender` | Male / Female / Unknown |
| Country | `country` | ISO code array |
| Language | `language` | ISO code array |
| Collaboration brands | `collaborate_brand` | Explicit commercial collaborations |
| Tagged brands | `tag_brand` | Brands mentioned in hashtags / tags |
| Content description | `description` | |

### Channel URLs (extracted from `links[]` after reveal)

| Column | Content |
|--------|---------|
| Instagram URL | Construct `https://instagram.com/{username}` directly when platform="insta". Otherwise extract from `links[]`. |
| TikTok URL | Construct `https://www.tiktok.com/@{username}` directly when platform="tiktok". Otherwise extract from `links[]`. |
| YouTube URL | Extract the YouTube entry from `links[]` |
| Blog URL | Extract the blog entry from `links[]` |

### Contact (extracted from `links[]` after reveal)

| Column | Content |
|--------|---------|
| Email | Extract the email entry from `links[]` |
| KakaoTalk | Extract the kakaotalk entry from `links[]` |

### Shopping links (add when relevant)

| Column | Content |
|--------|---------|
| Shopping links | Extract shopping-type entries from `links[]` (shopee / coupang / amazon, etc.) |

### Selection reason (always included)

Always include the two columns below as **separate cells** in the Excel output. See the "Selection-reason columns" section of `classification-and-reporting.md` for authoring guidance.

| Column | Content |
|--------|---------|
| Selection conditions | Campaign filter conditions this influencer satisfies. Example: `email✓, ShopMy✓, followers 10K–100K✓` |
| Collaboration strength | One-sentence summary tuned to the campaign context, synthesized from both `note_for_brand_collaborate_point` (AI strength analysis) and `collaborate_brand` (actual collab history). 20–40 characters. See `classification-and-reporting.md` for the authoring rubric. |

### Other

| Column | Content |
|--------|---------|
| Notes | Free-form memo column |

---

## Column Selection Principles

- Include only what the user requested (e.g., if they only asked for email, keep only the contact section)
- Leave the cell blank when an influencer has no such link after reveal
- By default, omit channels the user did not mention (e.g., blog, kakaotalk)
- For detailed Excel formatting, see the **xlsx skill**

### Available `linkPlatforms` values

- **SNS**: instagram, youtube, blog, facebook, linkedin, twitter, tiktok, snapchat, threads
- **Contact**: email, kakaotalk, whatsapp, telegram
- **Shopping**: shopee, amazon, shopltk, rakuten, sephora, coupang, shopmy

### Available `linkTypes` values

`sns`, `contact`, `shopping`
