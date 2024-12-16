
### **Optimizing Fetch Requests in Core Data**

When working with Core Data, performance optimization is crucial, especially when dealing with large datasets. Below is a detailed explanation of key performance considerations with examples.

---

### **Performance Costs in Fetch Requests**
There are two primary performance costs associated with fetch requests:
1. **Looking Up Data in SQLite Database:**  
   Fetching data involves querying the underlying SQLite database. This process can be resource-intensive, especially for large datasets or complex queries.
2. **Moving Data into Memory:**  
   Once data is fetched, it is loaded into the row cache and subsequently into the managed object context. This transition can be expensive if a lot of objects are materialized.

