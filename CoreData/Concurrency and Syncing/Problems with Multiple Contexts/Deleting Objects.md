

<img width="592" alt="Screenshot 2024-12-17 at 10 55 25 AM" src="https://github.com/user-attachments/assets/1cb198e7-bace-4c0b-994b-88959e24e319" />
### **Deleting Objects in Core Data**

Handling object deletions in Core Data can be tricky when using multiple managed object contexts (MOCs), especially when they operate on different threads or queues. If not managed carefully, deletions can lead to runtime crashes or data inconsistencies. Below is a detailed explanation of how deletions work, common pitfalls, and strategies to handle deletions effectively.

---

### **The Problem with Deleting Objects Across Contexts**

1. **Faults and Deletions**:
   - A fault in one context represents a placeholder for an object whose data hasn’t been loaded yet.
   - If another context deletes the object corresponding to the fault, the first context will crash when it tries to fulfill the fault, as the data no longer exists.

2. **Concurrency and Timing**:
   - When one context deletes an object and saves changes, it sends a `context-did-save` notification.
   - Other contexts must merge these changes to mark the object as deleted. However, there’s a **window of time** where another context might attempt to access the fault before the merge occurs.

---

### **Solutions for Handling Deletions**

#### **1. Use `shouldDeleteInaccessibleFaults`**

Introduced in iOS 9/macOS 10.11, the `shouldDeleteInaccessibleFaults` property automatically marks faults as deleted if they cannot be fulfilled from the store.

**How it Works**:
- When the property is enabled (default behavior), attempting to access an inaccessible fault will mark the object as deleted.
- Attributes and relationships of the deleted object will return `nil` or `0`, depending on their types.

**Limitations**:
- **Non-Optional Attributes**: If any attribute is non-optional, accessing it will cause a crash.
- **Validation Rules**: Core Data won’t enforce non-optional validation rules when this property is enabled.

**Usage**:
```swift
context.shouldDeleteInaccessibleFaults = true
```

**When to Use**:
- When you’re okay with attributes being `nil` or `0` for deleted objects.
- When simplicity is preferred, and you don’t have strict validation requirements.

---

#### **2. Two-Step Deletion**

A more controlled strategy is **two-step deletion**, where objects are marked for deletion before being permanently removed.

**How It Works**:
1. **Mark for Deletion**:
   - Add a `markedForDeletionDate` attribute to your entities.
   - Set this attribute to the current date when marking an object for deletion.

2. **Filter Out Marked Objects**:
   - Update fetch requests or predicates to exclude objects with a non-`nil` `markedForDeletionDate`.

3. **Delayed Permanent Deletion**:
   - Periodically delete objects marked for deletion after a certain interval.

**Advantages**:
- Avoids crashes caused by accessing faults for deleted objects.
- Gives you full control over when objects are deleted.
- Allows other contexts to release references to the object before it is permanently deleted.

---

#### **Implementation of Two-Step Deletion**

1. **Define the `DelayedDeletable` Protocol**:
   - Add a `markedForDeletionDate` attribute and a `markForLocalDeletion` method to your entities.

```swift
public protocol DelayedDeletable: AnyObject {
    var markedForDeletionDate: Date? { get set }
    func markForLocalDeletion()
}

extension DelayedDeletable where Self: NSManagedObject {
    public func markForLocalDeletion() {
        guard isFault || markedForDeletionDate == nil else { return }
        markedForDeletionDate = Date()
    }
}
```

2. **Filter Out Marked Objects**:
   - Update fetch requests to exclude marked objects.

```swift
extension DelayedDeletable {
    public static var notMarkedForLocalDeletionPredicate: NSPredicate {
        return NSPredicate(format: "markedForDeletionDate == NULL")
    }
}
```

3. **Batch Delete Marked Objects**:
   - Use a batch delete operation to permanently remove marked objects after a delay.

```swift
extension DelayedDeletable where Self: NSManagedObject {
    static func batchDeleteMarkedObjects(in context: NSManagedObjectContext) {
        let fetchRequest = NSFetchRequest<NSFetchRequestResult>(entityName: Self.entity().name!)
        let cutoff = Date(timeIntervalSinceNow: -120) // Two minutes ago
        fetchRequest.predicate = NSPredicate(format: "markedForDeletionDate < %@", cutoff as NSDate)
        
        let batchDeleteRequest = NSBatchDeleteRequest(fetchRequest: fetchRequest)
        batchDeleteRequest.resultType = .resultTypeStatusOnly
        try? context.execute(batchDeleteRequest)
    }
}
```

4. **Trigger Batch Deletion**:
   - Perform the deletion when the app enters the background or during a periodic cleanup task.

```swift
func cleanUpDeletedObjects() {
    MyEntity.batchDeleteMarkedObjects(in: context)
}
```

---

#### **3. Delete Propagation with Two-Step Deletion**

When using two-step deletion, Core Data’s automatic delete rules (like cascade) won’t trigger until the object is permanently deleted. You must manually update relationships for objects marked for deletion.

**Example**: Removing a country from its continent when the country is marked for deletion.
```swift
public override func willSave() {
    super.willSave()
    if changedForDelayedDeletion {
        removeFromContinent()
    }
}

extension DelayedDeletable where Self: NSManagedObject {
    public var changedForDelayedDeletion: Bool {
        return changedValue(forKey: "markedForDeletionDate") != nil
    }
}
```

---

### **Example Workflow: Deleting Objects Safely**

1. **Mark an Object for Deletion**:
   ```swift
   let country = context.object(with: countryID) as! Country
   country.markForLocalDeletion()
   try? context.save()
   ```

2. **Exclude Marked Objects from the UI**:
   ```swift
   fetchRequest.predicate = Country.notMarkedForLocalDeletionPredicate
   ```

3. **Permanently Delete Objects**:
   ```swift
   Country.batchDeleteMarkedObjects(in: backgroundContext)
   ```

4. **Update Relationships**:
   - Use the `willSave` method to update related objects for marked entities.

---

### **Best Practices**

1. **Understand Your Use Case**:
   - Use `shouldDeleteInaccessibleFaults` for simple apps where attributes can be `nil` for deleted objects.
   - Use two-step deletion for complex apps where safety and control are critical.

2. **Keep Relationships Consistent**:
   - Manually handle relationships when using two-step deletion.

3. **Merge Changes Properly**:
   - Always merge `context-did-save` notifications into all contexts to propagate deletions.

4. **Test Thoroughly**:
   - Simulate real-world scenarios with multiple contexts and deletions to ensure stability.

---

### **Key Takeaways**

- **Faults and Deletions**:
  - Faults referencing deleted objects can crash the app if not handled properly.

- **Solutions**:
  - Use `shouldDeleteInaccessibleFaults` for simplicity but be mindful of its limitations.
  - Implement two-step deletion for better control and safety.

- **Batch Deletion**:
  - Use batch delete operations for efficient permanent deletion of marked objects.

- **Relationship Management**:
  - When using two-step deletion, manually update related objects to maintain consistency.

By understanding and applying these techniques, you can safely manage deletions in Core Data across multiple contexts and avoid runtime crashes or data inconsistencies.
