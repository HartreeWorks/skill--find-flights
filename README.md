# Find flights

A skill that finds the best flight for you. Your agent searches five days around your target date, checks Google Flights (via SerpApi) against Skyscanner, and ranks the options against your own rules for times, cabin, baggage, seats and how much an hour saved is worth. Once you've booked, it can add the flight and a busy "Travel day" to Google Calendar.

## Example

> Find me a flight from Montpellier to London around 18 November.

An excerpt from a real run (SerpApi results only, searched 7 October 2026). A full run also cross-checks Skyscanner and adds baggage and seat costs.

| # | €Total | Airline / flight | Route | Duration | Stops |
|---|---|---|---|---|---|
| 1 | €58 | easyJet U2 8390 | 18 Nov 11:15 MPL → 12:10 LGW | 1:55 | Direct |
| 2 | €67 | easyJet U2 8390 | 16 Nov 13:15 MPL → 14:10 LGW | 1:55 | Direct |
| 3 | €90 | easyJet U2 8390 | 20 Nov 18:05 MPL → 19:00 LGW | 1:55 | Direct |

**#1** — Why ranked: cheapest direct option, on the target date, inside the preferred 10:00–21:00 window.

**Longer shortlist**
- Air France, 18 Nov 10:55 → 15:45, €90, 1 stop: €32 more and 3h55 longer than #1, so it loses on both price and time.

## Requirements

- A [SerpApi](https://serpapi.com) key (free tier available), plus Python with `google-search-results` and `python-dotenv`.
- Optional: the claude-in-chrome extension for the Skyscanner cross-check, and the [gogcli](https://github.com/steipete/gogcli) CLI for adding flights to Google Calendar.

## Installation

```bash
npx skills add HartreeWorks/skill--find-flights
```

Then copy `.env.example` to `.env` in the skill folder and add your SerpApi key. Open `SKILL.md` and edit the traveller preferences—home airport, cabin, time windows, the value of an hour—to match your own.

## Try it

Ask your agent:

```text
Find me a flight to Lisbon around 12 March.
```

Your agent searches one-way flights from your home airport for 10–14 March, then presents its top three options in a table, each with a short rationale. A longer shortlist of other viable flights follows.

## Documentation

See [SKILL.md](./SKILL.md) for complete documentation and usage instructions.

## About

Created by [Peter Hartree](https://x.com/peterhartree) of [AI Wow](https://wow.pjh.is).

Find more skills at [HartreeWorks/skills](https://github.com/HartreeWorks/skills).
