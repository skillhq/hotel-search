# 🏨 hotel-search

Google Hotels search skill for [Claude Code](https://claude.com/claude-code). Find hotel prices, ratings, and availability using browser automation via [agent-browser](https://github.com/nicobailey/agent-browser).

## 📦 Install

```bash
claude skill add skillhq/hotel-search
```

## 🔍 What it does

Ask Claude to search for hotels and it will:

1. Open Google Hotels via `agent-browser`
2. Search using a direct URL (fast path for location) + interactive date setting
3. Extract results and present them as a formatted table with prices, ratings, and amenities

## 💬 Triggers

- "Find hotels in Bangkok, March 15-20"
- "Cheapest 4-star hotels in Paris near the Eiffel Tower"
- "Where to stay in Tokyo Shibuya for 3 nights"
- "Hotel prices in New York for 2 adults and 1 child"
- "Compare hotels in Shibuya vs Shinjuku"

## ✅ Capabilities

| Feature | Method | Notes |
|---------|--------|-------|
| Location search | URL fast path | ~5 commands (location via URL, dates interactive) |
| Check-in / check-out dates | Interactive | Calendar picker after URL loads |
| Guests & rooms | Interactive | Supports multiple rooms, children with ages |
| Star rating filter | Interactive | Filter by 2★ to 5★ |
| Price range filter | Interactive | Min/max price |
| Amenities filter | Interactive | Pool, WiFi, Spa, Breakfast, etc. |
| Free cancellation filter | Interactive | Toggle on/off |
| Hotel detail drill-down | Interactive | Provider price comparison, room types |
| Area comparison | Parallel sessions | e.g., Shibuya vs Shinjuku |

## ⚙️ Requirements

- [agent-browser](https://github.com/nicobailey/agent-browser) CLI installed and available in PATH

## 📁 Files

```
SKILL.md                              # Main skill (triggers, workflow, rules)
references/
  interaction-patterns.md             # Deep-dive cookbook for tricky interactions
```

## 📄 License

MIT
