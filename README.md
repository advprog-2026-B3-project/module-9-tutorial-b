# Maira Azma Shaliha (2406408086) - CRUD Kebun Sawit

## Component Diagram (C4 Level 3)
![alt text](palmerycomponentdiagram.png)

## Code Diagrams (C4 Level 4)

### Class Diagram: PlantationServiceImpl
![alt text](PlantationServiceImpl_codediagram.png)

### Class Diagram: Plantation Entity & DTOs
![alt text](PlantationModelandDTO_codediagram.png)

### Sequence Diagram: Create Plantation
![alt text](createsequencediagram.png)

### Sequence Diagram: Delete Plantation
![alt text](deletesequencediagram.png)

---

### Module Integration with Other Modules

The CRUD Kebun Sawit module supports the broader Palmery system by:
- Providing plantation existence validation that the Harvest module depends on (harvests can only happen in valid plantations).
- Providing plantation and assignment data that the Delivery module depends on for scheduling.
- Enforcing data integrity (unique codes, no overlapping areas, no deletion of active plantations) that protects downstream modules from operating on invalid state.
