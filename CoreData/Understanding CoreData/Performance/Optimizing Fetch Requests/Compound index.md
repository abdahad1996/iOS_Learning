### **Understanding Compound Indexes in Core Data**

Compound indexes in Core Data allow SQLite to optimize queries and sorting based on a combination of two or more attributes. These indexes are especially useful for complex queries or sorting operations that involve multiple attributes.

---
<img width="561" alt="Screenshot 2024-12-16 at 5 03 03 PM" src="https://github.com/user-attachments/assets/cecfacce-1bcd-4b38-a85c-89efb1385764" />

### **Why Use Compound Indexes?**

1. **Efficiency for Multi-Attribute Queries**:
   - A compound index speeds up searches and sorting for queries involving a specific combination of attributes.
   - Example: Fetching or sorting cities by whether they are capitals (`isCapital`) and then by population (`population`).

2. **Reduction in Query Time**:
   - Instead of scanning rows for each attribute, SQLite uses the compound index to perform efficient lookups and sorting.

3. **Better Performance for Specific Use Cases**:
   - Compound indexes are optimized for queries involving attributes in the exact order defined in the index.

---

### **Tradeoffs of Compound Indexes**

1. **Increased Database Size**:
   - Compound indexes add to the size of the database because SQLite creates an additional data structure.

2. **Higher Write Costs**:
   - Insertions, updates, and deletions become more expensive because the database must update the compound index along with the data.

3. **Index Usage Rules**:
   - Compound indexes are only used for queries that align with their attribute order.
   - Example: An index on `isCapital,population` works for sorting by both attributes together, or just by `isCapital`. It does **not** help with sorting by `population` alone.

4. **Avoid Redundant Indexes**:
   - Having both a compound index (e.g., `isCapital,population`) and a single index on `isCapital` is unnecessary because SQLite can only use one index at a time. This redundancy increases database size and write costs without improving query performance.

---

### **How to Create Compound Indexes in Core Data**

#### **Scenario**
You have a `City` entity with the following attributes:
- **name**: The name of the city.
- **isCapital**: Boolean (1 if the city is a capital, 0 otherwise).
- **population**: Integer representing the city’s population.

You want to:
- Display all cities with capitals first, followed by non-capitals.
- Within each group, cities should be sorted by population.

#### **Steps to Create the Compound Index**

1. Open the Core Data Model Editor.
2. Select the **City** entity.
3. Go to the **Indexes** section in the Data Model Inspector.
4. Add a new compound index and include the attributes `isCapital` and `population` in that order.

---

### **Using the Compound Index in a Query**

#### **Fetch Request Example**
To fetch and sort cities with capitals first and by population within each group:

```swift
let fetchRequest: NSFetchRequest<City> = City.fetchRequest()
fetchRequest.sortDescriptors = [
    NSSortDescriptor(key: "isCapital", ascending: false), // Capitals first
    NSSortDescriptor(key: "population", ascending: false) // Sort by population within each group
]

do {
    let cities = try context.fetch(fetchRequest)
    for city in cities {
        print("\(city.name): Capital - \(city.isCapital), Population - \(city.population)")
    }
} catch {
    print("Failed to fetch cities: \(error)")
}
```

#### **Performance Impact**
- The compound index (`isCapital,population`) ensures that the query performs efficiently.
- SQLite leverages the index to sort by `isCapital` first and then by `population`, without scanning the entire table.

---

### **Key Considerations**

1. **Order of Attributes Matters**:
   - A compound index works only if the query matches the order of attributes in the index.
   - Example: An index on `isCapital,population` supports:
     - Sorting by `isCapital` alone.
     - Sorting by `isCapital` and then `population`.
   - It does **not** support sorting by `population` alone.

2. **Avoid Redundant Indexes**:
   - If a compound index exists (e.g., `isCapital,population`), avoid creating single-attribute indexes for either `isCapital` or `population` unless they are frequently used in isolation.

3. **Measure Performance**:
   - Use profiling tools to determine if the compound index improves query performance for your specific use case.

---

### **Advanced Profiling with SQLite**

To ensure that SQLite is using the compound index, use the **EXPLAIN QUERY PLAN** command:

1. Export the SQLite database from your app.
2. Open the database using a tool like `sqlite3`.
3. Run your query with `EXPLAIN QUERY PLAN` to check if the compound index is being used:
   ```sql
   EXPLAIN QUERY PLAN
   SELECT * FROM City
   WHERE isCapital = 1
   ORDER BY isCapital DESC, population DESC;
   ```

If the output indicates that the query uses the compound index, the optimization is working as intended.

---

### **Example Summary**

#### **Without Indexes**
- Query: Fetch cities, sorted by capitals first and population second.
- SQLite scans the entire table, performing multiple comparisons for each row.
- Performance degrades as the dataset grows.

#### **With a Compound Index**
- Query: Same as above.
- SQLite uses the `isCapital,population` index, efficiently retrieving and sorting results.
- Query performance remains consistent, even for large datasets.

---

### **Best Practices for Compound Indexes**

1. **Use Compound Indexes for Common Query Patterns**:
   - Identify frequently used queries that involve sorting or filtering by multiple attributes in a specific order.

2. **Avoid Redundancy**:
   - Do not create single-attribute indexes for attributes already covered by a compound index unless they are frequently queried independently.

3. **Balance Read and Write Performance**:
   - Consider the tradeoff between faster reads (lookups and sorting) and slower writes (inserts and updates).

4. **Profile and Test**:
   - Use profiling tools and real-world data to measure the impact of compound indexes.
   - Remove unnecessary indexes to reduce write overhead and database size.

---

### **Conclusion**

Compound indexes are a powerful way to optimize multi-attribute queries in Core Data. They improve query and sorting performance for large datasets but come with tradeoffs in terms of database size and write costs. By carefully designing your indexes based on usage patterns and profiling their impact, you can achieve significant performance gains in your Core Data application.
