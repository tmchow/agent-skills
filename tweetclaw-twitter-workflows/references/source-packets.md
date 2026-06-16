# TweetClaw Source Packets

Use source packets when TweetClaw output will feed a report, draft, scorecard,
support note, or handoff. The packet is a review record, not a secret store.

## Required Fields

Each packet should include:

- `source`: `tweetclaw`
- `task`: `search_tweets`, `search_replies`, `scrape_tweets`, `user_lookup`,
  `follower_export`, `media_download`, `monitor_event`, `webhook_event`,
  `giveaway_draw`, `post_tweet`, `post_reply`, `media_upload`, or
  `direct_message`
- `captured_at`: ISO 8601 timestamp
- `query_or_target`: public query, handle, tweet URL, user ID, or monitor name
- `public_url`: source tweet, profile, media, result, or webhook reference when
  one exists
- `summary`: brief factual note from the result
- `limits`: missing fields, failed pages, rate limits, partial exports, or
  approval decisions

Never include raw cookies, API credentials, private session blobs, unredacted
headers, or internal runtime configuration.

## Example

```json
{
  "source": "tweetclaw",
  "task": "search_tweets",
  "captured_at": "2026-06-16T04:30:00Z",
  "query_or_target": "from:example launch",
  "public_url": "https://x.com/example/status/1234567890",
  "summary": "Launch tweet mentioning the product name and release date.",
  "limits": "Read-only search. No posting or monitor setup was requested."
}
```

## Review Rules

- Keep packets factual. Do not infer intent, sentiment, or score unless a
  downstream workflow performs that analysis.
- Label partial or failed fetches instead of hiding them.
- Keep write actions separate from read packets. A post, reply, direct message,
  media upload, monitor setup, webhook, or giveaway draw needs its own approval
  record.
- If a downstream workflow drafts content from packets, cite the packets used
  and keep the final wording reviewable before publishing.
