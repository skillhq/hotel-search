# Google Hotels Interaction Patterns

Deep-dive cookbook for automating Google Hotels with agent-browser. Covers the tricky interactions that require careful handling.

**Related**: [../SKILL.md](../SKILL.md) for the main workflow.

## Contents

- [Full Annotated Walkthrough](#full-annotated-walkthrough)
- [Location Autocomplete Deep Dive](#location-autocomplete-deep-dive)
- [Date Picker Calendar Navigation](#date-picker-calendar-navigation)
- [Guest & Room Selector Deep Dive](#guest--room-selector-deep-dive)
- [Applying Filters](#applying-filters)
- [Scrolling for More Results](#scrolling-for-more-results)
- [Drilling Into a Hotel](#drilling-into-a-hotel)

---

## Full Annotated Walkthrough

A complete command-by-command walkthrough for: **Hotels in Bangkok, March 15-20, 2 adults, 1 room**.

```bash
# ── Step 1: Open Google Hotels via fast path ──
agent-browser --session hotels open "https://www.google.com/travel/search?q=Hotels+in+Bangkok"
agent-browser --session hotels wait --load networkidle
# Google loads hotel results for Bangkok with default dates and guests.
# Prices may show as "starting from" estimates or "View prices" until dates are set.

agent-browser --session hotels snapshot -i
# Expected output: interactive elements including:
#   @e1 [textbox] "Bangkok"             ← location field (pre-filled from URL)
#   @e2 [button] "Check-in"             ← check-in date
#   @e3 [button] "Check-out"            ← check-out date
#   @e4 [button] "Number of travelers"   ← travelers/rooms selector
#   @e5 [link] "Sukhothai Bangkok"       ← first hotel result
#   @e6 [link] "Centara Grand"           ← second hotel result
#   ... more hotel listings
# NOTE: Actual ref numbers WILL vary. Always read the snapshot.

# ── Step 2: Check for consent banner ──
# If you see a consent/cookie dialog, handle it first:
# agent-browser --session hotels click @eN  (Accept all / Reject all)
# agent-browser --session hotels wait 2000
# agent-browser --session hotels snapshot -i

# ── Step 3: Set Check-in Date ──
agent-browser --session hotels click @eN
# Click the check-in date button/field.
# This opens a calendar overlay.

agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Calendar appears showing current and next month.
# Look for month headers and individual day buttons.
# Each day button has a label like "Saturday, March 15"

# Navigate to March if not already visible:
# agent-browser --session hotels click @eN   # Next month arrow
# agent-browser --session hotels wait 1000
# agent-browser --session hotels snapshot -i

# Click March 15 (check-in)
agent-browser --session hotels click @eN   # The "15" button under March
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# Check-in date is now highlighted. Calendar stays open for check-out.

# ── Step 4: Set Check-out Date ──
# Calendar should still be open. March 20 should be visible.
agent-browser --session hotels click @eN   # The "20" button under March
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# Both dates are now selected. A date range is highlighted.

# Click "Done" to confirm dates
agent-browser --session hotels click @eN   # "Done" button
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
# Results now show actual prices per night instead of "View prices".

# ── Step 5: Extract Results ──
# Parse the snapshot for hotel listings.
# Each hotel entry typically contains:
#   - Hotel name (link element)
#   - Star rating (★ symbols or "N-star hotel" text)
#   - Guest rating and review count
#   - Price per night and/or total price
#   - Booking provider (Hotels.com, Booking.com, etc.)
#   - Key amenities

# ── Step 6: Close ──
agent-browser --session hotels close
```

---

## Location Autocomplete Deep Dive

The location autocomplete is the primary source of failures when using the interactive workflow (not an issue with the URL fast path since location is pre-filled).

### How It Works

Google Hotels location fields work as search/combobox elements:
1. Clicking the field focuses it and may show recent searches
2. Typing triggers a debounced API call (~500ms)
3. A dropdown renders with matching locations (cities, neighborhoods, landmarks, hotels)
4. Each suggestion includes location name, type, and sometimes a brief description

### Location Types in Suggestions

| Type | Example | When to Use |
|------|---------|-------------|
| City | "Bangkok, Thailand" | General city-wide search |
| Neighborhood | "Shibuya, Tokyo, Japan" | Specific area |
| Landmark | "Near Eiffel Tower, Paris" | Proximity-based |
| Airport | "Near BKK Airport" | Airport hotels |
| Hotel name | "Sukhothai Bangkok" | Searching for a specific hotel |

### Failure Mode 1: Dropdown Doesn't Appear

**Symptom**: After typing and waiting 2s, snapshot shows no dropdown.

**Recovery**:
```bash
# Try pressing Escape and starting over
agent-browser --session hotels press Escape
agent-browser --session hotels wait 500
agent-browser --session hotels click @eN     # Re-click the field
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

# Clear and retype
agent-browser --session hotels fill @eN "Bangkok"
agent-browser --session hotels wait 3000     # Wait longer this time
agent-browser --session hotels snapshot -i
```

If still no dropdown:
```bash
# Try a more specific query
agent-browser --session hotels fill @eN "Bangkok Thailand"
agent-browser --session hotels wait 3000
agent-browser --session hotels snapshot -i
```

### Failure Mode 2: Wrong Location Selected

**Symptom**: Results show hotels in the wrong city or area.

**Recovery**:
```bash
# Click the location field to re-open it
agent-browser --session hotels click @eN     # Location field
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

# Clear and re-enter
agent-browser --session hotels fill @eN "Shibuya Tokyo"
agent-browser --session hotels wait 2000
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN     # Click correct suggestion
```

### Failure Mode 3: Ambiguous Location

**Symptom**: Multiple similar locations appear (e.g., "Portland, OR" vs "Portland, ME").

**Recovery**: Look at the full text of each suggestion — they usually include state/country:
```
@e12 [listitem] "Portland, Oregon, United States"
@e13 [listitem] "Portland, Maine, United States"
```
Click the correct one based on context.

### Failure Mode 4: "Near X" Queries Not Working

**Symptom**: Typing "near Eiffel Tower" returns no useful suggestions.

**Recovery**:
```bash
# Try the landmark name directly
agent-browser --session hotels fill @eN "Eiffel Tower"
agent-browser --session hotels wait 2000
agent-browser --session hotels snapshot -i
# Look for a location suggestion (not a hotel name)
```

### Best Practices Summary

| Practice | Why |
|----------|-----|
| Prefer URL fast path for location | Skips autocomplete entirely |
| Use `fill` not `type` | Clears existing text first |
| Wait 2-3 seconds after typing | Autocomplete needs API roundtrip |
| Always click the suggestion | Enter doesn't reliably select |
| Use city + country for ambiguous names | Avoids wrong city |
| Re-snapshot after every interaction | DOM changes invalidate refs |

---

## Date Picker Calendar Navigation

### Calendar Structure

The Google Hotels calendar renders as a panel with month grids. Each date is a button with a full label. Key elements:

- **Month headers**: "March 2026", "April 2026"
- **Day buttons**: Each labeled like "Saturday, March 15" — some may show price annotations
- **Navigation arrows**: "<" and ">" to move between months
- **Done button**: Confirms the date selection
- **Reset/Clear**: Clears selected dates

### Setting Check-in and Check-out

The calendar flow for hotels is similar to flights:
1. First click sets **check-in** date
2. Second click sets **check-out** date
3. The range between them is highlighted
4. Click "Done" to confirm

```bash
# Open calendar
agent-browser --session hotels click @eN   # Check-in field
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i

# Click check-in date
agent-browser --session hotels click @eN   # Day button for check-in
agent-browser --session hotels wait 500

# Click check-out date
agent-browser --session hotels click @eN   # Day button for check-out
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Confirm
agent-browser --session hotels click @eN   # "Done"
agent-browser --session hotels wait --load networkidle
```

### Navigating to a Far-Future Month

The calendar may only show 1-2 months at a time. Navigate forward with the arrow:

```bash
agent-browser --session hotels click @eN   # ">" / next month arrow
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Repeat until target month is visible
```

**Tip**: Search the snapshot text for your target date string (e.g., "March 15") to quickly find if it's visible and get the correct ref.

### Dates That Span Months

For a stay from March 28 to April 5:

```bash
# 1. Make sure March is visible
# 2. Click March 28 (check-in)
agent-browser --session hotels click @eN    # "28" in March
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# 3. April should now be visible (calendar may auto-advance)
#    If not, click ">" to navigate
agent-browser --session hotels click @eN    # ">" if needed
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# 4. Click April 5 (check-out)
agent-browser --session hotels click @eN    # "5" in April
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# 5. Confirm
agent-browser --session hotels click @eN    # "Done"
```

### Common Calendar Issues

**Issue: Day numbers are ambiguous** — Snapshot may show "15" twice (once for each visible month).
**Solution**: Look at surrounding context — day elements are grouped under their month header. Pick the one under the correct month.

**Issue: Calendar doesn't close** — Click the "Done" button explicitly. Unlike some date pickers, it doesn't auto-close.

**Issue: Dates seem set but prices still show "View prices"** — Results may need a moment to refresh. Wait for networkidle or add a short wait after clicking "Done".

---

## Guest & Room Selector Deep Dive

The guest/room selector is the **most complex widget** in Google Hotels. It supports:
- Multiple rooms (each with independent guest counts)
- Adults per room
- Children per room (with individual age dropdowns)

### Default State

- 1 room
- 2 adults
- 0 children

### Opening the Selector

```bash
agent-browser --session hotels click @eN   # Button showing "Number of travelers"
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# Expected: a panel with:
#   Room 1:
#     Adults: [−] 2 [+]
#     Children: [−] 0 [+]
#   [Add a room]
#   [Done]
```

### Adjusting Adults

```bash
agent-browser --session hotels click @eN   # "+" next to Adults (Room 1)
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# Adults count increments. Repeat clicking "+" for more.
# Use "−" to decrease.
```

### Adding Children

```bash
agent-browser --session hotels click @eN   # "+" next to Children (Room 1)
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# A child age dropdown may appear immediately or after clicking
```

### Selecting Child Ages

When children are added, Google Hotels requires an age for each child. This creates a dropdown per child:

```bash
# After adding a child, look for an age dropdown
agent-browser --session hotels click @eN   # Age dropdown (may say "Age needed" or show default)
agent-browser --session hotels snapshot -i
agent-browser --session hotels click @eN   # Select age (e.g., "8")
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
```

**Each child has a separate age dropdown.** If you add 2 children, you'll need to set 2 ages.

### Adding Multiple Rooms

```bash
agent-browser --session hotels click @eN   # "Add a room" button
agent-browser --session hotels wait 1000
agent-browser --session hotels snapshot -i
# A new "Room 2" section appears with its own Adults/Children controls
# Repeat adult/child adjustments for Room 2
```

### Removing a Room

```bash
agent-browser --session hotels click @eN   # "Remove room" or "X" next to Room 2
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
```

### Confirming

```bash
agent-browser --session hotels click @eN   # "Done" or "Close"
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Common Guest/Room Issues

**Issue: Age dropdown doesn't open** — Try clicking the dropdown ref, wait, re-snapshot.

**Issue: "+" button doesn't respond** — There may be a maximum (e.g., 14 adults per room). Check the current count.

**Issue: Adding a room resets previous room** — This shouldn't happen, but if it does, set rooms in order and verify each after adding.

---

## Applying Filters

Filters appear as buttons/toggles above the hotel results list. They update results dynamically — no separate "Apply" or "Search" button needed.

### Star Rating Filter

```bash
agent-browser --session hotels click @eN   # "Star rating" or "Stars" button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# Options like: "2+", "3+", "4+", "5"
agent-browser --session hotels click @eN   # Desired star level
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

**Tip**: Some interfaces show individual star checkboxes (2★, 3★, 4★, 5★) — click each one you want. Others show cumulative filters (3+ stars).

### Price Range Filter

```bash
agent-browser --session hotels click @eN   # "Price" filter button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i
# May show a slider or min/max input fields

# If input fields:
agent-browser --session hotels fill @eN "100"   # Min price
agent-browser --session hotels fill @eN "300"   # Max price
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i

# If slider: use click at approximate position or drag
```

### Free Cancellation

```bash
agent-browser --session hotels click @eN   # "Free cancellation" checkbox/toggle
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Amenities Filter

```bash
agent-browser --session hotels click @eN   # "Amenities" or "More filters" button
agent-browser --session hotels wait 500
agent-browser --session hotels snapshot -i

# Common amenity options:
# - Pool, Spa, Gym/Fitness center
# - Free WiFi, Free breakfast, Free parking
# - Restaurant, Bar, Kitchen
# - Air conditioning, Pet-friendly

agent-browser --session hotels click @eN   # Click desired amenity
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### Clearing Filters

To remove a filter:
```bash
# Look for an "X" next to the active filter chip, or click the filter again to deselect
agent-browser --session hotels click @eN   # "X" on active filter or toggle off
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

---

## Scrolling for More Results

Google Hotels initially shows ~10-20 results. To see more:

```bash
# Check if there's a "View more" or "Show more hotels" button
agent-browser --session hotels snapshot -i
# Look for "View more", "Show more", or similar

# If found, click it
agent-browser --session hotels click @eN    # "Show more hotels"
agent-browser --session hotels wait 3000
agent-browser --session hotels snapshot -i

# If no button, try scrolling down
agent-browser --session hotels scroll down 1000
agent-browser --session hotels wait 2000
agent-browser --session hotels snapshot -i
```

Repeat as needed to load more results. Each "Show more" click typically adds another batch.

---

## Drilling Into a Hotel

To view detailed information for a specific hotel:

### Opening Hotel Details

```bash
agent-browser --session hotels click @eN   # Click hotel name or listing
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

### What the Detail Page Shows

The hotel detail page typically contains:

1. **Provider price comparison** — Multiple booking sites with their prices:
   ```
   Hotels.com:  $185/night
   Booking.com: $190/night
   Expedia:     $188/night
   Hotel direct: $180/night
   ```

2. **Room types** — Different room categories with prices:
   ```
   Standard Room:  $185/night
   Deluxe Room:    $220/night
   Suite:          $350/night
   ```

3. **Full amenity list** — Comprehensive list of hotel amenities

4. **Guest reviews** — Rating breakdown, review highlights

5. **Location** — Map, nearby attractions, transit info

### Navigating Back

```bash
agent-browser --session hotels click @eN   # Back arrow or "Back to results" link
agent-browser --session hotels wait --load networkidle
agent-browser --session hotels snapshot -i
```

**Important**: After navigating back, results may have shifted. Re-snapshot and re-identify refs before interacting with other hotels.

### Extracting Provider Comparison

When drilling into a hotel, present the provider comparison:

```
## Sukhothai Bangkok — Provider Comparison

| Provider | Room Type | Price/Night | Total (5 nights) | Cancellation |
|----------|-----------|-------------|-------------------|--------------|
| Hotel direct | Deluxe | $175 | $875 | Free until Mar 10 |
| Hotels.com | Deluxe | $185 | $925 | Free until Mar 12 |
| Booking.com | Deluxe | $190 | $950 | Non-refundable |
| Expedia | Standard | $165 | $825 | Free until Mar 8 |
```
