### **Summary**

Core Data provides great flexibility in configuring its stack to meet your requirements. However, not all possible setups are equally practical. Below are the most common and recommended setups discussed in this chapter:

1. **Single (Main Queue) Context**:  
   - Use this if you don’t have to perform tasks that could block the UI.

2. **Separate Main Queue and Private Queue Contexts with One Coordinator**:  
   - Ideal for most background work, such as syncing with a web service.  
   - This is the default setup provided by `NSPersistentContainer`.

3. **Main Queue and Private Queue Contexts with Separate Coordinators**:  
   - A viable option for heavy background work prior to iOS 10/macOS 10.12.

4. **Main Queue Context as a Child of a Private Queue Context**:  
   - Suitable for deferred background saving of large changes originating from the UI.

5. **Main Queue Contexts as Children of a Master Main Queue Context**:  
   - Best used as discardable scratchpads for temporary edits.

---

### **Key Takeaways**

1. **Use the Simplest Setup Possible**:  
   - Avoid unnecessary complexity unless your use case requires it.

2. **Always Dispatch onto a Context’s Queue**:  
   - Before accessing a context or its managed objects, ensure you’re on the correct queue.

3. **Separate Work on Different Contexts**:  
   - Isolate tasks on different contexts and use `context-did-save` notifications as the only point of contact.

4. **Pass Only Object IDs Between Contexts**:  
   - If passing objects between contexts, use only their `objectID` property to avoid breaking Core Data's concurrency rules.

5. **Be Cautious with Nested Contexts**:  
   - Nested contexts add significant complexity. Use them only if they clearly solve your problem better than simpler setups.
