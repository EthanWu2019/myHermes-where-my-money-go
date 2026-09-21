# Where My Money Go

A personal expense tracker where the input mechanism is "tell the AI agent" instead of a form.

I tried a handful of personal finance apps over the years and they all share the same problem: every entry is a form — pick a date, pick a category, type a project name, type an amount. By the third entry of the day, I had stopped logging. Bank aggregators (Rocket Money style) solve the input half by reading transactions directly, but they don't cover my case: multiple bank accounts, a Chinese RMB account, USD on a debit card, and recurring charges that split across accounts.

So I built this around a simple idea: I already talk to an AI agent (Hermes) all day. If the entry point is just *saying* "I spent $40 on lunch", the form disappears.

## How it works

```
you say "spent $40 on lunch"
        │
        ▼
  pending.json ............ local JSON queue (single source of truth)
        │
        ▼
  23:55 cron → sync.py .... pushes queue to Notion (账目明细 database)
        │
        ▼
  23:10 cron → generate_data.py ... pulls everything back, writes data/all.json,
                                      commits + pushes to this repo
        │
        ▼
  GitHub Pages ............ serves the static dashboard
```

Three layers, each with one job and one failure mode:

1. **Input** — an expense-tracker skill in Hermes. Natural-language entry, keyword rules infer category and currency (`$` → USD, `块` → RMB, `原神/Steam/月卡` → 游戏, `AI/Token/ChatGPT` → AI). Ambiguous entries get one clarifying question, not five.
2. **Queue + sync** — `pending.json` on the Mac mini. Every entry point (WebUI, QQ bot, future CLI) writes to the same file. The only process that reads it is the nightly cron, which POSTs entries to Notion at 2 req/s and clears the queue on success.
3. **Render** — `generate_data.py` paginates the Notion data source, writes `data/all.json`, and `update_pages.sh` commits and pushes. GitHub Pages serves the dashboard.

## The dashboard

- USD ↔ RMB switch with live FX rate (open.er-api.com)
- Overview / weekly / monthly views with prev/next navigation
- Chart.js pie chart for spend distribution
- 5 hand-rolled CSS themes (Violet / Sharp / Minimal / Neon / Sunset)
- Click a category card to filter the transaction list

Live at **https://ethanwu2019.github.io/myHermes-where-my-money-go/**

The spend data is public on purpose — it's a single `data/all.json` you can fork. I don't mind.

## Data format

`data/all.json`:

```json
{
  "updatedAt": "2026-05-26T00:00:00Z",
  "rate": 6.7987,
  "entries": [
    {
      "name": "明日方舟月卡",
      "amount": 60,
      "currency": "RMB",
      "type": "支出",
      "category": "游戏 🎮",
      "date": "2026-05-26",
      "note": ""
    }
  ]
}
```

Categories: 房租 🏠 / 游戏 🎮 / 饮食 🍜 / 大件 🖥️ / AI 🤖 / 电商 🛒 / 月扣 🔄 / 交通 🚕 / 其他 📦 (支出), 工资 💰 / 拨款 💸 / 零头 🪙 (收入).

## Recurring charges

`recurring.py` + three cron jobs auto-log monthly charges: iCloud on the 3rd, Apple Care on the 10th, split phone bill on the 14th. Each job only writes charges matching today's date, so adding a fourth recurring charge is one new cron + one line in a dict.

## Local development

The repo itself is a static site — no build step. `index.html` + `script.js` + `style.css` + `data/all.json`.

To regenerate data locally (requires Notion credentials):

```bash
python3 generate_data.py
./update_pages.sh
```

## Notes

- If you edit `script.js`, bump the `?v=N` query param in `index.html` — GitHub Pages caches aggressively.
- The sync/render scripts live on the Mac mini, not in this repo; this repo is the public output end of the pipeline.
