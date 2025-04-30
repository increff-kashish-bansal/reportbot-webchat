# Chapter 19: InventoryComputationUtil

Welcome back! In the [previous chapter](18_skustyleconversionutil_.md), we learned about [SkuStyleConversionUtil](18_skustyleconversionutil_.md), a tool that helps us correctly handle product data when there are parent-child relationships (like a T-shirt style and its different size SKUs). This ensures our sales and stock data accurately reflects the right product level.

Now that we have accurate data, let's think about a fundamental question in retail: "Did we actually have this product in stock on a particular day in the past?" Just knowing the current stock isn't enough if we want to analyze *past* performance accurately.

**Use Case:** Imagine you're analyzing why a specific blue T-shirt (SKU #12345) sold poorly last month in your downtown store (Store #101). You look at the sales data and see zero sales on many days. Was it because customers didn't want it, or was it simply because the T-shirt was **out of stock** on those days? To know this, you need to figure out the inventory level for SKU #12345 in Store #101 for *every single day* last month.

## What Problem Does InventoryComputationUtil Solve?

Calculating historical inventory levels day-by-day isn't trivial. Stock doesn't stay constant; it changes daily due to various events:
*   **Sales:** Stock decreases when items are sold.
*   **Returns:** Stock increases when items are returned.
*   **Inwards:** Stock increases when new deliveries arrive (from warehouses or suppliers).
*   **Outwards:** Stock decreases due to transfers to other stores, write-offs for damage, etc.

Manually tracking this for thousands of SKUs across hundreds of stores is impossible. We need an automated engine that can simulate this daily stock flow based on historical data.

**`InventoryComputationUtil`** is this engine. It's the core calculator in `irisx-algo` responsible for reconstructing the historical inventory levels for every SKU in every store, day by day. It takes starting stock information and daily transaction data to figure out on which specific days a product actually had stock (> 0) and was available for sale (a "live day").

Knowing these "live days" is critical. You can't sell something you don't have! Accurate performance metrics, like Rate of Sale (ROS), depend heavily on knowing how many days an item was *actually* available to be sold.

## Core Idea: The Day-by-Day Stock Simulator

Think of `InventoryComputationUtil` like keeping a detailed daily diary for the stock of each product in each store.

1.  **Starting Point:** We need a known stock level on a specific date. This is called a **keyframe** snapshot (`InvKeyFrameRow`). It's like knowing your bank balance on the 1st of the month.
2.  **Daily Changes:** We then process the events for each following day:
    *   Sales (-)
    *   Returns (+)
    *   Inwards (+)
    *   Outwards (-)
3.  **Calculate Next Day's Stock:** We apply these changes to the previous day's stock using a simple formula:
    `Stock[Today] = Stock[Yesterday] - Sales[Today] + Returns[Today] + Inwards[Today] - Outwards[Today]`
    *(For simplicity, returns, inwards, and outwards are often grouped)*
4.  **Check "Live" Status:** If the calculated `Stock[Today]` is greater than 0, we mark that SKU as "live" in that store on that day.
5.  **Repeat:** We repeat this for every day, for every SKU, in every store, building a complete history of inventory levels and live days.

**Inputs Needed:**

*   **Stock Snapshots:** `InvKeyFrameRow` data providing starting stock levels at specific dates (often the start of a week).
*   **Sales Data:** `SalesRow` data showing daily sales quantities.
*   **(Implicit) Other Movements:** The provided code often calculates implied "inwards" based on sales exceeding opening stock, but conceptually, explicit inwards/outwards/returns data could also be used.
*   **Product Relationships:** Maps defining SKU-Style relationships (from [SkuStyleConversionUtil](18_skustyleconversionutil_.md)).
*   **Configuration:** Which stores/SKUs to process, date ranges, the start day of the week (e.g., Monday).

**Outputs Produced:**

The main outputs are maps stored in memory (often managed by the [Cache](05_cache_.md)):
*   **Live Day Map (`skuStoreMap`):** Records which stores an SKU was live in on each day (e.g., `Map<SkuId, Map<Date, Set<StoreId>>>`).
*   **Live Quantity Map (`qtyLiveMap`):** Records the calculated stock quantity for an SKU in specific stores on specific days (e.g., `Map<SkuId, Map<Date, Map<StoreId, Quantity>>>`).

This computed information is then used by other parts of `irisx-algo`, especially the [View](10_view_.md) helper, to answer questions about historical availability.

## How It's Used (The Process)

`InventoryComputationUtil` contains the core calculation logic as static methods. It doesn't run on its own but is called by other modules responsible for data preparation, specifically `InvComputationModule` (which is often part of `InvComputationGroupModule`).

Here's the typical flow:

1.  **Data Gathering (`InvComputationModule`):**
    *   Loads necessary data: `InvKeyFrameRow`, `SalesRow`, `EndDateStockRow`, etc. from the database or files.
    *   Uses [SkuStyleConversionUtil](18_skustyleconversionutil_.md) to filter and aggregate this data based on product hierarchies (e.g., roll up child SKU keyframes/sales to parents).
    *   Gets configuration like the analysis end date and week start day.
    *   Determines the date ranges (`computePeriods`) for which inventory needs calculation.
    *   Groups sales data efficiently, often by week (`weeklySalesMap`).
2.  **Prepare Metadata (`InvComputationModule`):**
    *   Bundles all the prepared input data (filtered keyframes, weekly sales map, store/SKU sets, date ranges, week start day) into a special container object called `InvComputeMetadata`.
    *   This metadata object also holds references to the empty maps where the results (`skuStoreMap`, `qtyLiveMap`) will be stored.
3.  **Execute Computation (`InvComputationModule` calls `InventoryComputationUtil`):**
    *   Calls the main static method: `InventoryComputationUtil.computeInventory(invComputeMetadata)`.
4.  **Simulation (`InventoryComputationUtil`):**
    *   The `computeInventory` method orchestrates the simulation. It iterates through the defined `computePeriods`, styles, and stores.
    *   For each combination, it simulates the stock flow week by week, day by day, using the data provided in the `invComputeMetadata`.
    *   As it calculates the daily stock, it populates the `skuStoreMap` (marking live days) and `qtyLiveMap` (storing quantities) *directly within the passed `invComputeMetadata` object*.
5.  **Store Results (`InvComputationModule`):**
    *   After `computeInventory` finishes, `InvComputationModule` retrieves the populated `skuStoreMap` and `qtyLiveMap` from the `invComputeMetadata` object.
    *   It then typically stores these maps in the [Cache](05_cache_.md) (`cache.setLiveDateMap(...)`, `cache.setQtyLiveMap(...)`) for other modules to access.

## Under the Hood: The Simulation Logic

Let's trace the core simulation process within `InventoryComputationUtil`.

**Data Preparation (Done by `InvComputationModule`):**

*   **Keyframes (`InvKeyFrameData`):** Starting stock snapshots are loaded and organized for quick lookup by SKU, store, and the start date of the week.
*   **Weekly Sales (`weeklySalesMap`):** Daily sales (`SalesRow`) are grouped by SKU, store, and the start date of the week they occurred in. This allows the computation to efficiently get all sales for a specific SKU/store within a given week.

**Simulation Steps (`InventoryComputationUtil.computeInventory` -> `computeStyleInventory` -> `computeWeekInventory` -> `addLiveDates`):**

1.  **Outer Loops:** The code iterates through calculated time periods, then styles, then the stores relevant to each style.
2.  **Weekly Processing:** Inside these loops, the simulation focuses on one week at a time for a specific style-store combination.
3.  **Get Weekly Metadata (`getWeekMetadata`):** Before simulating the week, it calculates:
    *   `inwardsQtyMap`: For each SKU in the style, it estimates the minimum "inwards" quantity needed during the week. This is done by comparing total weekly sales to the opening inventory (from the keyframe). If sales > opening, the difference is assumed to be covered by some form of inward movement during the week. *This is a simplification/estimation used when detailed inwards data isn't available.*
    *   `saleQtyMap`: Maps the exact day within the week to the quantity sold for each SKU.
    *   `inwardsDate`: Estimates the first day within the week when an inward movement might have occurred (often the first day a sale happened, adjusted slightly).
4.  **Daily Simulation (`addLiveDates`):** This is the core loop that simulates day-by-day within the week for *each SKU* belonging to the style:
    *   **Get Opening Stock:** Retrieves the `openingInv` for the SKU at the start of the week from the `keyFrameData` (via `metadata`).
    *   **Loop Days:** Iterates from the first day of the week to the last.
    *   **Apply Inwards (Estimated):** If the current day matches the estimated `inwardsDate`, adds the calculated `inwardsQty` for that SKU to the `openingInv`.
    *   **Check Live Status:** If the current `openingInv` is greater than 0:
        *   Mark the SKU as live for this day and store: `metadata.addSkuDayLiveStore(sku, dateInWeek, store)`.
        *   If this day is the *final end date* of the overall analysis, store the quantity: `metadata.addSkuDayStoreQty(sku, dateInWeek, store, openingInv)`.
    *   **Apply Sales:** Subtract the sales quantity for the current day (obtained from `saleQtyMap`) from `openingInv` to get the opening stock for the *next* day.
    *   **Optimization:** If `openingInv` drops to 0 or below, and the estimated `inwardsDate` has passed, the loop can break early for this week, as the SKU won't be live for the remaining days.

**Sequence Diagram (Simplified Weekly Simulation for one SKU):**

```mermaid
sequenceDiagram
    participant ICM as InvComputationModule
    participant Util as InventoryComputationUtil
    participant Meta as InvComputeMetadata
    participant Cache as Cache

    ICM ->> Util: computeInventory(metadata)
    Util ->> Util: Loop through Periods/Styles/Stores
    Note over Util, Meta: Processing Week W for SKU S, Store T
    Util ->> Meta: getKeyFrameQty(S, T, StartOfWeekW)
    Meta -->> Util: Return openingInv
    loop Day in Week W (currentDay)
        Util ->> Meta: Get weekly sales info for S, T, Week W
        Meta -->> Util: Return salesQty for currentDay, estimated inwardsDate/Qty
        alt currentDay == inwardsDate
             Util ->> Util: openingInv += inwardsQty
        end
        alt openingInv > 0
            Util ->> Meta: addSkuDayLiveStore(S, currentDay, T)
            alt currentDay == ActualEndDate
                Util ->> Meta: addSkuDayStoreQty(S, currentDay, T, openingInv)
            end
        end
        Util ->> Util: openingInv -= salesQty
        alt openingInv <= 0 AND currentDay >= inwardsDate
            Note over Util: Stop processing rest of week for this SKU
            break
        end
    end
    Note over Util, Meta: Week W done, continue loops...
    Util -->> ICM: Computation complete (results in metadata)
    ICM ->> Cache: Set LiveDayMap from metadata.getSkuStoreMap()
    ICM ->> Cache: Set QtyLiveMap from metadata.getQtyLiveMap()
```

**Code Dive:**

Let's look at the key parts involved.

*   **Orchestration (`InvComputationModule.computeInv`):**

    ```java
    // Simplified from InvComputationModule.java
    private void computeInv(Properties properties) {
        // ... (Load data, get args, dates, parent maps) ...

        // Filter keyframes and sales using SkuStyleConversionUtil
        InvKeyFrameData keyFrameData = SkuStyleConversionUtil.getFilteredKeyFrame(
                db().select(InvKeyFrameRow.class), skuParentMap, cache.getFilteredSkus());
        Map<Integer, Map<Integer, Map<LocalDate, List<Key>>>> weeklySalesMap =
            InventoryComputationUtil.getSalesDayQtyMap(
                SkuStyleConversionUtil.getFilteredSales(
                    db().select(SalesRow.class), skuParentMap, cache.getFilteredSkus()
                ), commonArgs.weekStart);

        // Create the metadata object and populate inputs
        InvComputeMetadata invComputeMetadata = new InvComputeMetadata()
                .setActualEndDate(endDate)
                .setWeekStart(commonArgs.weekStart)
                .setKeyFrameData(keyFrameData)
                .setWeeklySalesMap(weeklySalesMap)
                // ... (set styleSkuMap, storeSet, skuStyleMap, computePeriods) ...
                ;

        // *** Call the computation utility ***
        InventoryComputationUtil.computeInventory(invComputeMetadata);

        // ... (Handle End Date Stock Override - complex logic omitted) ...

        // *** Store results in Cache ***
        cache.setLiveDateMap(invComputeMetadata.getSkuStoreMap());
        cache.setQtyLiveMap(invComputeMetadata.getQtyLiveMap());

        // ... (Clear intermediate data) ...
    }
    ```
    **Explanation:** This shows `InvComputationModule` preparing inputs (using helpers like `SkuStyleConversionUtil`), packaging them into `InvComputeMetadata`, calling the core `InventoryComputationUtil.computeInventory` method, and finally storing the results (which are now inside `invComputeMetadata`) into the Cache.

*   **Metadata Container (`InvComputeMetadata.java`):**

    ```java
    // Simplified from InvComputeMetadata.java
    public class InvComputeMetadata {
        // --- Inputs ---
        private LocalDate actualEndDate;
        private DayOfWeek weekStart;
        private InvKeyFrameData keyFrameData; // Holds keyframe data
        private Map<Integer, Map<Integer, Map<LocalDate, List<Key>>>> weeklySalesMap; // Holds weekly sales
        private Map<Integer, List<Integer>> styleSkuMap; // Style -> List<SKU ID>
        private Set<Integer> storeSet; // Stores to consider
        private Map<Integer, Integer> skuStyleMap; // SKU -> Style ID
        private List<Key> computePeriods; // Date ranges

        // --- Outputs (Populated by InventoryComputationUtil) ---
        // sku -> (day -> live stores)
        private Map<Integer, Map<LocalDate, Set<Integer>>> skuStoreMap;
        // sku -> (day -> (store -> qty))
        private Map<Integer, Map<LocalDate, Map<Integer, Integer>>> qtyLiveMap;

        // Constructor initializes output maps (e.g., as ConcurrentHashMap)
        public InvComputeMetadata() { /* ... */ }

        // --- Getters for inputs (e.g., getKeyFrameQty, getWeeklySales) ---
        // Used by InventoryComputationUtil to access data

        // --- Methods to add output results ---
        // Called by InventoryComputationUtil to populate maps
        public void addSkuDayLiveStore(int sku, LocalDate day, int store) {
           skuStoreMap.computeIfAbsent(sku, /*...*/)
                      .computeIfAbsent(day, /*...*/)
                      .add(store);
        }
        public void addSkuDayStoreQty(int sku, LocalDate day, int store, int qty) {
           qtyLiveMap.computeIfAbsent(sku, /*...*/)
                     .computeIfAbsent(day, /*...*/)
                     .put(store, qty);
        }

        // --- Getters for outputs (e.g., getSkuStoreMap, getQtyLiveMap) ---
        // Used by InvComputationModule to retrieve results
    }
    ```
    **Explanation:** This class acts as a structured container. It holds all the input data needed for the computation and also the maps where the results will be stored. It provides methods for the computation utility to access inputs and add outputs.

*   **Core Logic (`InventoryComputationUtil.addLiveDates`):**

    ```java
    // Simplified from InventoryComputationUtil.java
    /** Function which adds live days to live day map and to qty map (only for end date). */
    private static void addLiveDates(int sku, Key dateStoreStyleKey, InvComputeMetadata metadata,
                                     int openingInv, InvCompWeekMetadata weekMetadata, boolean jitStoreStyle) {
        LocalDate weekStartDate = (LocalDate) dateStoreStyleKey.part(0);
        int store = (Integer) dateStoreStyleKey.part(1);
        LocalDate weekEndDate = LocalDateProvider.plusDays(weekStartDate, DAYS_QUANTUM); // Usually 7 days

        // Loop through each day in the week
        for (LocalDate dateInWeek = weekStartDate; dateInWeek.isBefore(weekEndDate); dateInWeek = LocalDateProvider.plusDays(dateInWeek, 1)) {

            // Optimization: break if stock is <= 0 and estimated inwards is done/not happening
            if (openingInv <= 0 && (dateInWeek.isAfter(weekMetadata.inwardsDate) || /* inwards not happening */ )) {
                break;
            }

            // Apply estimated inwards if it's the day
            openingInv += (weekMetadata.inwardsDate.equals(dateInWeek) ?
                           weekMetadata.inwardsQtyMap.getOrDefault(sku, 0) : 0);

            // Check if live (stock > 0)
            if (openingInv > 0) {
                // Add to live day map
                metadata.addSkuDayLiveStore(sku, dateInWeek, store);
                // Add to quantity map IF it's the final analysis end date
                if (metadata.getActualEndDate().isEqual(dateInWeek))
                    metadata.addSkuDayStoreQty(sku, dateInWeek, store, openingInv);
            }

            // Subtract sales for the current day to get stock for start of next day
            // (Skip sales subtraction for JIT items - Just-In-Time assumes stock always available if ordered)
            if (!jitStoreStyle) {
                 openingInv -= weekMetadata.saleQtyMap.getOrDefault(new Key(sku, dateInWeek), 0);
            }
        }
    }
    ```
    **Explanation:** This method simulates day-by-day within a week. It adjusts the `openingInv` based on estimated inwards and daily sales. If `openingInv > 0`, it calls `metadata.addSkuDayLiveStore` to record the live day and potentially `metadata.addSkuDayStoreQty` to record the quantity on the final date.

## Conclusion

**`InventoryComputationUtil`** is the core engine in `irisx-algo` for reconstructing historical daily inventory levels.

*   It **simulates stock flow** day by day using keyframe snapshots and daily sales data (and potentially other movements).
*   Its primary purpose is to determine the **"live days"** for each SKU in each store – days when stock was available (> 0).
*   It uses the formula: `Stock[Today] = Stock[Yesterday] - Sales + Returns + Inwards - Outwards`.
*   It's typically called by `InvComputationModule`, which prepares the input data (using helpers like [SkuStyleConversionUtil](18_skustyleconversionutil_.md)) and stores the results (live day map, quantity map) in the [Cache](05_cache_.md).
*   The resulting historical inventory information is crucial for accurate performance analysis and other downstream calculations.

Understanding how this utility works is key to understanding how `irisx-algo` determines product availability over time.

Now that we've seen the utility that performs the core calculation, the next chapter looks at the module that orchestrates this process: the [Inventory Creation Module](20_inventory_creation_module_.md).

[Next Chapter: Inventory Creation Module](20_inventory_creation_module_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)