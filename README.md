# Maira Azma Shaliha (2406408086) - CRUD Kebun Sawit

## Component Diagram (C4 Level 3)
![alt text](palmerycomponentdiagram.png)

## Code Diagrams (C4 Level 4)

### Class Diagram: PlantationServiceImpl
![alt text](PlantationServiceImpl_codediagram.png)

### Class Diagram: Plantation Entity & DTOs
![alt text](PlantationModelandDTO_codediagram.png)


### Sequence Diagram: Create Plantation

```mermaid

```

### Explanation

The create flow shows the actual implementation path:
1. `JwtAuthenticationFilter` parses the JWT locally using the shared secret (no call to repo-auth at runtime).
2. Bean validation on `PlantationRequestDto` handles coordinate range checks at the controller level.
3. The service first validates code uniqueness via `existsByCode`, then fetches all existing plantations and passes them to `PlantationCoordinateValidator.hasOverlapWithAny()`.
4. If both validations pass, `PlantationMapper` converts the DTO to an entity and the repository persists it.

---

### 2d. Sequence Diagram: Delete Plantation

```mermaid
sequenceDiagram
    participant Admin as Admin Utama
    participant Filter as JwtAuthenticationFilter
    participant Controller as PlantationController
    participant Service as PlantationServiceImpl
    participant Repo as PlantationRepository
    participant DB as PostgreSQL

    Admin->>Controller: DELETE /kebun/{id} (Bearer JWT)
    Controller->>Filter: doFilterInternal()
    Filter->>Filter: Parse JWT, set ROLE_ADMIN
    Filter-->>Controller: Authentication set

    Controller->>Controller: @PreAuthorize("hasRole('ADMIN')") passes
    Controller->>Service: deletePlantation(id)

    Service->>Repo: findById(id)
    Repo->>DB: SELECT FROM plantation WHERE id = ?
    DB-->>Repo: Plantation entity
    Repo-->>Service: Plantation found

    Service->>Service: ensureNoActivePersonnel(plantation)

    alt Plantation has active Mandor
        Service->>Service: throw PlantationHasActivePersonnelException
        Service-->>Controller: Exception thrown
        Controller-->>Admin: HTTP 400 Bad Request\n"Kebun masih memiliki mandor aktif\ndan tidak dapat dihapus"
    else No active personnel
        Service->>Repo: delete(plantation)
        Repo->>DB: DELETE FROM plantation WHERE id = ?
        DB-->>Repo: Deleted
        Repo-->>Service: void
        Service-->>Controller: void
        Controller-->>Admin: HTTP 204 No Content
    end
```

### Explanation

The delete flow enforces the business rule that a plantation cannot be deleted if it still has an active Mandor assigned. The `ensureNoActivePersonnel()` method in `PlantationServiceImpl` performs this check. If the guard fails, a `PlantationHasActivePersonnelException` is thrown and handled by the `GlobalExceptionHandler`. On success, the controller returns HTTP 204 No Content (matching the actual `ResponseEntity.noContent().build()` in the code).

---

## Design Decision Notes

### Separation of Concerns

The module follows a clean layered architecture where each component has a single responsibility:
- **PlantationController** handles only HTTP concerns (routing, status codes, `@PreAuthorize` role checks).
- **PlantationServiceImpl** orchestrates business logic without knowing about HTTP or persistence details.
- **PlantationCoordinateValidator** is a dedicated `@Component` that encapsulates polygon overlap detection using `java.awt.geom.Path2D`.
- **PlantationMapper** is a dedicated `@Component` that handles all entity-to-DTO and DTO-to-entity conversions.
- **PlantationRepository** handles data access with custom JPQL for filtered search.

### Why Coordinate Validation is Handled at Two Levels

**Bean validation on PlantationRequestDto** handles format-level checks:
- `@NotNull` ensures all coordinate fields are present.
- `@DecimalMin` / `@DecimalMax` ensures latitude is within [-90, 90] and longitude within [-180, 180].
- This runs automatically at the controller layer before reaching the service.

**PlantationCoordinateValidator** handles domain-level validation:
- Builds `Path2D` polygons from four corner coordinates.
- Checks vertex containment between the incoming polygon and all existing plantations.
- This requires database access (fetching existing plantations) and geometric computation.

Separating these two levels means format errors are caught early (fast, no DB call), while overlap detection only runs when the input is already structurally valid.

### Module Integration with Other Modules

The CRUD Kebun Sawit module supports the broader Palmery system by:
- Providing plantation existence validation that the Harvest module depends on (harvests can only happen in valid plantations).
- Providing plantation and assignment data that the Delivery module depends on for scheduling.
- Enforcing data integrity (unique codes, no overlapping areas, no deletion of active plantations) that protects downstream modules from operating on invalid state.
