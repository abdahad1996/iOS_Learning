### **Query Generations in Core Data**

Query generations, introduced in iOS 10/macOS 10.12, address the issue of **data inconsistency** in scenarios where multiple managed object contexts (MOCs) interact with the same persistent store. By using query generations, you can ensure that all read operations (fetch requests and fault fulfillments) within a managed object context are based on a consistent snapshot of the database.

---

### **Why Query Generations Matter**

When multiple contexts operate on the same Core Data store, conflicts can occur during **read and write operations**. For example:

1. **Scenario Without Query Generations**:
   - A context fetches data (e.g., fetch request).
   - Before it triggers a fault for related data, another context saves changes to the store.
   - The fetched data may now be **inconsistent** with the data fetched by the fault, as the store’s state has changed in the meantime.

2. **Scenario With Query Generations**:
   - A context is pinned to a **specific generation** of the store.
   - All read operations (fetches and faults) within the context are guaranteed to reflect the **same consistent snapshot** of the database, regardless of changes made by other contexts.

---

### **How Query Generations Work**

1. **Pinning a Context to a Query Generation**:
   - You pin a context to the current state of the persistent store using:
     ```swift
     try! moc.setQueryGenerationFrom(NSQueryGenerationToken.current)
     ```
   - Once pinned, all fetches and faults fulfilled by the context are based on the same snapshot of the database.

2. **Effects of Pinning**:
   - Changes made by other contexts are **invisible** to the pinned context until:
     - The pinned context explicitly merges changes from the `context-did-save` notification.
     - The pinned context itself is saved, which updates its query generation to the current state.

3. **Updating Query Generations**:
   - When a pinned context merges a `context-did-save` notification, it updates to the latest query generation, making new changes visible.
   - Similarly, when a context saves its changes, it synchronizes with the current store state.

---

### **Example: Using Query Generations**

#### **1. Basic Setup**

Imagine an app where two contexts (`mainContext` and `backgroundContext`) operate on the same store. We want to ensure that `mainContext` remains consistent during a fetch and fault operation, even if `backgroundContext` saves changes during that time.

```swift
import CoreData

// Persistent container setup
let container = NSPersistentContainer(name: "Model")
container.loadPersistentStores { _, error in
    if let error = error {
        print("Error loading persistent stores: \(error)")
    }
}

// Main and background contexts
let mainContext = container.viewContext
let backgroundContext = container.newBackgroundContext()

// Pin the main context to the current query generation
do {
    try mainContext.setQueryGenerationFrom(NSQueryGenerationToken.current)
} catch {
    print("Error setting query generation: \(error)")
}
```

---

#### **2. Fetch and Fault Operations with Query Generations**

```swift
mainContext.perform {
    // Fetch objects from the store
    let fetchRequest: NSFetchRequest<MyEntity> = MyEntity.fetchRequest()
    if let fetchedObjects = try? mainContext.fetch(fetchRequest) {
        print("Fetched objects: \(fetchedObjects)")
    }

    // Trigger a fault on a fetched object
    if let firstObject = fetchedObjects?.first {
        print("Triggered fault for first object: \(firstObject.someAttribute)")
    }
}
```

**Behavior**:
- The context will use the same query generation for both the fetch request and the fault trigger, ensuring consistent data.

---

#### **3. Handling `context-did-save` Notifications**

To keep the context updated with changes made by other contexts, merge `context-did-save` notifications:

```swift
NotificationCenter.default.addObserver(
    forName: .NSManagedObjectContextDidSave,
    object: backgroundContext,
    queue: nil
) { notification in
    mainContext.perform {
        mainContext.mergeChanges(fromContextDidSave: notification)
    }
}
```

When merging, the query generation of `mainContext` will update to reflect the latest state of the database.

---

### **Key Considerations**

1. **Consistency in a Pinned Context**:
   - While pinned, the context operates as if the store is **unchanged**, even if another context modifies the store.

2. **Updating Query Generations**:
   - A context updates its query generation when:
     - It merges a `context-did-save` notification.
     - It saves changes, ensuring consistency between its in-memory data and the store.

3. **Temporary Window of Inconsistency**:
   - After a context saves but before another context merges the changes, there’s a brief window where the second context may read outdated data. This is particularly important for handling deletions.

---

### **Handling Special Cases**

#### **Deletions and Query Generations**

When an object is deleted in one context, the following might occur in a pinned context:
1. The pinned context still sees the object because it’s pinned to an earlier query generation.
2. Attempting to access the deleted object after merging changes may cause a fault or crash.

**Solution**:
- Use `refresh(_:mergeChanges:)` to refresh objects in the pinned context after merging changes:
  ```swift
  mainContext.refresh(deletedObject, mergeChanges: true)
  ```

---

### **When to Use Query Generations**

1. **Read Consistency**:
   - Use query generations when you need consistent data during multiple read operations, such as in reporting or analytics.

2. **Isolation of Changes**:
   - Use in scenarios where one context must remain unaffected by changes made in other contexts until explicitly updated.

---

### **Key Takeaways**

1. **Query Generations Ensure Consistency**:
   - They guarantee that all fetches and faults within a context reflect the same state of the database.

2. **Pinning Contexts**:
   - Pinning a context to a query generation isolates it from changes made by other contexts until the context updates itself.

3. **Updating Query Generations**:
   - Query generations automatically update when:
     - A context merges a `context-did-save` notification.
     - A context saves changes to the store.

4. **Deletions Require Special Handling**:
   - Always refresh objects in pinned contexts after merging changes to handle deletions gracefully.

---

### **Practical Example**

Let’s consider a **task management app** with two contexts:
- `mainContext`: Displays tasks in the UI.
- `backgroundContext`: Handles syncing tasks with a remote server.

```swift
// Pin the main context
try! mainContext.setQueryGenerationFrom(NSQueryGenerationToken.current)

// Background context saves new data
backgroundContext.perform {
    let newTask = Task(context: backgroundContext)
    newTask.title = "New Task"
    try? backgroundContext.save()
}

// Main context reads data without seeing changes yet
mainContext.perform {
    let fetchRequest: NSFetchRequest<Task> = Task.fetchRequest()
    if let tasks = try? mainContext.fetch(fetchRequest) {
        print("Tasks before merging: \(tasks.count)") // Will not include "New Task"
    }
}

// Merge changes to update the main context
NotificationCenter.default.addObserver(
    forName: .NSManagedObjectContextDidSave,
    object: backgroundContext,
    queue: nil
) { notification in
    mainContext.perform {
        mainContext.mergeChanges(fromContextDidSave: notification)

        // Read data after merging
        let fetchRequest: NSFetchRequest<Task> = Task.fetchRequest()
        if let tasks = try? mainContext.fetch(fetchRequest) {
            print("Tasks after merging: \(tasks.count)") // Includes "New Task"
        }
    }
}
```

**Outcome**:
- The `mainContext` initially sees a consistent snapshot of the store.
- After merging changes, it updates to reflect the new state.
