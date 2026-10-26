<p align="center"><a href="https://apify.com/DevilScrapes/hacker-news-scraper"><img src="assets/banner.svg" alt="Hacker News Scraper — JavaScript example" width="100%"></a></p>

# Hacker News API client for JavaScript and Node.js: ranked stories as JSON

<p align="center">
<a href="https://apify.com/DevilScrapes/hacker-news-scraper"><img alt="Run on Apify" src="https://img.shields.io/badge/Run%20on-Apify-DC2626?logo=apify&logoColor=white"></a>
<img alt="JavaScript" src="https://img.shields.io/badge/Node.js-20%2B-339933?logo=nodedotjs&logoColor=white">
<a href="LICENSE"><img alt="MIT license" src="https://img.shields.io/badge/license-MIT-0A0606"></a>
<a href="https://github.com/devilscrapes/hacker-news-scraper-javascript/actions/workflows/ci.yml"><img alt="CI" src="https://github.com/devilscrapes/hacker-news-scraper-javascript/actions/workflows/ci.yml/badge.svg"></a>
</p>

The official Hacker News feeds return bare arrays of item numbers, which means a Node app has to fan out hundreds of follow-up requests before it can show a single headline. This repository hands that job to the **[Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper)** on Apify. A single ES module wraps Apify's official JavaScript client: you name a feed and a limit, and you receive story objects with the feed position, headline, outbound link, discussion permalink, points, comment count, author handle and ISO timestamps. Use it from an Express route, a serverless cron, a Slack bot or a GitHub Actions schedule. When you run the script directly the stories are also written to `results.json` and `results.csv`. The front page, minus the procrastination.

> **Actor:** [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper) · **Publisher:** [Devil Scrapes](https://apify.com/DevilScrapes) · **Also in Python:** [hacker-news-scraper-python](https://github.com/devilscrapes/hacker-news-scraper-python)

## 🔥 What you get

- **Six feeds behind one switch**: `feed` picks the front page (`top`), the newest submissions, the best-of ranking, Ask HN, Show HN or the YC jobs board.
- **Front-page position captured**: every row carries `rank`, the story's place in the chosen feed at the moment of the run, a number the official item records do not include.
- **Feed and items stitched together**: the feed only lists IDs, so the Actor fetches each item record in parallel (`concurrency`, 1 to 32) and returns finished rows.
- **Two time formats**: the raw Unix `time` the site stores plus an ISO 8601 UTC `posted_at`, and a `scraped_at` stamp so repeated snapshots can be ordered.
- **A ready discussion link**: `permalink` points at the item's `news.ycombinator.com` thread, while `url` holds the outbound article and is `null` for text posts.
- **Self-post bodies on request**: with `includeText` on, Ask HN and Show HN rows include the post body as the site's HTML.
- **Dead and deleted items dropped**: flagged or removed stories never reach your dataset, and an item that fails to load is logged and left out instead of stopping the run.

## 🎯 Use cases

### Alert a channel when a story mentions you

Run the `new` feed every few minutes from a serverless timer, test each `title` and `url` against your product names and post matches to a Slack or Discord webhook. Developer-tool companies often learn about a launch thread this way before the comments pile up.

### Publish a daily tech digest

A morning job can take the `best` feed, keep the ten highest `score` values and render them into an email or a static page. Link both `url` and `permalink`, so readers can choose between the article and the discussion.

### Track your own Show HN launch

Poll the `show` feed while your post is live and store `rank`, `score` and `descendants` for its `id` on every run. A small chart in your admin panel then shows exactly when the post peaked and when it fell off the list.

### Feed job listings into a hiring board

The `jobs` feed lists the openings posted by Y Combinator companies. Upsert each object by `id` into your database and expose them through your API next to listings from other sources.

## ⚡ Quick start

You need an Apify account and an API token from [Console → Settings → Integrations](https://console.apify.com/settings/integrations). Apify's free plan includes monthly usage credit; this Actor's event prices are listed under Pricing below.

```bash
git clone https://github.com/devilscrapes/hacker-news-scraper-javascript.git
cd hacker-news-scraper-javascript
npm install
export APIFY_TOKEN=your_token_here
node index.js                       # uses input.json
node index.js my-input.json --out data --max-charge 2
```

The script starts the [Hacker News Scraper](https://apify.com/DevilScrapes/hacker-news-scraper), waits for it to finish, prints a link to the run in Apify Console and saves every result to `results.json` and `results.csv`. It exits with code 1 and the run's status message if the run fails.

## 🧩 The core call

This is the whole integration once argument parsing and file output are taken away. The first awaited call returns when the run has completed; the second reads its stored items.

```javascript
import { ApifyClient } from 'apify-client';

const client = new ApifyClient({ token: 'YOUR_APIFY_TOKEN' });
const run = await client.actor('DevilScrapes/hacker-news-scraper').call({"feed": "top", "maxResults": 10, "includeText": true, "concurrency": 8, "proxyConfiguration": {"useApifyProxy": false}});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
console.log(items);
```

## 📥 Input

`index.js` forwards `input.json` to the Actor unchanged. The example asks for the top ten stories of the front page with self-post text included, enough to see the object shape for a few cents. Set `feed` to another list, or lower `concurrency` if you want the run to fetch items more gently.

```json
{
  "feed": "top",
  "maxResults": 10,
  "includeText": true,
  "concurrency": 8,
  "proxyConfiguration": {
    "useApifyProxy": false
  }
}
```

| Field | Type | Required | Default | Allowed values |
|---|---|---|---|---|
| `feed` | string | no | `"top"` | `top`, `new`, `best`, `ask`, `show`, `jobs` |
| `maxResults` | integer | no | `100` | — |
| `includeText` | boolean | no | `true` | — |
| `concurrency` | integer | no | `8` | — |
| `proxyConfiguration` | object | no | `{"useApifyProxy": false}` | — |

## 📤 Output

You receive one object per story, job or poll in the requested feed. Parallel fetching means the array is not guaranteed to be in rank order, so sort on `rank` before you render a list. `text` arrives as HTML from the site; sanitise it before putting it into a page.

| Field | Type | Always present |
|---|---|---|
| `id` | integer | yes |
| `type` | string | no |
| `title` | string | yes |
| `url` | string or null | no |
| `permalink` | string | yes |
| `by` | string | yes |
| `score` | integer or null | no |
| `descendants` | integer or null | no |
| `text` | string or null | no |
| `time` | integer | yes |
| `posted_at` | string | no |
| `scraped_at` | string | no |
| `rank` | integer | no |

## 💰 Pricing

| Event | Price (USD) |
|---|---|
| Actor start | $0.2 |
| Result emitted | $0.002 |

Prices as configured on 2026-08-20. The [Actor page](https://apify.com/DevilScrapes/hacker-news-scraper) always shows the current price. Runs are billed to the Apify account that owns the token. Use `--max-charge` to cap what a single run can cost.

## 🛠️ Working with the data in JavaScript

This snippet compares the fresh results with the IDs from the previous run, keeps new stories above a points threshold and posts them to a webhook if `WEBHOOK_URL` is set. Only Node.js built-ins are used, including the global `fetch`.

```javascript
import { readFile, writeFile } from 'node:fs/promises';

const stories = JSON.parse(await readFile('results.json', 'utf8'));
const seen = new Set(JSON.parse(await readFile('seen-ids.json', 'utf8').catch(() => '[]')));
const minScore = Number(process.env.MIN_SCORE ?? 100);

const fresh = stories
    .filter((s) => !seen.has(s.id) && (s.score ?? 0) >= minScore)
    .sort((a, b) => a.rank - b.rank);

for (const story of fresh) {
    const line = `#${story.rank} ${story.title} (${story.score} points, ${story.descendants ?? 0} comments) ${story.permalink}`;
    if (process.env.WEBHOOK_URL) {
        await fetch(process.env.WEBHOOK_URL, {
            method: 'POST',
            headers: { 'content-type': 'application/json' },
            body: JSON.stringify({ text: line }),
        });
    } else {
        console.log(line);
    }
}

for (const story of stories) seen.add(story.id);
await writeFile('seen-ids.json', JSON.stringify([...seen]));
```

## ❓ FAQ

### Do I need a Hacker News account?

No. The Actor reads the same open data that Hacker News publishes for anyone, so the only credential involved is your Apify token. Nothing is posted, voted on or logged into.

### Should I turn on Apify Proxy?

Usually not. The example leaves it off, and the Actor runs fine without it. Enable it in `proxyConfiguration` only if your organisation requires all outbound traffic to go through Apify.

### How often can I poll the feeds?

As often as your budget allows, since each run pays a start fee plus a small amount per story. For mention alerts, a short run of the `new` feed every few minutes with a low `maxResults` keeps the cost small.

### Is there a TypeScript type for the rows?

Not shipped in this repo, but the Output section maps one to one onto an interface: numbers for `id`, `score`, `descendants`, `time` and `rank`, strings for the rest, with `url`, `score`, `descendants` and `text` allowed to be `null`.

## 🧯 Troubleshooting

- **`Set APIFY_TOKEN first`**: export the token in the same shell, or put it in a local `.env` file your shell loads. Never commit it.
- **Run finished with `FAILED` or `ABORTED`**: open the printed Console link; the run log says why. Input validation errors name the field.
- **Fewer rows than expected**: a `--max-charge` cap also limits how many results can be charged. Raise the cap or lower the limits in your input.
- **Something looks wrong in the data**: open an issue on the [Actor page](https://apify.com/DevilScrapes/hacker-news-scraper) so the maintainers see it.

## 🔗 More from Devil Scrapes

- [Hacker News Scraper on Apify Store](https://apify.com/DevilScrapes/hacker-news-scraper)
- [Python version of this example](https://github.com/devilscrapes/hacker-news-scraper-python)
- [All Devil Scrapes Actors](https://apify.com/DevilScrapes)
- [Apify web scraping cookbook](https://github.com/devilscrapes/apify-web-scraping-cookbook)

Related Actors:

- [DEV.to Scraper — Articles by Tag & Author](https://apify.com/DevilScrapes/dev-to-articles-scraper) — developer blog posts with tags and reactions from DEV Community.
- [Lemmy Scraper — Posts, Comments & Community Data](https://apify.com/DevilScrapes/lemmy-community-scraper) — posts and votes from federated Lemmy communities.
- [GitHub Repo Scraper](https://apify.com/DevilScrapes/github-repo-scraper) — stars, forks and topics for the repositories that Show HN posts link to.

---

MIT licensed example code. The Actor itself runs on [Apify](https://apify.com/DevilScrapes/hacker-news-scraper) and is maintained by [Devil Scrapes](https://apify.com/DevilScrapes). Not affiliated with Hacker News or Y Combinator.
