# Chapter 12: MathUtil

Welcome back! In the [previous chapter](11_objectmaps_.md), we saw how [ObjectMaps](11_objectmaps_.md) helps organize our data lists into efficient maps for quick lookups, especially useful for the [Cache](05_cache_.md). Now that we have our data organized, many `irisx-algo` modules need to perform calculations – things like finding percentages, rounding numbers correctly, or calculating averages.

Imagine you're working in merchandising. You constantly need to calculate things like:
*   "What was the sell-through rate for this T-shirt?" (Units Sold / Total Initial Units) * 100
*   "What's the average selling price (ASP)?" (Total Revenue / Units Sold)
*   "We calculated a suggested order quantity of 12.34 units, but we need to round that to a whole number, or maybe to two decimal places for a cost."
*   "How consistent are the weekly sales for this item?" (Standard Deviation)

Doing these calculations repeatedly across many different modules can lead to duplicated code and potential errors (like accidentally dividing by zero!).

## What Problem Does MathUtil Solve?

Think of `MathUtil` as the standard **calculator toolbox** for the entire `irisx-algo` application. It gathers many common mathematical functions needed for merchandising calculations into one single, reliable place.

Instead of each module writing its own code to:
*   Calculate percentages safely.
*   Round numbers to specific decimal places (very important for currency or metrics).
*   Find the average (mean) of a set of numbers.
*   Calculate the standard deviation (to measure how much sales numbers vary).
*   Perform division without crashing if the denominator is zero.

...they can simply use the ready-made tools provided by `MathUtil`. This ensures calculations are done **consistently** and **safely** everywhere.

## Core Idea: A Central Toolbox of Math Functions

`MathUtil` is a utility class containing **static methods**. This means you don't need to create an instance of `MathUtil`; you just call the methods directly using the class name, like `MathUtil.round(...)` or `MathUtil.divide(...)`.

It provides functions for:

*   **Percentages:** Calculating what percentage one number is of another (e.g., `getPercent`, `percent`).
*   **Rounding:** Adjusting a number to a specific number of decimal places, with control over how rounding occurs (e.g., `round`).
*   **Averages:** Calculating the mean (sum of values / number of values) (e.g., `mean`, `meanInteger`).
*   **Standard Deviation:** Measuring the spread or variability in a list of numbers (e.g., `standardDeviation`, `standardDeviationInteger`). A low standard deviation means numbers are close to the average (e.g., consistent sales), while a high standard deviation means they are spread out (e.g., volatile sales).
*   **Safe Division:** Performing division while automatically handling cases where the divisor is zero (returning 0 instead of causing an error) (e.g., `divide`, `ratio`).

## How to Use MathUtil

Using `MathUtil` is straightforward. You just call the static method you need and provide the required numbers as input.

**Use Case:** Let's calculate the sell-through rate (STR) for a product. We sold 70 units, and the starting inventory was 100 units. We also need to calculate the average selling price (ASP) if the total revenue was $1500.50 from those 70 units, rounded to 2 decimal places.

```java
// Inside some module's logic...
import com.increff.irisx.util.MathUtil; // Import the toolbox
import java.math.RoundingMode; // To specify how to round
import java.util.Optional; // Used for optional parameters

// --- Sell-Through Rate (STR) Calculation ---
double unitsSold = 70.0;
double startingInventory = 100.0; // Note: STR often uses Sold / (Sold + Ending Stock) or Sold / Starting Stock. Using starting stock here for simplicity.

// Calculate STR percentage, rounded to 1 decimal place
double strPercentage = MathUtil.getPercent(unitsSold, startingInventory, 1); // numerator, denominator, scale

System.out.println("Sell-Through Rate: " + strPercentage + "%");

// --- Average Selling Price (ASP) Calculation ---
double totalRevenue = 1500.50;
double quantitySold = 70.0;

// Calculate ASP using safe division
double aspRaw = MathUtil.divide(totalRevenue, quantitySold);
System.out.println("Raw ASP: " + aspRaw);

// Round the ASP to 2 decimal places (typical for currency)
// Optional.empty() means use default rounding (HALF_UP)
double aspRounded = MathUtil.round(aspRaw, 2, Optional.empty());
System.out.println("Rounded ASP: $" + aspRounded);

// --- Example: What if we had 0 sales? ---
double zeroSalesRevenue = 0.0;
double zeroQuantity = 0.0;
double aspWithZeroSales = MathUtil.divide(zeroSalesRevenue, zeroQuantity);
System.out.println("ASP with zero sales (safe division): " + aspWithZeroSales);
```

**Explanation:**

*   `MathUtil.getPercent(70.0, 100.0, 1)` calculates (70 / 100) * 100 and rounds the result to 1 decimal place.
*   `MathUtil.divide(1500.50, 70.0)` safely calculates 1500.50 / 70.0. If `quantitySold` were 0, it would return 0.0 instead of throwing an error.
*   `MathUtil.round(aspRaw, 2, Optional.empty())` takes the raw ASP and rounds it to 2 decimal places using the default rounding mode (usually rounding up from .5).
*   The last example shows `MathUtil.divide` returning 0 when dividing by zero, preventing a program crash.

**Expected Output:**

```
Sell-Through Rate: 70.0%
Raw ASP: 21.435714285714286
Rounded ASP: $21.44
ASP with zero sales (safe division): 0.0
```

## Under the Hood: Simple, Static Functions

The methods inside `MathUtil` are typically direct implementations of standard mathematical formulas, with added checks for edge cases like division by zero or invalid inputs.

Let's look at how `divide` and `round` might work internally.

**Walkthrough: `MathUtil.divide(n, d)`**

1.  **Input:** Receives the numerator `n` and the denominator `d`.
2.  **Check Denominator:** It first checks if `d` is equal to 0.
3.  **Handle Zero:** If `d` is 0, it immediately returns `0.0` to avoid a division-by-zero error.
4.  **Perform Division:** If `d` is not 0, it performs the division `n / d`.
5.  **Round Result:** It often calls its own `round` method internally to ensure the result has a standard number of decimal places (defined by `GenericConstants.ROUNDING_SCALE`).
6.  **Return Result:** Returns the calculated (and potentially rounded) result.

**Walkthrough: `MathUtil.round(n, scale, mode)`**

1.  **Input:** Receives the number `n` to round, the desired `scale` (number of decimal places), and an `Optional<RoundingMode>` specifying the rounding behavior (e.g., `HALF_UP`, `DOWN`).
2.  **Use BigDecimal:** To handle rounding accurately, especially with floating-point numbers, it converts the input `double n` into a `java.math.BigDecimal` object. `BigDecimal` is designed for precise numerical calculations.
3.  **Set Scale:** It calls the `setScale()` method on the `BigDecimal` object, passing the desired `scale` and the `RoundingMode` (using `RoundingMode.HALF_UP` if the `mode` parameter is empty).
4.  **Convert Back:** It converts the resulting `BigDecimal` (which is now correctly rounded) back to a `double`.
5.  **Return Result:** Returns the rounded `double` value.

**Code Dive (`MathUtil.java`):**

Here are simplified versions of some methods from `MathUtil.java`:

```java
// Simplified from: src/main/java/com/increff/irisx/util/MathUtil.java
package com.increff.irisx.util;

import com.increff.irisx.constants.GenericConstants; // For default scale
import java.math.BigDecimal;
import java.math.RoundingMode;
import java.util.Optional;
import java.util.List; // For methods like mean, stddev

public class MathUtil {

    /** Safely divides n by d, returning 0 if d is 0. Rounds to default scale. */
    public static double divide(double n, double d) {
        // Check for zero denominator!
        if (d == 0) {
            return 0.0;
        }
        // Perform division and round using the default scale and mode
        return round(n / d, GenericConstants.ROUNDING_SCALE, Optional.empty());
    }

    /** Rounds a double n to a specified scale using a specific RoundingMode. */
    public static double round(double n, int scale, Optional<RoundingMode> mode) {
        // Use BigDecimal for accurate rounding
        BigDecimal bigDecimal = BigDecimal.valueOf(n);

        // Determine the rounding mode - use HALF_UP if none provided
        RoundingMode roundingMode = mode.orElse(RoundingMode.HALF_UP);

        // Set the scale (number of decimal places) and apply rounding
        BigDecimal roundedBigDecimal = bigDecimal.setScale(scale, roundingMode);

        // Convert back to double and return
        return roundedBigDecimal.doubleValue();
    }

    /** Calculates percentage, handling invalid inputs. */
    public static double getPercent(double n, double d, int scale) {
        // Handle edge cases: numerator negative, denominator zero/negative, n > d
        if (n < 0 || d <= 0 || n > d) {
            return 0.0;
        }
        // Calculate raw percentage
        double p = n * 100.0 / d;
        // Round to specified scale
        p = round(p, scale, Optional.empty());
        // Ensure percentage doesn't exceed 100 (due to potential rounding)
        p = Math.min(p, 100.0);
        return p;
    }

    /** Calculates the average (mean) of a list of integers. */
    public static double meanInteger(List<Integer> data) {
        if (data == null || data.isEmpty()) {
            return 0.0; // Avoid division by zero if list is empty
        }
        int sum = 0;
        for (int d : data) {
            sum += d;
        }
        // Use safe division
        return divide(sum, data.size());
    }

    // ... other methods like standardDeviation, sum, ratio, etc. ...
}
```

**Explanation:**

*   `divide`: Clearly shows the check `if (d == 0)` returning 0, otherwise calculating `n / d` and calling `round`.
*   `round`: Shows the core logic using `BigDecimal.valueOf(n).setScale(scale, roundingMode).doubleValue()`.
*   `getPercent`: Includes checks for invalid inputs (`n < 0`, `d <= 0`, `n > d`) before calculating the percentage and rounding.
*   `meanInteger`: Shows how it sums the list and then uses the safe `divide` method to calculate the average, handling empty lists.

## Conclusion

**`MathUtil`** serves as the central **mathematical toolkit** for `irisx-algo`, providing essential functions needed for various merchandising calculations.

*   It offers consistent and safe methods for **percentages, rounding, averages, standard deviation, and division**.
*   Using `MathUtil` avoids code duplication and ensures calculations are performed reliably across different modules.
*   Methods are **static**, allowing easy use via `MathUtil.methodName(...)`.
*   Key features include **safe division** (handling zero denominators) and precise **rounding** using `BigDecimal`.

By relying on `MathUtil`, developers can focus on the specific logic of their modules without worrying about the underlying implementation details of common mathematical operations.

Now that we have a grasp of basic data structures, caching, and utility functions like `ObjectMaps` and `MathUtil`, we can start looking at more specific data representations used in the algorithms. The next chapter introduces [ProductSalesRow](13_productsalesrow_.md), a key object for representing product sales data.

[Next Chapter: ProductSalesRow](13_productsalesrow_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)