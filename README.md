# BotanIQal ERP System (Freeze-Dried Microgreens & Supplements)

This repo documents the Google Sheets + Apps Script system I built to run **BotanIQal**, my freeze-dried microgreens & supplement line.

The system handles:

- Monthly **production planning** and feasibility checks (resources + machine hours)
- **Batch traceability** into a master log
- **Resource inventory** with planned vs actual usage and cycle counts
- **Product inventory** with per-batch bottle counts & expiration dates
- Tray production alerts for the microgreen inputs that feed the freeze dryer

This code is not meant as a generic plug-and-play tool for others.  
It’s primarily here to showcase **how I designed and automated my own operations.**

---

## 🧱 Core Components

### 1. Production Log (Monthly)

**File:** `Production Log TEMPLATE`  
This is the main monthly planning and execution workbook.

Key sheets & features:

- **`Overview`**
  - Month & Batch ID (e.g., `December`, `1225`)
  - High-level cost and revenue summary
  - Trays per cycle and freeze dryer hours for feasibility
- **`Products (Demand)`**
  - Per-product units to make and total powder required
- **`Input Guide`**
  - Maps each microgreen input to **fresh yield (oz)** per tray
  - Fresh yields pulled automatically from the global *Harvests Yield Summary* via script
- **`FD Planner`**
  - Freeze dryer capacity and cycle time
  - Calculates whether the monthly plan is feasible given machine hours
- **`Resource Usage`**
  - Planned resource consumption (e.g., powders, bottles, labels, capsules)
  - Script compares planned usage vs current inventory and flags shortages

**Key Apps Script functions:**

- `onOpen()`  
  Creates the **"BotanIQ-Alls Tools"** custom menu with actions like:
  - Update fresh yields
  - Push monthly summary to master
  - Check resource plan against inventory
  - Push resource usage to inventory
  - Apply packaging variance (actuals)
  - Push bottles produced to product inventory

- `updateFreshYieldsFromMaster()`  
  - Reads average yields from the global *Harvests Yield Summary* sheet.
  - Updates the **Fresh Yield (oz)** column in `Input Guide`.
  - Ensures production planning is based on real harvest data.

- `checkResourcePlanAgainstInventory()`  
  - Compares planned usage in `Resource Usage` against on-hand inventory in the BotanIQal **Resource Ledger**.
  - Writes **“Order X unit”** or **“No”** into an `ORDER MORE` column.
  - Applies conditional formatting: green = no issue, red = needs ordering.

- `pushResourceUsageToInventory()`  
  - Aggregates monthly planned usage by resource.
  - Pushes usage into the inventory workbook’s **Resource Usage Log**.
  - Subtracts used quantities from **On Hand** in the **Resource Ledger**.
  - De-duplicates by `(Batch ID, Resource)` so you can’t accidentally double-log usage.

- `applyPerProductPackagingVariance()`  
  - Reads packaging variance from `Production Actuals` (e.g., extra bottles used, extra capsules used).
  - Rolls up total variance for bottles and capsules.
  - Adjusts inventory for items like **Bottles, Labels, Lids, Shrink Seals, Seals, Capsules** in the Resource Ledger.

- `pushMonthToMaster()`  
  - Creates a per-batch summary of:
    - Month, Batch ID
    - Product name, units produced, powder used (g)
    - Trays produced, freeze dryer hours used
    - Feasibility flag
    - Total cost & total revenue
  - Appends these rows into the **MASTER PRODUCTION SHEET** so each batch is traceable in one place.

- `pushProductionActualsToProductInventory()`  
  - Reads per-product **bottles produced** from `Production Actuals`.
  - Pushes / upserts rows into the **Product Inventory** workbook’s `Production Actuals (Batch Log)` with:
    - Batch ID
    - Product name
    - Bottles produced
    - Date produced
    - Expiration date (currently fixed at production date + 2 years)

---

### 2. MASTER PRODUCTION SHEET

**File:** `MASTER PRODUCTION SHEET`  
**Tab:** `Monthly Summary`

- Stores every monthly production log as a **flattened record**.
- Each row = one product line from a given batch, plus shared batch-level fields.
- Acts as a **batch traceability table**:
  - Which month and batch produced which product
  - How many units and grams of powder were used
  - Total trays, freeze dryer hours, cost & revenue
  - Whether the plan was considered “Feasible?”

Populated by `pushMonthToMaster()`.

---

### 3. Resource Ledger (BotanIQal Inventory)

**File:** `Resource Ledger` (BotanIQal)  
Key sheets:

- **`Resource Ledger`**
  - Resource name
  - Unit
  - On Hand quantity
  - Last count date
  - Notes
- **`Resource Usage Log`**
  - Batch ID
  - Resource
  - Unit
  - Quantity used
  - Cost used
- **`Cycle Count (Responses)`**
  - Google Form responses with point-in-time counts per resource.

**Key Apps Script functions:**

- `autoApplyCycleCount(e)`  
  - Triggered on **form submission**.
  - Reads the latest cycle count response:
    - Resource name
    - Counted quantity
    - Unit
    - Timestamp
  - If the resource exists in the ledger:
    - Updates **On Hand** and **Last Count Date**.
  - If it doesn’t exist:
    - Creates a new row with that resource, unit, on-hand quantity, and date.
  - This turns the cycle count form into a one-click inventory adjustment tool.

- The Resource Ledger is also updated by:
  - `pushResourceUsageToInventory()` (planned usage)
  - `applyPerProductPackagingVariance()` (actual packaging variance)

---

### 4. Product Inventory

**File:** `Product Inventory`  
Key sheet:

- **`Production Actuals (Batch Log)`**
  - Batch ID
  - Product name
  - Bottles produced
  - Date produced
  - Expiration date (based on a fixed shelf life)

Populated by `pushProductionActualsToProductInventory()` from each monthly **Production Log**.

This gives a straightforward view of how many bottles were created when, and when they expire.

---

### 5. Tray Production Schedule & Alerts

**Sheet:** `Tray Production Schedule` (in a relevant workbook)  
**Key Apps Script function:**

- `sendBotanIQProductionAlerts()`  
  - Reads each tray **run**:
    - Microgreen
    - Run #
    - Trays this run
    - Sowing date
    - Harvest date
  - Uses soaking + germination logic (similar to MiniLeaf) to compute daily tasks:
    - Seeds to **soak** (for soak crops: beet, pea, sunflower, etc.)
    - Seeds to **drain**
    - Trays to **sow**
    - Trays to **move to light**
    - Trays to **harvest**
  - Sends a daily email like:
    - “Soak These Seeds Today”
    - “Drain These Soaked Seeds Today”
    - “Sow These Trays Today”
    - “Move These Trays to Light Today”
    - “Harvest These Trays Today”

This schedule drives the **fresh microgreen inputs** that are then freeze-dried for BotanIQal products.

---

## How the Pieces Fit Together

See [`docs/overview.md`](docs/overview.md) and [`docs/data-flow.md`](docs/data-flow.md) for a deeper explanation of:

- How data moves from **tray yields → fresh yields → production log → inventory**
- How batch traceability is enforced
- How planned usage vs actual usage vs cycle counts all reconcile into one inventory picture

---

## Notes / Limitations

- Many IDs (spreadsheet IDs, sheet names, cell references, etc.) are **hard-coded** for my setup.
- This is tailored specifically for **BotanIQal** and not meant as a general ERP template.
- That said, the structure shows how to:
  - Use Google Sheets as a lightweight ERP backbone
  - Add Apps Script to handle integrations, batch logging, and inventory logic
