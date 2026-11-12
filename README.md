# Review Sentiment Analyzer

[![Run on Apify](https://img.shields.io/badge/Run%20on-Apify-00a76f?style=for-the-badge)](https://apify.com/jobdata_labs/review-sentiment-analyzer)

> **Hosted version:** run it in the cloud, no install, pay per result: **[Review Sentiment Analyzer: Bulk Sentiment & Topic Tags](https://apify.com/jobdata_labs/review-sentiment-analyzer)**

Add a sentiment score, a sentiment label, topics and defect flags to reviews **you already have**. Paste them in, or point the Actor at any Apify dataset (for example the output of a review scraper you ran), and get one enriched row per review plus an aggregate summary.

This Actor does not scrape any website. It only processes the data you give it.

## What it does

- **Sentiment score and label** for every review: a score from -1.0 to +1.0 and `POSITIVE`, `NEUTRAL` or `NEGATIVE`.
- **Topic tags**: Quality & Build, Price & Value, Shipping & Delivery, Durability & Defects, Customer Service, Usability.
- **Defect flag** when the reviewer reports a broken or non-working product.
- **Rating/text mismatch flag** when stars and wording strongly disagree (5 stars with a complaint), which is useful for spotting mis-rated or suspicious reviews.
- **Summary report** with the sentiment breakdown, average score, average rating and sentiment per topic.
- **Works with any field names**: common names are auto-detected, or you can map them yourself.
- **Five languages**: English, Italian, Spanish, German and French.

## How the analysis works

The engine is rule-based, not a large language model. It counts positive and negative terms from built-in word lists, handles simple negations ("not good", "doesn't work"), and blends the result with the star rating when one is present (60% rating, 40% text). The `sentimentBasis` field tells you which signals were used for each review.

What that means in practice:

- Results are deterministic, fast and cheap: thousands of reviews take seconds.
- Sarcasm, irony and subtle wording are not understood. Reviews with no rating and no known sentiment words come back as `NEUTRAL` with `sentimentBasis: "none"`.
- Languages other than the five listed are scored from the rating only.

If you need nuance on every single review, use an LLM-based tool. If you need a fast, consistent first pass over a large volume, this fits.

## Input

| Field | Type | Description |
| :--- | :--- | :--- |
| `reviews` | Array | Reviews as plain strings or objects. Optional if `datasetId` is set. |
| `datasetId` | String | ID or name of an Apify dataset in your account. Read only. |
| `textField` | String | Field with the review text. Dot paths (`review.body`) work. Auto-detected if empty. |
| `titleField` | String | Field with the review title. Auto-detected if empty. |
| `ratingField` | String | Field with the rating. Numbers and strings like `4.0 out of 5 stars` work. Auto-detected if empty. |
| `idField` | String | Field with the review ID, copied to the output. Auto-detected if empty. |
| `ratingScale` | Integer | Highest possible rating: `5` for stars, `10` for a 1-10 scale. Default `5`. |
| `maxItems` | Integer | Stop after this many reviews. `0` means no limit. Default `1000`. |
| `keepFields` | Array | Extra fields to copy unchanged to each result, for example `productId`. |

Auto-detected field names: text (`text`, `reviewText`, `review`, `body`, `reviewBody`, `content`, `comment`, `message`, `snippet`, `description`), title (`title`, `reviewTitle`, `headline`, `subject`), rating (`rating`, `stars`, `ratingScore`, `score`, `starRating`, `reviewRating`, `ratingValue`), ID (`id`, `reviewId`, `review_id`, `uuid`).

### Example: inline reviews

```json
{
  "reviews": [
    { "id": "r1", "title": "Great value", "text": "Excellent build quality, arrived early.", "rating": 5 },
    { "id": "r2", "text": "Broken after three weeks. Refund refused.", "rating": 1 },
    "Not good, the packaging was damaged."
  ]
}
```

### Example: a dataset from another Actor

```json
{
  "datasetId": "YOUR_DATASET_ID",
  "textField": "reviewText",
  "ratingField": "stars",
  "keepFields": ["productId", "date"],
  "maxItems": 5000
}
```

## Output

One dataset row per analyzed review:

```json
{
  "reviewId": "r2",
  "reviewTitle": "Stopped working",
  "reviewText": "Broken after three weeks. Customer service refused a refund. Waste of money.",
  "rating": 1,
  "sentimentScore": -1,
  "sentimentLabel": "NEGATIVE",
  "sentimentBasis": "rating+text",
  "positiveTerms": 0,
  "negativeTerms": 3,
  "detectedTopics": ["Price & Value", "Durability & Defects", "Customer Service"],
  "isDefectReported": true,
  "ratingTextMismatch": false,
  "source": "input"
}
```

The aggregate report is saved in the run's key-value store under the key `SUMMARY`, and is also printed in the log:

```json
{
  "reviewsAnalyzed": 5,
  "averageSentimentScore": 0.06,
  "averageRating": 3.2,
  "sentimentBreakdown": {
    "positive": { "count": 2, "percent": 40 },
    "neutral": { "count": 1, "percent": 20 },
    "negative": { "count": 2, "percent": 40 }
  },
  "topics": [
    { "topic": "Shipping & Delivery", "mentions": 3, "positive": 2, "neutral": 0, "negative": 1 }
  ],
  "defectsReported": { "count": 1, "percent": 20 }
}
```

## Privacy

Only the review ID, title, text and rating are copied to the output. Reviewer names, profile links and every other field in your source data are left out unless you list them in `keepFields`.

## Pricing

You pay per analyzed review; the current rate is shown on the Actor's pricing tab. Rows skipped because they have neither text nor rating are not charged. Use `maxItems` to cap the cost of a run.

## Common uses

- Triage the output of a review scraper: sort by `sentimentScore`, filter `isDefectReported`.
- Track sentiment per topic over time by running the same dataset mapping on a schedule.
- Pre-filter a large review set before sending only the negative or mismatched ones to an LLM.
- Feed alerts in Make, Zapier or n8n when a run finds new negative reviews.

## FAQ

**Does it collect reviews from Amazon, Google or other sites?**
No. It analyzes data you provide. Collect reviews with a source you are allowed to use, then run this Actor on the result.

**Can I read a dataset from another account?**
Only datasets your account can access. The Actor gets read-only access to the one you select.

**My fields have unusual names. What do I do?**
Set `textField`, `ratingField`, `titleField` and `idField`. Nested fields use dot paths, for example `review.rating.value`.

**Why is a review NEUTRAL when it reads as negative?**
The wording was probably outside the built-in word lists and there was no rating. Open an issue with the example and the lists can be extended.

---

## Run it via API

```bash
curl -X POST "https://api.apify.com/v2/acts/jobdata_labs~review-sentiment-analyzer/run-sync-get-dataset-items" \
  -H "Authorization: Bearer $APIFY_TOKEN" -H "Content-Type: application/json" \
  -d '{"reviews": [{"id": "r1", "title": "Great value", "text": "Excellent build quality and it arrived two days early. Easy to set up, highly recommend.", "rating": 5}, {"id": "r2", "title": "Stopped working", "text": "Broken after three weeks. Customer service refused a refund. Waste of money.", "rating": 1}, {"id": "r3", "text": "It is ok for the price, nothing special.", "rating": 3}, {"id": "r4", "title": "Ottimo prodotto", "text": "Materiali solidi e spedizione veloce, lo consiglio.", "rating": 5}, {"id": "r5", "text": "Not good. The packaging was damaged and the seller was slow to reply.", "rating": 2}]}'
```

```js
import { ApifyClient } from 'apify-client';
const client = new ApifyClient({ token: process.env.APIFY_TOKEN });
const run = await client.actor('jobdata_labs/review-sentiment-analyzer').call({
    "reviews": [
        {
            "id": "r1",
            "title": "Great value",
            "text": "Excellent build quality and it arrived two days early. Easy to set up, highly recommend.",
            "rating": 5
        },
        {
            "id": "r2",
            "title": "Stopped working",
            "text": "Broken after three weeks. Customer service refused a refund. Waste of money.",
            "rating": 1
        },
        {
            "id": "r3",
            "text": "It is ok for the price, nothing special.",
            "rating": 3
        },
        {
            "id": "r4",
            "title": "Ottimo prodotto",
            "text": "Materiali solidi e spedizione veloce, lo consiglio.",
            "rating": 5
        },
        {
            "id": "r5",
            "text": "Not good. The packaging was damaged and the seller was slow to reply.",
            "rating": 2
        }
    ]
});
const { items } = await client.dataset(run.defaultDatasetId).listItems();
```

## Use it from AI agents (MCP)

```json
{
  "mcpServers": {
    "review-sentiment-analyzer": {
      "url": "https://mcp.apify.com?tools=jobdata_labs/review-sentiment-analyzer",
      "headers": { "Authorization": "Bearer <YOUR_APIFY_TOKEN>" }
    }
  }
}
```

## More tools by jobdata_labs

All tools: [apify.com/jobdata_labs](https://apify.com/jobdata_labs) · [SEO Audit Crawler](https://apify.com/jobdata_labs/seo-audit-crawler) · [Sitemap Change Monitor](https://apify.com/jobdata_labs/sitemap-change-monitor) · [RSS Feed Monitor](https://apify.com/jobdata_labs/rss-feed-monitor) · [Tech Stack & DNS Detector](https://apify.com/jobdata_labs/tech-stack-dns-detector)
