### **Detailed Explanation of `NSFetchedResultsController`**

The `NSFetchedResultsController` is a powerful Core Data class that bridges the gap between the **model layer** (Core Data) and the **view layer** (e.g., `UITableView` or `UICollectionView`). It simplifies displaying Core Data objects by managing:
1. **Data retrieval** with fetch requests.
2. **Live updates** for changes in the underlying data.

Here’s a detailed walkthrough of how it works and how to use it effectively.

---

### **1. What is `NSFetchedResultsController`?**

#### **Purpose**
- Keeps the UI (e.g., a table view) in sync with the Core Data store.
- Monitors changes in the managed object context and notifies its delegate of:
  - Insertions
  - Deletions
  - Updates
  - Moves

#### **Advantages**
1. **Automatic Updates**:
   - Automatically updates the UI when the underlying data changes.
2. **Efficiency**:
   - Only reloads the affected rows instead of reloading the entire table view.
3. **Simplifies Code**:
   - Handles sorting and sectioning directly from the fetch request.

---

### **2. Setting Up a Fetched Results Controller**

#### **Initialization**
Create the fetched results controller using a fetch request:

```swift
fileprivate func setupTableView() {
    let request = Mood.sortedFetchRequest
    request.fetchBatchSize = 20
    request.returnsObjectsAsFaults = false

    let frc = NSFetchedResultsController(
        fetchRequest: request,
        managedObjectContext: managedObjectContext,
        sectionNameKeyPath: nil, // Grouping into sections (optional)
        cacheName: nil           // Caching (optional)
    )

    // Assign to a property
    fetchedResultsController = frc
}
```

#### **Key Parameters**
1. **Fetch Request**:
   - Describes the data to fetch (e.g., `Mood` entities, sorted by `date`).
   - Includes:
     - Sort descriptors.
     - Batch size (`request.fetchBatchSize`).

2. **Managed Object Context**:
   - The context through which the Core Data stack is accessed.

3. **Section Name Key Path**:
   - Used to group fetched objects into sections.
   - E.g., grouping moods by the `date` attribute.

4. **Cache Name**:
   - Optional name for a cache to improve performance.

---

### **3. Delegate Methods**

The `NSFetchedResultsControllerDelegate` informs about changes in the data and updates the UI accordingly. Three essential methods must be implemented:

#### **1. `controllerWillChangeContent(_:)`**
Called before changes are applied to the UI. Typically used to prepare the table view for updates:

```swift
func controllerWillChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.beginUpdates()
}
```

---

#### **2. `controller(_:didChange:at:for:newIndexPath:)`**
Handles individual changes to the data (insertions, deletions, moves, updates):

```swift
func controller(
    _ controller: NSFetchedResultsController<NSFetchRequestResult>,
    didChange anObject: Any,
    at indexPath: IndexPath?,
    for type: NSFetchedResultsChangeType,
    newIndexPath: IndexPath?
) {
    switch type {
    case .insert:
        tableView.insertRows(at: [newIndexPath!], with: .automatic)
    case .delete:
        tableView.deleteRows(at: [indexPath!], with: .automatic)
    case .update:
        tableView.reloadRows(at: [indexPath!], with: .automatic)
    case .move:
        tableView.moveRow(at: indexPath!, to: newIndexPath!)
    @unknown default:
        fatalError("Unhandled case")
    }
}
```

---

#### **3. `controllerDidChangeContent(_:)`**
Called after changes are applied to the UI. Ends table view updates:

```swift
func controllerDidChangeContent(_ controller: NSFetchedResultsController<NSFetchRequestResult>) {
    tableView.endUpdates()
}
```

---

### **4. Encapsulating the Fetched Results Controller**

#### **Reusable Data Source**

To avoid cluttering the view controller with delegate code, encapsulate the fetched results controller logic in a reusable class (`TableViewDataSource`):

```swift
class TableViewDataSource<Delegate: TableViewDataSourceDelegate>: NSObject, UITableViewDataSource, NSFetchedResultsControllerDelegate {
    typealias Object = Delegate.Object
    typealias Cell = Delegate.Cell

    private let tableView: UITableView
    private let cellIdentifier: String
    private let fetchedResultsController: NSFetchedResultsController<Object>
    private let delegate: Delegate

    required init(
        tableView: UITableView,
        cellIdentifier: String,
        fetchedResultsController: NSFetchedResultsController<Object>,
        delegate: Delegate
    ) {
        self.tableView = tableView
        self.cellIdentifier = cellIdentifier
        self.fetchedResultsController = fetchedResultsController
        self.delegate = delegate

        super.init()

        fetchedResultsController.delegate = self
        try! fetchedResultsController.performFetch()
        tableView.dataSource = self
        tableView.reloadData()
    }

    func tableView(_ tableView: UITableView, numberOfRowsInSection section: Int) -> Int {
        return fetchedResultsController.sections?[section].numberOfObjects ?? 0
    }

    func tableView(_ tableView: UITableView, cellForRowAt indexPath: IndexPath) -> UITableViewCell {
        let object = fetchedResultsController.object(at: indexPath)
        guard let cell = tableView.dequeueReusableCell(withIdentifier: cellIdentifier, for: indexPath) as? Cell else {
            fatalError("Unexpected cell type")
        }
        delegate.configure(cell, for: object)
        return cell
    }
}
```

---

#### **Advantages of Encapsulation**
1. **Code Reuse**:
   - `TableViewDataSource` can be reused across multiple table views.
2. **Separation of Concerns**:
   - The fetched results controller logic is isolated from the view controller.
3. **Simplified View Controller**:
   - Focuses on high-level UI logic.

---

### **5. Configuring the Table View**

In the view controller, set up the data source and delegate:

```swift
fileprivate func setupTableView() {
    dataSource = TableViewDataSource(
        tableView: tableView,
        cellIdentifier: "MoodCell",
        fetchedResultsController: fetchedResultsController,
        delegate: self
    )
}
```

#### **Delegate Method for Cell Configuration**
The view controller implements the `TableViewDataSourceDelegate` protocol to configure cells:

```swift
extension MoodsTableViewController: TableViewDataSourceDelegate {
    func configure(_ cell: MoodTableViewCell, for object: Mood) {
        cell.configure(for: object)
    }
}
```

---

### **6. Key Points to Remember**

1. **Efficient Updates**:
   - `NSFetchedResultsController` only updates the rows affected by changes, improving performance.
2. **Batch Fetching**:
   - Use `fetchBatchSize` to limit the number of objects loaded into memory.
3. **Encapsulation**:
   - Delegate methods should be encapsulated in reusable classes for cleaner code.

---

### **7. Final Workflow**
<img width="588" alt="Screenshot 2024-12-03 at 3 47 29 PM" src="https://github.com/user-attachments/assets/4cab6657-f911-427e-90f1-d09510fbe789">

1. **Core Data Stack**:
   - Set up the managed object context and pass it through the view hierarchy.

2. **Fetch Request**:
   - Define the fetch request for the `Mood` entity.

3. **Fetched Results Controller**:
   - Create and configure the fetched results controller.
   - Implement delegate methods to handle data changes.

4. **Data Source**:
   - Use a reusable `TableViewDataSource` class to manage the table view and fetched results controller.

5. **UI Updates**:
   - Automatically reflect Core Data changes in the table view using the fetched results controller.

By following this workflow, your app remains efficient, modular, and easy to maintain as it scales.
