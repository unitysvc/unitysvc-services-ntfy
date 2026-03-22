# Getting Started with ntfy Push Notifications on UnitySVC

ntfy delivers push notifications via simple HTTP requests. Publish from any script, CI pipeline, or application — receive instantly on your phone, desktop, or via streaming API.

This tutorial walks you through the entire process: from enrolling in the service to sending your first notification, setting up mobile and streaming subscriptions, and using rich notification features for real-world use cases.

## What You Get After Enrolling

| Feature | Details |
|---------|---------|
| **Endpoint URL** | `https://api.unitysvc.com/ntfy/{TOPIC_CODE}` |
| **Topic code** | Unique 6-character alphanumeric code (e.g., `NHKVGT`) |
| **Price** | $0.01 per notification |
| **Message size** | 4 KB max |
| **Message retention** | 12 hours |

## Prerequisites

Before you begin, you'll need:

1. **A UnitySVC account** — [Sign up at unitysvc.com](https://unitysvc.com) if you don't have one
2. **A funded wallet** — Add funds from the Wallets page in your dashboard
3. **A SVCPASS API key** — Generate one from **Settings > API Keys** in your dashboard

Your API key will look something like: `svcpass_faketestkey1234567890abcdefexample`

Set it as an environment variable so you can use it throughout this tutorial:

```bash
export SVCPASS_API_KEY="svcpass_YOUR_KEY_HERE"
```

---

## Step 1: Find the Service in the Marketplace

Navigate to the **Marketplace** in your UnitySVC dashboard and search for **ntfy Push Notifications**.

<!-- ![ntfy in the marketplace](./images/01-marketplace-ntfy.png) -->

You'll see the service listing with:
- **Provider:** ntfy.sh
- **Price:** $0.01 per notification
- **Description:** Push notifications via HTTP — publish from anywhere, receive on any device

Click the service to view its full details.

---

## Step 2: Enroll

Click **Enroll** to create your subscription. There are no parameters to configure — just submit the form.

<!-- ![Enrollment form](./images/02-enrollment-form.png) -->

After enrolling, you'll receive a **topic code** — a 6-character string (e.g., `NHKVGT`) that identifies your private notification channel.

<!-- ![Enrollment details with topic code](./images/03-enrollment-details.png) -->

Save it as an environment variable:

```bash
export TOPIC_CODE="YOUR_TOPIC_CODE"
```

Your notification endpoint is:

```
https://api.unitysvc.com/ntfy/{TOPIC_CODE}
```

---

## Step 3: Send Your First Notification (curl)

Publish a plain-text message to your topic:

```bash
curl -X POST "https://api.unitysvc.com/ntfy/${TOPIC_CODE}" \
  -H "Authorization: Bearer ${SVCPASS_API_KEY}" \
  -H "Title: Hello from UnitySVC" \
  -H "Priority: default" \
  -H "Tags: wave" \
  -d "Your first ntfy notification!"
```

You should see a JSON response confirming the message:

```json
{
    "id": "uxUBNJ7R9Cew",
    "time": 1774136777,
    "expires": 1774179977,
    "event": "message",
    "topic": "NHKVGT",
    "title": "Hello from UnitySVC",
    "message": "Your first ntfy notification!",
    "tags": [
        "wave"
    ]
}
```

**Verify:** The response contains `"event": "message"` and your topic code. The message body is sent as plain text in the request body — metadata (title, priority, tags) goes in HTTP headers.

---

## Step 4: Send Your First Notification (Python)

```python
import os
import requests

api_key = os.environ["SVCPASS_API_KEY"]
topic = os.environ["TOPIC_CODE"]

response = requests.post(
    f"https://api.unitysvc.com/ntfy/{topic}",
    headers={
        "Authorization": f"Bearer {api_key}",
        "Title": "Hello from Python",
        "Priority": "default",
        "Tags": "snake",
    },
    data="Notification sent from a Python script!",
)

result = response.json()
print(f"Message ID: {result['id']}")
print(f"Topic:      {result['topic']}")
print(f"Message:    {result['message']}")
```

Run it:

```bash
pip install requests  # if not already installed
python notify.py
```

Expected output:

```
Message ID: abc123def456
Topic:      NHKVGT
Message:    Notification sent from a Python script!
```

---

## Step 5: Set Up the Mobile App

Install the official ntfy app to receive notifications on your phone.

| Platform | App |
|----------|-----|
| **Android** | [ntfy on Google Play](https://play.google.com/store/apps/details?id=io.heckel.ntfy) or [F-Droid](https://f-droid.org/packages/io.heckel.ntfy/) |
| **iOS** | [ntfy on App Store](https://apps.apple.com/us/app/ntfy/id1625396347) |

Once installed, add a subscription:

1. Tap **+** (or "Add subscription")
2. Set **Server URL** to `https://ntfy.svcpass.com`
3. Set **Topic** to your 6-character code (e.g., `NHKVGT`)
4. Tap **Subscribe**

<!-- ![Mobile app subscription setup](./images/04-mobile-app-subscribe.png) -->

> **Important:** The mobile app connects directly to `https://ntfy.svcpass.com`, NOT through the UnitySVC gateway. No API key is needed for receiving notifications — only for publishing them.

Send a test notification from your terminal (Step 3) and verify it appears on your phone within seconds.

---

## Step 6: Subscribe via HTTP JSON Stream

Open a long-lived connection to stream messages in real time:

```bash
curl -s "https://api.unitysvc.com/ntfy/${TOPIC_CODE}/json" \
  -H "Authorization: Bearer ${SVCPASS_API_KEY}"
```

This outputs newline-delimited JSON — one event per line — until you disconnect:

```json
{"event":"open","topic":"NHKVGT"}
{"event":"message","topic":"NHKVGT","message":"Hello!","title":"Test","priority":3}
{"event":"keepalive","topic":"NHKVGT"}
```

**Verify:** You should see an `"open"` event immediately. Send a message from another terminal and it appears as a `"message"` event. Keepalives arrive every ~30 seconds.

### Event types

| Event | Description |
|-------|-------------|
| `open` | Connection established |
| `message` | A published message (contains `message`, `title`, `priority`, `tags`, etc.) |
| `keepalive` | Periodic heartbeat (~30 seconds) to keep the connection alive |

---

## Step 7: Subscribe via WebSocket

For persistent, bidirectional connections, use WebSocket:

```python
import asyncio
import json
import os
import websockets

async def listen():
    topic = os.environ["TOPIC_CODE"]
    api_key = os.environ["SVCPASS_API_KEY"]
    uri = f"wss://api.unitysvc.com/ntfy/{topic}/ws"
    headers = {"Authorization": f"Bearer {api_key}"}

    async with websockets.connect(uri, additional_headers=headers) as ws:
        print(f"Connected — listening on topic {topic}")
        async for raw in ws:
            event = json.loads(raw)
            if event["event"] == "message":
                title = event.get("title", "Notification")
                print(f"[{title}] {event['message']}")

asyncio.run(listen())
```

Run it:

```bash
pip install websockets
python listen.py
```

Then send a message from another terminal — it should appear instantly.

> **Important:** The browser WebSocket API does not support custom headers. For browser-based apps, use the HTTP JSON stream (EventSource) approach instead.

---

## Step 8: Rich Notifications

ntfy supports rich notification features via HTTP headers:

| Header | Example | Description |
|--------|---------|-------------|
| `Title` | `Server Alert` | Notification title |
| `Priority` | `urgent` | min, low, default, high, urgent |
| `Tags` | `warning,server` | Comma-separated emoji tags |
| `Click` | `https://example.com` | URL opened when notification is tapped |
| `Delay` | `30m` | Delayed delivery (e.g., `30m`, `2h`, `tomorrow 9am`) |

### Example: All features combined

```bash
curl -X POST "https://api.unitysvc.com/ntfy/${TOPIC_CODE}" \
  -H "Authorization: Bearer ${SVCPASS_API_KEY}" \
  -H "Title: Deploy Complete" \
  -H "Priority: high" \
  -H "Tags: white_check_mark,rocket" \
  -H "Click: https://github.com/myorg/myrepo/actions" \
  -d "Production deploy v2.1.0 succeeded"
```

Expected response:

```json
{
    "id": "F3BmDlSUk6ru",
    "time": 1774136786,
    "expires": 1774179986,
    "event": "message",
    "topic": "NHKVGT",
    "title": "Deploy Complete",
    "message": "Production deploy v2.1.0 succeeded",
    "priority": 4,
    "tags": [
        "white_check_mark",
        "rocket"
    ],
    "click": "https://github.com/myorg/myrepo/actions"
}
```

### Priority levels

| Priority | Name | Behavior |
|----------|------|----------|
| 1 | `min` | No sound or vibration |
| 2 | `low` | No sound |
| 3 | `default` | Normal notification |
| 4 | `high` | Prominent notification |
| 5 | `urgent` | Bypasses Do Not Disturb on mobile |

### JSON body format

You can also send structured JSON instead of using headers. The JSON format requires posting to the **base ntfy endpoint** (without your topic code in the URL) and including `topic` in the body:

```bash
curl -X POST "https://api.unitysvc.com/ntfy/" \
  -H "Authorization: Bearer ${SVCPASS_API_KEY}" \
  -H "Content-Type: application/json" \
  -d "{
    \"topic\": \"${TOPIC_CODE}\",
    \"message\": \"Disk usage above 90%\",
    \"title\": \"Server Alert\",
    \"priority\": 4,
    \"tags\": [\"warning\", \"disk\"]
  }"
```

Expected response:

```json
{
    "id": "4qDYRlZIGJyd",
    "time": 1774136890,
    "expires": 1774180090,
    "event": "message",
    "topic": "NHKVGT",
    "title": "Server Alert",
    "message": "Disk usage above 90%",
    "priority": 4,
    "tags": [
        "warning",
        "disk"
    ]
}
```

> **Note:** When posting to the topic URL (`/ntfy/{TOPIC_CODE}`), the body is treated as the message text and metadata goes in headers. When posting to the base URL (`/ntfy/`), you can use a JSON body with all fields including `topic`.

### Delayed delivery

Schedule a message for later:

```bash
curl -X POST "https://api.unitysvc.com/ntfy/${TOPIC_CODE}" \
  -H "Authorization: Bearer ${SVCPASS_API_KEY}" \
  -H "Delay: 30m" \
  -d "Reminder: check on the deploy"
```

Accepts values like `30m`, `2h`, `tomorrow 9am`, or a Unix timestamp.

---

## Step 9: Realistic Use Cases

### CI/CD Pipeline Notification

Send a notification when a GitHub Actions workflow finishes:

```yaml
# .github/workflows/deploy.yml
- name: Notify on success
  if: success()
  run: |
    curl -s -X POST "https://api.unitysvc.com/ntfy/${{ vars.TOPIC_CODE }}" \
      -H "Authorization: Bearer ${{ secrets.SVCPASS_API_KEY }}" \
      -H "Title: Deploy Succeeded" \
      -H "Tags: white_check_mark" \
      -d "Deployed ${{ github.sha }} to production"

- name: Notify on failure
  if: failure()
  run: |
    curl -s -X POST "https://api.unitysvc.com/ntfy/${{ vars.TOPIC_CODE }}" \
      -H "Authorization: Bearer ${{ secrets.SVCPASS_API_KEY }}" \
      -H "Title: Deploy Failed" \
      -H "Priority: high" \
      -H "Tags: x" \
      -d "Deploy failed for ${{ github.sha }}"
```

### Server Health Alert

Add to cron — sends a notification only when a health check fails:

```bash
#!/bin/bash
curl -sf https://myapi.com/health || \
  curl -X POST "https://api.unitysvc.com/ntfy/${TOPIC_CODE}" \
    -H "Authorization: Bearer ${SVCPASS_API_KEY}" \
    -H "Title: Health Check Failed" \
    -H "Priority: urgent" \
    -H "Tags: rotating_light" \
    -d "myapi.com health check failed at $(date)"
```

### Python Notification Helper

A reusable function for any Python project:

```python
import requests

def notify(message, title=None, priority=None, tags=None):
    headers = {"Authorization": f"Bearer {SVCPASS_API_KEY}"}
    if title:
        headers["Title"] = title
    if priority:
        headers["Priority"] = str(priority)
    if tags:
        headers["Tags"] = ",".join(tags)

    requests.post(
        f"https://api.unitysvc.com/ntfy/{TOPIC_CODE}",
        headers=headers,
        data=message,
    )

# Usage
notify("Order #1234 shipped", title="Order Update", tags=["package"])
notify("Payment failed", title="Billing Alert", priority=4, tags=["warning"])
```

---

## Step 10: Limits and Troubleshooting

### Limits

| Limit | Value |
|-------|-------|
| **Message size** | 4 KB max |
| **Message retention** | 12 hours |
| **Topic isolation** | Each enrollment gets a unique topic code — you cannot access other topics |

### Troubleshooting

| Symptom | Cause | Fix |
|---------|-------|-----|
| 401 Unauthorized | Invalid or missing API key | Verify `SVCPASS_API_KEY` is set and starts with `svcpass_` |
| 404 Not Found | Not enrolled or wrong topic code | Check enrollment on dashboard; topic codes are case-sensitive |
| Messages don't appear on mobile | Wrong server URL in app | Set server to `https://ntfy.svcpass.com` (not ntfy.sh) |
| Messages don't appear on mobile | Wrong topic code | Verify topic matches your enrollment code exactly |
| Empty response body | Message too large | Keep messages under 4 KB |
| WebSocket disconnects | Client-side timeout | ntfy sends keepalives every ~30s — check your client timeout settings |

---

## Endpoints Reference

All endpoints use your topic code. Replace `{TOPIC_CODE}` with your 6-character enrollment code.

| Method | Path | Description |
|--------|------|-------------|
| `POST` | `/ntfy/{TOPIC_CODE}` | Publish a message |
| `PUT` | `/ntfy/{TOPIC_CODE}` | Publish a message (same as POST) |
| `GET` | `/ntfy/{TOPIC_CODE}/json` | Subscribe via HTTP JSON stream |
| `GET` | `/ntfy/{TOPIC_CODE}/ws` | Subscribe via WebSocket |

---

## Next Steps

- **Explore rich features** — Attachments, action buttons, and more at [docs.ntfy.sh/publish](https://docs.ntfy.sh/publish/)
- **Pair with monitoring** — Combine ntfy with [Uptime Monitoring](../../unitysvc-uptime/docs/tutorials/getting-started.md) for instant downtime alerts
- **Add more topics** — Enroll additional times for separate notification channels (CI, alerts, personal)
- **Full ntfy documentation** — [docs.ntfy.sh](https://docs.ntfy.sh/)
