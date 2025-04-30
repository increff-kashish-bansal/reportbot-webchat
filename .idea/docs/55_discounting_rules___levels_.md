# Chapter 55: Discounting Rules & Levels

Welcome back! In the [previous chapter](54_dynamic_discounting_module_.md), we explored the **Dynamic Discounting Module**, which acts like an automated pricing analyst, deciding whether to increase, decrease, or continue the current discount for a product based on its performance.

But how does it make those decisions? Just like our savvy fruit stand owner from the last chapter needs some guidelines ("If the bananas are getting spotty AND I have lots left, lower the price by 20 cents"), the Dynamic Discounting module needs a set of instructions. It needs a "brain" that tells it *what* action to take under *which* specific circumstances.

## What Problem Does This Concept Solve?

The Dynamic Discounting module looks at several factors for each product: how fast it's selling (ROS), how much stock is left (DOH/Ageing), how well it's displayed (Health Status), and how much has sold compared to what was received (Sell-Through). But knowing these numbers isn't enough. The system needs rules to interpret them.

For example:
*   Is a ROS of 2 units/day "HIGH" or "LOW"? It depends on the product!
*   If ROS is LOW and DOH (inventory cover) is HIGH, should we always increase the discount? What if the display health is poor?
*   How much should we increase the discount by? 5%? 10%?
*   Should the discount apply to all sizes of a T-shirt (Style level) or just the slow-moving XXL size (SKU level)?

Manually coding all these specific rules directly into the discounting module would make it very rigid and hard to change. If the business wants to try a different discounting strategy next season, programmers would have to rewrite the code.

**Discounting Rules & Levels** solve this by providing a **configurable "brain"** for the Dynamic Discounting module. Instead of hard-coding the logic, we define the rules, benchmarks, limits, and application level through configuration data. This allows the business to easily adjust the discounting strategy without changing the core program code.

## Core Concepts: The Ingredients of the "Brain"

Think of this configuration as the recipe book the Dynamic Discounting module follows. It includes several key ingredients:

1.  **The Decision Matrix (`DiscountRulesRow`):** This is the core rulebook. It's like a big table or flowchart that says: "IF the Rate of Sale (ROS) level is X, AND the Ageing/DOH level is Y, AND the Health Status is Z, AND the Sell-Through level is W, THEN the suggested Discount Action is A (Increase, Decrease, or Continue)."
    *   **Inputs:** Uses performance levels ([`RosLevel`](#roslevel-enum), [`AgeingLevel`](#ageinglevel-enum), [`HealthStatus`](#healthstatus-enum), [`SellThroughLevel`](#sellthroughlevel-enum)).
    *   **Output:** Defines the [`DiscountAction`](#discountaction-enum).
    *   It might also specify *how much* discount to apply based on rules (`percOfMaxDiscount`, `percOfDiscountPerc`).

2.  **Benchmarks (`SellthroughBenchmarkRow`):** How do we know if a Sell-Through rate of 30% after 4 weeks is HIGH, MEDIUM, or LOW? Benchmarks provide the context. This input defines target sell-through percentages for different stages of a product's lifecycle (e.g., after 4 weeks, 8 weeks, 12 weeks) for different categories. The module compares the actual sell-through against these benchmarks to determine the `SellThroughLevel`.
    *   *Example:* For T-shirts after 4 weeks, maybe 0-20% is LOW, 21-50% is MEDIUM, and 51%+ is HIGH.

3.  **Guardrails & Increments (`DiscountIncrementRow`):** These set the boundaries and step sizes for discounts.
    *   **Min/Max Discount:** Defines the absolute lowest and highest discount percentage allowed for a group of products (e.g., Brand/Category/Season).
    *   **Start Discount:** The initial discount applied if an item qualifies for its first markdown.
    *   **Incremental Discount:** How much the discount should change with each "INCREASE" or "DECREASE" action (e.g., change by 5% each time).

4.  **Discount Application Level (`DiscountLevel` enum):** This simple configuration tells the module whether the calculated discount recommendation should apply to:
    *   **`STYLE`:** The same discount percentage is applied to all SKUs (sizes) belonging to that style.
    *   **`SKU`:** Each individual SKU (size) gets its own discount calculation and recommendation based on its specific performance.

These configurable pieces together form the complete logic the Dynamic Discounting module uses to make its recommendations.

## How It Works: Using the Rules

Let's revisit the workflow from the [Dynamic Discounting Module](54_dynamic_discounting_module_.md) chapter, focusing on how these configurations are used:

1.  **Calculate Metrics:** The module calculates ROS, DOH, Age, Sell-Through, and gets Health Status for an item (Style or SKU).
2.  **Determine Levels:**
    *   Compares calculated ROS against peer group performance to determine `RosLevel` (HIGH/MEDIUM/LOW).
    *   Compares calculated DOH/Age against benchmarks to determine `AgeingLevel` (LEADING/ON_COURSE/LAGGING).
    *   Compares calculated Sell-Through against `SellthroughBenchmarkRow` data (considering lifecycle stage) to determine `SellThroughLevel` (HIGH/MEDIUM/LOW).
    *   Gets `HealthStatus` (HEALTHY/UNHEALTHY).
3.  **Lookup Action:** Uses the combination of `RosLevel`, `AgeingLevel`, `HealthStatus`, and `SellThroughLevel` as a key to find the matching rule in the `DiscountRulesRow` data. This gives the suggested `DiscountAction` (INCREASE, DECREASE, CONTINUE).
4.  **Calculate New Discount:**
    *   Retrieves the current discount for the item.
    *   Looks up the applicable `DiscountIncrementRow` based on Brand/Category/Store Group etc.
    *   If action is INCREASE, adds the `incrementalDiscount`.
    *   If action is DECREASE, subtracts the `incrementalDiscount`.
    *   If action is CONTINUE, keeps the current discount.
    *   Ensures the result is not below 0% and not above the `maxDiscount` defined in `DiscountIncrementRow`.
5.  **Output Recommendation:** Records the `suggestedDiscountAction` and the final calculated `discountRecommendation` percentage in the `DDDiscountingOutputRow`. This happens for every Style or SKU, depending on the configured `DiscountLevel`.

**Example Rule Application:**

*   **Item:** Style 123 (T-Shirt) in Store Group "Urban"
*   **Metrics:** ROS Level=LOW, Ageing Level=LAGGING, Health=HEALTHY, SellThrough Level=LOW
*   **Rule Lookup (`DiscountRulesRow`):** Find rule where `ros=LOW`, `doh_ageing=LAGGING`, `health=HEALTHY`, `sellThroughLevel=LOW`. Let's say this rule specifies `action=INCREASE`.
*   **Increment Lookup (`DiscountIncrementRow`):** Find rule for T-Shirts in "Urban" stores. Let's say `incrementalDiscount=10.0`, `maxDiscount=50.0`.
*   **Current Discount:** Let's say it's currently 20%.
*   **Calculation:** New Discount = 20% + 10% = 30%. Check limits: 30% is <= 50% (max).
*   **Recommendation:** Action=INCREASE, Recommendation=30.0%.

## Under the Hood: Data Structures and Lookup

The core "brain" components are represented by simple Row classes, loaded into memory (often in the `DynamicDiscountingData` object) for quick access by the computation modules.

**1. Code Dive: Input Row Structures:**

*   **`DiscountRulesRow.java`:** Defines the decision matrix.
    ```java
    // File: src/main/java/com/increff/irisx/row/input/dynamicDiscounting/DiscountRulesRow.java
    package com.increff.irisx.row.input.dynamicDiscounting;

    import com.increff.irisx.constants.dynamicDiscounting.*; // Import the Level enums

    // Represents one rule in the decision matrix
    public class DiscountRulesRow {
        // --- Conditions (Input Levels) ---
        public RosLevel ros;                 // e.g., LOW, MEDIUM, HIGH
        public AgeingLevel doh_ageing;        // e.g., LAGGING, ON_COURSE, LEADING
        public HealthStatus health;          // e.g., HEALTHY, UNHEALTHY
        public SellThroughLevel sellThroughLevel; // e.g., LOW, MEDIUM, HIGH

        // --- Outcome ---
        public DiscountAction action;        // e.g., INCREASE, DECREASE, CONTINUE

        // --- Optional: Specific Discount Calculation Rules ---
        public double percOfMaxDiscount; // Target a % of the max allowed discount
        public double percOfDiscountPerc; // Target a % increase/decrease? (Less common)

        // Constructors...
    }
    ```
    **Explanation:** Each row represents one possible combination of performance levels and the corresponding action to take. The module finds the row that matches the item's current state.

*   **`SellthroughBenchmarkRow.java`:** Defines targets for SellThrough classification.
    ```java
    // File: src/main/java/com/increff/irisx/row/input/dynamicDiscounting/SellthroughBenchmarkRow.java
    package com.increff.irisx.row.input.dynamicDiscounting;

    // Defines benchmarks for classifying Sell-Through level
    public class SellthroughBenchmarkRow {
        public String cat;    // Category
        public String subcat; // Subcategory
        public String gender; // Gender
        public int lifecycle;  // Weeks into lifecycle (e.g., 4, 8, 12)
        public int minLiveDays; // Minimum days item must be live for benchmark to apply
        // Thresholds (Cumulative Sell-Through %)
        public double high;   // % needed to be considered HIGH sell-through
        public double medium; // % needed to be considered MEDIUM sell-through (LOW is below this)
    }
    ```
    **Explanation:** This defines, for a given product group (Cat/Subcat/Gender) at a certain point in its life (`lifecycle` weeks), what sell-through percentage qualifies as HIGH or MEDIUM.

*   **`DiscountIncrementRow.java`:** Defines the guardrails and step size.
    ```java
    // File: src/main/java/com/increff/irisx/row/input/dynamicDiscounting/DiscountIncrementRow.java
    package com.increff.irisx.row.input.dynamicDiscounting;

    // Defines discount limits and increments for a group
    public class DiscountIncrementRow {
        public String brand;
        public String styleGroup; // Broader style grouping
        public String category;
        public String storeGroup; // Group of stores
        public String seasonType; // e.g., FullPrice, EOSS

        public double startDiscount;       // Initial discount % if action=INCREASE from 0%
        public double maxDiscount;         // Maximum allowed discount %
        public double incrementalDiscount; // % to add/subtract per action
    }
    ```
    **Explanation:** This sets the rules for how much the discount can change and the maximum it can reach for specific product/store groups and season types.

*   **`DiscountLevel.java`:** Simple enum for the application level.
    ```java
    // File: src/main/java/com/increff/irisx/constants/dynamicDiscounting/DiscountLevel.java
    package com.increff.irisx.constants.dynamicDiscounting;

    // Defines whether discounts apply at Style or SKU level
    public enum DiscountLevel {
        STYLE("STYLE"),
        SKU("SKU");

        private String val;
        DiscountLevel(String val) { this.val = val; }
        // ... getValue() might exist ...
    }
    ```

**2. Conceptual Lookup and Application:**

```java
// Conceptual snippet within AbstractDiscountComputationModule

// 1. Determine Levels for item 'itemData'
RosLevel rosL = classifyRos(itemData);
AgeingLevel ageL = classifyAgeing(itemData);
SellThroughLevel stL = classifySellThrough(itemData, lifecycleWeek);
HealthStatus healthS = getHealthStatus(itemData);

// 2. Lookup Action from rules map (loaded from DiscountRulesRow)
DiscountAction action = dynamicDiscountingData.getDiscountAction(rosL, ageL, healthS, stL);
itemData.setSuggestedDiscountAction(action);

// 3. Get Increment/Limits from increment map (loaded from DiscountIncrementRow)
DiscountIncrementData incData = dynamicDiscountingData.getDiscountIncrementData(/* key */);
double increment = incData.getIncrementalDiscount();
double maxDiscount = incData.getMaxDiscount();
double currentDiscount = itemData.getExistingDiscount();

// 4. Calculate New Discount based on action
double potentialNewDiscount;
if (action == DiscountAction.INCREASE) {
    potentialNewDiscount = currentDiscount + increment;
} else if (action == DiscountAction.DECREASE) {
    potentialNewDiscount = currentDiscount - increment;
} else {
    potentialNewDiscount = currentDiscount;
}

// 5. Apply Guardrails
double finalDiscount = Math.min(Math.max(potentialNewDiscount, 0.0), maxDiscount); // Clamp between 0 and max
itemData.setSuggestedDiscount(finalDiscount);

// 6. Apply at Style or SKU level based on DiscountLevel argument
if (args.discountLevel == DiscountLevel.STYLE) {
    // Apply finalDiscount to all SKUs of the style
} else {
    // Apply finalDiscount only to this specific SKU
}
```
**Explanation:** This conceptual flow shows how the calculated performance levels are used to look up an action from the pre-loaded rules. Then, the corresponding increment/decrement rules are retrieved, the new discount is calculated based on the action, and finally, it's clamped within the min/max limits. The application level (Style/SKU) would determine how this final value is stored or applied.

## Conclusion

**Discounting Rules & Levels** provide the essential configuration and logic that drive the [Dynamic Discounting Module](54_dynamic_discounting_module_.md).

*   They define **what action to take** (`DiscountRulesRow`) based on combinations of performance levels (ROS, Ageing/DOH, Health, Sell-Through).
*   They provide **benchmarks** (`SellthroughBenchmarkRow`) to classify performance metrics into these levels.
*   They set **guardrails** (`DiscountIncrementRow`) defining min/max allowable discounts and the step size for adjustments.
*   They determine **where** the discount applies (`DiscountLevel` enum - Style or SKU).

By externalizing this logic into configurable inputs, `irisx-algo` allows businesses to flexibly define and adapt their dynamic discounting strategies without needing code changes, making the system responsive to different market conditions and business goals.

Beyond optimizing pricing and inventory for individual items, how can we analyze the overall performance of our assortment plan compared to actual results or identify gaps?

[Next Chapter: Gap Analysis Module](56_gap_analysis_module_.md)
```

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)