### **Change Processors in Core Data Sync Architecture**

Change processors are responsible for handling specific tasks related to syncing data between the local Core Data store and the cloud. They encapsulate **domain-specific knowledge**, allowing for modular, maintainable, and focused code. In the **Moody example app**, there are three change processors:

1. **MoodDownloader**: Downloads moods from the cloud.
2. **MoodUploader**: Uploads new moods from the local store to the cloud.
3. **MoodRemover**: Removes moods from the cloud when they are deleted locally.

---

### **Core Responsibilities of Change Processors**

1. **Processing Local Changes**:
   - Detect and handle new or updated local objects that need to be synced to the cloud.

2. **Processing Remote Changes**:
   - Fetch and integrate updates from the cloud into the local Core Data store.

3. **Tracking Objects in Progress**:
   - Prevent duplicate processing of the same object while it’s being handled.

4. **Predicate-Based Filtering**:
   - Define which objects are relevant for the change processor (e.g., moods waiting for upload).

---

### **Key Methods in Change Processors**

1. **Processing Local Changes**:
   ```swift
   func processChangedLocalObjects(_ objects: [NSManagedObject], in context: ChangeProcessorContext)
   ```
   - Handles objects that have been inserted or updated locally.

2. **Processing Remote Changes**:
   ```swift
   func processRemoteChanges<T>(
       _ changes: [RemoteRecordChange<T>],
       in context: ChangeProcessorContext,
       completion: () -> ()
   )
   ```
   - Handles changes received from the cloud.

3. **Startup Initialization**:
   - When the app launches, the sync coordinator ensures any pending tasks (e.g., unsynced objects) are picked up:
   ```swift
   func entityAndPredicateForLocallyTrackedObjects(in context: ChangeProcessorContext) -> EntityAndPredicate<NSManagedObject>?
   func fetchLatestRemoteRecords(in context: ChangeProcessorContext)
   ```

---

### **How Change Processors Work**

#### **1. Process Local Changes**
The sync coordinator passes local changes to the change processor.

```swift
func processChangedLocalObjects(_ objects: [NSManagedObject], in context: ChangeProcessorContext) {
    let moodsToUpload = objects.compactMap { $0 as? Mood }
    processInsertedMoods(moodsToUpload, in: context)
}
```

The processor filters the objects using a **predicate** to identify those it is responsible for (e.g., moods without a remote identifier).

---

#### **2. Process Remote Changes**
When changes are received from CloudKit, they are passed to the appropriate change processor.

```swift
func processRemoteChanges<T>(
    _ changes: [RemoteRecordChange<T>],
    in context: ChangeProcessorContext,
    completion: () -> ()
) {
    // Handle remote changes (e.g., download new moods or remove outdated ones)
    completion()
}
```

---

#### **3. Startup Initialization**
At startup, the sync coordinator ensures that change processors resume any incomplete tasks.

- **Fetch Local Objects Pending Sync**:
   ```swift
   func entityAndPredicateForLocallyTrackedObjects(in context: ChangeProcessorContext) -> EntityAndPredicate<NSManagedObject>? {
       return EntityAndPredicate(entity: Mood.entity(), predicate: Mood.waitingForUploadPredicate)
   }
   ```

- **Fetch Remote Records**:
   ```swift
   func fetchLatestRemoteRecords(in context: ChangeProcessorContext) {
       // Fetch new records from the cloud
   }
   ```

---

### **Example: MoodUploader Change Processor**

The **MoodUploader** change processor is responsible for uploading new moods from the local store to CloudKit.

#### **Core Implementation**

```swift
final class MoodUploader: ElementChangeProcessor {
    var elementsInProgress = InProgressTracker<Mood>()

    func setup(for context: ChangeProcessorContext) {
        // No-op: Setup logic if needed
    }

    func processChangedLocalElements(_ objects: [Mood], in context: ChangeProcessorContext) {
        processInsertedMoods(objects, in: context)
    }

    func processRemoteChanges<T>(
        _ changes: [RemoteRecordChange<T>],
        in context: ChangeProcessorContext,
        completion: () -> ()
    ) {
        // No-op: MoodUploader does not process remote changes
        completion()
    }

    func fetchLatestRemoteRecords(in context: ChangeProcessorContext) {
        // No-op: MoodUploader does not fetch remote records
    }

    var predicateForLocallyTrackedElements: NSPredicate {
        return Mood.waitingForUploadPredicate
    }
}
```

- **Predicate**:
   ```swift
   extension Mood {
       static var waitingForUploadPredicate: NSPredicate {
           return NSPredicate(format: "%K == NULL", "remoteIdentifier")
       }
   }
   ```
   This predicate identifies moods that haven’t been uploaded to CloudKit yet.

---

#### **Uploading Moods**

When a mood is identified as needing upload, the `processInsertedMoods` method sends the mood to CloudKit.

```swift
extension MoodUploader {
    fileprivate func processInsertedMoods(_ insertions: [Mood], in context: ChangeProcessorContext) {
        context.remote.upload(insertions, completion: context.perform { remoteMoods, error in
            guard !(error?.isPermanent ?? false) else {
                // Permanent error: Mark for local deletion
                insertions.forEach { $0.markForLocalDeletion() }
                self.elementsInProgress.markObjectsAsComplete(insertions)
                return
            }

            // Update local objects with remote identifiers
            for mood in insertions {
                guard let remoteMood = remoteMoods.first(where: { mood.date == $0.date }) else { continue }
                mood.remoteIdentifier = remoteMood.id
                mood.creatorID = remoteMood.creatorID
            }

            context.delayedSaveOrRollback()
            self.elementsInProgress.markObjectsAsComplete(insertions)
        })
    }
}
```

---

#### **Key Steps in Uploading**

1. **Filter Objects**:
   - Identify objects needing upload using the `waitingForUploadPredicate`.

2. **Send to CloudKit**:
   - Use the remote interface to upload objects.

3. **Handle Success**:
   - Update the `remoteIdentifier` field with the identifier returned by CloudKit.

4. **Handle Permanent Errors**:
   - Mark objects for local deletion if they cannot be uploaded.

---

### **Helper Classes**

1. **InProgressTracker**:
   - Tracks objects currently being processed to avoid duplicate uploads.

   ```swift
   final class InProgressTracker<T: NSManagedObject> {
       private var inProgressObjects = Set<NSManagedObjectID>()

       func markObjectAsInProgress(_ object: T) {
           inProgressObjects.insert(object.objectID)
       }

       func markObjectAsComplete(_ object: T) {
           inProgressObjects.remove(object.objectID)
       }
   }
   ```

2. **ChangeProcessorContext**:
   - Provides access to the sync context and remote interface.

   ```swift
   class ChangeProcessorContext {
       let remote: RemoteInterface
       let managedObjectContext: NSManagedObjectContext

       init(remote: RemoteInterface, managedObjectContext: NSManagedObjectContext) {
           self.remote = remote
           self.managedObjectContext = managedObjectContext
       }

       func perform(_ block: @escaping () -> Void) {
           managedObjectContext.perform(block)
       }

       func delayedSaveOrRollback() {
           // Save changes or rollback in case of failure
       }
   }
   ```

---

### **Summary**

Change processors are specialized components in the sync architecture that handle specific tasks:

1. **Responsibilities**:
   - **MoodUploader**: Uploads local moods to CloudKit.
   - **MoodDownloader**: Downloads remote moods to Core Data.
   - **MoodRemover**: Removes moods from CloudKit.

2. **Core Features**:
   - Use **predicates** to identify relevant objects.
   - Track objects in progress using `InProgressTracker`.
   - Leverage `ChangeProcessorContext` for thread safety and network operations.

3. **Benefits**:
   - Simplifies syncing logic by isolating domain-specific responsibilities.
   - Modular design makes the code easy to maintain and extend.

By splitting tasks across change processors, the architecture ensures that each component is focused and testable, contributing to a robust syncing solution.
