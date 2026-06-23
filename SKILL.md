---
name: find-flights
description: >
  This skill should be used when the user asks to search for flights,
  find the best flight, book a flight, or compare flight options.
  Triggers: "find flights", "search flights", "book a flight", "what flights",
  "fly to", "flight options". Returns ranked options with prices, times, and
  your preference rules applied.
---

# Skill: Find flights

## Preferences

**Origin:** MPL (Montpellier) by default. If MPL returns no viable results, also search MRS (Marseille, ~1.5h by train)—flag this substitution. When proposing an MRS departure, budget the ground access realistically: the airport train station (Vitrolles–Aéroport Marseille Provence) is **not** walking distance to the terminals—there's a ~10-minute shuttle to Terminal 1 running only every 10–20 minutes. Add this shuttle time (plus wait) on top of the train journey when working back from the check-in/boarding deadline.

**Ticket type:** Assume one-way unless the destination is USA/transatlantic, in which case use round-trip. If round-trip is needed and no return date is given, pause and ask before searching.

**Cabin:**
* Premium Economy for transatlantic flights
* Economy for all others
* **Cheap upgrades:** if a higher cabin (premium economy / business / first) costs ≤20% more than the base fare, take it automatically; above that, surface it and let the user decide.

**Date flexibility:** ±2 days unless told otherwise. Always search all five candidate dates.

**Preferred times:**
* Depart ≥10:00 (absolute earliest 07:00)
* Arrive ≤21:00 (flexible for transatlantic)

**Connections:** Direct strongly preferred. If no direct options exist across the full ±2-day window, consider 1-stop. Never multi-day stopovers.

**Baggage:** Personal item + overhead carry-on. No hold luggage assumed. Flag any checked-bag requirements.

**Seats:** Always include front-of-cabin / extra-legroom options. Flag if seat fee >€50 short-haul or >€100 long-haul.

**Fast-track security:** Whenever booking a flight or confirming an itinerary, check whether the departure airport (and any connection/return airport) sells a paid fast-track / priority security lane, and book it alongside the flight.

**Price vs time:** Value 1 hour of travel-time reduction at €50. Show time-vs-price deltas explicitly in the output.

**Fare type:** **Non-refundable is always the default**—for both flights and any accommodation booked alongside them—unless the user explicitly asks for refundable/flexible. Pick the cheapest non-refundable fare/rate. No insurance add-ons.

**Accommodation alongside a flight:** If the user also wants a hotel (common for stopovers and late arrivals), use the **find-hotel** skill—it owns the accommodation preferences (default to Booking.com but cross-check direct; non-refundable by default but surface flexible rates when arrival is late or same-day). For Airbnb / short-term rentals use **find-airbnb**.

**Third-party agents:** If a third-party agent is >20% cheaper than buying direct, pause and ask the user before proceeding.

---

## Step 1: Clarify inputs (if needed)

Before searching, confirm:
1. Destination airport/city
2. Target departure date
3. Return date (transatlantic only—if not given, ask)

If the request is clear, proceed without asking.

---

## Step 2: Primary search—SerpApi

Ensure the `google-search-results` package is installed: `pip install google-search-results`

Search each of the 5 candidate dates (target ±2 days). For one-way flights, use `type: '2'`; for round-trip, use `type: '1'`. Fire the 5 dates concurrently.

**Python pattern:**

```python
from dotenv import load_dotenv
from concurrent.futures import ThreadPoolExecutor
import os
from serpapi import GoogleSearch

load_dotenv('.env')          # this skill's .env — see .env.example for the key it needs
api_key = os.environ['SERPAPI_KEY']

CANDIDATE_DATES = [           # target ±2 days
    '2026-03-23', '2026-03-24', '2026-03-25', '2026-03-26', '2026-03-27',
]

def search_date(outbound_date, departure_id='MPL'):
    params = {
        'engine': 'google_flights',
        'departure_id': departure_id,   # adjust as needed
        'arrival_id': 'REY',            # destination IATA code
        'outbound_date': outbound_date,
        # 'return_date': '2026-04-01',  # include for round-trips (type '1')
        'type': '2',                    # '1'=round-trip, '2'=one-way
        'currency': 'EUR',
        'hl': 'en',
        'api_key': api_key,
    }
    r = GoogleSearch(params).get_dict()
    return outbound_date, r.get('best_flights', []) + r.get('other_flights', [])

# Fire all 5 dates concurrently
with ThreadPoolExecutor(max_workers=5) as pool:
    results = dict(pool.map(lambda d: search_date(d), CANDIDATE_DATES))

flights = [f for date_flights in results.values() for f in date_flights]

# Write results to a file so this script can run in the background (see below)
import json
with open('/tmp/find-flights-serpapi.json', 'w') as fh:
    json.dump(flights, fh)
```

**Run it in the background to parallelise with Step 3.** Launch this script with `run_in_background`, then do the Skyscanner browser work (Step 3) while it runs. Collect the JSON output at Step 3b.

**Key notes:**
* `return_date` is required when `type` is `'1'` (round-trip)
* The return leg is NOT included in the outbound response—run a separate concurrent batch for it
* Collect across all dates and deduplicate results
* If MPL returns 0 results across all dates, re-run the same concurrent batch with `departure_id='MRS'` and note the substitution

**Filtering:** After collecting results, discard flights that:
* Depart before 07:00 or after (dep time cut-off for routing constraints)
* Arrive after 21:00 (short-haul; relax for transatlantic)
* Have more than 1 stop

---

## Step 3: Cross-check—Skyscanner via browser (main thread)

Always run a Skyscanner search. SerpApi is faster, but it is not always comprehensive; use Skyscanner as a cross-check for missing routes, airlines, providers, or cheaper fares. If SerpApi fails (quota exhausted, auth error, or 0 results across all dates and both MPL/MRS), use Skyscanner as the primary source instead.

**This step runs on the main thread, not in a subagent**—claude-in-chrome cannot be reliably driven from a subagent. To parallelise, start Step 2's Python script in the background (`run_in_background`) and do this browser work while it runs; reconcile at Step 3b.

**Always access Skyscanner via claude-in-chrome using your _personal_ Chrome profile**—never any other profile, and never plain WebFetch/curl (Skyscanner needs the real browser session to render all providers). For best results, pin the specific profile by filling in your own values (redacted here): display name `1. ******`, on-disk directory `Default`, account `******@*****.***`, extension profile ID `********-****-****-****-************`. Select/confirm this profile (e.g. via `mcp__claude-in-chrome__select_browser` / `list_connected_browsers`) before navigating; if only a non-personal profile is connected, stop and ask the user to connect the personal one rather than proceeding on the wrong profile.

1. Open Skyscanner using `mcp__claude-in-chrome__navigate`
2. URL pattern:
   ```
   # One-way (default):
   https://www.skyscanner.net/transport/flights/{from}/{to}/{YYMMDD}/?adultsv2=1&cabinclass=economy&rtn=0
   # Round-trip:
   https://www.skyscanner.net/transport/flights/{from}/{to}/{YYMMDD}/{YYMMDD_return}/?adultsv2=1&cabinclass=economy&rtn=1
   ```
   `{from}`/`{to}` are lowercase IATA codes; `{YYMMDD}` is the date (e.g. `260624` = 24 Jun 2026). The page loads ~5–7s of JS before all providers report — wait before reading. Prices render in the browser's locale currency (often £ GBP), not necessarily EUR.
3. Search target date ±2 days
4. Apply filters: direct only first; if none, 1-stop
5. Apply time filters matching the preferences above
6. Manually collect flight details from the page

---

## Step 3b: Reconcile (barrier)

Wait for **both** sources before merging: the background SerpApi script (read its `/tmp/find-flights-serpapi.json` output once it finishes) and the Step 3 Skyscanner browser results. Add any viable options that Skyscanner surfaced but SerpApi missed, and flag material price or routing disagreements between the two sources. Only proceed to ranking once both sources are in.

---

## Step 4: Output

Present the **top 3 options** as a table, one row per option:

| # | €Total (incl. add-ons) | Airline / flight | Cabin | Route | Duration | Stops | Baggage | Seats |
|---|---|---|---|---|---|---|---|---|
| 1 | €XXX | [Airline] [Flight no.] | [Cabin] | [Date] [Dep] [DEP]→[Date] [Arr] [ARR] | [h:mm] | [Direct / 1 stop XXX Xh Xm] | [incl + add-ons €] | [front/extra-legroom availability + €] |

Then, beneath the table, for each option:

**#n** — Why ranked: [time vs price delta using €50/hr rule]. Pros: [one line]. Cons: [one line].

Then add a **Longer shortlist** section: a bullet list of remaining viable options with airline, times, price, and 1-line note.
