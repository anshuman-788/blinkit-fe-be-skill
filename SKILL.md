---
name: blinkit-fe-be
description: >
  Analyses Blinkit Backend vs Frontend inventory for a defined list of PIDs across all Feeder
  Warehouses. Flags each facility as DEEP RED (FE% < 30%), AMBER (FE% 30–50%), NEUTRAL (50–70%),
  or GREEN (FE% > 70%). For GREEN facilities with no active PO, raises a "Raise PO" alert.
  Use when the user says "run blinkit FE BE analysis", "check blinkit inventory split",
  "blinkit backend frontend", "FE BE check", or "blinkit inventory health".
---

# Blinkit FE/BE Inventory Analysis

Reads the **Backend v Frontend Blinkit** tab and the **Blinkit PO Dump** tab from a single
Google Sheet. Analyses each target PID across all Feeder Warehouses and flags inventory health.

---

## Classification Rules

| Condition | Flag | Meaning |
|-----------|------|---------|
| FE / (BE + FE) < 30% | 🔴 DEEP RED | Frontend critically undersupplied — dark stores nearly empty |
| FE / (BE + FE) 30–50% | 🟡 AMBER | Frontend low — push stock from feeder to dark stores |
| FE / (BE + FE) 50–70% | ⚪ NEUTRAL | Balanced — no action required |
| FE / (BE + FE) > 70% | 🟢 GREEN | Frontend well-stocked — feeder running low, check PO |
| BE = 0 AND FE = 0 | — | Zero stock both sides — skip |

**GREEN + PO check:** For every GREEN facility, check the Blinkit PO Dump for an active PO
(`po_state` = Scheduled or Unscheduled) for that `item_id` × `facility_name` combination.
If none exists → add `⚠ Raise PO` alert. Quantity decision is left to the user.

**Scope:** Feeder Warehouses only. Filter: include only rows where `backend_facility_name`
contains the string "Feeder Warehouse". Exclude CPC, Super Store, CC-DTS, and other facility types.

---

## Step 0 — First-Run Setup

```bash
cat ~/.claude/skills/blinkit-fe-be/config.json 2>/dev/null || echo "NO_CONFIG"
```

**If config exists:** load it and skip to Step 1.

**If NO_CONFIG:** run the setup wizard. Tell the user:
> "Setting up /blinkit-fe-be for the first time. I need your Google Sheet URL, credentials, and the PIDs to analyse."

Ask using AskUserQuestion one at a time:

**Q1:** "Full path to your Google service account JSON key file?"

**Q2:** "Google Sheet URL containing the 'Backend v Frontend Blinkit' and 'Blinkit PO Dump' tabs."

**Q3:** "Comma-separated list of Blinkit item_ids (PIDs) to analyse.
Example: 10169926,10169932,10195000"

Save config:
```bash
python3 - <<'PYEOF'
import json, os, re

creds_path = "CREDS_PATH"
sheet_url  = "SHEET_URL"
pids_raw   = "PIDS_RAW"

def sheet_id(url):
    m = re.search(r'/spreadsheets/d/([a-zA-Z0-9_-]+)', url)
    return m.group(1) if m else url.strip()

pids = [int(p.strip()) for p in pids_raw.split(",") if p.strip().isdigit()]

config = {
    "creds_path": os.path.expanduser(creds_path.strip()),
    "sheet_id":   sheet_id(sheet_url),
    "pids":       pids,
    "fe_be_tab":  "Backend v Frontend Blinkit",
    "po_tab":     "Blinkit PO Dump",
}
path = os.path.expanduser("~/.claude/skills/blinkit-fe-be/config.json")
os.makedirs(os.path.dirname(path), exist_ok=True)
with open(path, "w") as f:
    json.dump(config, f, indent=2)
print(json.dumps(config, indent=2))
PYEOF
```

---

## Step 1 — Fetch and Analyse

Run this single script. It fetches both tabs, filters to Feeder Warehouses and target PIDs,
computes the FE% ratio, cross-references active POs, and prints the full report.

```python
from googleapiclient.discovery import build
from google.oauth2 import service_account
from datetime import date, timedelta
from collections import defaultdict
import json, os

config_path = os.path.expanduser("~/.claude/skills/blinkit-fe-be/config.json")
with open(config_path) as f:
    cfg = json.load(f)

creds = service_account.Credentials.from_service_account_file(
    cfg["creds_path"],
    scopes=["https://www.googleapis.com/auth/spreadsheets.readonly"]
)
svc = build("sheets", "v4", credentials=creds)
SID  = cfg["sheet_id"]
PIDS = set(cfg["pids"])

def fetch(tab):
    r = svc.spreadsheets().values().get(
        spreadsheetId=SID, range=f"'{tab}'!A:Z",
        valueRenderOption="UNFORMATTED_VALUE"
    ).execute().get("values", [])
    if not r: return []
    h = [str(x).strip() for x in r[0]]
    return [{h[i]: (row[i] if i < len(row) else "") for i in range(len(h))} for row in r[1:]]

def sf(v):
    try: return float(str(v).replace(",","").strip())
    except: return None

def to_date(v):
    if isinstance(v, (int, float)) and v > 40000:
        return (date(1899, 12, 30) + timedelta(days=int(v))).isoformat()
    return str(v).strip()

def is_feeder(name):
    return "Feeder Warehouse" in str(name)

# ── Fetch FE/BE data ──────────────────────────────────────────────────────────
febe_rows = fetch(cfg["fe_be_tab"])

# Find latest date in the data
all_dates = sorted(
    set(to_date(r.get("created_at","")) for r in febe_rows if r.get("created_at","")),
    reverse=True
)
latest_date = all_dates[0] if all_dates else None

# Filter: latest date + target PIDs + Feeder Warehouses only
filtered = [
    r for r in febe_rows
    if to_date(r.get("created_at","")) == latest_date
    and sf(r.get("item_id","")) is not None
    and int(sf(r.get("item_id",0))) in PIDS
    and is_feeder(r.get("backend_facility_name",""))
]

# ── Fetch PO Dump ─────────────────────────────────────────────────────────────
po_rows = fetch(cfg["po_tab"])

# Active POs: Scheduled or Unscheduled only
ACTIVE_STATES = {"scheduled", "unscheduled"}
active_po_keys = set()   # (item_id, facility_name)
for r in po_rows:
    state = str(r.get("po_state","")).strip().lower()
    if state not in ACTIVE_STATES: continue
    pid = sf(r.get("item_id",""))
    fac = str(r.get("facility_name","")).strip()
    if pid is not None and int(pid) in PIDS:
        active_po_keys.add((int(pid), fac))

# ── Classify each row ─────────────────────────────────────────────────────────
def classify(fe, be):
    total = fe + be
    if total == 0: return None, None          # both zero — skip
    ratio = fe / total
    if ratio < 0.30:  return "🔴 DEEP RED", ratio
    if ratio < 0.50:  return "🟡 AMBER",    ratio
    if ratio <= 0.70: return "⚪ NEUTRAL",   ratio
    return "🟢 GREEN", ratio

# Build structured data: pid → list of facility records
data = defaultdict(list)
pid_names = {}
for r in filtered:
    pid  = int(sf(r.get("item_id", 0)))
    name = str(r.get("item_name","")).strip()
    fac  = str(r.get("backend_facility_name","")).strip()
    be   = sf(r.get("backend_inv_qty","")) or 0
    fe   = sf(r.get("frontend_inv_qty","")) or 0
    flag, ratio = classify(fe, be)
    if flag is None: continue   # skip both-zero rows
    has_active_po = (pid, fac) in active_po_keys
    po_note = ""
    if flag == "🟢 GREEN" and not has_active_po:
        po_note = "⚠ Raise PO"
    pid_names[pid] = name[:70]
    data[pid].append({
        "facility": fac, "be": int(be), "fe": int(fe),
        "ratio": ratio, "flag": flag, "po_note": po_note
    })

# ── PRINT REPORT ──────────────────────────────────────────────────────────────
today = date.today().isoformat()
print("=" * 72)
print(f"  BLINKIT FE/BE INVENTORY ANALYSIS  |  Data: {latest_date}  |  Run: {today}")
print("=" * 72)
print(f"  Scope: Feeder Warehouses only  |  PIDs: {len(PIDS)}  |  Active rows: {len(filtered)}")

FLAG_ORDER = {"🔴 DEEP RED": 0, "🟡 AMBER": 1, "⚪ NEUTRAL": 2, "🟢 GREEN": 3}

# Overall priority summary across all PIDs
print("\n\n━━━ PRIORITY SUMMARY — DEEP RED FACILITIES (all PIDs) ━━━\n")
all_deep_red = []
for pid, rows in data.items():
    for r in rows:
        if r["flag"] == "🔴 DEEP RED":
            all_deep_red.append((pid, pid_names.get(pid,""), r))
all_deep_red.sort(key=lambda x: x[2]["ratio"])  # worst ratio first

if all_deep_red:
    print(f"  {'PID':>10}  {'Facility':<40} {'BE':>5} {'FE':>5} {'FE%':>6}")
    print(f"  {'─'*10}  {'─'*40} {'─'*5} {'─'*5} {'─'*6}")
    for pid, name, r in all_deep_red:
        print(f"  {pid:>10}  {r['facility']:<40} {r['be']:>5} {r['fe']:>5} {r['ratio']*100:>5.1f}%")
else:
    print("  No DEEP RED facilities across any target PID. ✓")

# PO alerts — all GREEN with no active PO
print("\n\n━━━ PO ALERTS — GREEN FACILITIES WITH NO ACTIVE PO ━━━\n")
po_alerts = []
for pid, rows in data.items():
    for r in rows:
        if r["po_note"]:
            po_alerts.append((pid, pid_names.get(pid,""), r))

if po_alerts:
    print(f"  {'PID':>10}  {'Facility':<40} {'BE':>5} {'FE':>5} {'FE%':>6}")
    print(f"  {'─'*10}  {'─'*40} {'─'*5} {'─'*5} {'─'*6}")
    for pid, name, r in po_alerts:
        print(f"  {pid:>10}  {r['facility']:<40} {r['be']:>5} {r['fe']:>5} {r['ratio']*100:>5.1f}%")
else:
    print("  All GREEN facilities have an active PO. ✓")

# Per-PID detailed breakdown
print("\n\n━━━ FULL BREAKDOWN — PER PID ━━━")
for pid in sorted(data.keys()):
    rows = sorted(data[pid], key=lambda x: FLAG_ORDER[x["flag"]])
    name = pid_names.get(pid, "Unknown")

    counts = defaultdict(int)
    for r in rows: counts[r["flag"]] += 1
    summary = " | ".join(f"{k}: {v}" for k,v in sorted(counts.items(), key=lambda x: FLAG_ORDER[x[0]]))

    print(f"\n  {'─'*70}")
    print(f"  PID {pid}  —  {name}")
    print(f"  {summary}")
    print(f"  {'─'*70}")
    print(f"  {'Facility':<42} {'BE':>5} {'FE':>5} {'FE%':>6}  Status")
    print(f"  {'─'*42} {'─'*5} {'─'*5} {'─'*6}  {'─'*22}")
    for r in rows:
        po = f"  {r['po_note']}" if r["po_note"] else ""
        print(f"  {r['facility']:<42} {r['be']:>5} {r['fe']:>5} {r['ratio']*100:>5.1f}%  {r['flag']}{po}")

# Final action list
print("\n\n━━━ ACTION LIST ━━━\n")
print("  IMMEDIATE (DEEP RED — push stock from feeder to dark stores):")
seen = set()
n = 1
for pid, name, r in all_deep_red:
    key = f"{pid}_{r['facility']}"
    if key not in seen:
        seen.add(key)
        print(f"  {n}. PID {pid} | {r['facility']} | FE% {r['ratio']*100:.1f}%")
        n += 1
if not all_deep_red:
    print("  None.")

print("\n  RAISE PO (GREEN — feeder running low, no active PO):")
n = 1
for pid, name, r in po_alerts:
    print(f"  {n}. PID {pid} | {r['facility']} | FE% {r['ratio']*100:.1f}%  ⚠ Raise PO")
    n += 1
if not po_alerts:
    print("  None.")
```

---

## Arguments

- `/blinkit-fe-be` — full analysis for all configured PIDs
- `/blinkit-fe-be <PID>` — single PID analysis only

To update the PID list without full reconfigure, edit `~/.claude/skills/blinkit-fe-be/config.json`
and change the `"pids"` array.

## Reconfigure

```bash
rm ~/.claude/skills/blinkit-fe-be/config.json
```
