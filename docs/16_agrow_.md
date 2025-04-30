# Chapter 16: AgRow

Welcome back! In the [previous chapter](15_price_bucket_creation_module_.md), we learned how the [Price Bucket Creation Module](15_price_bucket_creation_module_.md) helps us group products based on their price points, creating simpler categories like "Budget" or "Premium". This is one way to group similar items.

But what if we want to group products based on *many* characteristics at once? For example, maybe we want to analyze or plan for all "Men's, Cotton, V-Neck, Premium-Priced, Blue T-Shirts from Brand X". Handling such complex combinations individually is difficult. How can we give each unique combination a simple label or ID?

## What Problem Does AgRow Solve?

Imagine you're organizing a massive library. You could group books just by genre ("Fiction", "Science") or just by author. But sometimes you need more specific groups, like "Science Fiction books by Isaac Asimov published in the 1950s". Manually tracking every book that fits this description is cumbersome. What if you could create a unique label or catalog number specifically for *this exact combination* of genre, author, and publication decade?

That's what **`AgRow`** (Attribute Group Row) does for products in `irisx-algo`. Products have many attributes:
*   Category (e.g., "T-Shirt")
*   Subcategory (e.g., "V-Neck")
*   Brand (e.g., "BrandX")
*   Price Bucket (e.g., "Premium")
*   Gender (e.g., "Men")
*   Theme (e.g., "CORE")
*   Specific attributes like Fabric, Color, Sleeve Length, etc. (Attribute1 to Attribute10)

Analyzing or applying rules based on specific *combinations* of these attributes (like our "Men's, Cotton, V-Neck, Premium-Priced, Blue T-Shirts from Brand X" example) is very common in merchandising. `AgRow` solves the problem of representing these complex combinations simply.

It acts like a **unique summary tag** or **catalog number** for each distinct combination of attributes found in your products.

## Core Idea: The Attribute Combination Tag

An `AgRow` object represents one specific, unique combination of key product attributes. It holds two main things:

1.  **Unique ID (`id`):** An integer that uniquely identifies this specific combination of attributes. No other combination will have the same ID.
2.  **Attribute Values:** It stores the actual values for *all* the attributes that define this group (category, subcategory, brand, price bucket, gender, theme, and the 10 generic attributes `a1` through `a10`).

Think of it like this:

*   **Combination:** Men's / T-Shirt / V-Neck / BrandX / Premium / Blue / Cotton ...
*   **`AgRow` ID:** 1052 (A unique number assigned to *only* this combination)
*   **`AgRow` Object:** Holds the ID `1052` AND all the attribute values ("Men's", "T-Shirt", "BrandX", "Premium", "Blue", "Cotton", etc.).

Modules that group products based on these attributes (like `AgGroupModule`, covered in the [next chapter](17_attribute_grouping__aggroupmodule___agcomputemodule__.md)) are responsible for *creating* these `AgRow` objects and assigning the unique IDs.

Other modules then use the `AgRow` object itself, or more commonly, just its **`id`**, to:
*   **Aggregate Data:** Sum up sales, inventory, or forecasts for all products belonging to that specific attribute group (AgRow ID).
*   **Apply Rules:** Set inventory targets, define planning parameters, or apply promotion rules at the group level.

Using the `AgRow` ID (like `1052`) is much simpler than listing out all the individual attributes ("Men's", "T-Shirt", etc.) every time you want to refer to that group.

## How AgRow is Used

Modules typically don't create `AgRow`s themselves. They either receive them as part of their input data or look them up (often from the [Cache](05_cache_.md)).

**Use Case:** An inventory planning module needs to calculate the target stock level for a group of products defined by `AgRow` ID 1052. It might need to know the brand associated with this group to apply a brand-specific rule.

**1. Getting the AgRow (Conceptual):**
   The module might have a way to get the `AgRow` object corresponding to an ID, perhaps from a map loaded into the Cache.

   ```java
   // Assume 'agRowMap' is a Map<Integer, AgRow> loaded in the Cache
   Map<Integer, AgRow> agRowMap = cache.getAgRowMap(); // Conceptual cache access

   int targetAgId = 1052;
   AgRow attributeGroup = agRowMap.get(targetAgId);
   ```

**2. Accessing AgRow Fields:**
   Once the module has the `AgRow` object, it can easily access the specific attribute values stored within it.

   ```java
   // Continuing from above...
   import com.increff.irisx.row.output.ag.AgRow; // Import the class definition

   if (attributeGroup != null) {
       // Access the fields directly
       String category = attributeGroup.cat;
       String brand = attributeGroup.brand;
       String priceBucket = attributeGroup.priceBucket;
       String fabric = attributeGroup.a1; // Assuming 'a1' represents Fabric

       System.out.println("Planning for AG ID: " + attributeGroup.id);
       System.out.println("  Category: " + category); // e.g., T-Shirt
       System.out.println("  Brand: " + brand);       // e.g., BrandX
       System.out.println("  Price Bucket: " + priceBucket); // e.g., Premium
       System.out.println("  Fabric (a1): " + fabric);   // e.g., Cotton

       // Now use 'brand' to apply a brand-specific planning rule...
       // if ("BrandX".equals(brand)) { ... }

   } else {
       System.out.println("Attribute Group ID " + targetAgId + " not found.");
   }
   ```

**Explanation:**

*   The code retrieves the `AgRow` object for ID `1052` (presumably created by an earlier grouping module and stored in the Cache).
*   It then accesses the `cat`, `brand`, `priceBucket`, and `a1` fields directly from the `attributeGroup` object.
*   This allows the planning module to easily get the context (like the brand) associated with the group ID without needing complex lookups.

**Expected Output:**

```
Planning for AG ID: 1052
  Category: T-Shirt
  Brand: BrandX
  Price Bucket: Premium
  Fabric (a1): Cotton
```

**Using the ID for Aggregation:**
Often, modules use the `AgRow` ID as a key to aggregate results.

```java
// Example: Aggregating sales revenue by AgRow ID

import com.increff.irisx.row.input.transactional.ProductSalesRow;
import java.util.List;
import java.util.Map;
import java.util.HashMap;

// Assume 'salesList' is a List<ProductSalesRow>
// Each ProductSalesRow has an 'ag' field holding the AgRow object (or just its ID)
Map<Integer, Double> revenueByAgId = new HashMap<>();

for (ProductSalesRow psRow : salesList) {
    if (psRow.ag != null && psRow.sale != null) {
        int agId = psRow.ag.id; // Get the unique ID for the product's group
        double revenue = psRow.sale.revenue;

        // Add revenue to the total for this AgRow ID
        revenueByAgId.merge(agId, revenue, Double::sum);
    }
}

// Now 'revenueByAgId' holds total revenue grouped by attribute group ID
System.out.println("Revenue aggregated by AgRow ID: " + revenueByAgId);
// Example Output: {1052=5500.75, 1053=1200.0, 2100=800.50, ...}
```

**Explanation:**
This code iterates through enriched sales records. For each sale, it gets the `AgRow` ID (`psRow.ag.id`) associated with the product sold and uses that ID as a key to sum up revenue in the `revenueByAgId` map. This is a common pattern where the `AgRow` ID serves as the identifier for a specific group of products.

## Under the Hood: The AgRow Structure

`AgRow` itself is a straightforward data container, a [Row Input/Output Class](09_row_input_output_classes_.md).

**Structure (`AgRow.java`):**

```java
// Simplified from: src/main/java/com/increff/irisx/row/output/ag/AgRow.java
package com.increff.irisx.row.output.ag;

import com.increff.irisx.constants.ap.StyleTheme;
import java.util.Objects;
// ... other imports

// Represents a finalized Attribute Group combination
public class AgRow {

    // The unique identifier for this attribute combination
    public int id;

    // Fields holding the values for the attributes defining this group
    public String cat;
    public String subcat;
    public String brandSegment;
    public String brand;
    public String priceBucket;
    public String gender;
    public StyleTheme theme; // Enum for Style Theme
    public String ag; // Often a concatenated string representation (legacy?)
    public String a1; // Attribute 1 (e.g., Fabric)
    public String a2; // Attribute 2 (e.g., Color)
    public String a3;
    public String a4;
    public String a5;
    public String a6;
    public String a7;
    public String a8;
    public String a9;
    public String a10; // Attribute 10

    // Default constructor
    public AgRow() {
        this.theme = StyleTheme.EMPTY; // Initialize theme
    }

    // Helper method to get a specific attribute value by number
    public String getAgAttribute(int agNo) {
        switch (agNo) {
            case 1: return a1;
            case 2: return a2;
            // ... cases 3 through 9 ...
            case 10: return a10;
            default: return null;
        }
    }

    // Standard equals() and hashCode() based on the unique 'id'
    @Override
    public boolean equals(Object o) {
        if (this == o) return true;
        if (!(o instanceof AgRow)) return false;
        AgRow agRow = (AgRow) o;
        return id == agRow.id; // Equality is determined solely by the ID
    }

    @Override
    public int hashCode() {
        return Objects.hash(id); // Hash code is based solely on the ID
    }
}
```

**Explanation:**

*   It's primarily composed of `public` fields storing the unique `id` and the values for all the defining attributes (`cat`, `brand`, `a1`, `a2`, etc.).
*   It includes a helper `getAgAttribute(int no)` to retrieve attributes `a1` through `a10` by number.
*   The `equals()` and `hashCode()` methods are crucially based *only* on the `id` field. This means two `AgRow` objects are considered equal if and only if they have the same ID, reinforcing that the ID is the unique identifier for the attribute combination.

**`AgInterimRow` vs `AgRow`:**
You might also see `AgInterimRow`. This is a helper class used *during* the attribute grouping process (e.g., in `AgGroupModule`) to hold attribute combinations *before* they are assigned a final, unique `id` and stored as an `AgRow`. `AgRow` is the final, stable output representation of an attribute group.

## Conclusion

The **`AgRow`** object serves as a crucial **summary tag** or **unique identifier** for a specific combination of product attributes in `irisx-algo`.

*   It bundles a unique **`id`** with the values of all defining attributes (category, brand, price bucket, gender, theme, a1-a10).
*   It solves the problem of referring to complex product groups in a simple way.
*   Modules use the `AgRow` object or, more commonly, its `id`, to **aggregate data** or **apply rules** to groups of products sharing the same characteristics.
*   It's typically created by specialized grouping modules and used by downstream modules for analysis and planning.

Understanding `AgRow` is key to seeing how `irisx-algo` manages and operates on detailed product segments.

Now that we know what an `AgRow` *is*, how are these unique attribute group combinations actually identified and how are the `AgRow` objects created? The next chapter dives into the modules responsible for this process: [Attribute Grouping (AgGroupModule & AgComputeModule)](17_attribute_grouping__aggroupmodule___agcomputemodule__.md).

[Next Chapter: Attribute Grouping (AgGroupModule & AgComputeModule)](17_attribute_grouping__aggroupmodule___agcomputemodule__.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)