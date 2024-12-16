
<img width="599" alt="Screenshot 2024-12-16 at 9 24 38 PM" src="https://github.com/user-attachments/assets/dce31d4b-b0f4-4ca8-a12f-c49d5312b06e" />

### **Setups with Nested Contexts in Core Data**

Nested contexts allow you to chain managed object contexts (MOCs) together in a parent-child relationship. Instead of being connected directly to the persistent store coordinator, a child context can be connected to another context. This feature provides flexibility for managing data changes and saves, with specific use cases including deferred saves for large changes and temporary scratchpads for isolated edits.

---

### **How Nested Contexts Work**

1. **Parent Context**:
   - The parent context is responsible for persisting changes to its parent (e.g., the persistent store coordinator or another parent context).

2. **Child Context**:
   - The child context performs in-memory operations and pushes its changes to the parent context upon saving.

3. **Saving Mechanism**:
   - Saving the **child context** only pushes changes to the **parent context** (an in-memory operation).
   - Saving the **parent context** commits those changes to the persistent store (involves I/O operations).

---

### **Use Case 1: Main Context as a Child of a Private Context**

This setup is used to **defer the saving of large data changes** from the main queue context (UI-related work) to a background queue context, reducing UI blocking during heavy save operations.

#### **Architecture**
```plaintext
Main Queue Context (Child)
Private Queue Context (Parent)
Persistent Store Coordinator
Persistent Store
SQLite
```

#### **How It Works**
1. The main queue context (UI context) is set as a child of a private queue context.
2. Changes in the main context are saved to the private context (in-memory).
3. The private context is saved to the persistent store (background I/O).

#### **Advantages**
- Reduces the impact of heavy save operations on the UI thread.
- Defers I/O operations (saving to the database) to a private background queue.

#### **Disadvantages**
- Contention can still occur if the main context tries to perform save or fetch operations while the private context is persisting data.
- Complications arise if additional child contexts or independent top-level contexts are introduced.

#### **Example: Deferring Large Saves**
Imagine a document editing app where a user pastes a large document. To avoid UI freezing during the save:

```swift
// Core Data setup
let persistentContainer = NSPersistentContainer(name: "ModelName")
persistentContainer.loadPersistentStores { _, error in
    if let error = error {
        print("Failed to load persistent stores: \(error)")
    }
}

// Private parent context
let privateContext = persistentContainer.newBackgroundContext()

// Main context as a child of the private context
let mainContext = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
mainContext.parent = privateContext

// Save large changes in the main context
mainContext.perform {
    let largeData = LargeEntity(context: mainContext)
    largeData.content = "A large pasted document"
    
    do {
        try mainContext.save() // Push changes to the private context (in-memory)
        privateContext.perform {
            do {
                try privateContext.save() // Persist changes to the database
            } catch {
                print("Error saving private context: \(error)")
            }
        }
    } catch {
        print("Error saving main context: \(error)")
    }
}
```

---

### **Use Case 2: Child Contexts as Scratchpads**

Child contexts can serve as **temporary scratchpads** for making changes without affecting other parts of the app. These changes can either be discarded or pushed to the parent context and saved.

#### **Architecture**
```plaintext
Main Queue Context (Parent)
Main Queue Context (Child)
Persistent Store Coordinator
Persistent Store
SQLite
```

#### **How It Works**
1. The child context is created as a child of the main context.
2. Changes are made in the child context without affecting the parent.
3. Upon saving:
   - Changes are pushed to the parent context.
   - You can either persist the changes to the database or discard them.

#### **Advantages**
- Isolates changes, allowing you to make temporary edits.
- Easy to discard changes by simply destroying the child context.

#### **Disadvantages**
- Saving the child context pushes **all changes** to the parent context, potentially overriding unsaved changes in the parent.
- Does not improve concurrency—it is only for temporary edits.

#### **Example: Edit Dialog in a Contacts App**
In a contacts app, the user edits a contact in an isolated child context. If the user cancels the edits, changes are discarded. If the user saves, changes are pushed to the parent context and persisted.

```swift
// Core Data setup
let persistentContainer = NSPersistentContainer(name: "ModelName")
persistentContainer.loadPersistentStores { _, error in
    if let error = error {
        print("Failed to load persistent stores: \(error)")
    }
}

// Main context
let mainContext = persistentContainer.viewContext

// Child context for editing
let childContext = NSManagedObjectContext(concurrencyType: .mainQueueConcurrencyType)
childContext.parent = mainContext

// Editing a contact in the child context
func editContact(contactID: NSManagedObjectID) {
    childContext.perform {
        guard let contact = try? childContext.existingObject(with: contactID) as? Contact else {
            return
        }

        // Modify contact properties in the child context
        contact.name = "Updated Name"
        contact.phoneNumber = "123-456-7890"

        // User decides to save changes
        do {
            try childContext.save() // Push changes to the main context
            try mainContext.save()  // Persist changes to the database
        } catch {
            print("Error saving changes: \(error)")
        }
    }
}

// Discarding changes if the user cancels
func discardChanges() {
    childContext.reset() // Discard all changes in the child context
}
```

---

### **Best Practices**

1. **Use Nested Contexts Only When Necessary**:
   - For example, when deferring large saves or isolating temporary edits.

2. **Be Aware of Save Propagation**:
   - Changes in a child context are automatically propagated to the parent context upon saving.

3. **Avoid Overcomplicating the Setup**:
   - Adding multiple nested contexts or top-level contexts can lead to unexpected behavior and performance issues.

4. **Handle Contention Carefully**:
   - In the deferred save setup, ensure that save or fetch operations in the main context do not overlap with heavy saves in the private context.

---

### **When to Use Each Setup**

#### **Deferred Saves**
- **Scenario**: Large changes (e.g., importing data, editing a large document).
- **Objective**: Reduce UI blocking during save operations.

#### **Scratchpads**
- **Scenario**: Temporary edits (e.g., editing a contact, creating a draft).
- **Objective**: Provide isolated changes that can be easily discarded or saved.

---

### **Conclusion**

Nested contexts provide powerful tools for managing Core Data operations. Whether you're deferring large saves to avoid UI blocking or creating scratchpads for isolated edits, nested contexts enable efficient workflows. However, they come with trade-offs, such as potential contention and complex save propagation, so use them judiciously.
