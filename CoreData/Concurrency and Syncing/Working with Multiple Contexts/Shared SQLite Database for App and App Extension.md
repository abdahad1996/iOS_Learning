### **Real-World Example: Shared SQLite Database for App and App Extension**

A **shared SQLite database** between a main app and an app extension is often used in scenarios where both need to access and modify the same data. A common example is a **to-do app** with a widget extension. Both the app and the widget display tasks, and changes made in one must reflect in the other.

---

### **Use Case: To-Do App with a Widget Extension**

1. **Main App**:
   - Allows users to create, edit, and delete tasks.
   - Stores tasks in a SQLite-backed Core Data database.
   - Updates the widget with the latest tasks.

2. **Widget Extension**:
   - Displays a summary of tasks (e.g., tasks due today).
   - Reads tasks directly from the shared SQLite database.
   - Updates automatically when tasks are modified in the app.

---

### **Shared Database Setup**

1. **Enable App Groups**:
   - In Xcode, enable the **App Groups** capability for both the main app and the widget extension.
   - Create a shared container, e.g., `group.com.example.todoapp`.

2. **Shared SQLite Location**:
   - Store the SQLite database in the shared container so both the app and the extension can access it.

**Example: Setting Up the Shared Container URL**
```swift
let sharedContainerURL = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.example.todoapp")!
let storeURL = sharedContainerURL.appendingPathComponent("Tasks.sqlite")
```

---

### **Core Data Stack for Main App and Widget Extension**

#### **Main App Core Data Stack**
The main app initializes the Core Data stack with the shared SQLite database.

```swift
import CoreData

class CoreDataStack {
    static let shared = CoreDataStack()

    private init() {}

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "TasksModel")
        let sharedContainerURL = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.example.todoapp")!
        let storeURL = sharedContainerURL.appendingPathComponent("Tasks.sqlite")
        
        let storeDescription = NSPersistentStoreDescription(url: storeURL)
        container.persistentStoreDescriptions = [storeDescription]
        
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Failed to load persistent store: \(error)")
            }
        }
        
        return container
    }()

    var viewContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
}
```

#### **Widget Extension Core Data Stack**
The widget extension uses the same shared SQLite database.

```swift
import CoreData

class WidgetCoreDataStack {
    static let shared = WidgetCoreDataStack()

    private init() {}

    lazy var persistentContainer: NSPersistentContainer = {
        let container = NSPersistentContainer(name: "TasksModel")
        let sharedContainerURL = FileManager.default.containerURL(forSecurityApplicationGroupIdentifier: "group.com.example.todoapp")!
        let storeURL = sharedContainerURL.appendingPathComponent("Tasks.sqlite")
        
        let storeDescription = NSPersistentStoreDescription(url: storeURL)
        container.persistentStoreDescriptions = [storeDescription]
        
        container.loadPersistentStores { _, error in
            if let error = error {
                fatalError("Failed to load persistent store: \(error)")
            }
        }
        
        return container
    }()

    var viewContext: NSManagedObjectContext {
        return persistentContainer.viewContext
    }
}
```

---

### **Synchronizing Data Between App and Extension**

To ensure that changes made in the app are reflected in the widget (and vice versa), you can use `NSManagedObjectContext`'s change notifications.

#### **Main App: Notify the Widget of Changes**
The app saves changes to Core Data and notifies the widget.

```swift
func saveTask(name: String) {
    let context = CoreDataStack.shared.viewContext
    let task = Task(context: context)
    task.name = name
    task.createdAt = Date()
    
    do {
        try context.save()
        
        // Notify the widget of changes
        WidgetCenter.shared.reloadAllTimelines()
    } catch {
        print("Failed to save task: \(error)")
    }
}
```

#### **Widget: React to Changes**
The widget reloads its timeline when changes are detected.

```swift
import WidgetKit

struct ToDoWidget: Widget {
    var body: some WidgetConfiguration {
        StaticConfiguration(kind: "ToDoWidget", provider: TaskProvider()) { entry in
            ToDoWidgetView(entry: entry)
        }
        .configurationDisplayName("To-Do List")
        .description("Shows your upcoming tasks.")
    }
}

struct TaskProvider: TimelineProvider {
    func placeholder(in context: Context) -> TaskEntry {
        TaskEntry(date: Date(), tasks: [])
    }

    func getSnapshot(in context: Context, completion: @escaping (TaskEntry) -> ()) {
        let tasks = fetchTasks()
        let entry = TaskEntry(date: Date(), tasks: tasks)
        completion(entry)
    }

    func getTimeline(in context: Context, completion: @escaping (Timeline<TaskEntry>) -> ()) {
        let tasks = fetchTasks()
        let entry = TaskEntry(date: Date(), tasks: tasks)
        let timeline = Timeline(entries: [entry], policy: .never)
        completion(timeline)
    }

    private func fetchTasks() -> [Task] {
        let context = WidgetCoreDataStack.shared.viewContext
        let fetchRequest: NSFetchRequest<Task> = Task.fetchRequest()
        
        do {
            return try context.fetch(fetchRequest)
        } catch {
            print("Failed to fetch tasks: \(error)")
            return []
        }
    }
}
```

---

### **Handling Potential Issues**

1. **Concurrency**:
   - SQLite allows one write and multiple reads simultaneously.
   - Ensure that the app and widget do not perform heavy write operations simultaneously to avoid contention.

2. **Changes Not Reflected Immediately**:
   - Use `WidgetCenter.reloadAllTimelines()` to update the widget when changes occur in the app.
   - Observe `NSManagedObjectContextObjectsDidChange` notifications in the app and widget to handle changes dynamically.

---

### **Summary**

- **Scenario**: A to-do app shares a SQLite database with its widget extension to display tasks in both.
- **Setup**:
  - Use **App Groups** to share a SQLite database.
  - Initialize Core Data stacks in the app and extension with the shared database location.
- **Synchronization**:
  - Notify the widget of changes using `WidgetCenter.reloadAllTimelines()`.
  - Use change notifications to dynamically fetch and display updates.

This setup ensures seamless data sharing between the main app and its extension while leveraging Core Data's robust features for persistence.
