<img width="561" alt="Screenshot 2024-12-18 at 9 51 50 PM" src="https://github.com/user-attachments/assets/210214cb-02b7-45c6-90f1-10d9d6b9983d" />
<img width="417" alt="Screenshot 2024-12-18 at 9 52 03 PM" src="https://github.com/user-attachments/assets/de0c4f90-bf71-4ab9-8297-3ceebcae347f" />

### **Custom Mapping Models in Core Data**

When lightweight migrations can't handle complex changes to your data model, **custom mapping models** provide a flexible solution. They allow you to specify detailed transformations between old and new models, making it possible to:

- Combine entities.
- Split entities.
- Create new relationships.
- Perform complex attribute transformations.

---

### **Creating Custom Mapping Models in Xcode**

1. **Add a Mapping Model**:
   - In Xcode, create a new file and select **Mapping Model**.
   - Specify the **source model** and **destination model**.
   - Xcode will prepopulate the mapping model with default mappings for unchanged entities and attributes.

2. **Entity Mappings**:
   - Each **entity mapping** specifies how a source entity maps to a destination entity.
   - If an entity is added in the new model, its source entity is empty.
   - If an entity is removed, its destination entity is empty or omitted.

3. **Property Mappings**:
   - Define how attributes and relationships in the destination model map to the source model.
   - Use **NSExpression** to specify transformations for each property.

#### **Example 1: Removing an Entity**
Suppose you remove a `Continent` entity and add an `isoContinent` attribute to `Country`:
- In the mapping model, set the value expression for `isoContinent` as:
  ```plaintext
  $source.continent.numericISO3166Code
  ```
  This pulls the `numericISO3166Code` from the source `continent` and maps it to the new `isoContinent` attribute in `Country`.

---

### **Complex Transformations with Custom Expressions**

For complex changes, you can use **functions** in property mappings.

#### **Example 2: Adding Relationships**
If a new one-to-many relationship between `Continent` and `Mood` is introduced:
- For the `continent` relationship on `Mood`:
  ```plaintext
  FUNCTION($manager, "destinationInstancesForEntityMappingNamed:sourceInstances:", "ContinentToContinent", $source.country.continent)
  ```
  This retrieves the destination `Continent` corresponding to the `Mood`’s `continent`.

- For the inverse `moods` relationship on `Continent`:
  ```plaintext
  FUNCTION($manager, "destinationInstancesForEntityMappingNamed:sourceInstances:", "MoodToMood", $source.countries.@distinctUnionOfSets.moods)
  ```
  This uses the `@distinctUnionOfSets` operator to aggregate all moods from the continent’s countries.

---

### **Custom Entity Mapping Policies**

For more control over the migration process, you can use a subclass of `NSEntityMigrationPolicy`. This allows you to define custom logic that goes beyond what the mapping model editor supports.

#### **Steps to Implement a Custom Policy**

1. **Create the Policy**:
   - Subclass `NSEntityMigrationPolicy`.
   - Override relevant methods to implement custom behavior (e.g., `createDestinationInstances(forSource:in:manager:)`).

2. **Specify the Policy**:
   - In the mapping model editor, assign the custom policy class to the appropriate entity mapping.

#### **Example 3: Splitting an Entity**
Suppose you split the `Country` entity into `Country` and `Continent`:
- Create a `Country5ToCountry6Policy` class:
  ```swift
  final class Country5ToCountry6Policy: NSEntityMigrationPolicy {
      override func createDestinationInstances(
          forSource sInstance: NSManagedObject,
          in mapping: NSEntityMapping,
          manager: NSMigrationManager
      ) throws {
          try super.createDestinationInstances(forSource: sInstance, in: mapping, manager: manager)

          guard let continentCode = sInstance.value(forKey: "isoContinent") as? Int else { return }
          guard let country = manager.destinationInstances(
              forEntityMappingName: mapping.name,
              sourceInstances: [sInstance]
          ).first else { fatalError("Country not created") }

          guard let context = country.managedObjectContext else { fatalError("Context is missing") }

          let continent = context.findOrCreateContinent(withISOCode: continentCode)
          country.setValue(continent, forKey: "continent")
      }
  }
  ```

3. **Logic Breakdown**:
   - **Call Superclass Logic**: Start by letting Core Data handle standard migrations.
   - **Custom Transformation**:
     - Retrieve the destination `Country` object.
     - Use the `isoContinent` attribute to find or create a new `Continent`.
     - Establish the relationship between `Country` and `Continent`.

4. **File-Private Extensions**:
   - Use file-private extensions on `NSManagedObject` to encapsulate key-value coding for better safety and readability.

---

### **Custom Mapping Models Workflow**

1. **Define the Mapping**:
   - Use Xcode to prepopulate mappings.
   - Add custom expressions or assign a custom migration policy.

2. **Run Migration**:
   - Core Data uses the mapping model to transform data from the source store to the destination store.

3. **Validate the Result**:
   - Ensure all data is correctly transformed and relationships are properly established.

---

### **Advantages of Custom Mapping Models**

1. **Flexibility**: Handle complex changes like splitting entities or creating new relationships.
2. **Granular Control**: Use custom policies for advanced transformations.
3. **Reusability**: Mapping models and custom policies can be reused across migrations.

---

### **Limitations**

1. **Time-Consuming**: Requires more effort than lightweight migrations.
2. **Complexity**: Managing custom logic can lead to maintenance challenges.
3. **Performance**: Complex migrations may increase migration time for large datasets.

---

### **When to Use Custom Mapping Models**

- When lightweight migrations cannot express the required changes (e.g., splitting or merging entities).
- For domain-specific transformations involving complex relationships or attribute derivations.
- When introducing new entities or refactoring existing ones significantly.

---

### **Conclusion**

Custom mapping models in Core Data empower developers to manage even the most intricate data model changes. By combining mapping model expressions and `NSEntityMigrationPolicy`, you can create robust migrations tailored to your app’s specific requirements. This approach ensures a smooth user experience, even with substantial schema changes.
