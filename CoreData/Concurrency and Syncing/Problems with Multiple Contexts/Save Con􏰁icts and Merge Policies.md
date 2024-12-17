
### **Save Conflicts and Merge Policies in Core Data (Simplified with Examples)**

When working with multiple Core Data contexts, **save conflicts** may arise. These conflicts occur when different versions of an object exist in memory (managed object contexts) and in the persistent store (SQLite database). Core Data uses **merge policies** to resolve these conflicts.

Here’s a detailed and simplified explanation of how Core Data handles save conflicts and how you can configure it for your needs.

---

### **What Are Save Conflicts?**

Save conflicts happen in two main scenarios:

1. **Between a Snapshot and the Persistent Store's Row Cache**:
   - Occurs when multiple contexts connected to the same persistent store coordinator modify the same object.

2. **Between the Row Cache and SQLite**:
   - Happens when multiple coordinators are accessing the same SQLite database (e.g., in multi-process setups).

---

### **How Save Conflicts Are Detected**

Core Data uses **snapshots** to detect changes:
- **Snapshots**: Represent the last saved state of an object.
- **Version Identifiers**: Each object has a version number that is incremented on every change.
- **Comparison**: Core Data compares these version numbers during a save operation to detect conflicts.

---

### **Merge Policies**

Merge policies determine how conflicts are resolved. Core Data provides predefined merge policies, and you can create custom ones for specific needs.

---

### **Predefined Merge Policies**

#### **1. `NSErrorMergePolicy` (Default)**
- **Behavior**:
  - Throws an error when a conflict occurs.
  - Leaves it to the developer to resolve the conflict manually.

- **Use Case**:
  - When you need full control over how conflicts are resolved.

- **Example**:
  ```swift
  do {
      try context.save()
  } catch let error as NSError {
      if let conflicts = error.userInfo[NSPersistentStoreSaveConflictsErrorKey] as? [NSMergeConflict] {
          for conflict in conflicts {
              print("Conflict: \(conflict)")
          }
      }
  }
  ```

#### **2. `NSRollbackMergePolicy`**
- **Behavior**:
  - Discards local changes in the conflicting object and keeps the persistent store version.
- **Use Case**:
  - When the persistent store is considered the source of truth.

#### **3. `NSOverwriteMergePolicy`**
- **Behavior**:
  - Overwrites the persistent store with the in-memory object.
  - Ignores conflicts entirely.
- **Use Case**:
  - When in-memory data is always the correct version.

- **Example**:
  ```swift
  context.mergePolicy = NSOverwriteMergePolicy
  ```

#### **4. `NSMergeByPropertyStoreTrumpMergePolicy`**
- **Behavior**:
  - Resolves conflicts on a property-by-property basis.
  - The persistent store's version wins if both versions modify the same property.

#### **5. `NSMergeByPropertyObjectTrumpMergePolicy`**
- **Behavior**:
  - Resolves conflicts on a property-by-property basis.
  - The in-memory object's version wins if both versions modify the same property.

---

### **Custom Merge Policies**

If the predefined merge policies don’t fit your needs, you can create a **custom merge policy** by subclassing `NSMergePolicy`.

---

#### **Example: Custom Merge Policy**

**Scenario**:  
You have a `Country` object with an `updatedAt` property. When resolving conflicts, you want to ensure the most recent `updatedAt` date wins.

1. **Create the Custom Merge Policy**
```swift
public class MoodyMergePolicy: NSMergePolicy {
    public enum MergeMode {
        case remote
        case local
    }

    private var mergeType: NSMergePolicyType {
        switch self {
        case .remote: return .mergeByPropertyObjectTrumpMergePolicyType
        case .local: return .mergeByPropertyStoreTrumpMergePolicyType
        }
    }

    required public init(mode: MergeMode) {
        super.init(merge: mode.mergeType)
    }

    override open func resolve(optimisticLockingConflicts list: [NSMergeConflict]) throws {
        var regionsAndLatestDates: [(UpdateTimestampable, Date)] = []
        
        for conflict in list {
            if let region = conflict.sourceObject as? UpdateTimestampable {
                let newestDate = max(
                    conflict.objectSnapshot?["updatedAt"] as? Date ?? .distantPast,
                    conflict.persistedSnapshot?["updatedAt"] as? Date ?? .distantPast
                )
                regionsAndLatestDates.append((region, newestDate))
            }
        }
        
        try super.resolve(optimisticLockingConflicts: list)
        
        for (region, date) in regionsAndLatestDates {
            region.updatedAt = date
        }
    }
}
```

2. **Use the Custom Merge Policy**
Assign the custom policy to your contexts:
```swift
let uiContext = container.viewContext
uiContext.mergePolicy = MoodyMergePolicy(mode: .local)

let syncContext = container.newBackgroundContext()
syncContext.mergePolicy = MoodyMergePolicy(mode: .remote)
```

---

### **Resolving Conflicts with `refresh(_:mergeChanges:)`**

If you encounter a conflict, you can refresh the object to update it with the latest values from the persistent store while keeping unsaved changes.

- **Example**:
```swift
context.refresh(managedObject, mergeChanges: true)
```

This updates the object's attributes and version identifier.

---

### **When to Use Each Merge Policy**

1. **UI Context**:
   - Use `NSMergeByPropertyStoreTrumpMergePolicy` if persistent data is the truth.
   - Use `NSMergeByPropertyObjectTrumpMergePolicy` if in-memory changes are the truth.

2. **Background Context**:
   - Use `NSOverwriteMergePolicy` for aggressive overwrites.
   - Use a custom policy for special requirements, like merging timestamps.

---

### **Key Takeaways**

1. **Conflicts Are Natural**:
   - Save conflicts arise when multiple contexts modify the same object.

2. **Predefined Policies Are Convenient**:
   - Core Data provides several out-of-the-box merge policies for common scenarios.

3. **Custom Policies Add Flexibility**:
   - Subclass `NSMergePolicy` to implement custom conflict resolution logic.

4. **Plan Your Merge Strategy**:
   - Choose a merge policy that aligns with your application's requirements, whether that’s prioritizing the persistent store, in-memory changes, or a hybrid approach.

---

By understanding merge policies and how Core Data handles conflicts, you can build robust and reliable applications that manage data consistency across multiple contexts.
