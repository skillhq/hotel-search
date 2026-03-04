---
name: google-hotels
description: Search Google Hotels for hotel prices, ratings, and availability using browser automation. Use when user asks to search hotels, find accommodation, compare hotel prices, check availability, or look up places to stay. Triggers include "search hotels", "find hotels", "hotels in", "where to stay", "accommodation", "hotel prices", "cheapest hotel", "best hotel", "places to stay", "hotel near", "book a hotel", "hotel ratings".
allowed-tools: Bash(agent-browser:*)
---

# Google Hotels Search

Search Google Hotels via agent-browser to find hotel prices, ratings, amenities, and availability.

## When to Use

- User asks to search/find/compare hotels or accommodation
- User wants to know hotel prices in a city, neighborhood, or near a landmark
- User asks about hotel availability for specific dates
- User wants to find the best-rated or cheapest hotel in an area

## When NOT to Use

- **Booking hotels**: This skill searches only. Do not attempt to complete a purchase.
- **Vacation rentals / Airbnb**: Google Hotels shows hotels, not rental properties.
- **Flights**: Use the flight-search skill instead.
- **Historical price data**: Google Hotels shows current prices, not historical.

## Session Convention

Always use `--session hotels` for isolation.

## Fast Path: URL-Based Search (Preferred)

Navigate to Google's travel search with a location query. This loads a hotel results page directly.

### URL Template

```
https://www.google.com/travel/search?q=Hotels+in+{LOCATION}
```

**Important limitation**: Unlike Google Flights, the hotel URL fast path only supports **location**. Dates, guests, and rooms must be set interactively after the page loads. Expect **~5-7 commands** (not 3 like flights).

### Example: Hotels in Bangkok, March 15-20

```bash
# ── Step 1: Open with location ──
agent-browser --session hotels open "https://www.google.com/travel/search?q=Hotels+in+Bangkok"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i

# ── Step 2: Set check-in date (click date field, pick dates) ──
agent-browser --session hotels click @eN   # Date field / "Check-in" button
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Click the check-in date in the calendar
agent-browser --session hotels click @eN   # March 15
agent-browser --session hotels wait 500
# Click the check-out date
agent-browser --session hotels click @eN   # March 20
agent-browser --session hotels wait 500
agent-browser --session hotels click @eN   # "Done" button
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i

# ── Step 3: Extract results ──
# Results are now visible. Parse the snapshot for hotel listings.

# ── Step 4: Close ──
agent-browser --session hotels close
```

### Location Formats That Work

| Format | Example | Notes |
|--------|---------|-------|
| City name | `Hotels+in+Bangkok` | Most reliable |
| City + country | `Hotels+in+Bangkok+Thailand` | Disambiguates |
| Neighborhood | `Hotels+in+Shibuya+Tokyo` | Specific area |
| Near landmark | `Hotels+near+Eiffel+Tower` | Proximity search |
| Region | `Hotels+in+Amalfi+Coast` | Broader area |
| Airport code | `Hotels+near+BKK+airport` | Near airport |

### What Works via URL

| Feature | URL syntax | Status |
|---------|-----------|--------|
| Location (city) | `Hotels+in+Bangkok` | ✅ Works |
| Location (neighborhood) | `Hotels+in+Shibuya+Tokyo` | ✅ Works |
| Near landmark | `Hotels+near+Eiffel+Tower` | ✅ Works |
| **Dates** | N/A | ❌ Must set interactively |
| **Guests/rooms** | N/A | ❌ Must set interactively |
| **Filters (stars, price)** | N/A | ❌ Must set interactively |

### What Requires Interactive Steps After URL Load

- **Check-in / check-out dates** — Always needed for accurate pricing
- **Number of guests / rooms** — Defaults to 1 room, 2 guests
- **Star rating filter** — Apply after initial results load
- **Price range filter** — Apply after initial results load
- **Amenities filter** (pool, WiFi, parking, etc.)
- **Free cancellation filter**

## Default: Single Session with Rich Results

Hotels don't have a natural binary comparison like economy vs business class. Instead, Google Hotels surfaces **per-hotel provider comparisons** (Hotels.com, Booking.com, Expedia, etc.) with price differences.

Present results as a single rich table:

```
| # | Hotel | Stars | Rating | Price/Night | Total | Via | Key Amenities |
|---|-------|-------|--------|-------------|-------|-----|---------------|
| 1 | Sukhothai Bangkok | ★★★★★ | 9.2 | $185 | $925 | Hotels.com | Pool, Spa, WiFi |
| 2 | Centara Grand | ★★★★★ | 8.8 | $165 | $825 | Booking.com | Pool, Gym, WiFi |
| 3 | Ibis Sukhumvit | ★★★ | 7.4 | $45 | $225 | Agoda | WiFi, Restaurant |
```

**Parallel sessions** — Only use when the user explicitly asks for a comparison between distinct searches:
- "Compare hotels in Shibuya vs Shinjuku"
- "Compare 4-star vs 5-star hotels in Paris"
- "Compare hotels near the beach vs city center in Barcelona"

```bash
# Parallel comparison example: two neighborhoods
agent-browser --session shibuya open "https://www.google.com/travel/search?q=Hotels+in+Shibuya+Tokyo" &
agent-browser --session shinjuku open "https://www.google.com/travel/search?q=Hotels+in+Shinjuku+Tokyo" &
wait

agent-browser --session shibuya wait --load networkidle &
agent-browser --session shinjuku wait --load networkidle &
wait

# Set dates in both sessions, then snapshot both
agent-browser --session shibuya snapshot -i
agent-browser --session shinjuku snapshot -i

# Close both
agent-browser --session shibuya close &
agent-browser --session shinjuku close &
wait
```

## Reading Results from Snapshot

Hotel listings appear in the snapshot as structured elements. Common patterns:

- **Hotel name**: Appears as a heading or `link` element with the hotel name
- **Star rating**: Shown as `★★★★★` or text like "5-star hotel"
- **Guest rating**: Numeric score (e.g., "9.2") or text like "Excellent (1,245 reviews)"
- **Price**: Per-night price and/or total price, with booking provider name
- **Amenities**: Listed as text items (Pool, WiFi, Spa, Free breakfast, etc.)
- **Images**: Thumbnail links (not useful for extraction)

Example snapshot pattern for a single hotel:

```
link "Sukhothai Bangkok"
text "5-star hotel"
text "9.2 Excellent · 2,451 reviews"
text "$185 per night · Hotels.com"
text "Pool · Spa · WiFi · Restaurant"
```

Parse all visible hotel entries into the results table format shown above.

**Tip**: Results may show "starting from" prices even without dates set. These are indicative. For accurate per-night and total pricing, always set check-in/check-out dates. If you see "View prices" links instead of actual prices, dates need to be set.

## Interactive Workflow (Fallback)

Use when the URL path fails (consent banner, CAPTCHA, wrong locale) or when you need to interact with the full form from scratch.

### Open and Snapshot

```bash
agent-browser --session hotels open "https://www.google.com/travel/hotels"
agent-browser --session hotels wait 3000
agent-browser --session hotels snapshot -i
```

If a consent banner appears, click "Accept all" or "Reject all" first.

### Enter Location

```bash
agent-browser --session hotels click @eN   # Location/search field
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

agent-browser --session hotels fill @eN "Bangkok"
agent-browser --session hotels wait 2000   # Wait for autocomplete suggestions
agent-browser --session hotels snapshot -i

agent-browser --session hotels click @eN   # Click the correct suggestion (NEVER press Enter)
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
```

### Set Dates

```bash
agent-browser --session hotels click @eN   # Check-in date field
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Calendar overlay appears showing months

# Navigate to target month if needed
agent-browser --session hotels click @eN   # ">" / next month arrow (if needed)
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

# Click check-in date
agent-browser --session hotels click @eN   # Target check-in day button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Click check-out date
agent-browser --session hotels click @eN   # Target check-out day button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Confirm dates
agent-browser --session hotels click @eN   # "Done" button
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Set Guests and Rooms (if non-default)

The guest/room selector is the most complex widget in hotel search. It supports multiple rooms, each with separate adult/child counts, plus child age dropdowns.

```bash
agent-browser --session hotels click @eN   # Travelers button (e.g., "Number of travelers")
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

# Adjust adults (default is usually 2)
agent-browser --session hotels click @eN   # "+" for Adults
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Add children (if needed)
agent-browser --session hotels click @eN   # "+" for Children
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# A child age dropdown may appear — select the age
agent-browser --session hotels click @eN   # Age dropdown
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Select age (e.g., "8")
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Add another room (if needed)
agent-browser --session hotels click @eN   # "Add a room" button
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Repeat adult/child selection for the new room

# Confirm
agent-browser --session hotels click @eN   # "Done" or "Close" button
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Apply Filters (Optional)

After results load, filters are available above the listings.

**Star rating:**
```bash
agent-browser --session hotels click @eN   # "Stars" or star filter button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # "4+" or "5" stars
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

**Price range:**
```bash
agent-browser --session hotels click @eN   # "Price" filter
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# Adjust slider or enter min/max values
agent-browser --session hotels fill @eN "100"   # Min price
agent-browser --session hotels fill @eN "300"   # Max price
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

**Free cancellation:**
```bash
agent-browser --session hotels click @eN   # "Free cancellation" toggle/checkbox
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

**Amenities:**
```bash
agent-browser --session hotels click @eN   # "Amenities" or "More filters"
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Specific amenity (Pool, Spa, etc.)
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Search / Refresh Results

Unlike flights, hotel results often update dynamically as you change filters and dates. There may not be a separate "Search" button — results refresh automatically after each interaction. Always re-snapshot after applying any filter or changing parameters.

### Drilling Into a Hotel

To view details for a specific hotel (room types, provider price comparison):

```bash
agent-browser --session hotels click @eN   # Click hotel name/link
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
# Hotel detail page shows:
# - Room types with prices
# - Multiple booking providers with price comparison
# - Full amenity list
# - Location map
# - Reviews summary

# Navigate back to results
agent-browser --session hotels click @eN   # Back button or "Back to results"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

## Key Rules

| Rule | Why |
|------|-----|
| Prefer URL fast path | Skips form interaction for location (~5 commands vs 15+) |
| `wait --load networkidle` | Smarter than fixed `wait 5000` — returns when network settles |
| Set dates after URL load | URL doesn't support dates — always set them interactively |
| Use `fill` not `type` for location | Clears existing text first |
| Wait 2s after typing location | Autocomplete needs API roundtrip |
| Always CLICK suggestions, never Enter | Enter is unreliable for autocomplete |
| Re-snapshot after every interaction | DOM changes invalidate refs |
| Check for "View prices" / "starting from" | Set dates for accurate per-night pricing |
| Results update dynamically | No separate "Search" button needed after filter changes |

## Troubleshooting

**Consent popups**: Click "Accept all" or "Reject all" in the snapshot.

**URL fast path didn't work**: Fall back to `google.com/travel/hotels` interactive flow. Some regions/locales handle the URL differently.

**No results / "View prices"**: Dates aren't set. Set check-in and check-out dates to get actual pricing.

**Stale pricing**: Hotel prices are real-time and change frequently. Prices shown are indicative — actual booking prices may differ. Mention this caveat to the user.

**Currency mismatch**: Google shows prices in the local currency of your browser locale. If the user expects a different currency, note the discrepancy.

**CAPTCHA / bot detection**: Inform the user. Do NOT solve CAPTCHAs. Retry after a short wait.

**Map view instead of list**: Google may default to map view. Look for a "List" or "View list" toggle in the snapshot and click it.

## Deep-Dive Reference

See [references/interaction-patterns.md](references/interaction-patterns.md) for:
- Full annotated walkthrough (every command + expected output)
- Location autocomplete failure modes and recovery
- Date picker calendar navigation for hotels
- Guest & room selector deep dive (the most complex widget)
- Applying filters (stars, price, amenities, cancellation)
- Scrolling for more results and pagination
- Drilling into a hotel for provider price comparison
