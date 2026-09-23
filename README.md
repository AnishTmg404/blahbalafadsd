# blahbalafadsd!!
abdiswbksbdka





# Your Project — Complete Explanation

---

## What It Is

A **single-vendor fashion e-commerce web application** (selling t-shirts, pants, shoes, accessories) with **5 warehouses** in different locations across Nepal. After a customer places an order, a **greedy cost optimization algorithm** automatically decides **which warehouse fulfills which item(s)** — including whether to **split the order** across 2 warehouses — to **minimize the store's operational fulfillment cost** (picking, packing, shipping, processing fees).

It is NOT optimizing what the customer pays. It's optimizing what **your business spends** to fulfill the order.

---

## What Problem It Solves

A fashion store with 5 warehouses (KTM, PKR, BRD, BPT, DRT) faces a decision on every order:

- Which warehouse picks and ships the items?
- Should the order be split across 2 warehouses or kept as one shipment?
- Is it cheaper to ship 1 extra item from a farther warehouse than to pay a second pre-shipment fee + packaging?

**Currently**, small/medium stores either:
- Use "nearest warehouse with stock" (no cost comparison)
- Manually decide (admin looks at stock, guesses)
- Use basic OMS rules (first match wins, no cost math)

**Your system** does the math automatically: computes the actual operational cost of every feasible option and picks the cheapest one.

---

## Who Uses It

| Role | What they do |
|------|-------------|
| **Customer** | Browse catalog → add to cart → checkout (enter address/zone) → pay (mock) → see order confirmation with fulfillment details |
| **Admin** | Manage products, warehouses, inventory; set cost parameters; view fulfillment decisions + cost breakdowns; run what-if simulations; generate reports |

---

## Full System Flow (End to End)

### Customer Side

```
1. Customer opens store (browser)
2. Browses products (catalog with search/filter by category: tops, bottoms, shoes, accessories)
3. Adds items to cart
4. Proceeds to checkout:
   - Enters delivery address → selects zone (KTM, PKR, BRD, etc.)
   - System geocodes to lat/long (from zone lookup table)
   - Shows estimated delivery info
5. Confers order (mock payment)
6. Order stored in DB with status = PENDING
7. Customer sees confirmation page:
   "Your order will be fulfilled from: KTM (2 items) + PKR (1 item)"
   "Estimated delivery: 1-2 days"
   [Small map showing warehouse locations + assignment lines]
```

### Backend (Algorithm Triggers Here)

```
8. Order enters PENDING queue
9. Greedy fulfillment engine fires (PHP function):
   a) Load order items
   b) Load all 5 warehouses + their inventory
   c) Load cost parameters from DB (rate_per_km, handling_fee, pre_shipment_fee, packaging_cost)
   d) Compute distance from customer location to each warehouse (Haversine)
   e) Generate all feasible options (single + splits)
   f) Compute operational cost for each option
   g) Pick minimum cost option (greedy choice)
   h) Deduct inventory from assigned warehouse(s)
   i) Create shipment record(s)
   j) Store decision + full cost breakdown in fulfillment_log
   k) Update order status → FULFILLED
```

### Admin Side

```
10. Admin opens dashboard:
    - Sees all orders with fulfillment status
    - Clicks any order → sees:
      "Option 1: All from KTM → Cost 385 ← CHOSEN"
      "Option 2: All from PKR → Cost 620"
      "Option 3: Split (2 KTM + 1 PKR) → Cost 490"
      "Reason: Single warehouse was cheapest"
    - Sees cost breakdown: shipping + handling + pre-ship + packaging
    - Can adjust cost parameters (e.g., "carrier rate increased to 2.0/km")
    - Can re-run pending orders with new parameters
    - Views reports: total fulfillment cost, split rate, warehouse utilization
```

---

## The Algorithm — How It Actually Works

### Input

- **Order**: list of items (e.g., [T-shirt, Pants, Shoes]), total weight, customer's lat/long
- **Warehouses**: 5 locations, each with lat/long, inventory (which SKUs + how many), remaining capacity
- **Cost Parameters** (from DB, admin-editable):
  - `rate_per_km` (carrier cost per km per kg)
  - `handling_fee` (per item — picking + packing labor)
  - `pre_shipment_fee` (per shipment — processing, labeling, handoff)
  - `packaging_cost` (per shipment — box, tape, filler)

### Step 1: Compute Distances (Haversine)

```
For each warehouse:
    distance = haversine(customer_lat, customer_lng, wh_lat, wh_lng)
```

Example (customer in KTM at 27.70, 85.30):
- KTM WH (27.71, 85.32): ~3 km
- PKR WH (28.21, 83.99): ~131 km
- BRD WH (26.47, 87.27): ~215 km
- BPT WH (27.70, 85.34): ~3 km
- DRT WH (27.86, 85.33): ~18 km

### Step 2: Check Feasibility

For each warehouse, check:
- Does it have **all** items in stock? (for single-warehouse options)
- Does it have **enough** of each item? (quantity check)
- Does it have **remaining capacity** for the order weight?

### Step 3: Generate Options

**Type A — Single warehouse (all items from one WH):**
```
For each warehouse that has ALL items + capacity:
    cost = pre_shipment_fee + packaging_cost
         + (num_items × handling_fee)
         + (distance × rate_per_km × total_weight)
```

**Type B — Split (items divided across 2 warehouses):**
```
For each way to split items into group A and group B:
    For each pair (wh1, wh2) where wh1 ≠ wh2:
        If wh1 has all of A AND wh2 has all of B:
            cost = 2 × pre_shipment_fee + 2 × packaging_cost
                 + (|A| × handling_fee) + (|B| × handling_fee)
                 + (dist_wh1 × rate_per_km × weight_A)
                 + (dist_wh2 × rate_per_km × weight_B)
```

**Critical case — MUST split:**
If NO single warehouse has all items, only Type B options are generated. The algorithm finds the cheapest split.

### Step 4: Greedy Decision

```
best = MIN(all_options, by = cost)
```

Pick the option with the lowest total operational cost. Done.

### Step 5: Execute

- Deduct inventory from assigned warehouse(s)
- Create 1 or 2 shipment records
- Log the decision: which options were evaluated, their costs, which was chosen, why
- Update order status → FULFILLED

### Complexity

For 5 warehouses and up to 5 items per order:
- Single options: max 5
- Split options: max C(5,2) × 2^(items-1) = 10 × 16 = 160
- Total: ~165 cost calculations per order
- **Execution time: < 1 millisecond**

---

## The Cost Model (Why It's Parameterized)

All cost values live in a `cost_params` table in MySQL:

| param_name | value | unit | meaning |
|-----------|-------|------|---------|
| rate_per_km | 1.5 | NPR/km/kg | Carrier shipping rate |
| handling_fee | 30 | NPR/item | Picking + packing labor |
| pre_shipment_fee | 200 | NPR/shipment | Processing, labeling, handoff |
| packaging_cost | 50 | NPR/shipment | Box, tape, filler |

**Why parameterized?**
- Next year, carrier rate might increase to 2.0 → admin updates one DB row
- New warehouse opens with a promotional pre-shipment fee of 50 → admin sets it
- No code changes needed. Algorithm reads values at runtime.

---

## The Map / Distance Component

- Each warehouse has fixed **lat/long** in the database (set once by admin)
- Customer selects a **zone** at checkout (dropdown: KTM, PKR, BRD, BPT, DRT, etc.)
- Each zone has a **representative coordinate** in a `zones` table
- Distance = **Haversine formula** (pure math, no external API)
- Optional frontend: **Leaflet.js map** (free, open-source) showing:
  - Customer location (blue dot)
  - 5 warehouse pins
  - Lines from customer → assigned warehouse(s)
  - If split: two colored lines with labels ("2 items — 3 km", "1 item — 131 km")

---

## What Makes It Different From Other BCA E-Commerce Projects

| Typical BCA E-Commerce Project | Yours |
|-------------------------------|-------|
| Single warehouse, flat shipping | 5 warehouses, distance-based cost |
| "Ship from warehouse" (no decision) | **Algorithm decides which WH + whether to split** |
| No cost logic | **Parameterized cost model with real math** |
| No algorithmic depth | **Greedy optimization with option evaluation** |
| No admin analytics | **Cost breakdown per decision, what-if, reports** |
| CRUD only | CRUD + **decision engine** |

---

## Tech Stack

| Layer | Technology |
|-------|-----------|
| Frontend | HTML, CSS, JavaScript (+ Leaflet.js for map) |
| Backend | PHP 8.x (plain, no framework) |
| Database | MySQL |
| Algorithm | PHP (greedy engine, ~250 lines) |
| Server | XAMPP (Apache + MySQL) |
| Version Control | Git + GitHub |

---

## Key Modules

1. **User Auth** — register/login, session-based
2. **Product Catalog** — CRUD, search, filter by category
3. **Cart & Checkout** — add to cart, enter address/zone, mock payment
4. **Greedy Fulfillment Engine** — the core algorithm (generates options, computes costs, picks minimum)
5. **Warehouse Management** — CRUD, inventory per warehouse, capacity
6. **Cost Parameter Manager** — admin edits rates/fees in DB
7. **Order Management** — status tracking, fulfillment log, decision history
8. **Analytics & Reports** — total cost, split rate, warehouse utilization, what-if simulation

---

## One-Sentence Summary

> A fashion e-commerce store with 5 warehouses where a greedy algorithm automatically decides the cheapest way to fulfill each order — from which warehouse(s) to ship and whether splitting saves money — using a parameterized cost model and map-based distance calculation.

