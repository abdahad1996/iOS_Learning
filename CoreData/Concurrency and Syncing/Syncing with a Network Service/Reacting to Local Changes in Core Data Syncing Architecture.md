### **Reacting to Local Changes in Core Data Syncing Architecture**

The syncing architecture for Core Data ensures that **local changes** (whether initiated by the UI or sync processes) are properly identified, processed, and propagated to a backend (e.g., CloudKit). This approach minimizes complexity and guarantees data consistency across contexts and sessions.

---
<img width="557" alt="Screenshot 2024-12-16 at 6 04 36 PM" src="https://github.com/user-attachments/assets/fbd935ed-962c-4e07-935d-e219ee7bd59c" />

### **Key Concepts**

1. **Unified Processing for Inserts and Updates**:
   - Treating **inserts** and **updates** uniformly simplifies logic.
   - Sync processors don't differentiate between new and modified objects, relying instead on **predicates** to determine which objects need action.

2. **Durability**:
   - By relying on predicates rather than ephemeral events (e.g., "newly inserted"), the system can recover from app crashes or suspensions.
   - After a crash, the system fetches objects matching the predicates and resumes syncing.

3. **Remapping Across Contexts**:
   - Changes from the **UI context** are mapped to the **sync context** to ensure consistency between the two.

---

### **Step-by-Step Workflow**

#### **1. Detecting Local Changes**

When a user modifies the data (e.g., creates or updates a `Mood` object) in the **main (UI) context**, a `did-save` notification is triggered.

```swift
let mood = Mood(context: mainContext)
mood.text = "Happy"
mood.createdAt = Date()
mood.remoteIdentifier = nil // Indicates it's unsynced
try? mainContext.save()
```

- **What Happens**:
  - Core Data saves the changes in the **main context** and triggers a `did-save` notification.
  - The sync architecture listens to these notifications and forwards the changes to the **sync context**.

---

#### **2. Forwarding Changes to the Sync Context**

The **SyncCoordinator** observes `did-save` notifications. When a change occurs, it remaps objects from the **main context** to the **sync context** and processes them.

##### **Code: Forwarding Changes**
```swift
fileprivate func notifyAboutChangedObjects(from notification: ContextDidSaveNotification) {
    syncContext.perform {
        // Remap objects from the main context to the sync context
        let updates = notification.updatedObjects.remap(to: self.syncContext)
        let inserts = notification.insertedObjects.remap(to: self.syncContext)

        // Process all changed objects (both inserts and updates)
        self.processChangedLocalObjects(updates + inserts)
    }
}
```

---

#### **3. Remapping Objects**

Core Data objects belong to a specific **context**. Before processing changes in the **sync context**, the objects from the main context must be remapped to their equivalent objects in the sync context using their **object IDs**.

##### **Code: Remapping**
```swift
extension Sequence where Iterator.Element: NSManagedObject {
    func remap(to context: NSManagedObjectContext) -> [Iterator.Element] {
        return map { object in
            guard object.managedObjectContext !== context else { return object }
            return context.object(with: object.objectID) as! Iterator.Element
        }
    }
}
```

- **Why Remapping Is Needed**:
  - Objects from one context cannot be directly used in another context.
  - Remapping ensures that changes are applied to the correct objects in the sync context.

---

#### **4. Processing Changes in Change Processors**

The **SyncCoordinator** forwards the changed objects to **Change Processors**, which act upon them based on their predicates.

##### **Example: MoodUploader Processor**
The `MoodUploader` processes unsynced `Mood` objects.

```swift
class MoodUploader {
    private let predicate = NSPredicate(format: "%K == NULL", "remoteIdentifier")

    func processChangedLocalObjects(_ objects: [NSManagedObject], in syncCoordinator: SyncCoordinator) {
        let unsyncedMoods = objects.compactMap { $0 as? Mood }.filter { predicate.evaluate(with: $0) }
        for mood in unsyncedMoods {
            uploadMood(mood)
        }
    }

    private func uploadMood(_ mood: Mood) {
        // Convert the `Mood` object to a CloudKit record and upload
        let record = CKRecord(recordType: "Mood")
        record["text"] = mood.text
        record["createdAt"] = mood.createdAt

        // Simulate CloudKit upload
        CloudKitManager.save(record) { result in
            switch result {
            case .success(let recordID):
                mood.remoteIdentifier = recordID
                try? mood.managedObjectContext?.save()
            case .failure(let error):
                print("Failed to upload mood: \(error)")
            }
        }
    }
}
```

---

#### **5. Predicate-Based Processing**

Each change processor operates on specific entities and predicates. For example:

1. **Uploading New Objects**:
   ```swift
   let notUploaded = NSPredicate(format: "%K == NULL", "remoteIdentifier")
   ```
   - Objects without a `remoteIdentifier` are unsynced and need uploading.

2. **Deleting Objects**:
   ```swift
   let pendingDeletion = NSPredicate(format: "%K == YES", "pendingRemoteDeletion")
   ```
   - Objects marked for deletion (`pendingRemoteDeletion = true`) are sent to the backend for removal.

---

#### **6. Handling App Crashes or Suspensions**

If the app is interrupted (e.g., crash or suspension), predicates ensure that incomplete operations are resumed upon relaunch:

1. During startup, the **SyncCoordinator** fetches objects matching the processors' predicates.
2. Change processors handle these objects just as they would for real-time changes.

##### **Example: Resuming Unsynced Moods**
```swift
func resumeSync(for context: NSManagedObjectContext) {
    let fetchRequest: NSFetchRequest<Mood> = Mood.fetchRequest()
    fetchRequest.predicate = NSPredicate(format: "%K == NULL", "remoteIdentifier")
    
    let unsyncedMoods = try? context.fetch(fetchRequest)
    unsyncedMoods?.forEach { uploadMood($0) }
}
```

---

#### **7. UI Updates from Sync Context Changes**

When the **sync context** saves, a `did-save` notification propagates changes to the **main context**. The UI observes these changes and updates automatically.

---

### **Why This Architecture Works**

1. **Predicate-Based Logic**:
   - Treating inserts and updates uniformly ensures robust and simplified logic.

2. **Resilience**:
   - Even after app interruptions, predicates identify pending actions, ensuring no data is lost or left unsynced.

3. **Thread Safety**:
   - All sync operations occur on a private queue, isolating them from the UI thread.

4. **Performance**:
   - Remapping avoids redundant fetches.
   - Change processors act only on relevant objects.

---

### **End-to-End Example**

#### **Scenario**:
1. User adds a new `Mood` with text `"Happy"`.
2. Main context saves the mood locally and triggers a `did-save` notification.
3. SyncCoordinator remaps the mood to the sync context.
4. `MoodUploader` detects that the mood has no `remoteIdentifier` and uploads it to CloudKit.
5. CloudKit returns a `recordID`, which is saved back to Core Data.
6. Sync context saves, triggering a notification that updates the main context and UI.

---

### **Summary**

Reacting to local changes in Core Data's syncing architecture involves:
- **Unified handling of inserts and updates** using predicates.
- **Change processors** that operate on domain-specific logic.
- **Resilience** to interruptions by relying on predicates instead of transient state.
- **Efficient context mapping** to propagate changes between UI and sync contexts.

This design ensures consistency, simplicity, and performance, making it suitable for robust sync implementations in Core Data.
