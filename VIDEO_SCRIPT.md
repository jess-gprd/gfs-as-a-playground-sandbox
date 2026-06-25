# GFS as a Playground Sandbox

## Metadata

| Field | Value |
|-------|-------|
| **Title** | Branch Your Database Like Code GFS Sandbox Demo |
| **Series** | GFS Use Cases |
| **Status** | `draft` |
| **Target Length** | 3–4 min |

**Summary**
> Developers branch code without thinking twice. Databases get no such luxury. This video shows how GFS brings the same branch-and-commit model to your database, using a loyalty tier backfill as the example: a migration that has to be verified against real data before it can be trusted. The viewer leaves knowing how to open a sandbox, catch a silent data bug inside it, fix it, and merge only what's validated.


## Production Setup

### Demo Stack

| Tool | Version / Notes |
|------|-----------------|
| `gfs` CLI | installed, authenticated, tracking local DB |
| Docker | running, GFS local mode |
| PostgreSQL | inside GFS, `main` branch seeded |
| Terminal | clean prompt, font ≥ 18, dark theme |

### Assets

- [ ] Database seeded: `customers` and `orders` tables with realistic data
- [ ] Customer `id = 482` pre-identified: high order count, mostly refunded, should land **Bronze** not Platinum
- [ ] `gfs status` confirmed clean on `main` before recording

### Pre-Roll Checklist

- [ ] Terminal: one pane open, history cleared
- [ ] No other `gfs` branches exist (clean slate visually)
- [ ] Notifications off
- [ ] OBS: all 4 scenes tested, PiP and split layout set
- [ ] Clock hidden from taskbar


## Scene Types

| Tag | Layout | When to use |
|-----|--------|-------------|
| `[INTRO]` | Brand animation | Auto, no action needed |
| `[CAM]` | Full-frame webcam, no screen | Greeting, concept explanations, CTA |
| `[SCREEN]` | Full screen, no cam | Dense code the viewer needs to read without distraction |
| `[SCREEN+CAM]` | Screen with small PiP webcam | Actively running commands or writing code |
| `[SPLIT]` | Cam beside screen, side by side | Talking about something on screen without actively coding |
| `[OUTRO]` | Brand animation | Auto, no action needed |


## Script


### [CAM] Introduction

**line:**
- greet: "Hi, my name is Jess from the Guepard community building team"
- today we're talking about using GFS as a database sandbox
- what the viewer will learn: open a branch, catch a real data bug, merge only what's validated


### [INTRO]


### [CAM] The Problem and the Fix

**line:**
- code is safe to experiment with because of Git, databases have no equivalent
- testing a migration means risking the real environment or skipping the test entirely
- GFS gives databases the same branch model: `gfs checkout -b`, isolated, main never touched

**line:**
- the scenario: marketing wants loyalty tiers based on lifetime spend
- need to add two columns and backfill every existing customer from order history
- the trap: naive `SUM(total)` counts refunded orders, a $200 net spender looks like Platinum


### [SCREEN+CAM] Open the Sandbox

**line:**
- confirm we're on main, then branch

**action:**
```bash
gfs status
gfs checkout -b feature/loyalty-tiers
```

**line:**
- not a dump or a storage clone, just an isolated branch
- main is untouched from this point on


### [SCREEN+CAM] Add the Columns

**line:**
- both columns default to safe values, unmatched rows stay Bronze / zero

**action:**
```bash
gfs query "ALTER TABLE customers ADD COLUMN lifetime_value NUMERIC DEFAULT 0;
ALTER TABLE customers ADD COLUMN loyalty_tier TEXT DEFAULT 'bronze';"
```


### [SCREEN+CAM] First Backfill

**line:**
- first pass: sum all orders per customer, obvious starting point

**action:**
```bash
gfs query "UPDATE customers c
SET lifetime_value = sub.total
FROM (
  SELECT customer_id, SUM(total) AS total
  FROM orders
  GROUP BY customer_id
) sub
WHERE c.customer_id = sub.customer_id;"
```

**line:**
- spot-check customer 482: high order count, mostly refunded

**action:**
```bash
gfs query "SELECT name, lifetime_value, loyalty_tier
           FROM customers
           WHERE customer_id = 482;"
```


### [SPLIT] The Bug

**line:**
- result is wrong, lands Platinum because refunds are included in the sum
- point to the inflated `lifetime_value` on screen
- on main this is already done, here it's just a number, fix and rerun


### [SCREEN+CAM] Fix, Assign, Validate

**line:**
- filter to `status = 'completed'` only, then assign tiers in one pass

**action:**
```bash
gfs query "UPDATE customers c
SET lifetime_value = sub.total
FROM (
  SELECT customer_id, SUM(total) AS total
  FROM orders
  WHERE status = 'completed'
  GROUP BY customer_id
) sub
WHERE c.customer_id = sub.customer_id;"

gfs query "UPDATE customers
SET loyalty_tier = CASE
  WHEN lifetime_value >= 1500 THEN 'platinum'
  WHEN lifetime_value >= 500  THEN 'gold'
  WHEN lifetime_value >= 200  THEN 'silver'
  ELSE                             'bronze'
END;"
```

**line:**
- recheck 482

**action:**
```bash
gfs query "SELECT name, lifetime_value, loyalty_tier
           FROM customers
           WHERE customer_id = 482;"
```


### [SPLIT] Validation

**line:**
- Bronze, correct
- point to the fixed `lifetime_value` on screen
- validated against real data, not intuition


### [SCREEN+CAM] Commit and Ship

**line:**
- commit the sandbox, diff against main to see exactly what changed
- apply the same statements to main, commit for real

**action:**
```bash
gfs commit -m "Add loyalty tier backfill"
gfs schema diff main HEAD --pretty
gfs checkout main
gfs query "-- same ALTER TABLE + UPDATE statements"
gfs commit -m "Add loyalty tier schema and backfill"
```


### [CAM] The Escape Hatch

**line:**
- if it hadn't worked: main was never touched
- just abandon the branch, no rollback, no data surgery
- the cost of failure was zero, that's why you could run the experiment at all


### [OUTRO]


## Notes & Cuts

- Customer 482 must be seeded intentionally: high order count, high refund rate, net spend < $200
- Pause on the schema diff output before moving to the checkout
- "Hope" is the throughline word: use it in the hook and echo it in the escape hatch beat
- Do not show or improvise remote/hosted commands on camera
