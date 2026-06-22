# GFS as a Playground Sandbox: Video Script

---

## 1. Introduction

A database is the one part of an app most people are afraid to touch. Code has Git, so you branch, experiment, and throw the branch away if it doesn't work. A database usually doesn't have that. If you want to try something against real data, you either risk your actual environment or you don't try it at all.

GFS brings the same branch-and-commit model Git gives you for code to your database. `gfs checkout -b` creates an isolated copy of your current database state in seconds, without duplicating the underlying data. You can run queries, change schema, and load test data, anything, inside that branch, and your main database is never touched. If the experiment works, you bring it back. If it doesn't, you just stop using the branch.

---

## 2. Use Case: Loyalty Tiers

In a team retro, marketing flags that the store treats every customer the same. There's no way to tell a regular who's spent hundreds of dollars across a dozen orders from someone who bought one $15 item and never came back. The proposal is to bucket every customer into a loyalty tier, Bronze, Silver, Gold, or Platinum, based on how much they've actually spent, so marketing can target Gold+ customers with early access and give Platinum customers free shipping.

To do this, the customers table needs two new fields: `lifetime_value` and `loyalty_tier`, computed from each customer's order history. That means every existing customer needs to be backfilled at once, not just new ones going forward.

**Why it can't just be run.** This isn't a single insert. It's a calculation across every order ever placed, applied to every existing customer simultaneously. Get the logic wrong and it doesn't fail loudly, it fails by silently mislabeling thousands of customers, which nobody notices until marketing has already emailed the wrong people a Platinum-only promo. The real trap: a naive `SUM(total)` over all of a customer's orders will include refunded and cancelled ones, so a customer who ordered $2,000 worth of items but refunded $1,800 of it still gets counted as a $2,000 spender, Platinum tier, when they should be Bronze.

**The tiers, for reference:**

| Tier | Lifetime spend |
|---|---|
| Bronze | $0 to 199 |
| Silver | $200 to 499 |
| Gold | $500 to 1,499 |
| Platinum | $1,500+ |

**Why it needs a sandbox.** The only way to know the calculation is correct is to run it against real order data and check the output. You write the backfill, run it, and look at how a known customer gets classified. If someone with mostly refunded orders comes out Platinum, something's wrong, fix it on the branch, rerun, and check again. Once real customers land in the right tiers, that's a validated migration, ready to apply to main with confidence instead of hope.

---

## 3. Tools

| Tool | Role |
|---|---|
| **gfs** (CLI) | Branches, commits, and queries the database |
| **Git** | Branches the application code that will consume the validated feature |

With GFS already installed, the first step is to initialize it and pull the actual database into a GFS hosted database:
```bash
gfs init
gfs pull --source <production-database-connection-string>
```

---

## 4. Step by Step

**Check where you stand.** Confirm GFS is tracking this database and that you're currently on `main`.
```bash
gfs status
```

**Open the sandbox.** Clone the current database state into a new branch instantly. Main is now completely isolated from anything that happens next.
```bash
gfs checkout -b feature/loyalty-tiers
```

**Add the new fields.** Both default to safe values, so any customer not touched by the backfill stays Bronze at zero.
```bash
gfs query "ALTER TABLE customers ADD COLUMN lifetime_value NUMERIC DEFAULT 0;
ALTER TABLE customers ADD COLUMN loyalty_tier TEXT DEFAULT 'bronze';"
```

**First backfill attempt.** Sum every order per customer and write it back.
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

**Spot-check a known customer.** Pull up someone who's had a lot of refunds and see what they got assigned.
```bash
gfs query "SELECT name, lifetime_value, loyalty_tier
           FROM customers
           WHERE customer_id = 482;"
```
The result comes back wrong, `lifetime_value` includes the refunded orders, so this customer looks like a Platinum spender despite having returned most of what they bought.

**Fix it.** Only completed orders should count. Customers with zero completed orders are simply never matched by this query, so they correctly stay at the Bronze default.
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
```

**Assign the tiers.**
```bash
gfs query "UPDATE customers
SET loyalty_tier = CASE
  WHEN lifetime_value >= 1500 THEN 'platinum'
  WHEN lifetime_value >= 500  THEN 'gold'
  WHEN lifetime_value >= 200  THEN 'silver'
  ELSE 'bronze'
END;"
```

**Recheck the same customer.** This time the refund-heavy customer correctly lands in Bronze.
```bash
gfs query "SELECT name, lifetime_value, loyalty_tier
           FROM customers
           WHERE customer_id = 482;"
```

**Lock it in.** The backfill is validated. Commit the sandbox state and diff it against main to confirm exactly what changed.
```bash
gfs commit -m "Add loyalty tier backfill"
gfs schema diff main HEAD --pretty
```

**Build the app side in parallel.** Open a matching Git branch and build the tier badge / promo logic against the sandbox.
```bash
git checkout -b feature/loyalty-tiers
```

**It's good, bring it back.** Switch to main, apply the same validated migration there, and commit it for real.
```bash
gfs checkout main
gfs query "<the same ALTER TABLE + UPDATE statements from above>"
gfs commit -m "Add loyalty tier schema and backfill"
```

**Merge the app code normally.**
```bash
git checkout main
git merge feature/loyalty-tiers
```

**If it hadn't worked out:** main was never touched. Just stop using the branch.
```bash
gfs checkout main
git checkout main && git branch -D feature/loyalty-tiers
```

---

## 5. Scalability

Everything above runs locally, `gfs` plus Docker on one machine, fine for a solo experiment. For a team, you don't want five people fighting over one local container, or a migration like this stuck on one laptop.

Guepard's hosted platform extends the same branch/commit model to cloud-hosted, production-identical databases the whole team, or CI, can spin up on demand, with idle environments shutting down automatically so nothing's running (or costing money) when no one's using it. Same workflow, same commands, just remote instead of local. *(Point viewers to docs.guepard.run for exact remote setup, don't improvise commands here on camera.)*

---

## 6. Other Use Cases

The pattern isn't specific to loyalty tiers or to e-commerce. The same shape, *a real backfill or migration that has to be correct before it touches every existing record*, shows up constantly:

- A SaaS team prototyping a new analytics widget before adding it to the main dashboard
- A fintech app testing a new fraud-scoring rule against real transaction history
- A content platform tweaking a recommendation algorithm and checking the results feel right
- A subscription business migrating all existing accounts to a new billing tier structure
- A platform team testing a GDPR-compliant account-deletion cascade against real account data before running it for real

