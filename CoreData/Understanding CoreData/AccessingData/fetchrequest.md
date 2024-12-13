### **Fetch Requests in Core Data: Detailed Explanation**

Fetch requests are a core feature of Core Data, used to retrieve objects from the persistent store. This process involves multiple layers of Core Data's architecture, including the managed object context, persistent store coordinator, and SQLite store. Here’s a detailed step-by-step explanation with key concepts like **faulting** and **uniquing**.

---

### **Basic Fetch Request Example**

```swift
let request = NSFetchRequest<Mood>(entityName: "Mood")
let moods = try! context.fetch(request)
```

1. **Create the Fetch Request**:
   - `NSFetchRequest<Mood>` specifies the entity type (`Mood`) you want to fetch.
   - The entity name is defined in the Core Data model.

2. **Execute the Fetch Request**:
   - `context.fetch(request)` executes the request and returns an array of managed objects.

---

### **How a Fetch Request Works**

1. **Managed Object Context Forwards the Request**:
   - The context forwards the fetch request to the **persistent store coordinator** via the `execute(_ request :with context:)` method.
   - The context passes itself as an argument for later use.

2. **Persistent Store Coordinator Distributes the Request**:
   - The persistentstorecoordinator forwards the requestto all of its persistent stores (in case you have multiple) by calling each store’s execute(_ request:with context:) method. Again: the context from which the fetch request originated is passed to the store.

3. **Persistent Store Processes the Request**:
   - The persistent store translates the fetch request into an **SQL query** and executes it on the SQLite database.

4. **SQLite Returns Raw Data**:
   - SQLite executes the query and returns:
     - **Object IDs**: Unique identifiers for each row (combination of store ID, table ID, and primary key).
     - **Raw Data**: Simple data types like numbers, strings, and binary blobs.

5. **Row Cache**:
   - The raw data is stored in the persistent store’s **row cache**, tied to the object IDs. 
   - A row cache entry with a certain object ID lives as long as a managed object with this object ID exists, whether it’s a fault or not.

6. **Instantiate Managed Objects**:
   - The persistent store creates managed objects for the object IDs it retrieves from the SQLite store and returns them to the coordinator. These managed objects are tied to a specific context, so the store must be able to call object(with:) on the originating context.

By default, fetch requests return managed objects, which are initially faults. Faults are lightweight placeholder objects that don’t contain actual data yet—they retrieve data only when it’s needed (more on faulting later).

If a managed object with the same object ID already exists in the context, Core Data uses the existing object without creating a new one. This behavior is called uniquing. It ensures that, within a single managed object context, there is only one instance of an object representing a specific piece of data. As a result, objects representing the same data within the same context are guaranteed to be identical, even when compared using pointer equality.

   - If the object with the same ID already exists in the context, the existing object is reused (**uniquing**).
6. The persistentstore coordinator returns to the context the array of managed objects it got from the persistent store.

7. **Context Updates Pending Changes**:
   - Before returning the results, the context considers any **pending changes** (inserts, updates, or deletions) made within the context that haven’t been saved yet.
   - The fetch results are updated to reflect these changes.

8. **Return Managed Objects**:
   - Finally, an array of managed objects is returned to the caller.

<img width="553" alt="Screenshot 2024-12-13 at 9 14 39 PM" src="https://github.com/user-attachments/assets/8acd0d6d-437c-4ecb-b929-6b909bdfabf0" />
---

### **Faulting and Uniquing**

#### **1. Faulting**
- **What It Is**:
  - Faulting creates lightweight, placeholder objects instead of fully loading data into memory.
  - Data is only fetched from the persistent store when you access the faulted object’s properties.

- **Advantages**:
  - Reduces memory usage.
  - Improves performance by only loading data when needed.

- **Example**:
  ```swift
  let mood = moods.first!
  print(mood.date) // Accessing a property triggers faulting.
  ```

#### **2. Uniquing**
- **What It Is**:
  - Core Data ensures there’s only one managed object instance for a given object ID in a managed object context.
  - If you fetch the same object multiple times, Core Data returns the same instance.

- **Advantages**:
  - Ensures data consistency.
  - Prevents duplicate objects in memory.

- **Example**:
  ```swift
  let firstFetch = try! context.fetch(request).first!
  let secondFetch = try! context.fetch(request).first!
  print(firstFetch === secondFetch) // true, due to uniquing.
  ```

---

### **Includes Pending Changes**

- By default, the fetch request’s `includesPendingChanges` property is `true`.
- This means the context includes unsaved changes (e.g., newly inserted or updated objects) in the fetch results.

- **Example**:
  ```swift
  let newMood = Mood(context: context)
  newMood.date = Date()
  
  let request = NSFetchRequest<Mood>(entityName: "Mood")
  let results = try! context.fetch(request)
  print(results.contains(newMood)) // true, due to includesPendingChanges.
  ```

---

### **Synchronous Execution**

- Fetch requests are **synchronous**.
- The managed object context is blocked until the fetch completes.
- Prior to iOS 10/macOS 10.12, the **persistent store coordinator** was also blocked during fetch requests.

---

### **Visualizing the Fetch Request Workflow**

**Fetch Request Lifecycle**:
1. **Application Layer**:
   - Context executes the fetch request.
2. **Core Data Layer**:
   - Coordinator forwards the request to persistent stores.
   - Persistent store converts request to SQL.
3. **Database Layer**:
   - SQLite executes the query and returns raw data.
4. **Core Data Layer**:
   - Managed objects are instantiated or reused (uniquing).
5. **Application Layer**:
   - Context updates results with pending changes and returns them.

**Diagram**:
```
Application Layer
  ↓
Context → Coordinator → Persistent Store → SQLite
  ↑                  ← Managed Objects ← Raw Data
```

---
<img width="553" alt="Screenshot 2024-12-13 at 9 14 39 PM" src="https://github.com/user-attachments/assets/8acd0d6d-437c-4ecb-b929-6b909bdfabf0" />

### **Important Options for Fetch Requests**

1. **`includesPropertyValues`**:
   - Default: `true`.
   - Determines whether property values are loaded or only object IDs are returned.
   - Set to `false` for lightweight fetching.

2. **`includesPendingChanges`**:
   - Default: `true`.
   - Includes unsaved changes in the fetch results.

3. **`fetchBatchSize`**:
   - Specifies the number of objects to fetch at a time.
   - Useful for managing memory with large datasets.

4. **`returnsObjectsAsFaults`**:
   - Default: `true`.
   - Determines whether fetched objects are faults or fully populated.

---


### **Key Takeaways**

1. **Faulting**:
   - Core Data optimizes memory usage by returning faults instead of fully populated objects.
   - Data is fetched lazily when accessed.

2. **Uniquing**:
   - Ensures only one managed object instance exists for a given object ID within a context.

3. **Includes Pending Changes**:
   - Fetch requests include unsaved changes by default.

4. **Performance**:
   - Fetch requests are synchronous, so consider optimizing large dataset queries with batching or background contexts.

5. **Customizing Fetch Requests**:
   - Adjust fetch options (e.g., `includesPropertyValues`, `fetchBatchSize`) to balance memory usage and performance.

By understanding the internals of fetch requests and leveraging options like faulting and uniquing, you can write efficient and robust Core Data queries.
