# Doppel Plugin Development Guide

Doppel plugins are JSON files that teach the browser extension how to interact with a website. No code required — just CSS selectors and configuration.

## Quick Start

1. Copy an existing plugin JSON from `zh-CN/` or `en/`
2. Modify the selectors and URLs for your target site
3. Test by importing the JSON in the extension's plugin manager
4. Submit a PR to share with the community

## Plugin Structure

```jsonc
{
  "id": "mysite-automate",        // Unique ID: {site}-{capability}
  "name_cn": "中文名称",           // Chinese name
  "name_en": "English Name",      // English name
  "description_cn": "中文描述",    // Chinese description
  "description_en": "Description", // English description
  "type": "community",            // "community" for user-contributed
  "version": "1.0.0",             // Semver
  "author": "Your Name",
  "lang": "zh-CN",                // Target site language: "zh-CN" or "en"
  "rule": { ... }                 // Site rule definition (see below)
}
```

## Capabilities

Each plugin declares what it can do via `rule.capabilities`:

| Capability | Description | What it enables |
|-----------|-------------|-----------------|
| `automate` | Auto-patrol | Reads posts, AI replies, likes — runs on autopilot |
| `import` | Content import | Export button on profile pages to import user history |
| `compose` | AI writing | AI button next to input boxes for writing assistance |
| `enhance` | Content enhancement | Interest scoring, summaries on list pages |
| `translate` | Translation | Auto-translate foreign content |
| `browse` | Browse enhancement | General browsing improvements |

You can combine capabilities. A "full" plugin typically has `automate + import + compose`.

## Rule Definition

### `rule.match` — URL Matching

```json
{
  "match": {
    "urlPatterns": ["*://example.com/*"],
    "excludePatterns": ["*://example.com/admin/*"]
  }
}
```

Uses [Chrome match patterns](https://developer.chrome.com/docs/extensions/develop/concepts/match-patterns).

### `rule.pageTypes` — Page Detection

Identify what type of page the user is on:

```json
{
  "pageTypes": [
    {
      "name": "home",
      "priority": 10,
      "conditions": {
        "urlMatch": "^https?://example\\.com/?$"
      }
    },
    {
      "name": "detail",
      "priority": 30,
      "conditions": {
        "urlContains": ["/post/"]
      }
    },
    {
      "name": "profile",
      "priority": 20,
      "conditions": {
        "urlContains": ["/user/", "/profile/"]
      }
    }
  ]
}
```

Higher `priority` wins when multiple types match. Common page types: `home`, `list`, `detail`, `profile`, `hot`.

**Condition operators:**
- `urlMatch` — regex against full URL
- `urlContains` — URL includes any of these strings
- `urlNotContains` — URL must not include these
- `selector` / `domExists` — CSS selector must exist in DOM
- `or` / `and` — combine conditions

### `rule.extractors` — Content Extraction

Define how to pull content from the page using CSS selectors.

**List extractor** (for feed/list pages):

```json
{
  "extractors": {
    "lists": {
      "home": {
        "container": ".feed-list",
        "item": ".feed-item",
        "fields": {
          "title": { "selector": ".post-title", "transform": "trim" },
          "url": { "selector": "a.post-link", "attribute": "href" },
          "author": { "selector": ".author-name", "transform": "trim" }
        }
      }
    }
  }
}
```

**Detail extractor** (for post detail pages):

```json
{
  "extractors": {
    "detail": {
      "title": { "selector": "h1.post-title", "transform": "trim" },
      "author": { "selector": ".author", "transform": "trim" },
      "content": { "selector": ".post-body" },
      "time": { "selector": ".post-time", "transform": "trim" },
      "replies": {
        "container": ".comments",
        "item": ".comment",
        "fields": {
          "author": { "selector": ".comment-author", "transform": "trim" },
          "content": { "selector": ".comment-body" }
        }
      }
    },
    "currentUser": {
      "selector": ".my-avatar",
      "attribute": "alt"
    }
  }
}
```

**Field extractor options:**
- `selector` — CSS selector (required)
- `attribute` — HTML attribute to read (default: `textContent`)
- `regex` + `regexGroup` — extract with regex
- `transform` — `trim`, `parseInt`, `parseFloat`, `toLowerCase`, `toUpperCase`, `boolean`
- `default` — fallback value

### `rule.actions` — Interactions

```json
{
  "actions": {
    "clickPost": {
      "selector": "a.post-link"
    },
    "reply": {
      "input": {
        "selector": "textarea.reply-input",
        "type": "textarea"
      },
      "submit": {
        "selector": "button.submit-reply",
        "waitAfterClick": 2000
      }
    },
    "like": {
      "selector": ".like-button",
      "activeClass": "liked",
      "waitAfterClick": 500
    }
  }
}
```

**Input types:** `textarea`, `contenteditable`, `input`

**Available actions:** `clickPost`, `reply`, `like`, `collect`, `follow`, `repost`

### `rule.onboarding` — Login & Setup

```json
{
  "onboarding": {
    "startUrl": "https://example.com",
    "loginCheck": {
      "selector": ".user-avatar",
      "urlNotContains": "/login"
    },
    "loginUrl": "https://example.com/login",
    "loginHint": "Please log in first"
  }
}
```

### `rule.options` — Behavior Config

```json
{
  "options": {
    "supportedActions": ["reply", "like"],
    "delays": {
      "afterPageLoad": [2000, 4000],
      "afterClick": [1000, 2000],
      "afterReply": [3000, 5000]
    },
    "patrolModes": [
      {
        "value": "home",
        "label": "Home Feed",
        "startUrl": "https://example.com"
      },
      {
        "value": "hot",
        "label": "Trending",
        "startUrl": "https://example.com/hot"
      }
    ]
  }
}
```

Delays are `[min, max]` in milliseconds — the extension picks a random value in range to appear human.

### `rule.uiInjection` — UI Elements (Optional)

**AI compose button:**

```json
{
  "uiInjection": {
    "aiButton": {
      "pageTypes": ["home", "detail"],
      "injections": [{
        "targetSelector": "button.submit",
        "position": "before",
        "inputSelector": "textarea",
        "inputType": "textarea"
      }]
    }
  }
}
```

**Export/import button:**

```json
{
  "uiInjection": {
    "exportButton": {
      "pageTypes": ["profile"],
      "injection": {
        "targetSelector": ".profile-name",
        "position": "after"
      },
      "buttonText": "Export Posts",
      "buttonIcon": "📤",
      "dataSource": {
        "type": "api",
        "apiUrl": "/api/posts?uid={uid}&page={page}",
        "uidFromUrl": "/user/(\\d+)",
        "responseMapping": {
          "list": "data.items",
          "hasMore": "data.items.length > 0"
        },
        "fields": {
          "id": "id",
          "content": "text",
          "time": "created_at"
        },
        "pagination": { "type": "page", "maxPages": 100 }
      },
      "exportTo": {
        "api": "/api/dna/memory/import",
        "platform": "mysite"
      }
    }
  }
}
```

### `rule.contentProcessing` — Cleanup (Optional)

```json
{
  "contentProcessing": {
    "cleanup": {
      "removeSelectors": [".ad-banner", ".toolbar"],
      "maxLength": 2000
    }
  }
}
```

## Plugin Naming Convention

Format: `{platform}-{capability}.json`

| Suffix | Capabilities |
|--------|-------------|
| `-automate` | `automate` only |
| `-import` | `import` only |
| `-compose` | `compose` only |
| `-full` | `automate` + `import` + `compose` |

## Directory Structure

```
plugins/
├── zh-CN/          # Chinese platform plugins
│   ├── weibo-automate.json
│   ├── weibo-import.json
│   ├── weibo-compose.json
│   └── weibo-full.json
├── en/             # English platform plugins
│   ├── reddit-automate.json
│   └── reddit-full.json
├── schema.json     # JSON Schema for validation
└── README.md       # This file
```

## Validation

Validate your plugin against the schema:

```bash
npx ajv validate -s plugins/schema.json -d plugins/en/my-plugin.json
```

## Tips

- Use browser DevTools to find the right CSS selectors
- Test selectors in the console: `document.querySelector('.my-selector')`
- Start with `-automate` (simplest), then add `-import` and `-compose`
- Set reasonable delays to avoid rate limiting
- Use `transform: "trim"` on text fields to clean whitespace
