---
name: find-flights
description: >
  Use this skill when the user asks to search for flights, find the best
  flight, book a flight, or compare flight options. It searches multiple
  candidate dates, ranks options by traveller preferences, and returns prices,
  times, baggage, seat, and routing trade-offs.
---

# Find flights

Search for flights, compare options, and return ranked recommendations with
prices, times, baggage, seat, and routing trade-offs.

## Traveller preferences

**Origin:** MPL (Montpellier) by default. If MPL returns no viable results for
warm or exotic destinations, also search MRS (Marseille, around 1.5 hours by
train) and flag this substitution to the user.

**Ticket type:** One-way unless the destination is USA/transatlantic, in which
case use round-trip. If round-trip is needed and no return date is given, pause
and ask before searching.

**Cabin:**

- Premium Economy for transatlantic flights
- Economy for all others
- **Cheap upgrades:** if a higher cabin (premium economy / business / first) is
  offered for a small marginal cost, roughly single-digit euros or a small
  fraction of the base fare, take it automatically. This overrides the baseline
  cabin above when the upgrade cost is trivial. For larger upgrade costs, more
  than around EUR15-20 or a meaningful fraction of the fare, surface it and let
  the user decide.

**Date flexibility:** +/- 2 days unless told otherwise. Always search all five
candidate dates.

**Preferred times:**

- Depart at or after 10:00, with 07:00 as the absolute earliest departure
- Arrive at or before 21:00, flexible for transatlantic routes

**Connections:** Direct strongly preferred. If no direct options exist across
the full +/- 2-day window, consider 1-stop. Never multi-day stopovers.

**Baggage:** Personal item plus overhead carry-on. No hold luggage assumed. Flag
any checked-bag requirements.

**Seats:** Always include front-of-cabin / extra-legroom options. Flag if seat
fee is more than EUR50 short-haul or EUR100 long-haul.

**Price vs time:** Value 1 hour of travel-time reduction at EUR50. Show
time-vs-price deltas explicitly in the output.

**Fare type:** Non-refundable is the default for both flights and any
accommodation booked alongside them unless the user explicitly asks for
refundable or flexible. Pick the cheapest non-refundable fare/rate. No insurance
add-ons.

**Third-party agents:** If a third-party agent is more than 30% cheaper than
buying direct, pause and ask before proceeding.

## Step 1: Clarify inputs if needed

Before searching, confirm:

1. Destination airport/city
2. Target departure date
3. Return date for transatlantic routes

If the request is clear, proceed without asking.

## Step 2: Primary search via SerpApi

Ensure the `google-search-results` package is installed:

```bash
pip install google-search-results
```

Create a local `.env` file with:

```bash
SERPAPI_KEY=your_serpapi_key_here
```

Search each of the five candidate dates: target date +/- 2 days. For one-way
flights, use `type: "2"`; for round-trip flights, use `type: "1"`.

```python
from dotenv import load_dotenv
import os
from serpapi import GoogleSearch

load_dotenv()
api_key = os.environ["SERPAPI_KEY"]

params = {
    "engine": "google_flights",
    "departure_id": "MPL",
    "arrival_id": "REY",
    "outbound_date": "2026-03-25",
    # "return_date": "2026-04-01",  # include for round-trips (type "1")
    "type": "2",                    # "1" = round-trip, "2" = one-way
    "currency": "EUR",
    "hl": "en",
    "api_key": api_key,
}

r = GoogleSearch(params).get_dict()
flights = r.get("best_flights", []) + r.get("other_flights", [])
```

Key notes:

- `return_date` is required when `type` is `"1"` for round-trip searches.
- The return leg is not included in the outbound response. Run a separate query
  for it.
- Run once per candidate date; collect and deduplicate results.
- If MPL returns no results across all dates, re-run with `departure_id: "MRS"`
  and note the substitution.

After collecting results, discard flights that:

- Depart before 07:00.
- Arrive after 21:00 for short-haul routes. Relax this for transatlantic
  routes.
- Have more than 1 stop.

## Step 3: Fallback via Skyscanner in the browser

Use this only if SerpApi fails because of quota exhaustion, auth errors, or zero
results across all dates and both MPL/MRS.

Open Skyscanner in a browser. URL patterns:

```text
# One-way
https://www.skyscanner.net/transport/flights/{from}/{to}/{YYMMDD}/?adults=1&cabinclass=economy

# Round-trip
https://www.skyscanner.net/transport/flights/{from}/{to}/{YYMMDD}/{YYMMDD_return}/?adults=1&cabinclass=economy
```

`{from}` and `{to}` are lowercase IATA codes. `{YYMMDD}` is the date, e.g.
`260624` for 24 June 2026. The page loads several seconds of JavaScript before
all providers report; wait before reading. Prices render in the browser's locale
currency, not necessarily EUR.

Search target date +/- 2 days, apply direct-only filters first, and if none
exist, search 1-stop options. Apply the same time filters as above, then collect
flight details from the page.

## Step 4: Output

Present the top 3 options in this exact format:

```text
**EUR TOTAL (incl. seat + carry-on add-ons)**
- [Airline] [Flight number(s)]
- [Cabin class]
- [Date] [Dep time] [Dep airport] -> [Date] [Arr time] [Arr airport]
- Duration [h:mm]
- [Direct / 1 stop at XXX for Xh Xm]
**Baggage:** [inclusions + assumed add-ons with EUR]
**Seats:** [front/extra-legroom availability and cost]
**Why ranked #n:** [time vs price delta using EUR50/hr rule]
**Pros:** [one line]
**Cons:** [one line]
```

Then add a **longer shortlist** section: a bullet list of remaining viable
options with airline, times, price, and a one-line note.

## Step 5: Red flags

Pause and ask before proceeding when:

- A transatlantic request lacks a return date.
- Only 1-stop options exist across the entire +/- 2-day window.
- Seat fees exceed the caps: more than EUR50 short-haul or EUR100 long-haul.
- The cheapest option violates the time-value rule but is otherwise attractive.
- A third-party agent is more than 30% cheaper than buying direct.
- A cabin upgrade is available but costs more than the cheap threshold of around
  EUR15-20. Surface it; do not auto-accept it.
