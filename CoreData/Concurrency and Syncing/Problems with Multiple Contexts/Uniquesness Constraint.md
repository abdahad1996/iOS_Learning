### **Uniqueness Constraints in Core Data**

Uniqueness constraints are a Core Data feature introduced in iOS 9 and macOS 10.11 to ensure that entities in the store adhere to specific uniqueness rules. These constraints prevent duplicate objects with the same unique identifier from being created, even when using multiple managed object contexts. Below is a detailed explanation of uniqueness constraints, how they work, and how you can implement them.

---

### **Why Are Uniqueness Constraints Needed?**

In Core Data, it's possible to create duplicate entries unintentionally when working with multiple contexts. For example:
1. **Scenario**:
   - A `Country` entity has a unique identifier, such as `numericISO3166Code`.
   - Two contexts (e.g., `mainContext` and `backgroundContext`) create `Country` objects for the same country (`numericISO3166Code = 250` for France) simultaneously.
   - Without uniqueness constraints, both objects could be saved to SQLite, creating duplicates.

2. **Problem**:
   - This leads to data inconsistencies, making it difficult to fetch or manipulate data reliably.

---

### **What Are Uniqueness Constraints?**

Uniqueness constraints ensure that:
1. **Within a Context**:
   - Objects in the same context must satisfy the uniqueness rule before saving.
2. **Across Persistent Store**:
   - When saving to the persistent store, objects in the context and the persisted data must satisfy the uniqueness rule.

If a conflict arises, Core Data resolves it using the context's **merge policy** or raises an error.

---

### **Defining Uniqueness Constraints**

1. **Using the Data Model Inspector**:
   - Open your Core Data model in Xcode.
   - Select the entity (e.g., `Country`).
   - Under the "Constraints" section, add an attribute (e.g., `numericISO3166Code`) as a uniqueness constraint.

2. **Defining Constraints Programmatically**:
   - Use the `NSEntityDescription` API to define constraints:
     ```swift
     let entity = NSEntityDescription()
     entity.name = "Country"
     entity.managedObjectClassName = "Country"
     entity.uniquenessConstraints = [["numericISO3166Code"]]
     ```

---

### **How Uniqueness Constraints Work**

1. **Validation During Save**:
   - Core Data validates the constraints during a `save()` operation.
   - If a conflict exists:
     - Within the same context: Core Data resolves it immediately.
     - Between the context and the persistent store: Core Data reports it as a conflict, which must be resolved using a merge policy.

2. **Conflict Handling**:
   - Uniqueness constraint conflicts are treated like **optimistic locking conflicts**.
   - Core Data provides details about the conflict in the error’s `userInfo` dictionary under the `NSConstraintConflict` key.

---

### **Predefined Merge Policies for Uniqueness Constraints**

Core Data’s predefined merge policies dictate how conflicts are resolved:

1. **`NSRollbackMergePolicy`**:
   - The **persisted state** always wins.
   - Conflicting changes in the context are rolled back.

2. **`NSOverwriteMergePolicy`**:
   - The **context’s changes** win.
   - The existing persisted object is deleted.

3. **`NSMergeByPropertyStoreTrumpMergePolicy`**:
   - Behaves like `NSRollbackMergePolicy`.
   - The persisted object remains unchanged, and conflicting changes are rolled back.

4. **`NSMergeByPropertyObjectTrumpMergePolicy`**:
   - The conflicting object in the context is merged into the persisted object.
   - Changed properties in the context overwrite those in the persisted object.

**Example**:
```swift
let context = persistentContainer.viewContext
context.mergePolicy = NSMergeByPropertyObjectTrumpMergePolicy
```

---

### **Practical Example**

#### **1. Scenario: Enforcing Unique Countries**

A `Country` entity has a unique `numericISO3166Code`. We want to prevent duplicates even when data is created in different contexts.

1. **Model Definition**:
   - Add a uniqueness constraint for `numericISO3166Code`.

2. **Creating Objects in Different Contexts**:
```swift
// Main Context
let mainContext = persistentContainer.viewContext
let france = Country(context: mainContext)
france.numericISO3166Code = 250
france.name = "France"

// Background Context
let backgroundContext = persistentContainer.newBackgroundContext()
backgroundContext.perform {
    let anotherFrance = Country(context: backgroundContext)
    anotherFrance.numericISO3166Code = 250
    anotherFrance.name = "France"

    try? backgroundContext.save()
}

// Save Main Context
try? mainContext.save()
```

3. **Outcome**:
   - Without uniqueness constraints: Two entries for France are created in the store.
   - With uniqueness constraints: Core Data detects the conflict and resolves it using the specified merge policy.

---

#### **2. Handling Save Errors**

If no merge policy is set, Core Data throws an error for uniqueness constraint conflicts.

**Error Handling**:
```swift
do {
    try context.save()
} catch let error as NSError {
    if let conflicts = error.userInfo[NSPersistentStoreSaveConflictsErrorKey] as? [NSConstraintConflict] {
        for conflict in conflicts {
            print("Conflict: \(conflict)")
        }
    }
}
```

---

### **Custom Merge Policies**

For advanced conflict resolution, you can create a custom merge policy by subclassing `NSMergePolicy`.

1. **Define Custom Rules**:
   - Implement specific criteria to determine which object should win in a conflict.

2. **Override `resolve(constraintConflicts:)`**:
```swift
class CustomMergePolicy: NSMergePolicy {
    override func resolve(constraintConflicts list: [NSConstraintConflict]) throws {
        for conflict in list {
            let persistedObject = conflict.databaseObject as? Country
            let conflictingObject = conflict.conflictingObjects.first as? Country
            
            // Custom resolution logic: prefer the object with the latest update
            if let persisted = persistedObject, let conflicting = conflictingObject {
                if persisted.updatedAt > conflicting.updatedAt {
                    // Persisted object wins
                    context.delete(conflicting)
                } else {
                    // Conflicting object wins
                    context.delete(persisted)
                }
            }
        }
        try super.resolve(constraintConflicts: list)
    }
}
```

3. **Set the Custom Merge Policy**:
```swift
context.mergePolicy = CustomMergePolicy()
```

---

### **Key Considerations**

1. **React to Changes**:
   - Use `objects-did-change` notifications to handle changes in deleted or merged objects.

2. **Test Uniqueness Rules**:
   - Ensure that fetch requests and context merges maintain data consistency.

3. **Use Predicates for Fetching**:
   - Fetch or display only the relevant object in case of uniqueness conflicts.

---

### **Takeaways**

- **Uniqueness Constraints**:
  - Prevent duplicate objects with the same unique identifier.
  - Enforced both within a context and across the persistent store.

- **Merge Policies**:
  - Predefined merge policies resolve conflicts automatically, with behaviors like rolling back changes or merging properties.

- **Custom Policies**:
  - Allow fine-grained control over conflict resolution.

- **Practical Scenarios**:
  - Use uniqueness constraints in apps where entities like users, countries, or products must have unique identifiers.

By leveraging uniqueness constraints and merge policies, you can maintain consistent and reliable data in Core Data, even in complex scenarios involving multiple contexts.
