<img width="578" alt="Screenshot 2024-12-16 at 6 31 34 PM" src="https://github.com/user-attachments/assets/cef3b254-3143-4121-88f0-be1c0a90f426" />



<img width="584" alt="Screenshot 2024-12-16 at 6 32 15 PM" src="https://github.com/user-attachments/assets/0cdc154d-e088-46f8-9a47-4d72e8180583" />

### **Passing Objects Between Contexts in Core Data**

When working with multiple Core Data contexts, you must pass objects between them safely and correctly. Directly using `NSManagedObject` instances from one context in another is unsafe because objects are bound to their original context's queue. Instead, you should rely on the **object ID** of the managed object to transfer it between contexts. Let’s explore this process with examples.

---

### **1. Passing Object IDs Between Contexts**

**Key Rule**: Pass the `NSManagedObjectID` from one context to another and re-instantiate the object in the target context using the `object(with:)` method.

#### **Example 1: Using `object(with:)`**
```swift
func finishedBackgroundOperation(_ objects: [NSManagedObject]) {
    let ids = objects.map { $0.objectID } // Extract object IDs
    mainContext.perform {
        let results = ids.map { mainContext.object(with: $0) }
        // Use the results safely on the main queue
        print(results)
    }
}
```

**How It Works**:
1. Extract `NSManagedObjectID` for each object in the background context.
2. Pass the IDs to the main context via its queue using `perform`.
3. Use `mainContext.object(with:)` to fetch objects from their IDs.
   - This ensures that the objects are now safe to access and modify on the main context.

---

### **2. Optimizing Performance: Keeping Row Cache Entries Alive**

When passing managed objects between contexts, you can optimize performance by ensuring that the **row cache** remains alive. The row cache keeps fetched data in memory, reducing the need to query SQLite again.

#### **Technique**:
Pass the `NSManagedObject` instances directly to the target context **only to extract their IDs**. Do not use the objects in any other way.

#### **Example 2: Extracting Object IDs in the Target Context**
```swift
func finishedBackgroundOperation(_ objects: [NSManagedObject]) {
    mainContext.perform {
        let results = objects.map { mainContext.object(with: $0.objectID) }
        // Results can now be used safely in the main context
        print(results)
    }
}
```

**How It Works**:
1. Pass the objects directly to the target context.
2. Use the `objectID` property of each object to re-instantiate them in the target context.

> **Important**: Do not directly interact with the passed objects in the target context. Only use their `objectID`.

---

### **3. Passing Objects Between Contexts with Different Persistent Store Coordinators**

When contexts belong to **different persistent store coordinators (PSCs)**, `NSManagedObjectID` from one PSC cannot directly be used in another. Instead:
1. Extract the **URI representation** of the object ID.
2. Use `persistentStoreCoordinator.managedObjectID(forURIRepresentation:)` to create a valid `NSManagedObjectID` for the target PSC.
3. Use `object(with:)` to re-instantiate the object in the target context.

#### **Example 3: Using URI Representation for Different PSCs**
```swift
func finishedBackgroundOperation(_ objects: [NSManagedObject]) {
    let ids = objects.map { $0.objectID } // Extract object IDs

    separatePSCContext.perform {
        let results = ids.compactMap { sourceID -> NSManagedObject? in
            let uri = sourceID.uriRepresentation()
            let psc = separatePSCContext.persistentStoreCoordinator!
            guard let targetID = psc.managedObjectID(forURIRepresentation: uri) else { return nil }
            return separatePSCContext.object(with: targetID)
        }
        // Results are now usable in the separate PSC context
        print(results)
    }
}
```

**How It Works**:
1. Extract `URIRepresentation` from the object IDs in the source context.
2. Convert the URI to a valid `NSManagedObjectID` in the target PSC.
3. Use `object(with:)` to fetch the object in the target PSC.

> **Note**: This approach is necessary only when contexts are connected to different PSCs.

---

### **4. Practical Scenarios**

#### **Scenario 1: Background Search Operation**
Imagine you perform a search in a background context and need to update the UI with the results in the main context.

**Solution**:
- Perform the search in the background context.
- Pass the `objectID`s of the results to the main context.
- Fetch the objects using `object(with:)` in the main context.

**Code Example**:
```swift
func performSearchInBackground(query: String) {
    let backgroundContext = NSPersistentContainer(name: "ModelName").newBackgroundContext()

    backgroundContext.perform {
        let fetchRequest: NSFetchRequest<MyEntity> = MyEntity.fetchRequest()
        fetchRequest.predicate = NSPredicate(format: "name CONTAINS[cd] %@", query)

        if let results = try? backgroundContext.fetch(fetchRequest) {
            let objectIDs = results.map { $0.objectID }
            DispatchQueue.main.async {
                self.updateUIWithResults(objectIDs: objectIDs)
            }
        }
    }
}

func updateUIWithResults(objectIDs: [NSManagedObjectID]) {
    mainContext.perform {
        let objects = objectIDs.compactMap { mainContext.object(with: $0) as? MyEntity }
        print("Fetched objects on main context: \(objects)")
    }
}
```

---

#### **Scenario 2: Syncing Data Across Different PSCs**
You have two Core Data stacks (e.g., local and cloud databases) and need to transfer objects between them.

**Solution**:
- Extract the `URIRepresentation` of objects from the source PSC.
- Use `managedObjectID(forURIRepresentation:)` to fetch valid IDs for the target PSC.
- Fetch objects in the target context using these IDs.

**Code Example**:
```swift
func syncDataBetweenStacks(sourceContext: NSManagedObjectContext, targetContext: NSManagedObjectContext) {
    let fetchRequest: NSFetchRequest<MyEntity> = MyEntity.fetchRequest()
    
    sourceContext.perform {
        if let sourceObjects = try? sourceContext.fetch(fetchRequest) {
            let uris = sourceObjects.map { $0.objectID.uriRepresentation() }
            
            targetContext.perform {
                let targetObjects = uris.compactMap { uri -> NSManagedObject? in
                    let psc = targetContext.persistentStoreCoordinator!
                    guard let targetID = psc.managedObjectID(forURIRepresentation: uri) else { return nil }
                    return targetContext.object(with: targetID)
                }
                print("Synced objects: \(targetObjects)")
            }
        }
    }
}
```

---

### **Key Points to Remember**

1. **Pass Object IDs**:
   - Use `objectID` to transfer objects between contexts safely.
2. **Use `URIRepresentation` for Different PSCs**:
   - Convert `objectID` to URI for cross-PSC operations.
3. **Optimize with Row Cache**:
   - Pass objects to the target context only for extracting their IDs if within the same PSC.
4. **Dispatch to Correct Queue**:
   - Always use `perform` or `performAndWait` for context-specific operations.

---

### **Challenges of Multiple Contexts**

Concurrency introduces complications such as:
- **Conflicts**: Two contexts modifying the same data simultaneously.
- **Race Conditions**: Timing issues when saving or merging changes.

These challenges are handled using strategies like conflict resolution policies and proper synchronization, which are covered in subsequent chapters. By adhering to the rules of object ID passing and queue isolation, you can avoid many concurrency issues in Core Data.
