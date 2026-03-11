---
name: brevo-newsletter
description: "Send email newsletters via Brevo (formerly Sendinblue) REST API. Use when creating, testing, and sending email campaigns with audience segmentation. Requires a Brevo API key."
homepage: https://www.brevo.com
metadata:
  {
    "openclaw":
      {
        "emoji": "📨",
        "requires": { "env": ["BREVO_API_KEY"] },
      },
  }
allowed-tools: ["exec"]
---

# Brevo Newsletter Publishing

Send email newsletters via Brevo's REST API. Covers the full workflow: create campaign, select audience, test, and send.

## Prerequisites

1. Brevo account (free tier: 300 emails/day).
2. `BREVO_API_KEY` set in environment (generate at https://app.brevo.com/settings/keys/api).
3. At least one verified sender email.
4. At least one contact list with subscribers.

## API Endpoint

All requests go to:

```
https://api.brevo.com/v3/
api-key: $BREVO_API_KEY
Content-Type: application/json
```

## Workflow

The newsletter workflow follows this sequence:

1. **Create** the campaign (draft).
2. **Test** by sending to a small group.
3. **Review** the test (human approval gate).
4. **Send** to the full audience.

## Common Operations

### List Available Templates

```bash
curl -s https://api.brevo.com/v3/smtp/templates \
  -H "api-key: $BREVO_API_KEY" \
  -H "Content-Type: application/json" | jq '.templates[] | {id, name, subject}'
```

### List Contact Lists

```bash
curl -s https://api.brevo.com/v3/contacts/lists \
  -H "api-key: $BREVO_API_KEY" | jq '.lists[] | {id, name, totalSubscribers}'
```

### Create a Campaign (Draft)

```bash
curl -s -X POST https://api.brevo.com/v3/emailCampaigns \
  -H "api-key: $BREVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "name": "March 2026 Newsletter",
    "subject": "Monthly Update — March 2026",
    "sender": { "name": "Newsletter", "email": "newsletter@example.org" },
    "templateId": TEMPLATE_ID,
    "recipients": {
      "listIds": [LIST_ID],
      "segmentIds": [SEGMENT_ID],
      "excludeListIds": [UNSUBSCRIBE_LIST_ID]
    },
    "replyTo": "reply@example.org"
  }'
```

The response includes `id` — save it for testing and sending.

### Send a Test Email

Before sending to the full audience, test with a small group:

```bash
curl -s -X POST "https://api.brevo.com/v3/emailCampaigns/CAMPAIGN_ID/sendTest" \
  -H "api-key: $BREVO_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
    "emailTo": ["test1@example.org", "test2@example.org"]
  }'
```

Review the test email in the recipients' inboxes before proceeding.

### Send the Campaign

After test review and human approval:

```bash
curl -s -X POST "https://api.brevo.com/v3/emailCampaigns/CAMPAIGN_ID/sendNow" \
  -H "api-key: $BREVO_API_KEY" \
  -H "Content-Type: application/json"
```

### Check Campaign Status

```bash
curl -s "https://api.brevo.com/v3/emailCampaigns/CAMPAIGN_ID" \
  -H "api-key: $BREVO_API_KEY" | jq '{status, statistics}'
```

## Audience Segmentation

Brevo supports three mechanisms for targeting:

- **`listIds`**: include entire contact lists.
- **`segmentIds`**: include dynamic segments (filter by attributes).
- **`excludeListIds`**: exclude lists (typically your unsubscribe/opt-out list).

Always include an exclusion list for unsubscribed contacts. Example:

```json
{
  "recipients": {
    "listIds": [2, 5],
    "segmentIds": [1],
    "excludeListIds": [10]
  }
}
```

## Free Tier Considerations

Brevo's free tier allows 300 emails per day. For newsletters:

- Keep your audience under 300 contacts, or upgrade to a paid plan.
- Schedule sends during off-peak hours to avoid daily limit conflicts with transactional emails.
- Monitor usage: `GET https://api.brevo.com/v3/account` returns remaining credits.

## Pre-Send Checklist

Before every campaign send:

- [ ] Subject line reviewed (no placeholder text).
- [ ] Sender email is verified in Brevo.
- [ ] Template renders correctly (check test email).
- [ ] Audience is correct (`listIds` + `segmentIds`).
- [ ] Unsubscribe list excluded (`excludeListIds`).
- [ ] Reply-to address set.
- [ ] Test sent to at least 2 reviewers.
- [ ] Human approval received.

## Gotchas

- **Template IDs**: templates are versioned. If you update a template in the Brevo UI, the `templateId` stays the same but content changes. Always test after template edits.
- **Sender verification**: new sender emails must be verified before use. Campaigns with unverified senders fail silently.
- **Rate limiting**: Brevo enforces rate limits on API calls (not just email sends). Space out API requests in automation loops.
- **HTML content**: if using `htmlContent` instead of `templateId`, ensure the HTML includes an unsubscribe link (`{{ unsubscribe }}`). Brevo requires this for compliance.

## When NOT to Use This Skill

- Transactional emails (order confirmations, password resets) — use Brevo's transactional API instead.
- SMS campaigns — use Brevo's SMS API.
- Managing contacts or lists — use Brevo's contacts API directly.
