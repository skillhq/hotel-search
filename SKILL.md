---
name: google-hotels
description: Search Google Hotels for hotel prices, ratings, and availability using browser automation. Use when user asks to search hotels, find accommodation, compare hotel prices, check availability, or look up places to stay. Triggers include "search hotels", "find hotels", "hotels in", "where to stay", "accommodation", "hotel prices", "cheapest hotel", "best hotel", "places to stay", "hotel near", "book a hotel", "hotel ratings".
allowed-tools: Bash(agent-browser:*), Bash(echo*), Bash(printf*)
---

# Google Hotels Search

Search Google Hotels via agent-browser to find hotel prices, ratings, amenities, and availability.

## When to Use

- User asks to search, find, or compare hotels or accommodation
- User wants hotel prices in a city, neighborhood, or near a landmark
- User asks about hotel availability for specific dates
- User wants the best-rated or cheapest hotel in an area

## When NOT to Use

- **Booking**: This skill searches only — never complete a purchase
- **Vacation rentals / Airbnb**: Google Hotels shows hotels, not rentals
- **Flights**: Use the flight-search skill
- **Historical prices**: Google Hotels shows current prices only
- **NEVER leave Google Hotels** — do not navigate to Booking.com, Expedia, Hotels.com, or any third-party site. If Google Hotels isn't working, close the session and tell the user. Do not attempt workarounds on other sites.

## Session Convention

Always use `--session hotels` for isolation.

## URL Fast Path (Preferred)

Build a URL with location **and dates** encoded. Loads results directly — **3 commands total**.

### URL Template

```
https://www.google.com/travel/search?q={QUERY}&qs=CAE4AA&ts={TS_PARAM}&ap=MAE
```

Where `{QUERY}` is the location/hotel name and `{TS_PARAM}` is a base64-encoded date parameter.

### Encoding Dates in the URL

Google Hotels uses a protobuf-encoded `ts` parameter for dates. The byte layout is fixed for years 2025-2030. Use this bash function to generate it:

```bash
hotel_ts() {
  local ci_y=$1 ci_m=$2 ci_d=$3 co_y=$4 co_m=$5 co_d=$6 nights=$7
  local cyl=$(printf '%02x' $(( ($ci_y & 0x7f) | 0x80 )))
  local cyh=$(printf '%02x' $(($ci_y >> 7)))
  local col=$(printf '%02x' $(( ($co_y & 0x7f) | 0x80 )))
  local coh=$(printf '%02x' $(($co_y >> 7)))
  echo -n "08011a200a021a00121a12140a0708${cyl}${cyh}10$(printf '%02x' $ci_m)18$(printf '%02x' $ci_d)120708${col}${coh}10$(printf '%02x' $co_m)18$(printf '%02x' $co_d)18$(printf '%02x' $nights)32020801" \
    | xxd -r -p | base64 | tr -d '\n='
}
# Usage: hotel_ts CHECKIN_YEAR MONTH DAY CHECKOUT_YEAR MONTH DAY NIGHTS
# Example: hotel_ts 2026 3 15 2026 3 20 5
# Output: CAEaIAoCGgASGhIUCgcI6g8QAxgPEgcI6g8QAxgUGAUyAggB
```

**Constraints**: Years 2025-2030, months 1-12, days 1-31, nights 1-127. These cover all realistic hotel searches.

### Location Formats

| Format | Example |
|--------|---------|
| City | `Hotels+in+Bangkok` |
| City + country | `Hotels+in+Bangkok+Thailand` |
| Neighborhood | `Hotels+in+Shibuya+Tokyo` |
| Near landmark | `Hotels+near+Eiffel+Tower` |
| Region | `Hotels+in+Amalfi+Coast` |
| Airport | `Hotels+near+BKK+airport` |
| **Specific hotel** | `Haus+im+Tal+Munich` |

### URL vs Interactive Features

| Feature | Via URL? |
|---------|----------|
| Location | ✅ Yes (via `q=`) |
| **Dates** | **✅ Yes (via `ts=`)** |
| Guests / rooms | ❌ Set interactively (default: 1 room, 2 guests) |
| Filters (stars, price, amenities) | ❌ Set interactively |

### Example: Hotels in Bangkok, March 15-20

```bash
# Step 1: Generate ts parameter
ts=$(hotel_ts 2026 3 15 2026 3 20 5)

# Step 2: Open with location + dates — results load immediately
agent-browser --session hotels open "https://www.google.com/travel/search?q=Hotels+in+Bangkok&qs=CAE4AA&ts=${ts}&ap=MAE"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i

# Step 3: Extract results from snapshot, then close
agent-browser --session hotels close
```

### Example: Specific Hotel with Dates

```bash
ts=$(hotel_ts 2026 3 9 2026 3 12 3)
agent-browser --session hotels open "https://www.google.com/travel/search?q=Haus+im+Tal+Munich&qs=CAE4AA&ts=${ts}&ap=MAE"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
agent-browser --session hotels close
```

### Without Dates (Location Only)

If the user doesn't specify dates, omit the `ts`, `qs`, and `ap` parameters:

```bash
agent-browser --session hotels open "https://www.google.com/travel/search?q=Hotels+in+Bangkok"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

Results will show "starting from" prices. Set dates interactively if the user provides them later (see [deep-dive reference](#deep-dive-reference)).

For detailed interactive workflows (calendar navigation, guest/room selector, filters), see the [deep-dive reference](#deep-dive-reference).

## Result Format

Present results as a single rich table:

```
| # | Hotel | Stars | Rating | Price/Night | Total | Via | Key Amenities |
|---|-------|-------|--------|-------------|-------|-----|---------------|
| 1 | Sukhothai Bangkok | ★★★★★ | 9.2 | $185 | $925 | Hotels.com | Pool, Spa, WiFi |
| 2 | Centara Grand | ★★★★★ | 8.8 | $165 | $825 | Booking.com | Pool, Gym, WiFi |
| 3 | Ibis Sukhumvit | ★★★ | 7.4 | $45 | $225 | Agoda | WiFi, Restaurant |
```

## Parallel Sessions

Only use when the user explicitly asks for a comparison between distinct searches (e.g., "Compare hotels in Shibuya vs Shinjuku").

```bash
agent-browser --session shibuya open "https://www.google.com/travel/search?q=Hotels+in+Shibuya+Tokyo" &
agent-browser --session shinjuku open "https://www.google.com/travel/search?q=Hotels+in+Shinjuku+Tokyo" &
wait

agent-browser --session shibuya wait --load networkidle &
agent-browser --session shinjuku wait --load networkidle &
wait

# Set dates in both sessions, then snapshot both
agent-browser --session shibuya snapshot -i
agent-browser --session shinjuku snapshot -i

agent-browser --session shibuya close &
agent-browser --session shinjuku close &
wait
```

## Key Rules

| Rule | Why |
|------|-----|
| Prefer URL fast path with `ts=` | 3 commands with dates vs 10+ interactive |
| `wait --load networkidle` | Smarter than fixed `wait 5000` |
| Always encode dates in URL when available | Use `hotel_ts` to generate the `ts` parameter |
| Use `fill` not `type` for text | Clears existing text first |
| Wait 2s after typing location | Autocomplete needs API roundtrip |
| Click suggestions, never Enter | Enter is unreliable for autocomplete |
| Re-snapshot after every interaction | DOM changes invalidate refs |
| Check for "View prices" | Means dates aren't set yet |
| **Never leave Google Hotels** | Do not navigate to Booking.com, Expedia, etc. |
| Navigate calendar with "<" and ">" | Calendar may open on wrong month — use arrows to go backward or forward |

## Troubleshooting

| Problem | Solution |
|---------|----------|
| Consent popup | Click "Accept all" or "Reject all" |
| URL fast path fails | Fall back to `google.com/travel/hotels` interactive flow |
| No results / "View prices" | Set check-in and check-out dates |
| Calendar opens on wrong month | Use "<" arrow to navigate backward or ">" to go forward. Always re-snapshot after navigating to confirm target month is visible |
| Snapshot blocked by calendar overlay | Press Escape to close the calendar, re-snapshot, then re-open the date picker and try again |
| Stale pricing | Prices are real-time and indicative — mention this caveat |
| Currency mismatch | Google uses browser locale currency — note any discrepancy |
| CAPTCHA | Inform user. Do NOT solve. Retry after a short wait |
| Map view instead of list | Click "List" or "View list" toggle |
| Hotel not found | Try variations: full name, shorter name, name + city, name + neighborhood |

## Deep-Dive Reference

See [references/interaction-patterns.md](references/interaction-patterns.md) for:
- Full annotated walkthrough (every command + expected output)
- Location autocomplete failure modes and recovery
- Date picker calendar navigation
- Guest & room selector (the most complex widget)
- Filters (stars, price, amenities, cancellation)
- Scrolling for more results
- Hotel detail drill-down with provider price comparison
