### **Storing, Searching, and Sorting Text in Core Data**

Storing text in Core Data is straightforward, but searching and sorting text strings can be complex due to the intricacies of Unicode, different languages, and locale-specific rules. This guide explores these challenges and provides efficient solutions.

---

### **Complexity of Unicode**

Text strings are more than their byte representation, especially when dealing with:
1. **Diacritics and Accents**:
   - Example: The letter "É" can be represented as a single code point (`U+00C9`) or as a combination (`E` + combining acute accent, `U+0301 U+0045`).
   - Users expect `Saint-Étienne`, `saint-etienne`, and `Saint Etienne` to match.

2. **Locale-Specific Variations**:
   - In Danish, "Aarhus" is equivalent to "Århus".
   - Sorting may vary: German treats "ö" as "oe" or as its own letter, while Swedish places "ö" after "z".

3. **Mixed Scripts**:
   - Should the Cyrillic "Москва" match "Moscow"? Should it sort before or after Latin names?

---

### **Challenges in Searching**

#### **Case-Insensitive and Diacritic-Insensitive Searches**
Using Core Data predicates:
```swift
let predicate = NSPredicate(format: "%K BEGINSWITH[cd] %@", #keyPath(City.name), searchTerm)
```
- **`[cd]` Modifiers**:
  - **`c`**: Case-insensitive.
  - **`d`**: Diacritic-insensitive.
- Example: `saint-etienne` matches `Saint-Étienne`.

However, this approach can be inefficient for large datasets because:
- SQLite doesn’t support case- and diacritic-insensitive searches natively.
- Core Data has to fetch all rows from the file system and perform comparisons in memory.

---

### **String Normalization for Efficient Searching**

To improve search performance, normalize text during storage and search operations.

#### **Normalization Process**
Normalization involves:
1. **Lowercasing** text.
2. **Stripping diacritics**.
3. Optionally, **removing non-letter characters**.

---

#### **Data Model Adjustment**
Add a normalized attribute (`name_normalized`) to the `City` entity alongside the actual `name`.

#### **Normalization Logic**
Update the normalized attribute whenever the `name` changes:
```swift
final public class City: NSManagedObject, Managed {
    @NSManaged fileprivate var primitiveName: String
    public var name: String {
        set {
            willChangeValue(forKey: #keyPath(City.name))
            primitiveName = newValue
            updateNormalizedName(newValue)
            didChangeValue(forKey: #keyPath(City.name))
        }
        get {
            willAccessValue(forKey: #keyPath(City.name))
            let value = primitiveName
            didAccessValue(forKey: #keyPath(City.name))
            return value
        }
    }
    
    fileprivate func updateNormalizedName(_ name: String) {
        setValue(name.normalizedForSearch, forKey: "name_normalized")
    }
}
```

#### **String Normalization Helper**
Normalize text using `StringTransform`:
```swift
extension String {
    public var normalizedForSearch: String {
        let transformed = applyingTransform("Any-Latin; Latin-ASCII; Lower", reverse: false)
        return transformed ?? ""
    }
}
```

This transformation:
- Converts non-Latin scripts (e.g., Cyrillic) to Latin equivalents.
- Removes diacritics (e.g., `É` becomes `E`).
- Lowercases the text.

---

#### **Efficient Search with Normalized Strings**
Use the normalized attribute in fetch requests:
```swift
let predicate = NSPredicate(format: "%K BEGINSWITH[n] %@", #keyPath(City.name_normalized), searchTerm.normalizedForSearch)
```
- **`[n]` Modifier**: Tells Core Data that the comparison is already normalized.
- Example: Searching for `Béziers` becomes:
  ```sql
  WHERE name_normalized BEGINSWITH "beziers"
  ```
- SQLite performs the search directly, benefiting from indexes on the normalized attribute.

---

### **Challenges in Sorting**

Sorting text involves locale-specific collation rules. For example:
1. German treats `ö` as `oe`, while Swedish places `ö` after `z`.
2. Mixed scripts (e.g., Cyrillic and Latin) require app-specific sorting rules.

#### **Naive Sorting**
Using an `NSSortDescriptor` in a fetch request:
```swift
let sd = NSSortDescriptor(key: #keyPath(City.name), ascending: true)
request.sortDescriptors = [sd]
```
- This sorts by the byte values of the text, which may not align with user expectations.

---

#### **Localized Sorting in Memory**
For small datasets, sort in memory using `localizedStandardCompare`:
```swift
cities.sort { $0.name.localizedStandardCompare($1.name) == .orderedAscending }
```
- Considers locale-specific rules but is expensive for large datasets.

---

### **Optimizing Sorting**

#### **Pre-Sorted Arrays**
For frequently accessed data, maintain a sorted array in memory:
1. **Detect Changes**:
   - Listen to `.NSManagedObjectContextObjectsDidChange` notifications.
   - Check if any `City` objects were inserted or had their `name` updated.

2. **Update Sorted Array**:
   - Use binary search for inserting new items efficiently:
     ```swift
     let index = sortedArray.index(of: city, inSortedRange: range, options: .insertionIndex, usingComparator: comparator)
     ```

3. **Cache Sorted Array**:
   - Keep the sorted array updated in memory for quick access.

---

#### **Persisting Sorted Order**
To persist sorted order:
1. Use an **ordered to-many relationship** in Core Data:
   - Create a `SortedCityOwner` entity with an ordered relationship to `City`.
   - Maintain the relationship when cities are inserted or updated.

2. **Locale and Framework Changes**:
   - Persist the sorted array only if the locale and system framework remain unchanged:
     ```swift
     if Locale.currentCollatorIdentifier == persistedCollatorIdentifier &&
        NSFoundationVersionNumber == persistedFoundationVersion {
        // Use persisted order
     } else {
        // Re-sort
     }
     ```

---

### **Takeaways**

#### **For Searching:**
1. Normalize text to improve search performance in large datasets.
2. Use normalized attributes and the `[n]` modifier to leverage SQLite indexes.
3. Match locale and Unicode intricacies with appropriate transforms.

#### **For Sorting:**
1. Sort small datasets in memory using `localizedStandardCompare`.
2. For large datasets, maintain a pre-sorted array in memory.
3. Consider using ordered relationships for persistent sorted order.

---

### **Conclusion**

By combining normalization for efficient searches and localized sorting strategies, you can handle text-related complexities in Core Data while balancing performance and user experience. Always profile and test your implementation, especially when dealing with large datasets or multilingual content.
