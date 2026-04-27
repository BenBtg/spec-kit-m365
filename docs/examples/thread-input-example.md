# Example: Teams Thread -> Markdown Artifact

This example shows how a Teams channel conversation is fetched and saved as Markdown.

---

## Source: Teams Channel Messages (JSON)

The following is a sample of what the M365 CLI returns from `m365 teams message list`:

```json
[
  {
    "id": "1712900400000",
    "replyToId": null,
    "messageType": "message",
    "createdDateTime": "2026-04-09T10:00:00Z",
    "from": {
      "user": { "displayName": "Sarah Chen", "id": "user-001" }
    },
    "body": {
      "contentType": "text",
      "content": "Hey team — we need to finalize the approach for the new notification preferences feature. Users have been asking for more control over what emails and push notifications they receive. Let's discuss here and make a decision by EOW."
    },
    "importance": "high"
  },
  {
    "id": "1712900400001",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T10:15:00Z",
    "from": {
      "user": { "displayName": "Marcus Johnson", "id": "user-002" }
    },
    "body": {
      "contentType": "text",
      "content": "Good call. I think we should support per-category toggles (marketing, transactional, system alerts) plus a frequency option (instant, daily digest, weekly digest). We could also let users set quiet hours."
    }
  },
  {
    "id": "1712900400002",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T10:22:00Z",
    "from": {
      "user": { "displayName": "Priya Patel", "id": "user-003" }
    },
    "body": {
      "contentType": "text",
      "content": "Per-category toggles make sense. I'd push back on quiet hours for MVP though — that adds complexity with timezone handling. Can we do that in a later phase?"
    }
  },
  {
    "id": "1712900400003",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T10:30:00Z",
    "from": {
      "user": { "displayName": "Sarah Chen", "id": "user-001" }
    },
    "body": {
      "contentType": "text",
      "content": "Agreed — let's defer quiet hours to phase 2. For MVP: per-category toggles + frequency selection. @Marcus does the backend support per-category filtering already?"
    }
  },
  {
    "id": "1712900400004",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T11:05:00Z",
    "from": {
      "user": { "displayName": "Marcus Johnson", "id": "user-002" }
    },
    "body": {
      "contentType": "text",
      "content": "The notification service already tags messages by category. We'd need to add a user preferences table and a new API endpoint. Frequency/digest is trickier — we'd need a scheduled job to batch notifications. I'd estimate 2 sprints for the full thing."
    }
  },
  {
    "id": "1712900400005",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T11:20:00Z",
    "from": {
      "user": { "displayName": "Priya Patel", "id": "user-003" }
    },
    "body": {
      "contentType": "text",
      "content": "What about email vs push? Should users be able to toggle channels independently? Like get push for system alerts but email digest for marketing?"
    }
  },
  {
    "id": "1712900400006",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T11:35:00Z",
    "from": {
      "user": { "displayName": "Sarah Chen", "id": "user-001" }
    },
    "body": {
      "contentType": "text",
      "content": "Great point. Let's do per-channel AND per-category toggles. So the matrix is: [category] × [channel (email/push)] × [frequency]. That gives users full control."
    }
  },
  {
    "id": "1712900400007",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T11:40:00Z",
    "from": {
      "user": { "displayName": "Marcus Johnson", "id": "user-002" }
    },
    "body": {
      "contentType": "text",
      "content": "That works. Let's go with that. I'll start on the DB schema and API design. One question — should there be a 'reset to defaults' option?"
    }
  },
  {
    "id": "1712900400008",
    "replyToId": "1712900400000",
    "messageType": "message",
    "createdDateTime": "2026-04-09T14:00:00Z",
    "from": {
      "user": { "displayName": "Sarah Chen", "id": "user-001" }
    },
    "body": {
      "contentType": "text",
      "content": "Yes, include a 'Reset to defaults' button. Also, we need to define what the defaults are — I'd say all channels ON, frequency instant for transactional/system, daily digest for marketing. Confirmed: MVP scope is per-category × per-channel toggles + frequency + reset to defaults. Quiet hours deferred to phase 2."
    }
  }
]
```

---

## Generated Output: `teams-message-1712900400000.md`

The following is a sample output from `/speckit.m365.fetch-message`:

```markdown
---
Source:
  type: "m365-message"
  messageSource: "teams"
  messageId: "1712900400000"
  team: "Product Engineering"
  channel: "Backend"
  fetchedAt: "2026-04-09T15:30:00Z"
---

# Teams Thread: 1712900400000

- Team: Product Engineering
- Channel: Backend

## Messages

[2026-04-09T10:00:00Z] Sarah Chen

Hey team - we need to finalize the approach for the new notification preferences feature...

[2026-04-09T10:15:00Z] Marcus Johnson

Good call. I think we should support per-category toggles...

[2026-04-09T14:00:00Z] Sarah Chen

Yes, include a "Reset to defaults" button. Confirmed MVP scope is ...
```

## Summary

The fetch command produces a reviewable source artifact only.

Next step:

```text
speckit specify .specify/specs/teams-message-1712900400000.md
```
