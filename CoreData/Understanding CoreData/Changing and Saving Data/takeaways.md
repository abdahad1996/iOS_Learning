### **Summary**

Core Data keeps track of changes at both the **managed object level** and the **context level**, sending `objects-did-change` notifications when `processPendingChanges` is called. These notifications can be used to respond to data changes, similar to how fetched results controllers work. When saving changes:
1. Core Data validates the data against your rules.
2. It detects conflicts using a two-step **optimistic locking** approach.
3. If successful, Core Data posts a `context-did-save` notification.

**Batch updates** and **batch deletes** bypass Core Data’s usual mechanisms and operate directly on the persistent store. While efficient for large-scale updates (e.g., setting a “read” flag for thousands of objects), they require additional effort to ensure consistency in the **managed object context** and **row cache**.

---

### **Takeaways**

- **Change Tracking**:  
  Core Data tracks changes to managed objects between calls to `processPendingChanges` and between calls to `save`.

- **Notifications**:  
  `processPendingChanges` updates relationships and posts the `objects-did-change` notification. This can be paired with the managed object’s `changedValuesForCurrentEvent` API to identify specific changes.

- **Tracking Changes**:  
  Managed objects and contexts provide properties like:
  - `hasChanges`: Indicates if there are unsaved changes.
  - `changedValues`: Shows which properties have changed.
  - `insertedObjects`: Lists newly created objects.  
  These help track what has changed since the data was last persisted.

- **Validation Rules**:  
  Validation rules at the **property** and **object** levels run before saving. The save operation fails if any validation rule fails.

- **Conflict Resolution**:  
  Saving can fail due to conflicts between the managed object context and the persistent store. These conflicts can be resolved using **merge policies**, which are particularly important in setups involving multiple contexts.

- **Batch Operations**:  
  Batch updates and deletes are powerful tools but work differently from other Core Data APIs:
  - They bypass the managed object context and operate directly on the persistent store.
  - Additional steps (e.g., `mergeChanges(fromRemoteContextSave:into:)`) are needed to synchronize in-memory data with the persistent store. 

Let me know if you need further refinement!
