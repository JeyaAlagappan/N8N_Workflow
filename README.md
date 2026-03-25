# 📰 48-Hour Newsletter Automation

An n8n workflow that transforms a simple Slack message into a fully designed, AI-generated HTML newsletter — fetching real-time news from multiple sources, curating imagery, and delivering the final product straight to your inbox via Gmail.

---

## How It Works

Type a news category (e.g. `AI`, `Sports`, `Geopolitics`) into a designated Slack channel. Within seconds the workflow fetches articles published in the last 48 hours from three news APIs, pulls relevant stock photography, feeds everything into GPT-4o-mini to produce a polished HTML newsletter, and emails it to you — all while posting status updates back to Slack.

---

## Workflow Architecture

```
Slack Message
      │
      ▼
┌─────────────┐
│ Bot Filter  │──── (bot messages discarded)
└─────┬───────┘
      ▼
┌──────────────────┐
│ Parse Category & │
│ Calculate 48h    │
│ Time Window      │
└──────┬───────────┘
       │
       ├──► GNews API ─────────┐
       ├──► NewsAPI ───────────┤
       ├──► The Guardian API ──┤
       └──► Pexels Images ────┤
                               ▼
                    ┌──────────────────┐
                    │  Merge & Merge   │
                    │  All Results     │
                    └────────┬─────────┘
                             ▼
                    ┌──────────────────┐
                    │ Normalize,       │
                    │ Deduplicate &    │
                    │ Filter (Top 10)  │
                    └────────┬─────────┘
                             │
                    ┌────────┴─────────┐
                    │  Articles Found? │
                    ├─── No ──► Slack: "No articles found"
                    │
                    ▼ Yes
              ┌─────────────┐
              │ GPT-4o-mini │
              │ Generate    │
              │ HTML Email  │
              └──────┬──────┘
                     ▼
              ┌─────────────┐
              │ Clean HTML  │
              │ + Base64    │
              │ Attachment  │
              └──────┬──────┘
                     ▼
              ┌─────────────┐
              │  Send Gmail │
              ├─── OK ──► Slack: "✅ Newsletter Delivered!"
              └─── Fail ─► Slack: "❌ Email Failed"
```

---

## Nodes Breakdown

### 1. Slack Trigger
Listens for new messages in a configured Slack channel.

### 2. Is Human Message?
Filters out bot-generated messages by checking whether `bot_id` is empty, preventing infinite loops.

### 3. Code — Parse Category & Time Window
Normalizes the user's input into a known category (`Sports`, `Artificial Intelligence`, `Geopolitics`, or freeform), computes the 48-hour lookback window, maps category-specific search keywords for every API, and persists all metadata to n8n's global static data so downstream nodes can access it reliably.

### 4. Parallel Data Fetching (4 HTTP Request nodes)

| Node | API | What it returns |
|---|---|---|
| **Fetch GNews** | `gnews.io` | Up to 10 articles with title, description, image, source |
| **Fetch NewsAPI** | `newsapi.org` | Up to 10 articles with `urlToImage`, source metadata |
| **Fetch The Guardian** | `guardianapis.com` | Up to 10 results with headline, trail text, thumbnail |
| **Fetch Pexels Images** | `pexels.com` | 3 landscape photos matching the category keyword |

### 5. Merge Nodes
Two-stage merge combines GNews + NewsAPI, then Guardian + Pexels, and finally merges everything into a single stream.

### 6. Code — Normalize, Deduplicate & Filter
Identifies which API each input came from, normalizes all articles into a common schema (`title`, `description`, `url`, `publishedAt`, `source`, `image`), removes duplicates by URL, strips invalid entries, sorts by date descending, and caps the result at 10 articles. Also maps Pexels photos into hero / section image slots.

### 7. Articles Found?
Routes to the LLM path if articles exist, or posts a warning to Slack if none were found.

### 8. LLM Generate Newsletter (GPT-4o-mini)
Sends a detailed system + user prompt to OpenAI requesting a complete, mobile-responsive HTML email with a gradient header, category badge, hero image, article cards, section images, a trend highlight, and a professional footer. Temperature is set to `0.7` with a `4000` token limit.

### 9. Code — Clean HTML & Create Attachment
Strips markdown backticks from the LLM output, validates length, converts the cleaned HTML to a Base64 binary attachment, and prepares the email payload.

### 10. Send Gmail
Sends the newsletter to the configured recipient with a date-stamped subject line and the generated HTML embedded in a wrapper email template. Uses `continueErrorOutput` to gracefully handle failures.

### 11. Slack Notifications
Posts a success or failure message back to the originating Slack channel.

---

## Supported Categories

The workflow ships with built-in keyword mappings for three categories, but accepts any freeform topic:

| Input | Normalized Category | Search Keywords |
|---|---|---|
| `ai`, `artificial intelligence` | Artificial Intelligence | AI-specific queries across all APIs |
| `sports`, `sport` | Sports | Sport-focused queries |
| `geopolitics`, `geo`, `world` | Geopolitics | World politics queries |
| *(anything else)* | *(used as-is)* | Category name used directly as keyword |

---

## Prerequisites

### API Keys Required

| Service | Purpose | Get a key |
|---|---|---|
| **GNews** | News articles | [gnews.io](https://gnews.io/) |
| **NewsAPI** | News articles | [newsapi.org](https://newsapi.org/) |
| **The Guardian** | News articles | [open-platform.theguardian.com](https://open-platform.theguardian.com/) |
| **Pexels** | Stock images | [pexels.com/api](https://www.pexels.com/api/) |
| **OpenAI** | Newsletter generation (GPT-4o-mini) | [platform.openai.com](https://platform.openai.com/) |

### Credentials in n8n

| Credential Name | Type | Used By |
|---|---|---|
| `Newsletter_Slack` | Slack API | Trigger + all Slack notification nodes |
| `Gmail account` | Gmail OAuth2 | Send Gmail node |
| `OpenAi account 2` | OpenAI API | LLM Generate Newsletter node |

> **Note:** API keys for GNews, NewsAPI, The Guardian, and Pexels are passed as query/header parameters directly in the HTTP Request nodes. Update them inside each node's settings after import.

---

## Setup Instructions

1. **Import the workflow** — Open your n8n instance, go to *Workflows → Import from File*, and select `48-Hour_Newsletter_Automation.json`.

2. **Configure credentials** — Create or connect credentials for Slack, Gmail (OAuth2), and OpenAI in *Settings → Credentials*.

3. **Update API keys** — Open each HTTP Request node (`Fetch GNews`, `Fetch NewsApi`, `Fetch The Guardian`, `Fetch Pexels Images`) and replace the placeholder API keys with your own.

4. **Set your Slack channel** — The workflow listens on channel ID `C0AGC0EKE2W`. Update this in the Slack Trigger node and all Slack notification nodes to match your channel.

5. **Set the recipient email** — The default recipient is hardcoded in the `Send Gmail` node and as a fallback in the Code nodes. Change it to your desired address.

6. **Activate the workflow** — Toggle the workflow to *Active*. Send a message like `AI` or `Sports` in your Slack channel and watch the magic happen.

---
## Error Handling

The workflow includes three Slack notification paths:

| Scenario | Slack Message |
|---|---|
| No articles found for the category | ⚠️ *No Articles Found* |
| Gmail send fails | ❌ *Email Delivery Failed* |
| Newsletter sent successfully | ✅ *Newsletter Delivered!* |

All HTTP Request nodes are configured with `neverError: true` so the workflow continues gracefully even if an individual API is down.

---

## License

This workflow is provided as-is for personal and educational use. Respect the terms of service of each third-party API.
