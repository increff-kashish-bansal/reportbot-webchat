# Chapter 22: NoosCoreComputeModule

Welcome back! In the [previous chapter](21_noos_identification__noosgroupmodule__.md), we learned how the [NOOS Identification (NoosGroupModule)](21_noos_identification__noosgroupmodule__.md) coordinates the overall process of figuring out which products should be Never Out Of Stock (NOOS). It acts like a manager, calling different specialist modules to do the actual analysis.

Now, let's meet the first specialist: the `NoosCoreComputeModule`. Its specific job is to identify the **"Core"** styles.

## What Problem Does This Module Solve?

Think about a store selling clothes. Some items are the absolute basics – the plain white t-shirts, the classic blue jeans, the simple black socks. These items might not be the most exciting, but they form the foundation of the assortment. Customers expect them to be available pretty much all the time, they sell steadily throughout the year, and they usually aren't heavily discounted. These are what we often call **"Core"** items.

How can `irisx-algo` automatically figure out which products fit this "Core" description? Looking just at total sales isn't enough, because a trendy item might sell a lot for a short time. We need to look for specific characteristics:

*   **Consistency:** Do sales happen steadily over time, or are they very up-and-down?
*   **Low Discounts:** Are they usually sold near their full price?
*   **Reasonable Volume:** Do they sell enough units to be considered significant?
*   **Availability:** When they *are* in stock, do they actually sell?

The `NoosCoreComputeModule` solves the problem of analyzing historical data against these criteria to automatically flag styles that behave like Core items.

## Core Idea: Finding Stable, Low-Discount Essentials

The `NoosCoreComputeModule` dives into the historical data (sales and inventory availability) for each style and calculates several key metrics over a defined period (e.g., the last 6 months). It then compares these metrics against predefined benchmarks or thresholds to decide if a style qualifies as Core.

Here are the main factors it considers:

1.  **Sales Consistency (Coefficient of Variation - CoV):**
    *   It looks at the daily sales quantities (Rate of Sale, or ROS).
    *   It calculates the average daily sales (Mean ROS).
    *   It calculates how much the daily sales vary from the average (Standard Deviation).
    *   The **Coefficient of Variation (CoV)** is calculated as `Standard Deviation / Mean`. A *low* CoV means sales are very consistent relative to the average (like selling 2, 3, 2, 3 items each day). A *high* CoV means sales are volatile (like selling 0, 10, 0, 8 items).
    *   **Core Criteria:** Core styles should ideally have a *low* CoV compared to other similar products (e.g., within the same category/brand segment).

2.  **Discount Level:**
    *   It calculates the average discount percentage given for the style over the period.
    *   **Core Criteria:** Core styles should have a *low* average discount, close to or below the benchmark for similar products.

3.  **Sales Activity (Days Sold / Days Live):**
    *   It checks how many days the style was actually available ("live days", using the [View](10_view_.md) helper).
    *   It checks how many of those live days actually had sales ("days sold").
    *   **Core Criteria:** Core styles should have a reasonably high ratio of `Days Sold / Days Live`, indicating they sell consistently when available. There might be specific thresholds (e.g., if live for >90 days, must have sales on at least 70% of those days).

4.  **Minimum Sales Quantity:**
    *   It checks if the total quantity sold over the period meets a minimum threshold. This prevents very low-volume items from being marked as Core just because they are consistent.
    *   **Core Criteria:** Must sell at least a certain number of units (e.g., `minQtySold` from [NoosArgs](03_configuration___arguments__args_classes__.md), possibly adjusted by category).

If a style meets **all** these criteria, the `NoosCoreComputeModule` flags it by setting its theme to `StyleTheme.CORE`.

## How It Works (The Process)

This module runs as part of the sequence managed by [NOOS Identification (NoosGroupModule)](21_noos_identification__noosgroupmodule__.md).

**Inputs:**
*   Filtered, cleaned sales data ([ProductSalesRow](13_productsalesrow_.md) list, often from `NoosData` helper).
*   Historical availability data (via the [View](10_view_.md) helper, which uses results from the [Inventory Creation Module](20_inventory_creation_module_.md)).
*   Product Master Data ([Cache](05_cache_.md) for Style/SKU details).
*   Configuration ([NoosArgs](03_configuration___arguments__args_classes__.md) for `coreDuration`, discount/CoV multipliers, days live/sold percentages, `minQtySold`).
*   Potentially category-specific quantity thresholds (`CatQtyRow`).

**Calculation Steps:**
1.  Determine the analysis date range (e.g., last `coreDuration` months).
2.  Load relevant sales data for this period.
3.  Initialize a map to store results (`coreMap`).
4.  For each Style:
    a.  Calculate total revenue, quantity sold, total discount value over the period using [ProductSalesUtil](14_productsalesutil_.md).
    b.  Calculate the average discount percentage (`avgStyleDisc`) using [MathUtil](12_mathutil_.md).
    c.  Determine "live days" and "days sold" for the style (using [View](10_view_.md) and considering paramount sizes if configured).
    d.  Generate the Rate of Sale (ROS) list (daily quantities sold on days sold, padded with zeros for live days with no sales).
    e.  Calculate the Mean ROS and Standard Deviation of ROS using [MathUtil](12_mathutil_.md).
    f.  Calculate the Coefficient of Variation (`rosCoeff = SD / Mean`).
    g.  Store these calculated metrics in a `CoreRow` object associated with the style ID in `coreMap`.
5.  Calculate Benchmarks: For groups of similar products (e.g., same Category, Subcategory, Brand Segment), calculate the average discount (`discBenchmark`) and average CoV (`rosCoeffBenchmark`) across all styles in that group. Store these benchmarks in the corresponding `CoreRow` objects.
6.  Identify Core: Iterate through the `CoreRow` objects in `coreMap`:
    a.  Check if `qtySold >= minQtySold` criterion is met.
    b.  Check if `daysSold / daysLive` ratio meets the configured thresholds.
    c.  Check if `avgStyleDisc <= discBenchmark * multiplier`.
    d.  Check if `rosCoeff <= rosCoeffBenchmark * multiplier`.
    e.  If ALL checks pass, set `coreRow.theme = StyleTheme.CORE` and update the central `NoosData` theme map. Otherwise, set `theme = StyleTheme.FASHION`.
7.  Save Results: Persist the detailed `CoreRow` objects (often used for debugging/analysis) and ensure the `NoosData` theme map is updated.

**Outputs:**
*   `CoreRow` data is saved (contains calculated metrics and benchmarks for each style).
*   The central `NoosData.styleThemeMap` is updated, marking relevant styles with `StyleTheme.CORE`.

## Under the Hood: Calculations and Comparisons

Let's look at some key parts of the process within `NoosCoreComputeModule`.

**1. Calculating Style Metrics (`createCoreRow` helper):**
This helper function takes the sales data for a single style and calculates the core metrics needed for the evaluation.

```java
// Simplified helper method logic from NoosCoreComputeModule.java
private static CoreRow createCoreRow(StyleRow s, List<ProductSalesRow> sales,
                                     List<Integer> rosList, int daysLive,
                                     int daysSold, /*... other inputs ...*/) {
    CoreRow r = new CoreRow();
    r.style = s.id;
    r.daysLive = daysLive;
    r.daysSold = daysSold;

    // Calculate totals using ProductSalesUtil or simple loops
    r.rev = ProductSalesUtil.getTotalRevenue(sales);
    r.qtySold = ProductSalesUtil.getTotalQtySold(sales);
    r.discValue = ProductSalesUtil.getTotalDiscountValue(sales);

    // Calculate average discount using MathUtil
    r.avgStyleDisc = MathUtil.getPercent(r.discValue, r.discValue + r.rev, 1); // scale=1

    // Calculate ROS Mean and Standard Deviation using MathUtil
    r.ros = MathUtil.meanInteger(rosList);
    double rosStdDev = MathUtil.standardDeviationInteger(rosList);

    // Calculate Coefficient of Variation using MathUtil's safe division
    r.rosCoeff = MathUtil.divide(rosStdDev, r.ros);

    return r;
}
```
**Explanation:** This function calculates sum of revenue, quantity, and discount using helpers. It then uses [MathUtil](12_mathutil_.md) to compute the average discount percentage and the Coefficient of Variation (CoV) based on the provided Rate of Sale list (`rosList`).

**2. Calculating Benchmarks (`computeRolledUpMetadata` / `addRolledUpMetadata`):**
After calculating metrics for individual styles, the module groups styles by shared characteristics (like Category, Subcategory, Brand Segment) and calculates the average metrics for each group.

```java
// Simplified logic for benchmark calculation from NoosCoreComputeModule.java

// Group sales data by Cat/Subcat/BrandSeg Key
Map<Key, List<ProductSalesRow>> group = data.groupBy(o -> new Key(o.style.cat, o.style.subcat, o.style.brandSegment)).getMap();

group.forEach((key, productSalesRowsInGroup) -> {
    // Calculate average discount for the whole group
    double groupDiscPerc = ProductSalesUtil.getDiscountPercent(productSalesRowsInGroup);

    // Calculate average CoV for the group
    double groupRosCoeffSum = 0;
    List<Integer> stylesInGroup = catSubcatBrSegStyleMap.get(key); // Get styles in this group
    for(int styleId : stylesInGroup) {
        groupRosCoeffSum += coreMap.get(styleId).rosCoeff; // Sum individual CoVs
    }
    double groupRosCoeffAvg = MathUtil.divide(groupRosCoeffSum, stylesInGroup.size());

    // Store these benchmarks in each CoreRow belonging to the group
    stylesInGroup.forEach(styleId -> {
        CoreRow r = coreMap.get(styleId);
        r.discBenchmark = groupDiscPerc;
        r.rosCoeffBenchmark = groupRosCoeffAvg;
    });
});
```
**Explanation:** The code groups sales by a composite key (Category/Subcategory/Brand Segment). For each group, it calculates the overall discount percentage and the average CoV across all styles within that group. These calculated group averages (`groupDiscPerc`, `groupRosCoeffAvg`) are then stored as benchmarks (`discBenchmark`, `rosCoeffBenchmark`) in the `CoreRow` of every style belonging to that group.

**3. Identifying Core Styles (`identifyCore` and check methods):**
This is the final decision step where each style's metrics are compared against the criteria.

```java
// Simplified logic from NoosCoreComputeModule.java
private void identifyCore() {
    int coreCount = 0;
    Map<Key, Integer> catQtyMap = ObjectMaps.createCatQtySoldMap(db().select(CatQtyRow.class)); // Min Qty Map

    // Iterate through all calculated CoreRows
    for (CoreRow coreRow : coreMap.values()) {
        // Apply all check criteria
        if (checkMinQtySoldCriteria(coreRow, catQtyMap) &&
            checkDaysSoldCriteria(coreRow) &&
            checkDiscCriteria(coreRow) &&
            checkCoeffVarCriteria(coreRow)) {

            // If all pass, mark as CORE
            coreRow.theme = StyleTheme.CORE;
            noosData.setStyleTheme(coreRow.style, coreRow.theme); // Update shared data
            coreCount++;
        } else {
            // Otherwise, mark as FASHION (Bestseller check comes later)
            coreRow.theme = StyleTheme.FASHION;
        }
    }
    logger.info("Core style count: " + coreCount);
}

// --- Example Check Methods (using NoosArgs for thresholds/multipliers) ---

private boolean checkDiscCriteria(CoreRow r) {
    // Check if style's discount is <= benchmark discount * multiplier
    // Also considers an upper cap on the benchmark
    return r.avgStyleDisc <= (Math.max(r.discBenchmark, noosArgs.avgCatSubCatBrSegAvgDiscUpperCap)
                             * noosArgs.catSubcatBrSegAvgDiscMultiplier);
}

private boolean checkCoeffVarCriteria(CoreRow r) {
    // Check if style's CoV is <= benchmark CoV
    // (Note: Sometimes a multiplier is used here too, but simplified for example)
    return r.rosCoeff <= (r.rosCoeffBenchmark);
}

private boolean checkDaysSoldCriteria(CoreRow r) {
    // Check if daysSold / daysLive ratio meets configured percentage thresholds
    // based on how many days the item was live
    if (r.daysLive >= noosArgs.atleastDaysLive1)
        return r.daysSold >= MathUtil.divide(noosArgs.daysLive1Percentage, 100.0) * r.daysLive;
    if (r.daysLive >= noosArgs.atleastDaysLive2)
        // ... check against daysLive2Percentage ...
    // ... potentially more levels ...
    return false; // Didn't meet any threshold
}

private boolean checkMinQtySoldCriteria(CoreRow r, Map<Key, Integer> catQtyMap) {
    StyleRow styleRow = cache.getStyleRow(r.style);
    // Get min qty threshold for this style's category (or default)
    int minQty = catQtyMap.getOrDefault(new Key(styleRow.cat, StyleTheme.CORE), noosArgs.minQtySold);
    return r.qtySold >= minQty;
}
```
**Explanation:** The `identifyCore` method iterates through each style's `CoreRow`. It calls specific check methods (`checkDiscCriteria`, `checkCoeffVarCriteria`, etc.) which compare the style's calculated metrics (`avgStyleDisc`, `rosCoeff`, `qtySold`, `daysSold/daysLive`) against the benchmarks stored in the `CoreRow` and thresholds/multipliers defined in `NoosArgs`. If all checks pass, the style's theme is set to `CORE`.

## Conclusion

The **`NoosCoreComputeModule`** is the specialist responsible for identifying **Core** styles within the NOOS framework.

*   It analyzes historical sales and inventory data to evaluate styles based on **sales consistency (low CoV)**, **low average discounts**, sufficient **sales activity** (days sold vs. live), and a **minimum sales volume**.
*   It calculates these metrics for each style and compares them against benchmarks derived from similar product groups.
*   Styles meeting all criteria are assigned the `StyleTheme.CORE`.
*   This module relies heavily on utilities like [ProductSalesUtil](14_productsalesutil_.md), [MathUtil](12_mathutil_.md), and [View](10_view_.md), as well as configuration from [NoosArgs](03_configuration___arguments__args_classes__.md).

By systematically identifying these foundational items, `NoosCoreComputeModule` provides crucial input for strategic inventory and planning decisions.

But what about items that aren't necessarily basic but sell extremely well? The next chapter introduces the module that identifies these: the [NoosBestsellerComputeModule](23_noosbestsellercomputemodule_.md).

[Next Chapter: NoosBestsellerComputeModule](23_noosbestsellercomputemodule_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)