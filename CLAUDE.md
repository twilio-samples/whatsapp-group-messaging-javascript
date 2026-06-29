# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What This Is

A Twilio code sample demonstrating WhatsApp group messaging using the Twilio Conversations API and Sync. It consists of three standalone Twilio Functions (serverless) — there is no `package.json`, no build step, and no local test runner. The Twilio Functions runtime provides the Node.js helper library.

## Deployment

These functions are deployed to Twilio Functions, not run locally. Use the [Twilio CLI](https://www.twilio.com/docs/twilio-cli/quickstart) or the Twilio Console to deploy:

```bash
twilio serverless:deploy
```

## Architecture

Three independent serverless functions, each exported as `exports.handler`:

| File                    | Visibility | Trigger                                          | Purpose                                                                                                                |
| ----------------------- | ---------- | ------------------------------------------------ | ---------------------------------------------------------------------------------------------------------------------- |
| `createConversation.js` | Public     | HTTP GET (admin calls manually)                  | Creates the Conversation, stores `CONVERSATION_SID` as an env var, sends initial WhatsApp invites via Content Template |
| `joinConversation.js`   | Protected  | Messaging Service webhook (`onMessageReceived`)  | Adds participants who reply "Join the group chat" to the Conversation and records them in the Sync Map                 |
| `message.js`            | Protected  | Conversations Pre-Event webhook (`onMessageAdd`) | Prepends the sender's display name (from Sync Map) to every outbound message body                                      |

**Data flow:** Sync Map (`SYNC_MAP_SID`) is the shared state store — it maps participant phone numbers to profile names. `joinConversation.js` writes to it; `message.js` reads from it.

## Required Environment Variables

Set these in the Twilio Functions environment (`.env` for local Twilio CLI dev):

```
SYNC_MAP_SID          # SID of the Twilio Sync Map
TWILIO_SERVICE_SID    # SID of the Twilio Sync Service
WHATSAPP_NUMBER       # WhatsApp sender phone number (e.g. whatsapp:+1...)
MESSAGE_SERVICE_SID   # SID of the Messaging Service
CONTENT_SID           # SID of the pre-approved WhatsApp Content Template
SERVICE_SID           # Twilio Functions Service SID
ENVIRONMENT_SID       # Twilio Functions Environment SID
CONVERSATION_SID      # Set automatically by createConversation.js at runtime
```

## Webhook Configuration

After deploying:

1. Set the Messaging Service inbound webhook URL to the `joinConversation` function URL.
2. Set the Conversations Service Pre-Event webhook URL to the `message` function URL, scoped to `onMessageAdd`.

## Key Twilio API Patterns

- Functions receive `(context, event, callback)` — `context` exposes env vars and `getTwilioClient()`.
- `callback(null, response)` returns success; `callback(error)` returns failure.
- The Conversations API modifies message content via Pre-Event webhooks by returning a modified `body` in the callback response.
- Sync Map items are keyed by participant phone number and store `{ identity, name }`.
