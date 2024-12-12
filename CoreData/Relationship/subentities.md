### Subentities in Core Data: A Detailed Explanation

#### What Are Subentities in Core Data?

In Core Data, **subentities** allow you to organize entities hierarchically. This means an entity can inherit attributes and relationships from a parent entity. While this is conceptually similar to object-oriented inheritance, the implementation and purpose differ:

- **Entity hierarchy**: Defines data structures in your Core Data model.
- **Class hierarchy**: Defines how you interact with these structures in your Swift code.

---

### Use Case for Subentities

Imagine an app that needs to display **countries** and **continents** in the same table view, both together and separately. Instead of querying separate entities and merging results, subentities offer a clean solution by organizing the model hierarchically:

1. Create an **abstract entity** called `Region`.
2. Make `Continent` and `Country` subentities of `Region`.
3. Move shared attributes (`numericISO3166Code` and `updatedAt`) to the abstract parent entity.

This setup allows a single fetch request for the `Region` entity to return both `Country` and `Continent` instances.

---
 

### How to Implement Subentities

1. **Abstract Parent Entity**: In the Core Data model editor, define an abstract entity (`Region`) that is not directly instantiated.
2. **Subentities**: Set `Continent` and `Country` as subentities of `Region`.
3. **Fetch Request**: Use a fetch request on the `Region` entity to retrieve a mixed result of its subentities.

#### Region Class Example
Define a class for the `Region` entity if needed:

```swift
final class Region: NSManagedObject {}
```

While not mandatory, defining a class allows you to set the result type of fetch requests for `Region` and interact with it programmatically.

<img width="589" alt="Screenshot 2024-12-12 at 8 03 26 PM" src="https://github.com/user-attachments/assets/5b9451c0-d8d6-4d2d-8e3e-53bfe93093ce" />
---

### Independent Class and Entity Hierarchies

It's important to understand that **class hierarchies** and **entity hierarchies** are independent. For instance:

- The `Region` entity is abstract, and its subentities are `Country` and `Continent`.
- However, `Country` and `Continent` classes do not necessarily subclass `Region`.

This separation is reflected in Core Data's flexibility to manage entities and relationships without enforcing strict class inheritance.

---

### The Downside of Subentities

Subentities can cause performance issues due to how Core Data maps them to the underlying SQLite database:

1. **Single Database Table**: All subentities share one table in SQLite. This means all attributes of all subentities are stored in one large table.
2. **Performance Overhead**: Fetching data from this shared table can be slower and require more memory because Core Data retrieves all attributes, even those irrelevant to the specific subentity.

---

### When to Avoid Subentities

Ask yourself this question: **Would it feel wrong to collapse all entities with a common parent into a single entity with a type attribute?**

- If "yes," avoid subentities.
- If "no," subentities might be appropriate.

Instead of using subentities, consider these alternatives:

1. **Protocols**: Define common behavior across classes.
2. **Separate Entities**: Use relationships to link entities without inheritance.

---

### Using Protocols as an Alternative

Instead of relying on subentities, use Swift protocols to define common properties or methods. For example:

#### Protocol Definition
```swift
protocol LocalizedStringConvertible {
    var localizedDescription: String { get }
}
```

#### Conformance by `Country` and `Continent`
```swift
extension Country: LocalizedStringConvertible {
    var localizedDescription: String {
        return iso3166Code.localizedDescription
    }
}

extension Continent: LocalizedStringConvertible {
    var localizedDescription: String {
        return iso3166Code.localizedDescription
    }
}
```

With this protocol, both `Country` and `Continent` objects can be treated similarly for UI purposes, such as displaying their names in a table view.

---

### Summary

- **Subentities** are useful for creating hierarchical models when multiple object types need to appear in the same fetch request or relationship.
- Avoid subentities if they lead to large, unwieldy database tables or if a simpler solution (like a `type` attribute or protocol) is sufficient.
- Use **protocols** in Swift to define shared behavior or properties while keeping entities independent. 

By carefully weighing performance, maintainability, and requirements, you can decide whether subentities or alternative designs suit your app better.
