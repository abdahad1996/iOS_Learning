### **Summary: Core Data Data Types and Custom Data Storage**

In this chapter, we explored how Core Data handles **default data types** and supports storing **custom data types** through transformable attributes, custom value transformers, or custom accessors. Below is a breakdown of key points and takeaways.

---

### **Key Concepts**

1. **Core Data’s Default Data Types**:
   - Core Data supports a wide range of basic types such as integers, floats, strings, dates, booleans, and binary data.
   - Choose the simplest type that meets your use case to avoid wasting storage or computational resources.

2. **Storing Custom Data Types**:
   - Since Core Data can only persist its supported data types, custom types must be converted to one of these:
     - **NSCoding**: Simplifies storage of custom types by encoding them into a transformable attribute.
     - **Custom Value Transformer**: Allows more efficient encoding and decoding, typically for performance or storage optimization.
     - **Custom Accessors**: Enables lazy transformations or custom logic for property access.

3. **Performance Considerations**:
   - Avoid premature optimization; measure performance before implementing complex solutions.
   - Only opt for custom value transformers or accessors if storage size or performance is a genuine concern.

---

### **Takeaways**

#### **1. Default Data Types**
- Use the basic types Core Data supports natively, such as `Int16`, `String`, or `Date`.
- Optimize for storage and retrieval efficiency by picking the appropriate type (e.g., use `Int16` for small integers instead of `Int64`).

#### **2. Binary Data**
- When storing binary data (e.g., images), always enable the **"Allow External Storage"** option in the model editor.
- This lets Core Data store large binary data outside the main SQLite database, improving performance.

#### **3. Persisting Custom Data Types**
- **NSCoding**:
  - Simplifies storage by encoding objects into a property list.
  - Suitable for most cases unless storage size or performance is a concern.
  - Example: Use `NSCoding` for an array of colors (`[UIColor]`) in a transformable attribute.

- **Custom Value Transformer**:
  - Provides more control over how data is stored (e.g., storing colors as compact binary blobs).
  - Example: Serialize `UIColor` arrays into RGB bytes to save space.

- **Custom Accessors**:
  - Ideal for lazy transformations or advanced property handling.
  - Example: Convert binary data to a user-facing format (e.g., `[UIColor]`) only when accessed.

#### **4. Change Tracking with Custom Accessors**
- Always wrap access to primitive properties with Core Data lifecycle methods:
  - `willAccessValue(forKey:)` and `didAccessValue(forKey:)` for reads.
  - `willChangeValue(forKey:)` and `didChangeValue(forKey:)` for writes.
- This ensures Core Data tracks changes correctly and avoids inconsistencies.

#### **5. Non-Optional Attributes**
- By default, mark attributes as non-optional unless there’s a valid reason to allow `nil`.
- Non-optional numeric attributes, in particular, help maintain data integrity and simplify code.

#### **6. Default Values**
- Specify static default values in the model editor for attributes that should have initial values (e.g., a boolean defaulting to `true`).
- Set dynamic default values (e.g., the current date) by overriding `awakeFromInsert()` in the `NSManagedObject` subclass.

---

### **Best Practices Summary**

| **Feature**                  | **Best Practice**                                                                                     | **Example**                                                                                 |
|-------------------------------|-------------------------------------------------------------------------------------------------------|---------------------------------------------------------------------------------------------|
| **Default Data Types**        | Choose the simplest and most appropriate type for your data.                                          | Use `Int16` for small numbers; `NSDecimalNumber` for currency.                             |
| **Binary Data**               | Enable **Allow External Storage** for large binary data.                                              | Store images or audio files efficiently with Core Data managing storage location.          |
| **Custom Data Types**         | Use `NSCoding` for simple cases or custom transformers for optimized storage.                         | Serialize `[UIColor]` into RGB bytes using a custom transformer.                           |
| **Custom Accessors**          | Implement lazy transformations and wrap access with lifecycle methods.                               | Use `willAccessValue(forKey:)` and `didAccessValue(forKey:)` in custom getters.            |
| **Non-Optional Attributes**   | Make attributes non-optional by default; use optional only when necessary.                           | Mark `date` as non-optional and initialize with a default value in `awakeFromInsert()`.     |
| **Default Values**            | Set static defaults in the model editor or dynamic defaults in `awakeFromInsert()`.                   | Initialize `isPositive` to `true` or set `date` to the current time dynamically.            |

---

### **Conclusion**

Core Data provides flexibility for managing both default and custom data types:
- Leverage default data types where possible for simplicity and efficiency.
- Use custom transformers or accessors only when optimization is required.
- Apply best practices like enabling external storage for binary data, enforcing non-optional attributes, and setting appropriate default values.

By following these principles, you can design robust, efficient, and maintainable Core Data models.
