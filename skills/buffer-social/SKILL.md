---
name: buffer-social
description: "Post to social media via Buffer's GraphQL API. Use when scheduling or publishing to LinkedIn, Twitter/X, Facebook, or Instagram through Buffer. Requires a Buffer access token."
homepage: https://buffer.com
metadata:
  {
    "openclaw":
      {
        "emoji": "📣",
        "requires": { "env": ["BUFFER_ACCESS_TOKEN"] },
      },
  }
allowed-tools: ["exec"]
---

# Buffer Social Publishing

Publish and schedule social media posts via Buffer's GraphQL API. Buffer manages multi-platform posting to LinkedIn, Twitter/X, Facebook, Instagram, and more from a single API.

## Prerequisites

1. Buffer account with connected social channels.
2. `BUFFER_ACCESS_TOKEN` set in environment (generate at https://buffer.com/developers/api).
3. Channel IDs for each connected profile (see [Get channel IDs](#get-channel-ids)).

## API Endpoint

All requests go to Buffer's GraphQL endpoint:

```
POST https://graph.bufferapp.com/graphql
Authorization: Bearer $BUFFER_ACCESS_TOKEN
Content-Type: application/json
```

## Get Channel IDs

Query your connected channels to find their IDs:

```bash
curl -s https://graph.bufferapp.com/graphql \
  -H "Authorization: Bearer $BUFFER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"query": "{ channels { id name service } }"}' | jq '.data.channels'
```

Save the `id` for each channel you want to post to.

## Common Operations

### Publish a Post Immediately

Use `createPost` (not `createIdea` — ideas are internal drafts, not publishable).

```bash
curl -s https://graph.bufferapp.com/graphql \
  -H "Authorization: Bearer $BUFFER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreatePost($input: CreatePostInput!) { createPost(input: $input) { id status } }",
    "variables": {
      "input": {
        "channelIds": ["CHANNEL_ID_HERE"],
        "text": "Your post content here. #hashtag",
        "postingSchedule": "now"
      }
    }
  }'
```

### Schedule a Post

```bash
curl -s https://graph.bufferapp.com/graphql \
  -H "Authorization: Bearer $BUFFER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreatePost($input: CreatePostInput!) { createPost(input: $input) { id status scheduledAt } }",
    "variables": {
      "input": {
        "channelIds": ["CHANNEL_ID_HERE"],
        "text": "Scheduled post content.",
        "postingSchedule": "scheduled",
        "scheduledAt": "2026-03-15T14:00:00Z"
      }
    }
  }'
```

### Multi-Channel Post

Post to multiple channels simultaneously by passing multiple channel IDs:

```bash
curl -s https://graph.bufferapp.com/graphql \
  -H "Authorization: Bearer $BUFFER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "mutation CreatePost($input: CreatePostInput!) { createPost(input: $input) { id status } }",
    "variables": {
      "input": {
        "channelIds": ["LINKEDIN_CHANNEL_ID", "TWITTER_CHANNEL_ID"],
        "text": "Cross-posted content.",
        "postingSchedule": "now"
      }
    }
  }'
```

### Verify a Post Was Published

Query by post ID to confirm status:

```bash
curl -s https://graph.bufferapp.com/graphql \
  -H "Authorization: Bearer $BUFFER_ACCESS_TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "query": "{ post(id: \"POST_ID_HERE\") { id status text scheduledAt sentAt } }"
  }'
```

## Platform-Specific Tips

- **LinkedIn**: supports longer text (up to 3,000 chars). Include a link at the end for best engagement.
- **Twitter/X**: 280 char limit. No markdown. Hashtags count toward limit.
- **Facebook**: link previews auto-generate. Avoid URL shorteners.
- **Instagram**: requires an image for feed posts. Text-only posts are not supported on Instagram.

## Gotchas

- **`createPost` vs `createIdea`**: always use `createPost`. `createIdea` saves to Buffer's internal idea board and does not publish to any channel.
- **Duplicate detection**: Buffer may silently reject posts with identical text to a recent post on the same channel. Change the text slightly or wait before retrying.
- **Rate limits**: Buffer's API has rate limits per access token. Space out bulk operations.
- **Channel IDs change**: if you disconnect and reconnect a social account in Buffer, the channel ID changes. Re-query channels after reconnecting.

## When NOT to Use This Skill

- Direct posting without Buffer (use platform-native APIs instead).
- Reading social media feeds (Buffer is write-only).
- Managing Buffer team settings or billing.
