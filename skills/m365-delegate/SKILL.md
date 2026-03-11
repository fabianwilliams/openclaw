---
name: m365-delegate
description: "Read and send email, manage calendar via Microsoft 365 Graph API with delegate permissions. Use when the agent needs to read a mailbox, send on behalf of a user, or manage calendar events in M365. Requires Azure AD app registration."
homepage: https://learn.microsoft.com/graph/overview
metadata:
  {
    "openclaw":
      {
        "emoji": "📧",
        "requires":
          {
            "env":
              [
                "M365_TENANT_ID",
                "M365_CLIENT_ID",
                "M365_CLIENT_SECRET",
              ],
          },
      },
  }
allowed-tools: ["exec"]
---

# M365 Delegate (Graph API)

Read email, send messages, and manage calendar via Microsoft Graph API using application permissions with delegate scoping. Designed for agents that act on behalf of users in a Microsoft 365 tenant.

## Prerequisites

1. Azure AD app registration with the following **application** permissions:
   - `Mail.Read` — read mailbox contents.
   - `Mail.Send` — send email (as the delegate mailbox).
   - `Calendars.ReadWrite` — read and create calendar events.
2. Admin consent granted for the application permissions.
3. Application access policy restricting the app to specific mailboxes (see [Security](#security)).
4. Environment variables set:
   - `M365_TENANT_ID` — Azure AD tenant ID.
   - `M365_CLIENT_ID` — App registration client ID.
   - `M365_CLIENT_SECRET` — App registration client secret.

## Authentication

Obtain an access token using the client credentials flow:

```bash
TOKEN=$(curl -s -X POST \
  "https://login.microsoftonline.com/$M365_TENANT_ID/oauth2/v2.0/token" \
  -H "Content-Type: application/x-www-form-urlencoded" \
  -d "client_id=$M365_CLIENT_ID&scope=https://graph.microsoft.com/.default&client_secret=$M365_CLIENT_SECRET&grant_type=client_credentials" \
  | jq -r '.access_token')
```

Use `$TOKEN` in subsequent requests as `Authorization: Bearer $TOKEN`.

## Common Operations

### Read Inbox

```bash
curl -s "https://graph.microsoft.com/v1.0/users/delegate@example.org/mailFolders/Inbox/messages?\$top=10&\$orderby=receivedDateTime desc&\$select=subject,from,receivedDateTime,isRead,bodyPreview" \
  -H "Authorization: Bearer $TOKEN" | jq '.value[] | {subject, from: .from.emailAddress.address, received: .receivedDateTime, read: .isRead, preview: .bodyPreview}'
```

### Read a Specific Email

```bash
curl -s "https://graph.microsoft.com/v1.0/users/delegate@example.org/messages/MESSAGE_ID?\$select=subject,from,toRecipients,body,receivedDateTime" \
  -H "Authorization: Bearer $TOKEN" | jq '{subject, from: .from.emailAddress.address, body: .body.content}'
```

### Send Email (as Delegate)

Send from the delegate mailbox. Recipients see the delegate's email address as the sender:

```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/delegate@example.org/sendMail" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "subject": "Meeting Follow-up",
      "body": {
        "contentType": "Text",
        "content": "Thank you for attending. Here are the action items..."
      },
      "toRecipients": [
        { "emailAddress": { "address": "recipient@example.com" } }
      ]
    },
    "saveToSentItems": true
  }'
```

### Send on Behalf Of

To send on behalf of another user (recipient sees "Delegate on behalf of Principal"):

```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/delegate@example.org/sendMail" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "message": {
      "subject": "Weekly Report",
      "body": {
        "contentType": "Text",
        "content": "Please find the weekly report attached."
      },
      "from": {
        "emailAddress": { "address": "principal@example.org" }
      },
      "toRecipients": [
        { "emailAddress": { "address": "team@example.com" } }
      ]
    },
    "saveToSentItems": true
  }'
```

Requires `Send on Behalf` permission set in Exchange Online for the delegate.

### Read Calendar

```bash
curl -s "https://graph.microsoft.com/v1.0/users/delegate@example.org/calendarView?\$top=10&startDateTime=$(date -u +%Y-%m-%dT00:00:00Z)&endDateTime=$(date -u -d '+7 days' +%Y-%m-%dT23:59:59Z 2>/dev/null || date -u -v+7d +%Y-%m-%dT23:59:59Z)&\$select=subject,start,end,organizer,location" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Prefer: outlook.timezone=\"America/New_York\"" \
  | jq '.value[] | {subject, start: .start.dateTime, end: .end.dateTime, organizer: .organizer.emailAddress.name}'
```

### Create Calendar Event

```bash
curl -s -X POST "https://graph.microsoft.com/v1.0/users/delegate@example.org/events" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{
    "subject": "Team Standup",
    "start": { "dateTime": "2026-03-15T09:00:00", "timeZone": "America/New_York" },
    "end": { "dateTime": "2026-03-15T09:30:00", "timeZone": "America/New_York" },
    "attendees": [
      { "emailAddress": { "address": "colleague@example.org" }, "type": "required" }
    ],
    "location": { "displayName": "Conference Room A" }
  }'
```

## Security

### Application Access Policy (Critical)

Without scoping, application permissions grant access to **every mailbox in the tenant**. Always create an application access policy:

1. Create a mail-enabled security group in Azure AD containing only the mailboxes the agent should access.
2. Apply the policy:

```powershell
# Exchange Online PowerShell
New-ApplicationAccessPolicy `
  -AppId "$M365_CLIENT_ID" `
  -PolicyScopeGroupId "agent-mailboxes@example.org" `
  -AccessRight RestrictAccess `
  -Description "Restrict agent to delegate mailboxes only"
```

3. Test the policy:

```powershell
Test-ApplicationAccessPolicy `
  -Identity "delegate@example.org" `
  -AppId "$M365_CLIENT_ID"
# Should return: Granted

Test-ApplicationAccessPolicy `
  -Identity "ceo@example.org" `
  -AppId "$M365_CLIENT_ID"
# Should return: Denied
```

### Additional Security Measures

- **Rotate client secrets** on a regular schedule (90 days recommended).
- **Monitor sign-in logs** for the app registration in Azure AD.
- **Use certificate-based auth** instead of client secrets for higher security.
- **Audit mailbox access** via Microsoft 365 compliance center.

## Microsoft 365 for Nonprofits

Qualifying nonprofits get Microsoft 365 Business Basic for free, which includes:

- Exchange Online (email + calendar).
- Graph API access.
- Azure AD for app registrations.

Apply at https://nonprofit.microsoft.com. This makes the full delegate pattern available at zero cost for eligible organizations.

## Gotchas

- **Token lifetime**: client credentials tokens expire after 1 hour. Cache and refresh before expiry.
- **Application access policy propagation**: new policies can take up to 30 minutes to take effect. Test before relying on them.
- **Shared mailboxes**: Graph API treats shared mailboxes as regular users. Use the shared mailbox UPN in the URL path.
- **Large mailboxes**: use `$top` and `$skip` for pagination. Default page size is 10 messages.
- **Calendar timezone**: always set `Prefer: outlook.timezone` header or use explicit `timeZone` in event payloads to avoid UTC surprises.
- **Send on Behalf vs Send As**: "Send on Behalf" shows the delegate name. "Send As" hides it. For transparency, prefer "Send on Behalf."

## When NOT to Use This Skill

- Personal Microsoft accounts (outlook.com, hotmail.com) — use delegated auth flow instead.
- Reading from mailboxes outside the application access policy scope.
- Managing Azure AD users, groups, or app registrations.
