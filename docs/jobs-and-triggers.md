# Jobs & Triggers – BotanIQal ERP

This document lists the main Apps Script functions, how they are intended to run, and what they do.

---

## 1. Menu Actions (Manual, via onOpen)

**File:** Production Log (monthly)  
**Function:** `onOpen()`

On opening the Production Log, this function adds a custom menu:

> **BotanIQ-Alls Tools**

With the following menu items:

1. **Update Fresh Yields (oz)**
   - Runs: `updateFreshYieldsFromMaster()`
   - Use when:
     - You’ve updated the global *Harvests Yield Summary* and want the production log to reflect new average yields.

2. **Push Month to Master**
   - Runs: `pushMonthToMaster()`
   - Use when:
     - You’ve finalized your monthly plan and/or completed the batch and want to log it into the `MASTER PRODUCTION SHEET`.

3. **Check Resources vs Inventory**
   - Runs: `checkResourcePlanAgainstInventory()`
   - Use when:
     - You’ve filled out the `Resource Usage` tab and want to see which items require ordering.

4. **Push Resource Usage to Inventory**
   - Runs: `pushResourceUsageToInventory()`
   - Use when:
     - You’re ready to commit planned resource usage to the central BotanIQal inventory and log it in `Resource Usage Log`.

5. **Update Resource Usage based on Actuals**
   - Runs: `applyPerProductPackagingVariance()`
   - Use when:
     - You’ve entered bottle/capsule variance into `Production Actuals` and want to adjust inventory to match reality.

6. **Push Bottles Produced to Inventory**
   - Runs: `pushProductionActualsToProductInventory()`
   - Use when:
     - Bottling is complete for the batch and you want to log finished goods into `Product Inventory`.

These menu actions are **explicit** by design so you control when data flows between files.

---

## 2. Time-Based Triggers (Recommended)

### 2.1 Tray Production Alerts

**Function:** `sendBotanIQProductionAlerts()`  
**File:** Workbook containing `Tray Production Schedule`

**Recommended trigger:**

- Type: **Time-driven**
- Frequency: **Daily**
- Time: Early morning (e.g., 5–7 AM local time)

**What it does:**

- Reads all tray runs from `Tray Production Schedule`.
- Calculates for the current date:
  - Seeds to **soak** (soak crops only).
  - Seeds to **drain**.
  - Trays to **sow**.
  - Trays to **move to light**.
  - Trays to **harvest**.
- Sends a daily email to `lukasharacz524@gmail.com`, `morrowethan12@gmail.com`, and `shanegirtain@gmail.com`.

---

## 3. Form-Submit Triggers

### 3.1 Cycle Count Application

**Workbook:** BotanIQal Inventory (Resource Ledger)  
**Function:** `autoApplyCycleCount(e)`

**Recommended trigger:**

- Type: **From form**
- Event: `On form submit`  
  (Bound to the form feeding `Cycle Count (Responses)`.)

**What it does:**

- On each new cycle count submission:
  - Reads resource name, counted quantity, and timestamp.
  - Normalizes resource name to find a match in `Resource Ledger`.
  - If found:
    - Updates **On Hand** and **Last Count Date** for that resource.
  - If not found:
    - Creates a new row with resource, unit, on-hand quantity, and last count date.

This turns cycle counting into a one-click “snapshot” update for inventory.

---

## 4. Manual / As-Needed Scripts

Some functions don’t strictly require a trigger and are better run manually via the menu.

### 4.1 `updateFreshYieldsFromMaster()`

- Already covered under menu actions.
- Could be run weekly or monthly before planning a new batch.

### 4.2 `checkResourcePlanAgainstInventory()`

- Also a menu action.
- Should be run:
  - After setting up a new production plan.
  - Before ordering raw materials.

### 4.3 `pushResourceUsageToInventory()`

- Run once per batch (or per major update) after you finalize `Resource Usage`.

### 4.4 `applyPerProductPackagingVariance()`

- Run after entering actual variance numbers from production into `Production Actuals`.

### 4.5 `pushMonthToMaster()`

- Run once per batch to commit it to the **MASTER PRODUCTION SHEET**.

### 4.6 `pushProductionActualsToProductInventory()`

- Run once per batch to commit finished bottles to the **Product Inventory**.

---

## 5. Summary Table

| Function                               | Where It Lives                  | How It’s Triggered                | Purpose                                                  |
|----------------------------------------|---------------------------------|-----------------------------------|----------------------------------------------------------|
| `onOpen()`                             | Production Log                  | On open                           | Adds “BotanIQ-Alls Tools” menu                          |
| `updateFreshYieldsFromMaster()`        | Production Log                  | Manual (menu)                     | Sync FRESH YIELD (oz) from global Harvest Yields        |
| `pushMonthToMaster()`                  | Production Log                  | Manual (menu)                     | Log batch into MASTER PRODUCTION SHEET                  |
| `checkResourcePlanAgainstInventory()`  | Production Log                  | Manual (menu)                     | Check planned usage vs Resource Ledger inventory        |
| `pushResourceUsageToInventory()`       | Production Log + Inventory file | Manual (menu)                     | Log usage & subtract from inventory                     |
| `applyPerProductPackagingVariance()`   | Production Log + Inventory file | Manual (menu)                     | Adjust inventory for actual packaging variance          |
| `pushProductionActualsToProductInventory()` | Production Log + Product Inventory | Manual (menu)               | Log bottles produced with production & expiry date      |
| `sendBotanIQProductionAlerts()`        | Tray Schedule workbook          | Time-driven (daily)               | Email daily tray production tasks                       |
| `autoApplyCycleCount(e)`               | Resource Ledger workbook        | On form submit                    | Apply cycle counts directly to Resource Ledger          |

---

This setup keeps control in your hands:

- **Manual actions** for “big moves” (pushing to master logs, adjusting inventory).
- **Automated triggers** for repetitive daily tasks and form-driven updates.
