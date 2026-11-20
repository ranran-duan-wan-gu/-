# Reddit Demand Radar (n8n)

A read-only n8n workflow that mines a subreddit once a day for **manual, repetitive work people complain about**, extracts the complaints into structured fields with an LLM, groups them into themes, and emails you a summary.

It does not post, comment, vote, or log in. It reads a public JSON endpoint and sends you an email.

The point is not lead generation. It is finding out what people are already complaining about — before you build something nobody wants.

```
Schedule Trigger (09:01)
  → HTTP Request          arctic-shift /posts/search
  → Split Out             fieldToSplitOut: "data"
  → Pre-filter (Code)     drop removed / promo / zero-comment / short posts, trim to 7 fields
  → Limit                 max 8  (each item = 1 LLM call)
  → Basic LLM Chain       structured extraction, batchSize 4
      ↳ OpenAI Chat Model json_object, temperature 0.2
  → Build Report (Code)   group by theme, score, build HTML
  → Send an Email         subject + html
```

---

## What it extracts

For each post, the LLM returns these fields:

| Field | Meaning |
|---|---|
| `pain_theme` | 3–5 word name of the task, e.g. `invoice chasing` |
| `pain_quote` | verbatim snippet that carries the complaint |
| `who` | who is complaining, e.g. `small_business_owner` |
| `current_workaround` | how they cope today, e.g. `spreadsheet` |
| `product_idea` | one line: what could be built for this |
| `manual_hours` | hours per week, if stated |
| `demand_score` | 1–10, how badly they want it solved |
| `buildable` | 1–5, how easy it is to ship in n8n |

Then `Build Report` groups by `pain_theme`. Themes scoring below `THEME_MIN_SCORE = 3` are dropped. The rest are sorted by `comments + score * 4` (comment count is weighted high because it is the only crowd-validated signal available) and capped at 6.

The HTML email is built entirely inside that Code node and passed to the email node as `={{ $json.html }}`.

---

## Setup

1. Import `demand-radar-v3.json` into n8n.
2. **Open `OpenAI Chat Model` and pick a model.** The `model` field is intentionally empty in this file — no credentials or model selection are published.
3. Pick your OpenAI-compatible credential on the same node. Leave `responsesApiEnabled` at `false` unless your provider supports the Responses API; DeepSeek and most gateways do not.
4. Open `Send an Email`, set `fromEmail` / `toEmail` (both are `you@example.com` here), and select your SMTP credential.
5. Enable the workflow. It runs once a day at 09:01 in the workflow timezone (`Asia/Shanghai` in this file — change it in workflow settings).

Before enabling, run it once manually with **Execute Workflow** so you can see the item counts at each node.

---

## The query rotation

`arctic-shift` rate-limits hard. Four concurrent requests reliably returned `422 Timeout. Maybe slow down a bit` in testing, so this workflow makes **one request per run** and rotates the subreddit + query by day of week:

| Day | subreddit | query |
|---|---|---|
| Mon | smallbusiness | `still do manually` |
| Tue | smallbusiness | `chasing invoices` |
| Wed | smallbusiness | `hours every week` |
| Thu | smallbusiness | `data entry` |
| Fri | agency | `repetitive task` |
| Sat | smallbusiness | `still do manually` |
| Sun | smallbusiness | `chasing invoices` |

Measured relevance (share of returned posts whose title or body actually contained a manual-work signal):

| Query | Relevant |
|---|---|
| `still do manually` | 7/8 |
| `chasing invoices` | 5/5 |
| `data entry` | 7/9 |
| `hours every week` | 5/10 |

Single-word queries (`spreadsheet`, `manual`) pull in too much noise to be useful. Overly specific phrases (`by hand every`) tended to time out.

Both arrays are inline in the HTTP node's URL, indexed by `$now.weekday - 1`. Edit them there.

---

## Six things that cost me time

### 1. Luxon's `toSeconds()` returns a float, and some APIs reject it

Passing `after={{ $now.minus({days: 7}).toSeconds() }}` produced:

```
400 {"error":"'after' must be a valid date (epoch, ISO, or relative)"}
```

The value being sent was `1794377292.123`. The API wants integer seconds. `.0` fails too, so it is not about the fraction being non-zero.

Verified matrix against this endpoint:

| Sent | Result |
|---|---|
| `1794377292` | 200 |
| `1794377292.0` | 400 |
| `1794377292.123` | 400 |
| `2026-11-11` | 200 |
| `7d` | 200 |

`Math.floor()` fixes it. This workflow uses the relative form (`after=30d`) because it needs no arithmetic at all.

### 2. n8n expressions do not support `{% if %}`

I wrote a conditional block in an email template and it rendered literally. The expression engine handles `{{ }}` only. Any branch — "show the table when there are results, show a placeholder when there are none" — has to live in a Code node that returns a finished string.

That is why this workflow builds the entire HTML in `Build Report` and the email node is just `={{ $json.html }}`. Less elegant than a template, but it is testable outside n8n and the empty-result path is explicit.

### 3. A Chain node and an Agent node unwrap JSON differently

This one cost the most time.

- **Chain**, `typeVersion >= 1.6`, with `responseFormat: json_object` → the object is unwrapped to the top level. It is `$json.myField`.
- **Agent** with an output parser attached → it is `$json.output.myField`.

Same-looking LLM output, two different shapes. A sub-workflow that serves both needs `$json.output ?? $json`.

I found this by reading the source, not the docs.

### 4. `batching.batchSize` is parallel, not serial

A batch size of 4 does **not** mean "four at a time, waiting for each to finish". It is `Promise.allSettled`, so the batch waits for its slowest item and later batches can start while earlier ones are still pending.

Practical consequence: batch size gives you no protection against an API that throttles on concurrency. I confirmed it by firing four requests at an endpoint that returns `422` on overload — two of the four failed. One request per run with a long retry interval was the only thing that held.

### 5. "No error" does not mean "it worked"

The filter node uses `continue` to drop items. When I pointed a downstream node at a field name that did not exist, the next node received zero items and the workflow finished **green**. Nothing in the execution list looked wrong.

If a run can legitimately produce nothing, that path needs its own output. `Build Report` emits an explicit "no results this round" email for exactly this reason — otherwise you cannot tell a quiet day from a broken field name.

### 6. `query` needs `subreddit` or `author` on this API

There is no cross-Reddit full-text search here:

```
GET /api/posts/search?query=invoice+chasing
→ 400 "'query' query parameter requires one of: author, subreddit"
```

Multi-subreddit also does not work. `subreddit=a,b` and `a+b` both return `200` with `n=0`, which looks like "no posts" rather than an error. One subreddit per request.

---

## Tested state

What I actually ran:

- `Pre-filter` against 7 synthetic cases — removed posts, promo flair, zero-comment, short body and soft-promo were all dropped; the real complaint with a long body was kept; output ordered by comment count. One regex bug was caught this way: `/\[removed\]/i` missed the real title `[Removed by moderator]` and was changed to `/\[\s*(removed|deleted)/i`.
- `Build Report` against three scenarios: normal input (themes merged case-insensitively, e.g. `Invoice Chasing` + `invoice chasing`), all items empty/none, and zero input items. All three produce a valid email, and the zero-input case reaches the empty-state branch without throwing.
- One live end-to-end run against `r/smallbusiness` with the query `spreadsheet`: 8 posts fetched (after two `422` retries), 6 kept by the filter, LLM output mocked to reach `Build Report`, HTML rendered with 2 theme rows.

What is **not** tested:

- A second subreddit or a second LLM provider.
- The `agency` + `repetitive task` rotation slot has returned `0` posts more than once. That is a thin query, not necessarily a bug.
- No load or long-run testing.

---

## Known limits

- **Post body only.** The LLM never sees comments. A post with 40 comments often carries the real signal in the replies, and this pipeline misses it.
- **Comment count as demand proxy.** The API has no "sort by score" option, and newest posts have a median score of 1. Comment count is the only crowd signal available, so a post with 0 comments is dropped even if the complaint is genuine.
- **Theme grouping is string matching.** `invoice chasing` and `chasing invoices` become two themes. Normalising the theme to a controlled vocabulary in the prompt would fix this.
- **One subreddit per run.** See gotcha 6.

---

## License

MIT — see `LICENSE`.

arctic-shift is a third-party public archive of Reddit data. Check its terms before using it at higher volume.
