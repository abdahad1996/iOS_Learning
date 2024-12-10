Here’s a detailed breakdown of how to add and work with new entities in Core Data, following the given example:

---

### **1. Handling Data Model Changes During Development**
- **Key Insight**:
  - If you modify the Core Data model, the existing app version crashes due to schema mismatch between the model and the persistent store.
- **Development Convenience**:
  - During development, simply delete the app from the simulator or device and relaunch it to apply the updated schema.
- **Future Consideration**:
  - For production apps with existing installations, this approach isn’t viable. Instead, you’ll need to implement **Core Data migrations**, which is discussed in a separate chapter.

---

### **2. Adding New Entities: Country and Continent**

#### **Steps in Xcode’s Model Editor**:
1. Add two entities: **Country** and **Continent**.
2. Add the following attributes to each:
   - `numericISO3166Code` (type: `Int16`) to store the ISO 3166 code.
   - `updatedAt` (type: `Date`) to track when the entity was last updated.

#### **Managed Object Subclass for Country**
```swift
final class Country: NSManagedObject {
    @NSManaged var updatedAt: Date

    fileprivate(set) var iso3166Code: ISO3166.Country {
        get {
            guard let c = ISO3166.Country(rawValue: numericISO3166Code) else {
                fatalError("Unknown country code")
            }
            return c
        }
        set {
            numericISO3166Code = newValue.rawValue
        }
    }

    @NSManaged fileprivate var numericISO3166Code: Int16
}
```

- **Key Features**:
  - `numericISO3166Code` is marked as `fileprivate` because it’s only an internal implementation detail for persistence.
  - The public computed property `iso3166Code` uses the `ISO3166.Country` enum, providing a type-safe and readable interface.

#### **ISO 3166 Enum for Country Codes**
```swift
struct ISO3166 {
    enum Country: Int16 {
        case guy = 328 // Guyana
        case pol = 616 // Poland
        case ltu = 440 // Lithuania
        case nic = 558 // Nicaragua
        // Additional countries...
    }
}
```

- **Extensions**:
  - Add custom functionality like converting codes to strings, finding associated continents, or printing descriptions. These details are left as an exercise.

#### **Managed Object Subclass for Continent**
```swift
final class Continent: NSManagedObject {
    @NSManaged var updatedAt: Date

    fileprivate(set) var iso3166Code: ISO3166.Continent {
        get {
            guard let c = ISO3166.Continent(rawValue: numericISO3166Code) else {
                fatalError("Unknown continent code")
            }
            return c
        }
        set {
            numericISO3166Code = newValue.rawValue
        }
    }

    @NSManaged fileprivate var numericISO3166Code: Int16
}
```

- **Similarities**:
  - Mirrors the `Country` class, but uses `ISO3166.Continent` enum for continent codes.

---

### **3. Conforming Country and Continent to the Managed Protocol**

#### **Managed Protocol**:
This protocol centralizes metadata like `entityName` and default sort descriptors, improving maintainability.

#### **Country Conformance**:
```swift
extension Country: Managed {
    static var defaultSortDescriptors: [NSSortDescriptor] {
        return [NSSortDescriptor(key: #keyPath(updatedAt), ascending: false)]
    }
}
```

#### **Continent Conformance**:
```swift
extension Continent: Managed {
    static var defaultSortDescriptors: [NSSortDescriptor] {
        return [NSSortDescriptor(key: #keyPath(updatedAt), ascending: false)]
    }
}
```

- **Sort Descriptors**:
  - Sort `Country` and `Continent` objects by the `updatedAt` property in descending order.

---

### **4. Associating Moods with Countries**
To associate a `Mood` with a `Country`:
- Add a relationship between the `Mood` and `Country` entities in the data model.
- Define the inverse relationship for better integrity.

---

### **5. Storing Geolocation in the Mood Entity**
To store location data for moods:
1. Add two attributes to the `Mood` entity:
   - `latitude` (type: `Double`, optional)
   - `longitude` (type: `Double`, optional)

2. **Managed Object Subclass**:
   ```swift
   final class Mood: NSManagedObject {
       public var location: CLLocation? {
           guard let lat = latitude, let lon = longitude else { return nil }
           return CLLocation(latitude: lat.doubleValue, longitude: lon.doubleValue)
       }

       @NSManaged fileprivate var latitude: NSNumber?
       @NSManaged fileprivate var longitude: NSNumber?
   }
   ```

   - **Why Use `NSNumber?` Instead of `Double?`**:
     - Core Data requires Objective-C-compatible types for `@NSManaged` properties.
     - `NSNumber` provides compatibility while allowing optionality.

3. **Efficiency**:
   - Storing raw latitude and longitude as doubles is more efficient than storing a `CLLocation` object, which contains unnecessary data.

---

### **6. Benefits of This Approach**

#### **Type Safety**:
- Enums like `ISO3166.Country` ensure type-safe access to country codes, reducing errors.
  
#### **Encapsulation**:
- Implementation details like `numericISO3166Code` are hidden from public use, improving encapsulation.

#### **Protocol Conformance**:
- By conforming to the `Managed` protocol, new entities automatically benefit from utility methods like default fetch requests and insertions.

#### **Scalability**:
- The `Mood` class’s `location` property allows easy integration with location services or mapping features without modifying the database schema.

---

### **7. Considerations for Future Changes**
- **Migrations**:
  - Once the app is in production, adding or modifying entities requires Core Data migrations to avoid crashes.
  
- **Validation**:
  - Consider adding validation logic for geolocation data to ensure valid coordinates.

---

### **Conclusion**
This chapter demonstrates how to expand a Core Data model effectively while maintaining clean, type-safe, and efficient code. By using features like enums for type safety, protocols for reusability, and encapsulation for clean interfaces, you build a robust foundation for managing complex data relationships and attributes in your app.
