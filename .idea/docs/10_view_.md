# Chapter 10: View

Welcome back! In the [previous chapter](09_row_input_output_classes_.md), we learned about [Row Input/Output Classes](09_row_input_output_classes_.md), the blueprints that define how data like style details (`StyleRow`) or sales figures (`SalesRow`) are structured when moving into or out of `irisx-algo`.

Now that we have these structured data objects, often loaded into our handy [Cache](05_cache_.md), modules frequently need to ask specific questions about the *state* of things over time. For example, a planning module might need to know:

*   "Was this specific sneaker (SKU 54321) actually available for sale in the downtown store (Store 101) on July 15th?"
*   "For our new 'Summer Breeze' dress collection (Style 987), how many days was it actually live (available for sale) in any of our London stores during the entire month of June?"
*   "Which specific sizes of this shirt (Style 123) were available in the warehouse (Location 50) last Monday?"

Figuring this out by going back to raw sales or stock movements every time would be very slow and complicated. We've likely already processed this information, perhaps using the [Inventory Creation Module](20_inventory_creation_module_.md), and stored the results (like daily stock levels or 'live' status) in the [Cache](05_cache_.md). How can our modules easily *ask* these state-based questions without redoing all the hard work?

## What Problem Does View Solve?

Imagine you're an astronomer who has already spent hours capturing detailed images of the night sky (like our pre-calculated inventory data in the [Cache](05_cache_.md)). Now, you want to quickly answer specific questions like "Is Jupiter visible from London tonight?" or "How many hours was the Orion nebula above the horizon yesterday?". You don't want to re-process all the raw telescope data each time! You need a tool to quickly *view* or *query* the information you already have.

The **`View`** component in `irisx-algo` acts like those specialized astronomical query tools, or like a pair of powerful binoculars focused on our inventory landscape. It provides a simplified way for modules to **"view" or query the status of inventory across different times and locations**, using the information that's already been calculated and stored (usually in the [Cache](05_cache_.md)).

It answers questions like:
*   'Was this SKU live on this day in that store?'
*   'How many days was this style live in this region last month?'
*   'What percentage of a style's core sizes (pivotal SKUs) were available in Store X on Date Y?'

`View` prevents modules from having to implement complex logic just to check if something was available. It offers a set of convenient methods to get these answers quickly.

## Core Idea: Querying Pre-Calculated State

The `View` helper doesn't perform heavy calculations itself. Its main job is to:

1.  **Receive a Question:** A module calls a `View` method with specific parameters (e.g., style ID, store ID, date range).
2.  **Consult the Cache:** `View` uses its access to the [Cache](05_cache_.md) (which holds pre-calculated information like daily SKU-store live status) to find the answer.
3.  **Aggregate/Format:** It might do simple aggregation (like counting days over a range) or formatting.
4.  **Return the Answer:** It gives the result back to the module (e.g., a boolean `true`/`false`, a number of days, a set of sizes).

Think of the [Cache](05_cache_.md) as holding a detailed daily logbook of what SKU was available where. `View` is the librarian who knows how to quickly look up entries in that logbook to answer your specific questions about availability over time.

## How to Use View

Modules access the `View` helper using dependency injection, just like they access the [Cache](05_cache_.md) or [Common Data](06_common_data_.md).

**1. Getting Access (Injection):**

A module declares that it needs the `View` helper.

```java
// Inside a module class, e.g., a planning or analysis module

import com.increff.irisx.helper.View; // Import the View helper class
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
// ... other imports ...

@Component // Marks this class as a Spring-managed component
public class MyPlanningModule extends AbstractModule {

    @Autowired // Tell Spring to inject the View instance here
    private View view;

    // ... rest of the module code ...
}
```

**Explanation:**

*   `@Autowired private View view;`: This line tells the Spring framework, "When you create an instance of `MyPlanningModule`, please find the already existing instance of `View` and assign it to this `view` field."

**2. Using View Methods:**

Now the module can ask `View` questions about inventory status.

**Use Case:** Let's say our `MyPlanningModule` needs to know if SKU 54321 was live in Store 101 anytime between July 1st and July 15th, 2024. It also wants to know exactly how many days Style 987 was live in that same store during that period.

```java
// Inside a method within MyPlanningModule.java

import java.time.LocalDate;
import java.util.Optional; // Used for optional parameters

// ... inside some calculation logic ...

int skuIdToCheck = 54321;
int styleIdToCheck = 987;
int storeId = 101;
LocalDate startDate = LocalDate.of(2024, 7, 1);
LocalDate endDate = LocalDate.of(2024, 7, 15);

// --- Question 1: Was the SKU live AT ALL during the period? ---
boolean wasSkuLive = view.isSkuLiveInDurationInStore(skuIdToCheck, startDate, endDate, storeId);

if (wasSkuLive) {
    System.out.println("SKU " + skuIdToCheck + " was live in store " + storeId + " between " + startDate + " and " + endDate);
} else {
    System.out.println("SKU " + skuIdToCheck + " was NOT live in store " + storeId + " during " + startDate + " to " + endDate);
}

// --- Question 2: How many days was the STYLE live during the period? ---
// The Optional.of(storeId) passes the store ID. Optional.empty() would check across all stores.
int liveDays = view.getStyleLiveDays(styleIdToCheck, startDate, endDate, Optional.of(storeId));

System.out.println("Style " + styleIdToCheck + " was live for " + liveDays + " days in store " + storeId + " between " + startDate + " and " + endDate);

// --- Question 3: Was the Style live on a specific day? ---
LocalDate specificDay = LocalDate.of(2024, 7, 10);
boolean styleLiveOnDay = view.styleLiveStatus(styleIdToCheck, specificDay, Optional.of(storeId));
System.out.println("Style " + styleIdToCheck + " live on " + specificDay + " in store " + storeId + "? " + styleLiveOnDay);
```

**Explanation:**

*   `view.isSkuLiveInDurationInStore(...)`: Asks `View` if the given SKU was available in the specified store on *any* day within the date range. Returns `true` or `false`.
*   `view.getStyleLiveDays(...)`: Asks `View` to count how many days the given style was available in the specified store (using `Optional.of(storeId)`) within the date range. Returns an integer count.
*   `view.styleLiveStatus(...)`: Asks `View` if the style was live on one particular day in that store. Returns `true` or `false`.

**Expected Output (example, depends on the actual data in the Cache):**

```
SKU 54321 was live in store 101 between 2024-07-01 and 2024-07-15
Style 987 was live for 12 days in store 101 between 2024-07-01 and 2024-07-15
Style 987 live on 2024-07-10 in store 101? true
```

The module gets clear answers without needing to know the intricate details of how the daily live status is stored or checked in the [Cache](05_cache_.md).

## Under the Hood: Asking the Cache

Let's trace what happens when `view.getStyleLiveDays(styleId, startDate, endDate, Optional.of(storeId))` is called.

**High-Level Steps:**

1.  **`View` Receives Request:** The `getStyleLiveDays` method in `View` is called with the parameters.
2.  **Loop Through Dates:** `View` starts a loop, iterating through each day from `startDate` to `endDate`.
3.  **Ask About Style Status:** Inside the loop, for each `currentDate`, `View` needs to know if the `styleId` was live in the given `storeId` on that specific day. It calls another one of its own methods, like `styleLiveStatus(styleId, currentDate, Optional.of(storeId))`.
4.  **`styleLiveStatus` Asks About SKUs:** The `styleLiveStatus` method knows that a style is live if *any* of its SKUs are live. So, it gets the list of SKUs for the style from the [Cache](05_cache_.md) (`cache.getAllSkusInStyle(styleId)`).
5.  **Loop Through SKUs & Ask Cache:** `styleLiveStatus` then loops through these SKUs. For each `skuId`, it directly asks the [Cache](05_cache_.md): `cache.isLiveOnDayForStore(skuId, currentDate, storeId)`.
6.  **Cache Answers:** The [Cache](05_cache_.md) looks up its internal data structures (which store the pre-calculated daily SKU-store live status) and returns `true` or `false` for that SKU, date, and store.
7.  **`styleLiveStatus` Determines Result:** If the Cache returns `true` for *any* SKU, `styleLiveStatus` immediately knows the style was live on that day and returns `true`. If it checks all SKUs and finds none were live, it returns `false`.
8.  **`getStyleLiveDays` Counts:** Back in `getStyleLiveDays`, if `styleLiveStatus` returned `true` for the `currentDate`, it increments a counter.
9.  **Loop Continues:** The date loop continues to the next day.
10. **Final Result:** After checking all dates in the range, `getStyleLiveDays` returns the final value of the counter.

**Sequence Diagram:**

```mermaid
sequenceDiagram
    participant Mod as MyPlanningModule
    participant V as View
    participant C as Cache

    Mod->>V: getStyleLiveDays(987, 2024-07-01, 2024-07-15, store=101)
    loop Date Range (July 1 to July 15)
        V->>V: styleLiveStatus(987, currentDate, store=101)
        V->>C: getAllSkusInStyle(987)
        C-->>V: Return List<Integer> [skuA, skuB, skuC]
        loop SKUs [skuA, skuB, skuC]
            V->>C: isLiveOnDayForStore(currentSku, currentDate, 101)
            C-->>V: Return true/false
            alt If Cache returns true
                Note over V: Style is live for currentDate
                V-->>V: Return true (from styleLiveStatus)
                V->>V: Increment liveDays counter
                break SKU loop
            end
        end
        Note over V: If no SKU was live, styleLiveStatus returns false
    end
    V-->>Mod: Return final liveDays count (e.g., 12)
```

This shows that `View` orchestrates the query, but the core check (`isLiveOnDayForStore`) is delegated to the `Cache`, which holds the actual pre-computed state.

**Code Dive:**

Let's look at simplified versions of the relevant methods in `View.java`.

```java
// File: src/main/java/com/increff/irisx/helper/View.java
package com.increff.irisx.helper;

import com.increff.irisx.helper.Cache; // Needs the Cache
import com.increff.irisx.provider.LocalDateProvider; // For date math
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;
import java.time.LocalDate;
import java.util.List;
import java.util.Optional;

@Component
public class View {

    @Autowired
    private Cache cache; // Injected Cache dependency

    /** Checks if a style is live on a given day, optionally for a specific store */
    public boolean styleLiveStatus(int styleId, LocalDate day, Optional<Integer> storeId) {
        // Get all SKUs belonging to this style from the Cache
        List<Integer> skuList = cache.getAllSkusInStyle(styleId);
        for (Integer skuId : skuList) {
            // Check if a specific store was provided
            if (storeId.isPresent()) {
                // Ask Cache if this SKU is live for the specific store and day
                if (cache.isLiveOnDayForStore(skuId, day, storeId.get()))
                    return true; // Found a live SKU, style is live!
            } else {
                // No specific store - check if live in ANY store
                // Ask Cache for the set of stores where this SKU is live on this day
                if (!cache.getStoresLiveOnDay(skuId, day).isEmpty())
                    return true; // Found a live SKU, style is live!
            }
        }
        // If loop finishes, no live SKUs were found for this style on this day/store
        return false;
    }

    /** Counts the number of days a style was live within a date range */
    public int getStyleLiveDays(int styleId, LocalDate startDate, LocalDate endDate, Optional<Integer> storeId) {
        int liveDays = 0;
        LocalDate currentDate = startDate;
        // Loop through each day in the range
        while (!currentDate.isAfter(endDate)) {
            // Use styleLiveStatus to check if style was live on this day/store
            if (styleLiveStatus(styleId, currentDate, storeId)) {
                liveDays++; // Increment counter if live
            }
            // Move to the next day (using a provider for testability)
            currentDate = LocalDateProvider.plusDays(currentDate, 1);
        }
        return liveDays;
    }

    /** Checks if an SKU was live on any day within the duration for a specific store */
    public boolean isSkuLiveInDurationInStore(int skuId, LocalDate startDate, LocalDate endDate, int store) {
        LocalDate currentDate = startDate;
        while (!currentDate.isAfter(endDate)) {
            // Ask Cache directly if the SKU was live on this day/store
            if (cache.isLiveOnDayForStore(skuId, currentDate, store)) {
                return true; // Found a live day, no need to check further
            }
            currentDate = currentDate.plusDays(1); // Standard date increment
        }
        // If loop finishes, SKU was not live on any day in the range
        return false;
    }

    // ... many other helpful methods exist in View.java ...
    // e.g., getSkuLiveDays, getSizesLiveOnDayInStore, getPivotalPercentOnDay, etc.
}
```

**Explanation:**

*   The `View` class uses the injected `cache` object extensively.
*   Methods like `getStyleLiveDays` iterate through dates and call other helper methods like `styleLiveStatus`.
*   Methods like `styleLiveStatus` iterate through the relevant SKUs (obtained from the `cache`) and call specific `cache` methods (like `isLiveOnDayForStore`) to check the state.
*   The logic is focused on iterating through time or components (like SKUs) and asking the `Cache` for the status at each point.

## Conclusion

The **`View`** component acts as a crucial helper in `irisx-algo`, providing a simplified interface for modules to **query the status of inventory** (like availability or "liveness") across time and location dimensions.

*   It answers questions like "Was SKU X live in Store Y on Date Z?" or "How many days was Style A live in Region B?".
*   It works by leveraging **pre-calculated inventory data** stored in the [Cache](05_cache_.md), avoiding redundant complex calculations.
*   Modules use it via dependency injection (`@Autowired`) and call its methods (e.g., `getStyleLiveDays`, `isSkuLiveInDurationInStore`).
*   Internally, `View` methods often iterate through dates or SKUs and delegate the actual status checks to the [Cache](05_cache_.md).

By abstracting these common status queries, `View` makes the code in algorithm modules cleaner, simpler, and more focused on their core logic.

To efficiently store and retrieve data in the Cache (which View relies on), we often need ways to quickly convert lists of objects into maps keyed by specific IDs. The next chapter explores utility classes called [ObjectMaps](11_objectmaps_.md) that help with exactly this.

[Next Chapter: ObjectMaps](11_objectmaps_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)