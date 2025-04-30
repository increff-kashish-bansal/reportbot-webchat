# Chapter 3: Configuration & Arguments (Args Classes)

In the [previous chapter](02_worker_api_.md), we saw how the [Worker API](02_worker_api_.md) acts as an entry point for specific jobs, taking instructions (like client name, task ID, and which module to run) and using the [Application API (AppApi)](01_application_api__appapi__.md) to execute them.

But how do the individual algorithm modules know *exactly* how they should run? For example, if we're running the NOOS (Never Out Of Stock) calculation, how does it know:

*   What percentage of revenue defines a "top" product? (`top_x_rev_contri_percentage`)
*   How many past days of sales data should it consider for "bestsellers"? (`bestseller_duration`)
*   What's the minimum quantity sold for an item to be considered? (`min_qty_sold`)

These are like the specific settings or knobs for the algorithm. Where do these settings come from, and how does the module access them easily and reliably?

## What Problem Do Args Classes Solve?

Imagine you're ordering a custom pizza. You need to specify the size, crust type, sauce, cheese, and toppings. The pizza place needs a clear, structured way to receive your order and ensure they don't miss anything or misunderstand your choices.

Similarly, `irisx-algo` modules need specific **parameters** or **arguments** to function correctly. These parameters might come from:

*   **Configuration Files:** Often `.properties` or `.tsv` files containing key-value pairs (e.g., `bestseller_duration=180`).
*   **Task Inputs:** Parameters passed directly when a job is triggered.

Manually reading these settings directly from files or input objects inside *every* module would be messy and repetitive:

1.  You'd need code in each module to find and read the configuration source.
2.  You'd have to convert values from strings (which is how they are often stored) to the correct data type (like numbers, dates, or booleans).
3.  You'd need error handling in case a required setting is missing or has the wrong format.

**Args Classes** solve this by acting like a **structured settings sheet** or **order form** for each module (or group of related modules).

## Core Idea: The Args Class Pattern

The core idea is simple:

1.  **Define Expectations:** For each major module or area (like NOOS, Distribution, Optimum Depth), create a dedicated Java class (e.g., `NoosArgs`, `DistributionArgs`, `OdArgs`).
2.  **Inherit Structure:** These classes usually inherit from a base class called `AbstractArgs`. `AbstractArgs` provides helpful tools for reading and converting settings.
3.  **Declare Fields:** Inside the specific `Args` class (like `NoosArgs`), declare public fields for each expected parameter (e.g., `bestsellerDuration`, `minQtySold`).
4.  **Fetch and Convert:** The constructor of the `Args` class takes the raw configuration (usually as a `java.util.Properties` object) and uses methods from `AbstractArgs` to:
    *   Look up the value for each expected parameter key (e.g., find the value associated with the key `"bestseller_duration"`).
    *   Convert the value (usually a String) to the correct type (e.g., convert `"180"` to the integer `180`).
    *   Store the converted value in the corresponding field.
5.  **Easy Access:** The module code can then create an instance of its `Args` class and access the parameters directly through the well-typed fields (e.g., `noosArgs.bestsellerDuration`).

Think of `AbstractArgs` as the template for the order form, providing boxes and instructions on how to fill them. Each specific `Args` class (like `NoosArgs`) is a filled-out form for a particular order (the NOOS module's settings).

## How to Use Args Classes

Let's say the `NoosCoreComputeModule` needs its settings.

**1. Define the Args Class (`NoosArgs.java`):**

This class defines the expected parameters for NOOS calculations.

```java
// File: src/main/java/com/increff/irisx/args/NoosArgs.java
package com.increff.irisx.args;

import com.increff.iris.commons.module.AbstractArgs; // Base class
import java.util.Properties; // Input format

public class NoosArgs extends AbstractArgs { // Inherits from AbstractArgs

    // Define keys used in the properties file
    public static final String BESTSELLER_DURATION = "bestseller_duration";
    public static final String MIN_QTY_SOLD = "min_qty_sold";
    // ... other keys ...

    // Define public fields to hold the converted values
    public final int bestsellerDuration;
    public final int minQtySold;
    // ... other fields ...

    // Constructor takes the raw properties
    public NoosArgs(Properties props) {
        super(props); // Pass properties to the base class

        // Use AbstractArgs methods to get & convert values
        bestsellerDuration = getInteger(BESTSELLER_DURATION); // Reads "bestseller_duration", converts to int
        minQtySold = getInteger(MIN_QTY_SOLD);           // Reads "min_qty_sold", converts to int
        // ... fetch other values ...
    }
}
```

**Explanation:**

*   It inherits from `AbstractArgs`.
*   It defines constants (`BESTSELLER_DURATION`, `MIN_QTY_SOLD`) representing the names of the parameters expected in the configuration source (like a `.properties` file).
*   It declares `public final` fields (`bestsellerDuration`, `minQtySold`) to hold the values *after* they've been read and converted.
*   The constructor takes a `Properties` object (which holds the raw key-value settings).
*   It calls `super(props)` to initialize the base `AbstractArgs`.
*   It calls methods like `getInteger(KEY_NAME)` provided by `AbstractArgs`. These methods look up the `KEY_NAME` in the `props`, retrieve the string value, convert it to an integer, and return it. The result is stored in the corresponding field.

**2. Prepare Configuration (e.g., `algo_properties.tsv` or `Properties` object):**

Somewhere, the actual settings are defined. They might look like this in a properties format:

```properties
# Settings for NOOS module
bestseller_duration=180
min_qty_sold=140
top_x_rev_contri_percentage=50.0
# ... other NOOS settings ...

# Settings for other modules might also be present
relevant_weeks_for_od=20
# ...
```

**3. Instantiate and Use in a Module:**

Inside the `NoosCoreComputeModule` (or wherever these settings are needed), you would create an instance of `NoosArgs` and access the parameters:

```java
// Inside a module like NoosCoreComputeModule.java (simplified)
import com.increff.irisx.args.NoosArgs;
import java.util.Properties;

// ... inside a method where module logic runs ...

// Assume 'algoProps' is the Properties object containing all settings
Properties algoProps = // ... obtained from Module Runner or context ...

// Create the specific Args object for NOOS
NoosArgs noosArgs = new NoosArgs(algoProps);

// Now access the parameters easily and safely!
int duration = noosArgs.bestsellerDuration; // Gets the integer 180
int minQuantity = noosArgs.minQtySold;     // Gets the integer 140
// double topRev = noosArgs.topXRevContri; // Gets the double 50.0

System.out.println("NOOS Bestseller Duration: " + duration);
System.out.println("NOOS Min Quantity Sold: " + minQuantity);

// ... use these values in the NOOS calculation logic ...
```

**Explanation:**

*   The module gets the `Properties` object (`algoProps`) containing all the settings.
*   It creates a `NoosArgs` instance, passing `algoProps` to the constructor. The constructor does the hard work of looking up keys, converting types, and storing them in the `noosArgs` object's fields.
*   The module can then directly access `noosArgs.bestsellerDuration`, `noosArgs.minQtySold`, etc. The values are already the correct type (e.g., `int`) and ready to use.

**Expected Outcome:**

*   The `NoosArgs` object is created successfully.
*   The fields like `bestsellerDuration` and `minQtySold` hold the correctly typed values read from the `Properties`.
*   The module can proceed with its logic using these clear, accessible parameters.
*   If a required property was missing or had the wrong format (e.g., `min_qty_sold=abc`), the `NoosArgs` constructor would throw an `ArgsParseException`, stopping the process early with a clear error message.

## Under the Hood: `AbstractArgs`

The magic of reading, converting, and handling errors happens mostly in the base class, `AbstractArgs`.

**How it Works (Simplified):**

1.  **Store Properties:** When you call `super(props)` in your specific `Args` class constructor (like `NoosArgs`), the `AbstractArgs` constructor stores the `Properties` object internally.
2.  **Provide Getters:** `AbstractArgs` provides methods like `getInteger(key)`, `getDouble(key)`, `getBoolean(key)`, `getDate(key)`, `get(key)` (for strings).
3.  **Lookup:** When you call `getInteger("min_qty_sold")`, the `AbstractArgs` method looks up the key `"min_qty_sold"` in the stored `Properties`.
4.  **Conversion & Error Handling:**
    *   It retrieves the value associated with the key (e.g., the string `"140"`).
    *   It attempts to convert this string to the requested type (e.g., Integer).
    *   **If successful:** It returns the converted value (e.g., the integer `140`).
    *   **If the key is missing:** It usually throws an `ArgsParseException` indicating the missing parameter.
    *   **If the value cannot be converted** (e.g., trying to convert `"abc"` to an integer): It throws an `ArgsParseException` indicating the format error.

**Code Dive (`AbstractArgs.java` - Simplified):**

```java
// Simplified from iris-commons AbstractArgs
package com.increff.iris.commons.module;

import com.increff.iris.commons.ArgsParseException;
import java.util.Properties;
// ... other imports for date/time handling ...

public abstract class AbstractArgs {

    protected final Properties props; // Stores the raw properties

    public AbstractArgs(Properties props) {
        if (props == null) {
            // Handle case where no properties are provided
            throw new ArgsParseException("Properties object cannot be null");
        }
        this.props = props;
    }

    // Helper to get the raw string value, checking if it exists
    protected String getPropertyValue(String key) {
        String value = props.getProperty(key);
        if (value == null || value.trim().isEmpty()) {
            throw new ArgsParseException("Required property missing: " + key);
        }
        return value.trim();
    }

    // Method to get an Integer property
    public int getInteger(String key) {
        String value = getPropertyValue(key); // Get raw string, throws if missing
        try {
            return Integer.parseInt(value); // Attempt conversion
        } catch (NumberFormatException e) {
            // Throw specific error if conversion fails
            throw new ArgsParseException("Invalid integer format for key '" + key + "': " + value, e);
        }
    }

    // Method to get a Double property (similar logic)
    public double getDouble(String key) {
        String value = getPropertyValue(key);
        try {
            return Double.parseDouble(value);
        } catch (NumberFormatException e) {
            throw new ArgsParseException("Invalid double format for key '" + key + "': " + value, e);
        }
    }

    // Method to get a String property (simpler, just checks existence)
    public String get(String key) {
        return getPropertyValue(key);
    }

    // Method to get a Boolean property (handles "true"/"false")
    public boolean getBoolean(String key) {
        String value = getPropertyValue(key);
        // Note: Boolean.parseBoolean is lenient, returns false for non-"true" strings
        // A stricter check might be needed depending on requirements.
        return Boolean.parseBoolean(value);
    }

    // Method to get a LocalDate property (example, actual parsing might differ)
    public java.time.LocalDate getDate(String key) {
        String value = getPropertyValue(key);
        try {
            // Assumes standard ISO date format YYYY-MM-DD
            return java.time.LocalDate.parse(value);
        } catch (java.time.format.DateTimeParseException e) {
            throw new ArgsParseException("Invalid date format for key '" + key + "': " + value + ". Expected YYYY-MM-DD.", e);
        }
    }
}
```

**Explanation:**

*   The constructor stores the passed `Properties`.
*   `getPropertyValue(key)` is a crucial helper. It fetches the raw string value for a `key` and throws a clear `ArgsParseException` if the key is missing or the value is empty.
*   Methods like `getInteger`, `getDouble`, `getDate`, etc., first call `getPropertyValue` to ensure the property exists.
*   They then use standard Java methods (`Integer.parseInt`, `Double.parseDouble`, `LocalDate.parse`) to attempt the conversion.
*   If the conversion fails (e.g., `NumberFormatException`), they catch the exception and throw a new `ArgsParseException` with a detailed message about which key failed and why.

This pattern centralizes the logic for reading, converting, and validating parameters within `AbstractArgs` and the specific `Args` class constructors, keeping the main module code clean and focused on its core task.

## Conclusion

You've now learned about **Configuration & Arguments (Args Classes)** in `irisx-algo`. These classes provide a structured, reliable, and easy way for algorithm modules to access their required settings.

*   They act like **typed settings sheets** for modules.
*   Each major module often has its own `Args` class (e.g., `NoosArgs`, `DistributionArgs`) inheriting from `AbstractArgs`.
*   The `Args` class constructor reads parameters from a source (like `Properties`), performs **type conversions** (String to int, double, date, etc.), and handles **missing or invalid settings** by throwing `ArgsParseException`.
*   Modules instantiate their specific `Args` class and access parameters through simple, type-safe fields.

This keeps module code cleaner and makes configuration management much more robust.

With the settings ready via `Args` classes, how does the system actually find and run the specific code for a module like `NoosCoreComputeModule`? The next chapter dives into the [Module Runner](04_module_runner_.md), the component responsible for executing the right module based on its name.

[Next Chapter: Module Runner](04_module_runner_.md)

---

Generated by [AI Codebase Knowledge Builder](https://github.com/The-Pocket/Tutorial-Codebase-Knowledge)