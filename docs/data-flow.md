# Data Flow – BotanIQal ERP

This document shows how data moves through the BotanIQal ERP system:

1. Microgreen tray yields → fresh yield table
2. Fresh yields + demand → monthly production plan
3. Production plan → resource usage & feasibility
4. Execution → actuals, variances, and inventory updates
5. All batches → master production and product inventory logs

---

## 1. Microgreen Yields → Fresh Yield Table

**Source:**

- Global `Harvests Yield Summary` spreadsheet
  - Each row: `Microgreen`, `Avg Yield (oz)` plus other stats.

**Process:**

- `updateFreshYieldsFromMaster()` in the Production Log:
  1. Opens the master yield sheet via `MASTER_YIELD_SHEET_ID`.
  2. Builds a lookup table:
     - key: normalized microgreen name (lowercase, trimmed).
     - value: average fresh yield in ounces.
  3. Reads the Production Log’s `Input Guide` sheet.
  4. For each input:
     - Finds the matching average yield.
     - Writes it into the **FRESH YIELD (oz)** column.

**Destination:**

- Updated **FRESH YIELD (oz)** values in `Input Guide` drive:
  - Trays required to produce a target amount of powder.
  - Freeze dryer capacity calculations.

---

## 2. Demand → Monthly Production Plan

**Source:**

- `Products (Demand)` in the monthly Production Log:
  - Product name
  - Units to produce
  - Total powder (g) per product

**Internal calculations (within the Production Log):**

- Using fresh yields and desired powder per product, the log computes:
  - Total trays required per input.
  - Total powder required for all products.
  - Freeze dryer cycles and total hours (`FD Planner`).
  - Whether the plan is **Feasible?** given a one-month time window and FD capacity.

**Destination:**

- A planned **monthly production scenario** that answers:
  - “What do we want to make?”
  - “How many trays does that require?”
  - “Can our freeze dryer and time realistically handle this?”

---

## 3. Planned Resource Usage → Inventory & Ordering

**Source:**

- `Resource Usage` in the Production Log:
  - Resource
  - Unit
  - Quantity used (planned)
  - Cost used (planned)

**Processes:**

### 3.1 Check Feasibility Against On-Hand Inventory

- `checkResourcePlanAgainstInventory()`:
  1. Reads on-hand quantities from the BotanIQal `Resource Ledger`.
  2. For each resource in `Resource Usage`:
     - Compares planned usage vs on-hand quantity.
     - Writes into `ORDER MORE` column:
       - `"No"` if sufficient stock.
       - `"Order X unit"` if short.
  3. Applies conditional formatting to highlight shortages in red.

Result: a **pre-flight check**: “Do we have enough materials to run this batch?”

### 3.2 Log Planned Usage and Subtract from Inventory

- `pushResourceUsageToInventory()`:
  1. Aggregates rows from `Resource Usage` by normalized resource name.
  2. Opens the BotanIQal **Inventory** spreadsheet:
     - `Resource Ledger` (on-hand table).
     - `Resource Usage Log` (historical usage log).
  3. Builds a de-dupe key per `(Batch ID, Resource)` to avoid double logging.
  4. For each resource:
     - If already logged for this batch → skip.
     - Otherwise:
       - Appends a new row to `Resource Usage Log`:
         - `[Batch ID, Resource, Unit, Qty Used, Cost Used]`.
       - Subtracts quantity from `On Hand` in `Resource Ledger`.
       - If resource doesn’t exist in the ledger, creates a new row with negative balance.

Result: **planned resource usage** is recorded and reflected in inventory.

---

## 4. Execution → Actuals, Variance & Cycle Counts

Once production happens, there are three important realities:

1. Packaging usage may differ from the plan.  
2. Some inventory items will be physically counted.  
3. Final bottles produced must be logged for finished goods inventory.

### 4.1 Packaging Variance

**Source:**

- `Production Actuals` in the Production Log:
  - Per line: variance in bottles and capsules.

**Process:**

- `applyPerProductPackagingVariance()`:
  1. Sums all `VARIANCE BOTTLES` and `VARIANCE CAPS` across rows.
  2. Opens the BotanIQal inventory’s `Resource Ledger`.
  3. Adjusts On Hand for:
     - Bottles, Labels, Lids, Shrink Seals, Seals (based on bottles variance).
     - Capsules (based on capsules variance).
  4. Writes adjusted values back to the ledger.

Result: inventory reflects **actual packaging consumption**, not just the original plan.

---

### 4.2 Cycle Counts (Physical Inventory)

**Source:**

- Google Form responses into `Cycle Count (Responses)` sheet in the inventory workbook:
  - Resource
  - Counted quantity
  - Unit
  - Timestamp
  - Notes

**Process:**

- `autoApplyCycleCount(e)` (onFormSubmit trigger):
  1. Reads the new form response row.
  2. Normalizes the resource name to a key.
  3. If the resource exists in `Resource Ledger`:
     - Updates `On Hand` and `Last Count Date`.
  4. If it does not:
     - Inserts a new row for that resource, with the counted quantity and date.

Result: cycle counts become **live corrections** to inventory without manual spreadsheet editing.

---

### 4.3 Bottles Produced → Product Inventory

**Source:**

- `Production Actuals` in the Production Log:
  - Product name
  - Bottles produced per product line
- `Overview!B3`:
  - Batch ID

**Process:**

- `pushProductionActualsToProductInventory()`:
  1. Reads all rows where both Product Name and Bottles Produced are filled.
  2. Opens **Product Inventory** spreadsheet, tab `Production Actuals (Batch Log)`.
  3. Builds an index by `(Batch ID, Product Name)` to support upsert.
  4. For each product:
     - If a row already exists:
       - Updates bottles produced.
     - Else:
       - Appends a new row:
         - `[Batch ID, Product Name, Bottles Produced, Date Produced, Expiration Date]`
         - Expiration Date = Date Produced + fixed shelf life (currently 2 years).

Result: every run populates a **batch-level finished goods log** for traceability.

---

## 5. Master Batch Flow

Putting it together:

1. **Plan batch** in Production Log.
2. **Check** resource feasibility & machine hours.
3. **Approach production**:
   - Tray Schedule + `sendBotanIQProductionAlerts()` drive microgreen runs.
4. **Push plan**:
   - `pushResourceUsageToInventory()` logs planned usage and subtracts from inventory.
5. **Execute batch** and log:
   - `applyPerProductPackagingVariance()` fixes inventory for packaging differences.
   - `autoApplyCycleCount(e)` applies physical counts.
   - `pushProductionActualsToProductInventory()` logs finished goods.
6. **Summarize batch**:
   - `pushMonthToMaster()` sends a one-record-per-product snapshot to MASTER PRODUCTION SHEET.

This gives a complete chain from **fresh microgreens → freeze dryer → bottled product → inventory & traceability**.
