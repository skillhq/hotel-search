# Google Hotels Interaction Patterns

Deep-dive cookbook for automating Google Hotels with agent-browser. Covers tricky interactions that need careful handling.

**Quick reference**: [../SKILL.md](../SKILL.md) for the main workflow and key rules.

## Contents

- [Full Annotated Walkthrough](#full-annotated-walkthrough)
- [Specific Hotel Search](#specific-hotel-search)
- [Location Autocomplete](#location-autocomplete)
- [Date Picker Calendar](#date-picker-calendar)
- [Guest & Room Selector](#guest--room-selector)
- [Filters](#filters)
- [Scrolling for More Results](#scrolling-for-more-results)
- [Hotel Detail Drill-Down](#hotel-detail-drill-down)

---

## Full Annotated Walkthrough

Complete command-by-command example: **Hotels in Bangkok, March 15-20, 2 adults, 1 room**.

```bash
# ── Step 1: Open via fast path ──
agent-browser --session hotels open "https://www.google.com/travel/search?q=Hotels+in+Bangkok"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
# Expected elements include:
#   @e1 [textbox] "Bangkok"            ← pre-filled from URL
#   @e2 [button] "Check-in"
#   @e3 [button] "Check-out"
#   @e4 [button] "Number of travelers"
#   @e5 [link] "Sukhothai Bangkok"     ← first hotel result
# Actual ref numbers WILL vary. Always read the snapshot.

# ── Step 2: Handle consent banner (if present) ──
# agent-browser --session hotels click @eN  (Accept all / Reject all)
# agent-browser --session hotels wait 2000
# agent-browser --session hotels snapshot -i

# ── Step 3: Set check-in date ──
agent-browser --session hotels click @eN   # Check-in button — opens calendar
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Navigate to March if needed:
# agent-browser --session hotels click @eN   # Next month arrow

agent-browser --session hotels click @eN   # March 15
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# ── Step 4: Set check-out date ──
agent-browser --session hotels click @eN   # March 20
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

agent-browser --session hotels click @eN   # "Done"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
# Results now show actual per-night prices

# ── Step 5: Extract results ──
# Parse snapshot for: hotel name, star rating, guest rating,
# price/night, booking provider, amenities

# ── Step 6: Close ──
agent-browser --session hotels close
```

---

## Specific Hotel Search

When the user asks about a **specific hotel by name** (not a general area search), use the hotel name directly in the URL:

```bash
agent-browser --session hotels open "https://www.google.com/travel/search?q=Haus+im+Tal+Munich"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

Google typically shows the specific hotel in the sidebar or as the top result. Click into it for pricing and provider comparison, then set dates.

### If the Hotel Doesn't Appear

Try variations:
- Full official name: `Haus+im+Tal+Munich`
- Shorter name: `Haus+im+Tal`
- Name + neighborhood: `Haus+im+Tal+Isarvorstadt`
- Name + country: `Haus+im+Tal+Munich+Germany`

If none work, fall back to a general area search and look for the hotel in the results list.

---

## Location Autocomplete

Only relevant for the interactive workflow (the URL fast path pre-fills location).

### How It Works

1. Click the location field (focuses it, may show recent searches)
2. Type triggers a debounced API call (~500ms)
3. Dropdown renders with matching locations (cities, neighborhoods, landmarks, hotels)
4. Click the correct suggestion — **never press Enter**

### Suggestion Types

| Type | Example |
|------|---------|
| City | "Bangkok, Thailand" |
| Neighborhood | "Shibuya, Tokyo, Japan" |
| Landmark | "Near Eiffel Tower, Paris" |
| Airport | "Near BKK Airport" |
| Hotel name | "Sukhothai Bangkok" |

### Failure Modes

**Dropdown doesn't appear** after typing + 2s wait:

```bash
agent-browser --session hotels press Escape
agent-browser --session hotels wait 500
agent-browser --session hotels click @eN     # Re-click field
agent-browser --session hotels wait 1000
agent-browser --session hotels fill @eN "Bangkok"
agent-browser --session hotels wait 3000     # Wait longer
agent-browser --session hotels snapshot -i
```

Still nothing? Try a more specific query like "Bangkok Thailand".

**Wrong location selected** — results show wrong city:

```bash
agent-browser --session hotels click @eN     # Location field
agent-browser --session hotels wait 1000
agent-browser --session hotels fill @eN "Shibuya Tokyo"
agent-browser --session hotels wait 2000
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN     # Correct suggestion
```

**Ambiguous location** — multiple matches (e.g., Portland OR vs ME): Read the full suggestion text which includes state/country, then click the correct one.

**"Near X" not working**: Try the landmark name directly without "near".

---

## Date Picker Calendar

### Calendar Structure

- **Month headers**: "March 2026", "April 2026"
- **Day buttons**: Labeled like "Saturday, March 15" — some show price annotations
- **Navigation arrows**: "<" / ">" to move between months
- **Done button**: Confirms selection
- **Reset/Clear**: Clears selected dates

### Setting Dates

1. First click = check-in, second click = check-out
2. Range between them highlights
3. Click "Done" to confirm

```bash
agent-browser --session hotels click @eN   # Check-in field — opens calendar
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Check-in day
agent-browser --session hotels wait 500
agent-browser --session hotels click @eN   # Check-out day
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # "Done"
agent-browser --session hotels wait --load networkidle
```

### Navigating Between Months

The calendar may open on the **wrong month** (e.g., showing May when you need March). Use "<" to go backward and ">" to go forward.

```bash
# Navigate BACKWARD (to earlier months)
agent-browser --session hotels click @eN   # "<" previous month arrow
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Repeat until target month is visible

# Navigate FORWARD (to later months)
agent-browser --session hotels click @eN   # ">" next month arrow
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Repeat until target month is visible
```

**Tip**: Search the snapshot text for your target date string (e.g., "March 15") to check visibility. If the month header in the snapshot doesn't match your target, keep navigating.

**Important**: Always re-snapshot after each arrow click. The calendar re-renders and all refs change.

### Cross-Month Stays

For a stay from March 28 to April 5: click March 28 first. The calendar may auto-advance to show April. If not, navigate with ">". Then click April 5 and "Done".

### Common Issues

- **Calendar opens on wrong month**: Use "<" to navigate backward. This is common when Google defaults to a future month. Keep clicking "<" and re-snapshotting until the target month is visible.
- **Snapshot blocked by calendar overlay**: Press Escape to close the calendar, re-snapshot, then re-open the date picker and try again.
- **Ambiguous day numbers** (e.g., "15" in two visible months): Day elements are grouped under month headers — pick the one under the correct month.
- **Calendar doesn't close**: Click "Done" explicitly. It doesn't auto-close.
- **Still shows "View prices" after setting dates**: Wait for networkidle or add a short wait after "Done".

---

## Guest & Room Selector

The most complex widget. Supports multiple rooms, each with independent adult/child counts and per-child age dropdowns.

**Default**: 1 room, 2 adults, 0 children.

### Opening

```bash
agent-browser --session hotels click @eN   # "Number of travelers" button
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Panel shows: Room 1 → Adults [−] 2 [+], Children [−] 0 [+]
#              [Add a room] [Done]
```

### Adjusting Counts

Click "+" or "−" next to Adults/Children. Re-snapshot to confirm.

### Child Ages

Adding a child creates an age dropdown for that child. Each child needs a separate age selection:

```bash
agent-browser --session hotels click @eN   # "+" next to Children
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Age dropdown
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Select age (e.g., "8")
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
```

### Multiple Rooms

```bash
agent-browser --session hotels click @eN   # "Add a room"
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# New "Room 2" section appears — adjust adults/children for it
```

To remove a room, click "Remove room" or "X" next to that room.

### Confirming

```bash
agent-browser --session hotels click @eN   # "Done"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Common Issues

- **Age dropdown won't open**: Click the ref, wait, re-snapshot.
- **"+" not responding**: May have hit a maximum (e.g., 14 adults/room). Check current count.

---

## Filters

Filters appear as buttons above results. They update results dynamically — no "Apply" button.

### Star Rating

```bash
agent-browser --session hotels click @eN   # "Star rating" button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # e.g., "4+"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

Some interfaces show individual checkboxes (2-star, 3-star, etc.) instead of cumulative filters (3+).

### Price Range

```bash
agent-browser --session hotels click @eN   # "Price" filter
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels fill @eN "100"   # Min
agent-browser --session hotels fill @eN "300"   # Max
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Free Cancellation

```bash
agent-browser --session hotels click @eN   # "Free cancellation" toggle
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Amenities

```bash
agent-browser --session hotels click @eN   # "Amenities" or "More filters"
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Desired amenity (Pool, Spa, WiFi, etc.)
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Clearing Filters

Click the "X" on an active filter chip, or click the filter again to deselect.

---

## Scrolling for More Results

Google Hotels initially shows ~10-20 results.

```bash
agent-browser --session hotels snapshot -i
# Look for "Show more hotels" or similar

agent-browser --session hotels click @eN    # "Show more hotels"
agent-browser --session hotels wait 3000
agent-browser --session hotels snapshot -i
```

If no button exists, try scrolling:

```bash
agent-browser --session hotels scroll down 1000
agent-browser --session hotels wait 2000
agent-browser --session hotels snapshot -i
```

---

## Hotel Detail Drill-Down

### Opening Details

```bash
agent-browser --session hotels click @eN   # Hotel name/listing
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### What the Detail Page Shows

- **Provider price comparison**: Hotels.com, Booking.com, Expedia, hotel direct — each with their price
- **Room types**: Standard, Deluxe, Suite — each with prices
- **Amenities**: Full list
- **Reviews**: Rating breakdown and highlights
- **Location**: Map, nearby attractions, transit

### Provider Comparison Format

```
## Sukhothai Bangkok — Provider Comparison

| Provider | Room Type | Price/Night | Total (5 nights) | Cancellation |
|----------|-----------|-------------|-------------------|--------------|
| Hotel direct | Deluxe | $175 | $875 | Free until Mar 10 |
| Hotels.com | Deluxe | $185 | $925 | Free until Mar 12 |
| Booking.com | Deluxe | $190 | $950 | Non-refundable |
| Expedia | Standard | $165 | $825 | Free until Mar 8 |
```

### Navigating Back

```bash
agent-browser --session hotels click @eN   # Back arrow or "Back to results"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

After navigating back, results may have shifted — re-snapshot before interacting with other hotels.
