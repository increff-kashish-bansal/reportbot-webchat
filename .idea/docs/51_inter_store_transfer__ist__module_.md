# Chapter 51: Inter-Store Transfer (IST) Module

Welcome back! In the [previous chapter](50_inter_warehouse_transfer__iwht__module_.md), we saw how the **Inter-Warehouse Transfer (IWHT)** module helps optimize inventory by moving stock *between warehouses* to better position it for regional demand.

But even after inventory arrives at warehouses and gets allocated to stores ([Chapter 49](49_distribution_allocation_logic_.md)), imbalances can still occur *at the store level*. What happens if one store in a city ends up with too many Size Medium T-shirts, while another store just across town runs out and is missing sales? Sending stock all the way back to a warehouse and then out again is inefficient.

## What Problem Does This Module Solve?

Imagine you're baking cookies and realize you're out of sugar. Your neighbor happens to have an extra bag they don't need right now. Instead of driving all the way to the supermarket (the warehouse), wouldn't it be much faster and easier to just borrow the sugar from your neighbor?

Similarly, in retail, Store A might end up with excess stock of a particular SKU (like Size M T-shirts) perhaps due to lower-than-expected sales, while nearby Store B has sold out and has customers asking for it. The **Inter-Store Transfer (IST) Module** solves the problem of **balancing inventory directly between stores**.

It identifies stores with surpluses (potential outwards) and stores with deficits (potential inwards) for specific SKUs, typically within the same city or region. Based on rules (like proximity, stock levels, sales potential), it suggests optimal **transfers to move stock directly from overstocked locations to understocked ones**. This helps:
*   Reduce stock-outs in high-demand stores.
*   Clear excess inventory from low-demand stores.
*   Improve overall sell-through and customer satisfaction without involving the central warehouse.

## Core Concepts

1.  **Store Imbalances:** The starting point is identifying SKUs where some stores have more stock than needed, while others have less. This "need" vs. "excess" is often determined by comparing current stock against target stock levels or recent sales rates after initial warehouse allocations.
2.  **IST Groups:** Transfers usually don't happen randomly between any two stores. Stores are often grouped geographically (e.g., by city, region, or a defined "IST Group"). Transfers are typically only considered *within* the same group to keep logistics practical and costs down.
3.  **Identifying Sources (Outward Stores):** Stores with stock significantly above their target or with very slow sales for an item become potential sources for sending inventory out.
4.  **Identifying Destinations (Inward Stores):** Stores with stock below their target or experiencing stock-outs for an item become potential destinations for receiving inventory.
5.  **Matching Logic:** The core of IST is matching source stores with destination stores *within the same group* for the same SKU. This matching might prioritize:
    *   Stores with the biggest need.
    *   Stores with the largest excess stock to give.
    *   Geographical proximity (e.g., fulfill need from the closest store with excess first).
6.  **Transfer Quantity:** The amount transferred is usually the *minimum* of the source store's excess quantity and the destination store's needed quantity.
7.  **Pullbacks:** If a store still has excess stock after all possible ISTs within its group are considered (meaning no other nearby store needs it), this excess might be identified for a "pullback" – a transfer back to a designated warehouse.

## How It Works (The Process)

The IST calculation is typically performed by the `DistributionInterStoreTransferModule`, running as part of the `DistributionGroupModule` sequence, usually *after* warehouse allocation ([Chapter 49](49_distribution_allocation_logic_.md)) and inter-warehouse transfers ([Chapter 50](50_inter_warehouse_transfer__iwht__module_.md)). This ensures IST decisions are based on the most up-to-date stock picture after central distribution steps.

**Inputs:**
*   **Post-Allocation Store Stock:** The calculated stock levels in each store *after* warehouse allocations and IWHT have been considered (from `DistributionData`). This includes identifying potential "outward" quantities (excess) and "inward" needs (deficits) per Store-SKU.
*   **IST Group Information:** Mapping defining which stores belong to which city/region/IST group (often derived from `StoreRow` in the [Cache](05_cache_.md)).
*   **Store/Product Master Data:** ([Cache](05_cache_.md)).
*   **Configuration (`IstArgs`):** May contain specific rules or parameters for IST.

**Calculation Steps (Simplified):**
1.  **Identify Inward/Outward Quantities:** Based on the post-allocation stock levels, determine the `inwardQty` (need) and `outwardQty` (excess) for each SKU in each store. Store these in maps (e.g., `distribution.getSkuStoreInwardMap`, `distribution.getSkuStoreOutwardMap`).
2.  **Intra-Store Adjustment (Optional):** Sometimes, a store might paradoxically show both an inward need and an outward excess for the same SKU due to calculation nuances. A small adjustment might net these out first (`intraStoreAllocation`).
3.  **Inter-Store Matching (`interStoreAllocation`):**
    *   Iterate through each SKU with identified needs.
    *   For each store needing the SKU (`inStore`):
        *   Identify its IST Group (e.g., city, region).
        *   Find potential source stores (`outStore`) *within the same IST group* that have an excess (`outwardQty > 0`) of that SKU.
        *   Prioritize potential sources (e.g., stores in the same city first, then same region).
        *   For each potential source, calculate the transferable quantity: `transferQty = min(inStore need, outStore excess)`.
        *   If `transferQty > 0`:
            *   Record the IST: `addIstParentRow(outStore, inStore, sku, transferQty)`.
            *   Decrease the `inStore`'s need and the `outStore`'s excess by `transferQty`.
        *   Continue matching until the `inStore`'s need is met or no more suitable sources are found within the group.
4.  **Identify Pullbacks (`addPullBackStoreStyles`):** After attempting all ISTs, check which stores still have remaining `outwardQty` (excess). Flag this quantity for pullback to a designated warehouse.
5.  **Persist Outputs:** Save the calculated IST transfers (`IstOutputRow`, `IstParentOutputRow`) and potentially pullback information.

**Outputs:**
*   **`IstOutputRow` / `IstParentOutputRow`:** The primary output, listing recommended store-to-store transfers: `fromStore`, `toStore`, `sku`, `qty`.
*   **`ExportIstStoreOutputRow`:** A user-friendly, denormalized version of the IST plan.
*   Pullback data (often implicitly represented by remaining outward quantities or flagged in remarks).

These outputs guide store staff or logistics teams on executing the transfers.

## Under the Hood: Matching Needs and Excess

The core logic lies in efficiently matching stores needing stock with nearby stores having excess.

**1. Walkthrough (Conceptual IST for SKU 123):**
*   **Need:** Store A (City X) needs 15 units. Store B (City Y) needs 10 units.
*   **Excess:** Store C (City X) has 20 units excess. Store D (City X) has 5 units excess. Store E (City Y) has 30 units excess.
*   **Process Store A (Need=15, City X):**
    *   Look for excess in City X. Find Store C (20) and Store D (5).
    *   Prioritize (e.g., highest excess first): Use Store C.
    *   Transfer Qty = min(Need=15, Excess=20) = 15 units.
    *   Record: IST C -> A, SKU 123, Qty 15.
    *   Update: Store A need = 0. Store C excess = 20 - 15 = 5 units.
*   **Process Store B (Need=10, City Y):**
    *   Look for excess in City Y. Find Store E (30).
    *   Transfer Qty = min(Need=10, Excess=30) = 10 units.
    *   Record: IST E -> B, SKU 123, Qty 10.
    *   Update: Store B need = 0. Store E excess = 30 - 10 = 20 units.
*   **Leftover Excess:** Store C (5 units), Store D (5 units), Store E (20 units). These might be candidates for pullback if no other stores in their respective groups need them.

**Sequence Diagram (Simplified IST Matching):**
```mermaid
sequenceDiagram
    participant Mod as DistributionISTModule
    participant Needs as Map<SKU, Map<Store, NeedQty>>
    participant Excess as Map<SKU, Map<Store, ExcessQty>>
    participant Output as IST Output List

    Mod ->> Needs: Get SKU 123 needs (A:15, B:10)
    loop Each Needing Store (InStore)
        Mod ->> Mod: Identify InStore's IST Group (e.g., City)
        Mod ->> Excess: Find Stores (OutStore) in SAME Group with Excess for SKU 123
        Mod ->> Mod: Prioritize OutStores (e.g., by proximity, excess qty)
        loop Each Prioritized OutStore
            Mod ->> Mod: Get current needQty for InStore
            Mod ->> Mod: Get current excessQty for OutStore
            alt needQty > 0 AND excessQty > 0
                Mod ->> Mod: transferQty = min(needQty, excessQty)
                Mod ->> Output: Record IST(OutStore, InStore, SKU, transferQty)
                Mod ->> Needs: Decrease needQty for InStore by transferQty
                Mod ->> Excess: Decrease excessQty for OutStore by transferQty
            end
            If InStore need is now 0, break inner loop.
        end
    end
    Mod ->> Excess: Identify remaining excess for potential Pullback
```

**Code Dive:**

*   **Identifying Inward/Outward Candidates:** This happens implicitly after the main allocation. The `distribution.getSkuStoreInwardMap(sku)` and `distribution.getSkuStoreOutwardMap(sku)` methods return maps populated based on whether a store ended up with a deficit or surplus after warehouse allocation attempts.

*   **The Matching Loop (`interStoreAllocation`, `createIST`):**
    ```java
    // Simplified from DistributionInterStoreTransferModule.java
    private void interStoreAllocation() {
        // Loop through SKUs that have inward needs somewhere
        for (Integer sku : distribution.getSkuStoreInwardMapKeySet()) {
            // Get map of stores needing this SKU -> needed qty
            LinkedHashMap<Integer, Integer> storesNeedingSku = distribution.getSkuStoreInwardMap(sku);
            if (storesNeedingSku == null) continue;

            Set<Integer> inStoresList = storesNeedingSku.keySet();
            for (int inStore : inStoresList) { // For each store needing the SKU...
                int inwardQty = storesNeedingSku.get(inStore); // How much it needs
                if (inwardQty == 0) continue;

                // Find potential sources (stores with outward qty for this SKU)
                // Prioritize by City, then Region, then Group
                String city = cache.getStoreRow(inStore).city;
                String region = cache.getStoreRow(inStore).region;
                String istGroup = distribution.getISTGroupForStore(inStore);
                LinkedHashSet<Integer> outStoresInCity = getOutStoresInGroup(sku, city, istGroup);
                LinkedHashSet<Integer> outStoresInRegion = getOutStoresInGroup(sku, region, istGroup);
                LinkedHashSet<Integer> outStoresInGroup = getOutStoresInGroup(sku, istGroup, istGroup); // All in group

                // Try fulfilling from City first
                for (int outStore : outStoresInCity) {
                    inwardQty = createIST(inwardQty, sku, outStore, storesNeedingSku, inStore);
                    if (inwardQty == 0) break; // Need met
                }
                if (inwardQty == 0) continue; // Move to next needing store

                // Try fulfilling from Region (excluding City already tried)
                for (int outStore : outStoresInRegion) {
                    if(cache.getStoreRow(outStore).city.equals(city)) continue; // Skip same city
                    inwardQty = createIST(inwardQty, sku, outStore, storesNeedingSku, inStore);
                    if (inwardQty == 0) break;
                }
                if (inwardQty == 0) continue;

                // Try fulfilling from anywhere else in the IST Group
                for (int outStore : outStoresInGroup) {
                    if(cache.getStoreRow(outStore).region.equals(region)) continue; // Skip same region
                    inwardQty = createIST(inwardQty, sku, outStore, storesNeedingSku, inStore);
                    if (inwardQty == 0) break;
                }
            }
        }
    }

    // Helper to perform a single transfer
    private int createIST(int inwardQty, Integer sku, int outStore,
                           LinkedHashMap<Integer, Integer> storesNeedingSkuMap, int inStore) {
        if (inwardQty == 0) return 0;
        // Get excess qty at the source store
        int outwardQty = distribution.getOutwardQty(sku, outStore);
        if (outwardQty == 0) return inwardQty; // Source has no excess

        // Calculate transfer amount
        int transferQty = Math.min(outwardQty, inwardQty);

        // Update balances
        inwardQty -= transferQty;
        distribution.decreaseOutwardQty(sku, outStore, transferQty); // Decrease source excess
        storesNeedingSkuMap.put(inStore, inwardQty); // Decrease destination need

        // Record the transfer
        addIstParentRow(outStore, inStore, sku, transferQty);

        return inwardQty; // Return remaining need
    }

    // Helper to record the transfer (aggregates results)
    private void addIstParentRow(int fromStore, int toStore, Integer sku, int qty) {
        Key key = new Key(fromStore, toStore, sku);
        // Get existing row or create new, then add quantity
        istParentOutputRowMap.computeIfAbsent(key,
            k -> new IstParentOutputRow(fromStore, toStore, sku, 0)).qty += qty;
    }
    ```
    **Explanation:** The `interStoreAllocation` method loops through SKUs that are needed somewhere. For each store (`inStore`) needing the SKU, it identifies potential source stores (`outStore`) within progressively larger groups (City -> Region -> IST Group). It calls the `createIST` helper for each potential source. `createIST` checks if the source actually has excess (`outwardQty`), calculates the `transferQty` (minimum of need and excess), updates the remaining need and excess in the `DistributionData` object, and records the transfer using `addIstParentRow`. The loops continue until the need is met or all potential sources within the IST group are exhausted.

*   **Identifying Pullbacks (`addPullBackStoreStyles`):**
    ```java
    // Simplified from DistributionInterStoreTransferModule.java
    private void addPullBackStoreStyles() {
        // Loop through SKUs that might still have outward (excess) quantities
        for (Integer sku : distribution.getSkuStoreOutwardMapKeySet()) {
            // For each store that still has excess for this SKU...
            distribution.getSkuStoreOutwardMap(sku).forEach((store, qty) -> {
                if (qty > 0) { // If excess remains after IST...
                    // Flag this Store-SKU combination for pullback
                    distribution.addPullBackSkuStyle(store, sku, qty);
                    // Record a conceptual transfer to the pullback warehouse
                    addIstParentRow(store, GenericConstants.PULLBACK_WAREHOUSE, sku, qty);
                }
            });
        }
        // ... (The pullbacks might be reported or used by other systems) ...
    }
    ```
    **Explanation:** This code iterates through the remaining outward quantities after all IST matching is done. If a store still has an excess (`qty > 0`), it flags this quantity for potential pullback using `distribution.addPullBackSkuStyle` and records a conceptual transfer to a special `PULLBACK_WAREHOUSE` for reporting purposes.

## Conclusion

The **Inter-Store Transfer (IST) Module**, executed by `DistributionInterStoreTransferModule`, provides a mechanism for **balancing inventory directly between stores**, bypassing the warehouse for efficiency.

*   It identifies stores with **excess stock (sources)** and stores with **deficits (destinations)** for the same SKU, typically within defined geographical **IST groups**.
*   It matches sources to destinations based on **proximity rules** (e.g., City, Region).
*   It calculates the optimal **transfer quantity** (minimum of need and excess) and records the recommended movements (`IstOutputRow`).
*   It identifies any remaining excess stock for potential **pullback** to a warehouse.
*   IST helps **reduce local stock-outs and clear localized excess**, improving overall inventory health and sales potential within regions.

By enabling direct store-to-store movements, IST adds another layer of optimization to the distribution process, making the supply chain more responsive to local imbalances.

What if the goal isn't just regular replenishment, but specifically distributing stock for a major sales event like the End-of-Season Sale (EOSS)?

[Next Chapter: EOSS Distribution Module](52_eoss_distribution_module_.md)
```

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)