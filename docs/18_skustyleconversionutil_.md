# Chapter 18: SkuStyleConversionUtil

Welcome back! In the [previous chapter](17_attribute_grouping__aggroupmodule___agcomputemodule__.md), we saw how `AgComputeModule` groups products based on their shared attributes, creating unique `AgRow` tags for each combination. This helps us analyze product groups.

Now, let's consider another common complexity in product data: relationships between products. Think about a T-shirt, like a "Blue V-Neck T-Shirt". This is often considered a "Style". But this style comes in different sizes: Small, Medium, Large. These specific sizes are often called "SKUs" (Stock Keeping Units). Here, the "Blue V-Neck T-Shirt" is the **parent** (the Style), and the Small, Medium, and Large versions are the **children** (the SKUs).

Sometimes, things get even more complex. Maybe the "Blue V-Neck T-Shirt" itself is considered a child variation of an even broader parent style, like "Basic V-Neck T-Shirt".

## What Problem Does SkuStyleConversionUtil Solve?

Imagine you want to know the total sales for the "Blue V-Neck T-Shirt" style. You have sales data recorded for each individual SKU (Small, Medium, Large). To get the total for the style, you need to add up the sales from all its child SKUs.

Similarly, if you're looking at inventory, the stock might be stored at the SKU level (e.g., 10 Small, 15 Medium, 5 Large in the warehouse). If you want to know the total inventory for the "Blue V-Neck T-Shirt" style, you need to sum the inventory of its children.

What if some systems track data only at the parent level, while others track it at the child level? Or what if a product (like a child SKU) is mistakenly listed as its own parent? This can lead to:
*   **Double Counting:** Counting both the child's data and the parent's data separately when you only want the total.
*   **Missed Data:** Only looking at the parent level and missing the details from the children.
*   **Inconsistent Views:** Different reports showing different totals because they handle the parent-child relationships differently.

**`SkuStyleConversionUtil`** acts like a specialized tool for managing these **product family trees**. It provides utility functions to help correctly handle data (like sales, inventory, goods in transit, open orders) when these parent-child relationships exist between SKUs and Styles. Its main job is to ensure data from child items is correctly **aggregated** (summed up) or **mapped** to the parent level when needed, preventing errors and providing a consistent view.

## Core Idea: Rolling Up Data from Children to Parents

Think of `SkuStyleConversionUtil` as an accountant for product families. It knows which SKUs belong to which parent Styles (and potentially which Styles belong to parent Styles). When it processes a list of data (like sales records), it uses this family tree knowledge to:

1.  **Identify Children:** Recognize rows of data that belong to a child SKU or child Style.
2.  **Find the Parent:** Determine the correct parent SKU or Style ID for that child.
3.  **Aggregate/Remap:**
    *   For data like Sales or Inventory, it often *sums up* the quantities/values from the child rows and adds them to the corresponding parent row.
    *   It ensures that the final list of data represents the information at the correct parent level, removing or adjusting the child rows as needed.

This "roll-up" process gives you an accurate picture at the parent level, incorporating all the information from the descendants.

## How SkuStyleConversionUtil is Used

`SkuStyleConversionUtil` is primarily a **utility class with static methods**. It's typically used internally during the data loading and preparation phases, often called by components that populate the [Cache](05_cache_.md) or by modules that need to prepare data before performing calculations (like the [Inventory Creation Module](20_inventory_creation_module_.md)).

You wouldn't usually call its methods directly from high-level business logic modules like OTB or Distribution planning. Instead, those modules rely on the fact that the data they receive (e.g., from the Cache) has already been cleaned and aggregated correctly using utilities like this one.

**Key Inputs for the Utility:**

The methods in `SkuStyleConversionUtil` usually need:
*   **Data List:** The list of data rows to be processed (e.g., `List<SalesRow>`, `List<WhStockRow>`, `List<SkuRow>`).
*   **Parent Maps:** Maps that define the parent-child relationships:
    *   `skuParentMap`: A `Map<Integer, Integer>` where the key is a child SKU ID and the value is its parent SKU ID.
    *   `styleParentMap`: A `Map<Integer, Integer>` where the key is a child Style ID and the value is its parent Style ID.
*   **Valid SKUs/Styles:** A `Set<Integer>` containing the IDs of the SKUs or Styles that should be considered valid in the final output (often representing the parent-level items).

**Example Use Case (Conceptual): Processing Sales Data**

Imagine a module responsible for loading sales data needs to aggregate sales from child SKUs to their parents.

```java
// Conceptual code in a data loading/processing component

import com.increff.irisx.util.SkuStyleConversionUtil;
import com.increff.irisx.row.input.transactional.SalesRow;
import java.util.ArrayList;
import java.util.List;
import java.util.Map;
import java.util.Set;

// Assume these are loaded from configuration or earlier steps
Map<Integer, Integer> skuParentMap = loadSkuParentMap(); // e.g., {sku_S: parent_sku, sku_M: parent_sku, ...}
Set<Integer> validParentSkus = loadValidParentSkus(); // Contains only parent SKU IDs

// Raw sales data, potentially containing both parent and child SKU sales
ArrayList<SalesRow> rawSalesList = loadRawSalesData();

System.out.println("Raw sales records: " + rawSalesList.size());

// Use the utility to filter and aggregate sales to the parent SKU level
List<SalesRow> aggregatedSalesList = SkuStyleConversionUtil.getFilteredSales(
                                           rawSalesList,
                                           skuParentMap,
                                           validParentSkus
                                       );

System.out.println("Aggregated sales records (at parent level): " + aggregatedSalesList.size());

// Now 'aggregatedSalesList' contains sales data where:
// - Child SKU sales have been summed up and attributed to the parent SKU.
// - Rows corresponding only to child SKUs might have been removed.
// - The sku ID in each row represents a valid parent SKU.
// This list can now be safely used for analysis at the parent level.
```

**Explanation:**

1.  The process starts with `rawSalesList` which might contain sales for child SKUs (like size Small) and potentially parent SKUs.
2.  It calls `SkuStyleConversionUtil.getFilteredSales`, passing the raw list, the `skuParentMap` (telling it which SKU maps to which parent), and the `validParentSkus` set (defining which SKUs should appear in the final list).
3.  The utility method iterates through the raw sales, identifies child SKU sales using the map, finds or creates the parent SKU's sales record for that day/store, and adds the child's quantity and revenue to the parent's totals.
4.  It returns `aggregatedSalesList`, which contains sales data correctly rolled up to the parent SKU level, ready for further use.

## Under the Hood: Iterating, Mapping, and Aggregating

Let's trace how `getFilteredSales` might work internally.

**High-Level Steps (`getFilteredSales`):**

1.  **Input:** Receives `salesRows` (ArrayList), `skuParentMap`, `filteredSkus` (Set of valid parent SKUs).
2.  **Create Result Map:** Initializes an empty `Map<Key, SalesRow>`. The `Key` will represent the unique combination of `store`, `day`, and *parent* `sku`. The `SalesRow` will store the aggregated sales for that key.
3.  **Iterate Raw Sales:** Loops through each `row` in the input `salesRows`.
4.  **Filter by Valid SKUs:** Checks if the `row.sku` is present in the `filteredSkus` set. If not, it might skip this row (depending on exact logic, but usually we filter based on the *parent* SKU later).
5.  **Find Parent SKU:** Looks up `row.sku` in the `skuParentMap`.
    *   If found, `parentSku = skuParentMap.get(row.sku)`.
    *   If not found, the SKU is already a parent (or standalone), so `parentSku = row.sku`.
6.  **Create Key:** Creates a composite `Key` object using `row.store`, `row.day`, and the `parentSku` ID.
7.  **Aggregate in Map:** Uses the `Key` to look up an entry in the result map.
    *   **If Key exists:** Retrieves the existing `SalesRow` for the parent SKU on that day/store and adds the current `row.qty`, `row.revenue`, `row.discValue` to it.
    *   **If Key is new:** Creates a *new* `SalesRow`, sets its `store`, `day`, and `sku` (to the `parentSku`), initializes `qty`, `revenue`, `discValue` from the current `row`, and puts this new aggregated row into the map with the `Key`.
8.  **Return Final List:** After processing all input rows, the method takes all the `SalesRow` objects (which now contain aggregated data at the parent SKU level) from the values of the result map and returns them as a `List`.

**Sequence Diagram (`getFilteredSales` processing one child sale):**

```mermaid
sequenceDiagram
    participant Caller as Data Processor
    participant Util as SkuStyleConversionUtil
    participant SalesList as List<SalesRow> (Input)
    participant SkuMap as skuParentMap
    participant AggMap as Map<Key, SalesRow> (Internal Aggregation Map)
    participant FinalList as List<SalesRow> (Output)

    Caller->>Util: getFilteredSales(salesList, skuMap, validSkus)
    Util->>AggMap: Create empty map
    loop For each 'row' in SalesList
        Util->>SkuMap: Get parentSkuId for row.sku
        SkuMap-->>Util: Return parentSkuId (e.g., parent=100 for child=101)
        Util->>Util: Create Key(row.store, row.day, parentSkuId)
        Util->>AggMap: Check if Key exists
        alt Key Exists (Parent row already started)
            Util->>AggMap: Get existing SalesRow (aggRow) for Key
            Util->>AggMap: aggRow.qty += row.qty; aggRow.revenue += row.revenue
        else Key is New (First sale for this parent on this day/store)
            Util->>AggMap: Create new SalesRow (aggRow) with parentSkuId
            Util->>AggMap: aggRow.qty = row.qty; aggRow.revenue = row.revenue
            Util->>AggMap: Put(Key, aggRow)
        end
    end
    Util->>AggMap: Get all values (aggregated SalesRows)
    Util->>FinalList: Create List from map values
    Util-->>Caller: Return FinalList
```

**Code Dive (`SkuStyleConversionUtil.java`):**

Let's look at simplified snippets.

*   **Filtering SKUs/Styles:** These methods often just filter lists based on parent maps.

    ```java
    // Simplified from SkuStyleConversionUtil.java
    /** Filters SkuRows, removing children and remapping style IDs if needed */
    public static List<SkuRow> getFilteredSku(List<SkuRow> skuRows,
                                            Map<Integer, Integer> styleParentMap,
                                            Map<Integer, Integer> skuParentMap,
                                            Set<Integer> validSkus) {
        List<SkuRow> filteredSkuRows = new ArrayList<>();
        skuRows.stream()
                .filter(o -> validSkus.contains(o.id)) // Keep only valid SKUs
                .filter(o -> !skuParentMap.containsKey(o.id)) // Remove child SKUs
                .forEach(skuRow -> {
                    // Remap style ID to parent style ID if necessary
                    skuRow.style = styleParentMap.getOrDefault(skuRow.style, skuRow.style);
                    filteredSkuRows.add(skuRow);
                });
        return filteredSkuRows;
    }

    /** Filters StyleRows, removing children */
    public static List<StyleRow> getFilteredStyle(List<StyleRow> styleRows,
                                                 Map<Integer, Integer> styleParentMap) {
        List<StyleRow> filteredStyleRows = new ArrayList<>();
        styleRows.forEach(styleRow -> {
            // Keep only if it's NOT a child style (i.e., not a key in the map)
            if (!styleParentMap.containsKey(styleRow.id)) {
                filteredStyleRows.add(styleRow);
            }
        });
        return filteredStyleRows;
    }
    ```
    **Explanation:** `getFilteredSku` uses streams to first keep only SKUs present in `validSkus`, then removes SKUs that are keys in the `skuParentMap` (meaning they are children), and finally remaps the `style` ID using `styleParentMap`. `getFilteredStyle` simply removes styles that are listed as children in `styleParentMap`.

*   **Aggregating Transactional Data (Sales Example):**

    ```java
    // Simplified from SkuStyleConversionUtil.java
    /** Aggregates SalesRow data to the parent SKU level */
    public static List<SalesRow> getFilteredSales(ArrayList<SalesRow> salesRows,
                                                  Map<Integer, Integer> skuParentMap,
                                                  Set<Integer> filteredSkus) {
        // Map stores aggregated results: Key(store, day, parentSku) -> SalesRow
        Map<Key, SalesRow> salesRowMap = new HashMap<>();

        // Iterate through each raw sales row
        salesRows.stream()
                // Optional: Pre-filter if needed, though aggregation handles parents implicitly
                // .filter(o -> filteredSkus.contains(skuParentMap.getOrDefault(o.sku, o.sku)))
                .forEach(row -> {
                    // Determine the parent SKU ID
                    int parentSku = skuParentMap.getOrDefault(row.sku, row.sku);

                    // Create the aggregation key
                    Key key = getSalesRowKey(row, parentSku);

                    // Find or create the aggregated SalesRow in the map
                    // computeIfAbsent ensures we create a new row only if the key is new
                    SalesRow aggregatedRow = salesRowMap.computeIfAbsent(key, k -> createNewSalesRow(row, parentSku));

                    // Add current row's values to the aggregated row
                    // (Need to handle the initial row creation case inside computeIfAbsent or check here)
                    // Simplified: Assuming computeIfAbsent returns existing or newly created row
                    if (aggregatedRow != row) { // Avoid double-adding if it was just created
                         aggregatedRow.qty += row.qty;
                         aggregatedRow.discValue += row.discValue;
                         aggregatedRow.revenue += row.revenue;
                    }
                });

        // Return the values (aggregated SalesRows) from the map as a list
        return new ArrayList<>(salesRowMap.values());
    }

    // Helper to create the Key object
    private static Key getSalesRowKey(SalesRow row, int sku) {
        return new Key(row.store, row.day, sku);
    }

    // Helper to create a new SalesRow for aggregation
    private static SalesRow createNewSalesRow(SalesRow originalRow, int parentSku) {
        SalesRow salesRow = new SalesRow();
        salesRow.store = originalRow.store;
        salesRow.day = originalRow.day;
        salesRow.sku = parentSku; // Use the PARENT SKU ID
        // Initialize with values from the first row encountered for this key
        salesRow.revenue = originalRow.revenue;
        salesRow.discValue = originalRow.discValue;
        salesRow.qty = originalRow.qty;
        return salesRow;
    }
    ```
    **Explanation:** This method iterates through sales. For each `row`, it finds the `parentSku`. It creates a `Key` based on store, day, and `parentSku`. It uses `salesRowMap.computeIfAbsent` to get the `SalesRow` associated with this key. If the key is new, `createNewSalesRow` is called to initialize an aggregated row using the parent SKU ID and the current row's values. Then, the current `row`'s quantity, revenue, etc., are added to the `aggregatedRow`. Finally, the aggregated rows stored as values in the map are returned.

## Conclusion

**`SkuStyleConversionUtil`** is a critical utility in `irisx-algo` for handling the complexities arising from **parent-child relationships** between products (SKUs and Styles).

*   It acts like a manager for **product family trees**.
*   It provides static methods to **filter and aggregate** various types of data (Sales, Inventory, GIT, Orders, etc.) from child items up to their parent level.
*   It relies on **`skuParentMap`** and **`styleParentMap`** to understand the relationships.
*   Its primary goal is to ensure data used for analysis and planning is **consistent, accurate, and free from errors** like double-counting caused by product hierarchies.
*   It's mainly used during **data loading and preparation phases** to ensure downstream modules work with correctly aggregated data.

By correctly handling these relationships, `SkuStyleConversionUtil` ensures the integrity of the data used throughout the `irisx-algo` calculations.

With data accurately representing products (even with complex hierarchies), we can now calculate key metrics like inventory levels. The next chapter introduces [InventoryComputationUtil](19_inventorycomputationutil_.md), a utility focused on calculating inventory based on stock levels, sales, and returns.

[Next Chapter: InventoryComputationUtil](19_inventorycomputationutil_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)