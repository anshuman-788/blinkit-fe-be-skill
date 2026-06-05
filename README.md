# blinkit-fe-be — Blinkit Backend vs Frontend Inventory Analysis

A Claude Code skill that reads Blinkit's Backend v Frontend inventory data and flags the
health of each Feeder Warehouse for a defined set of PIDs.

## Classification

| FE / (BE + FE) | Flag | Action |
|----------------|------|--------|
| < 30% | 🔴 DEEP RED | Push stock from feeder to dark stores urgently |
| 30–50% | 🟡 AMBER | Frontend low — monitor and push soon |
| 50–70% | ⚪ NEUTRAL | Balanced — no action |
| > 70% | 🟢 GREEN | Frontend healthy — but check if feeder PO exists |
| BE=0, FE=0 | — | Skipped |

For GREEN facilities: if no Scheduled/Unscheduled PO exists in the PO Dump for that
`item_id × facility_name`, the skill flags **⚠ Raise PO**.

## Installation

```bash
npx skills add https://github.com/anshuman-788/blinkit-fe-be-skill
```

Then in Claude Code:
```
/blinkit-fe-be
```

On first run, the skill will ask for your Google Sheet URL, service account credentials,
and the PIDs to analyse.

## Required Google Sheet

A single Google Sheet containing two tabs:
- **Backend v Frontend Blinkit** — columns: `created_at`, `backend_facility_name`,
  `backend_facility_id`, `item_id`, `item_name`, `backend_inv_qty`, `frontend_inv_qty`
- **Blinkit PO Dump** — columns: `po_number`, `facility_name`, `item_id`, `po_state`,
  `units_ordered`, `remaining_quantity`, `appointment_date`

The service account needs read access to this sheet.

## Scope

Only **Feeder Warehouses** are analysed (facility names containing "Feeder Warehouse").
CPC facilities, Super Stores, and other location types are excluded.

Active POs = `po_state` in (Scheduled, Unscheduled). Fulfilled and Expired are ignored.

## Usage

```
/blinkit-fe-be           # full analysis for all configured PIDs
/blinkit-fe-be <PID>     # single PID only
```

## Update PID list

Edit `~/.claude/skills/blinkit-fe-be/config.json` and change the `"pids"` array.
No need to reconfigure from scratch.

## Reconfigure

```bash
rm ~/.claude/skills/blinkit-fe-be/config.json
```

## License

MIT
