# 🏨 hotel-search

Google Hotels search skill for [Claude Code](https://claude.com/claude-code). Find hotel prices, ratings, and availability using browser automation via [agent-browser](https://github.com/nicobailey/agent-browser).

## 📦 Install

```bash
npx skills add https://github.com/skillhq/hotel-search
```

## 🔍 What It Does

1. Opens Google Hotels via `agent-browser` (URL fast path for location, interactive date setting)
2. Applies filters (stars, price, amenities, cancellation)
3. Presents results as a formatted table with prices, ratings, and amenities

## 💬 Triggers

- "Find hotels in Bangkok, March 15-20"
- "Cheapest 4-star hotels in Paris near the Eiffel Tower"
- "Where to stay in Tokyo Shibuya for 3 nights"
- "Hotel prices in New York for 2 adults and 1 child"
- "Compare hotels in Shibuya vs Shinjuku"

## ✅ Capabilities

| Feature | Method |
|---------|--------|
| Location search | URL fast path |
| Check-in / check-out dates | Interactive (calendar picker) |
| Guests & rooms | Interactive (multiple rooms, children with ages) |
| Star rating filter | Interactive (2★–5★) |
| Price range filter | Interactive (min/max) |
| Amenities filter | Interactive (Pool, WiFi, Spa, etc.) |
| Free cancellation filter | Interactive (toggle) |
| Hotel detail drill-down | Interactive (provider price comparison) |
| Area comparison | Parallel sessions (e.g., Shibuya vs Shinjuku) |

## ⚙️ Requirements

- [agent-browser](https://github.com/nicobailey/agent-browser) CLI installed and available in PATH

## 📁 Files

```
SKILL.md                            # Quick reference (workflow, rules, troubleshooting)
references/
  interaction-patterns.md           # Deep-dive cookbook (complex widget interactions)
```

## 📄 License

MIT
