---
created: 2026-09-09
status: pendiente
proyecto: tsj-web
tags:
  - tsj-web
  - fase-1
  - plan
  - backend
---
# TSJ-Web — FASE 1: Mapeo de la Base de Datos

> **Objetivo:** Alinear backend con el esquema de BD, mapear las 5 tablas sin backend, y preparar tabla `users` para Fase 2.
> **Fecha:** 2026-09-09

---

## Decisiones tomadas

| Decisión | Elección |
|----------|----------|
| Naming de tablas | Usar `unidad_academica` (alinear backend con BD) |
| Tablas sin backend | Mapear las 5 tablas completas |
| Migraciones EF Core | No usar — solo scripts SQL para producción |

---

## Hallazgos de la auditoría

### Discrepancia de nombres (CRÍTICO)
- **BD:** tabla `unidad_academica`, FKs `unidad_academica_id`
- **Backend actual:** modelo `Campus`, FKs `CampusId`, tabla mapeada como `campus`
- **Seeds:** ya usan `campus` como nombre de tabla
- **Decisión:** Renombrar todo en el backend a `UnidadAcademica` / `unidad_academica_id`

### Tablas cubiertas (11/16)
| Tabla | Modelo EF | Estado |
|-------|-----------|--------|
| `unidad_academica` | `Campus` → **renombrar a `UnidadAcademica`** | Necesita rename |
| `document_archive` | `DocumentArchive` | OK |
| `department` | `Department` | OK |
| `administrative_staff` | `AdministrativeStaff` | FK `CampusId` → rename a `UnidadAcademicaId` |
| `researchers` | `Researcher` | FK `CampusId` → rename a `UnidadAcademicaId` |
| `study_programs` | `StudyProgram` | OK |
| `curriculum_courses` | `CurriculumCourse` | OK |
| `education_offer_appendices` | `EducationOfferAppendix` | OK |
| `campus_study_programs` | `CampusStudyProgram` | Tabla pivote — rename a `UnidadAcademicaStudyProgram` |
| `study_program_curriculum` | `StudyProgramCurriculum` | OK |
| `study_program_appendices` | `StudyProgramAppendix` | OK |

### Tablas NO cubiertas (5/16) — crear modelos
| Tabla | Descripción |
|-------|-------------|
| `customization_ua` | Banners y FAQs por unidad académica (JSONB) |
| `workshops_ua` | Talleres por unidad académica |
| `staff_workshops_ua` | Personal de talleres |
| `courses_ua` | Cursos en línea (JSONB) |
| `extra_curricular_activities` | Actividades extracurriculares |

### Tabla `users` — crear desde cero
- Script SQL + modelo EF Core (sin migración)

---

## Pasos de implementación

### PASO 1 — Renombrar `Campus` → `UnidadAcademica`

**Archivos a modificar:**

1. **`Data/Models/Campus.cs`** → renombrar archivo y clase a `UnidadAcademica.cs` / `UnidadAcademica`
   - Cambiar nombre de clase
   - Mantener mismas propiedades (id, name, cover_photo, icon_photo, address, phone, email, whatsapp)
   - Renombrar navegaciones: `CampusStudyPrograms` → `UnidadAcademicaStudyPrograms`, `Staff` → `AdministrativeStaff`, `Researchers` → `Researchers`

2. **`Data/Models/CampusStudyProgram.cs`** → renombrar a `UnidadAcademicaStudyProgram.cs`
   - `CampusId` → `UnidadAcademicaId`
   - `Campus` navigation → `UnidadAcademica`

3. **`Data/Models/AdministrativeStaff.cs`**
   - `CampusId` → `UnidadAcademicaId`
   - `Campus` navigation → `UnidadAcademica`

4. **`Data/Models/Researcher.cs`**
   - `CampusId` → `UnidadAcademicaId`
   - `Campus` navigation → `UnidadAcademica`

5. **`Data/AppDbContext.cs`**
   - `DbSet<Campus>` → `DbSet<UnidadAcademica>` + nombre `UnidadAcademicas`
   - `DbSet<CampusStudyProgram>` → `DbSet<UnidadAcademicaStudyProgram>` + nombre `UnidadAcademicaStudyPrograms`
   - Renombrar todos los métodos `ConfigureCampus*` → `ConfigureUnidadAcademica*`
   - Actualizar todos los `entity.ToTable("campus")` → `entity.ToTable("unidad_academica")`
   - Actualizar todos los FKs de `CampusId` → `UnidadAcademicaId`
   - Renombrar índices: `idx_campus_*` → `idx_unidad_academica_*`

6. **`Endpoints/Campus/CampusController.cs`** → renombrar a `UnidadAcademicaController.cs`
   - Clase `CampusController` → `UnidadAcademicaController`
   - Ruta `[Route("campus")]` → `[Route("unidad-academica")]`
   - Tipos de retorno: `CampusDto` → `UnidadAcademicaDto` (si aplica)

7. **`Endpoints/Campus/CampusModule.cs`** → renombrar a `UnidadAcademicaModule.cs`
   - Clase `CampusModule` → `UnidadAcademicaModule`

8. **`Endpoints/Campus/CampusService.cs`** → renombrar a `UnidadAcademicaService.cs`
   - Clase `CampusService` → `UnidadAcademicaService`

9. **`Endpoints/Campus/CampusVerifier.cs`** → renombrar a `UnidadAcademicaVerifier.cs`

10. **`Endpoints/Campus/PublicCampusService.cs`** → renombrar a `PublicUnidadAcademicaService.cs`

11. **`Endpoints/Campus/Dto/`** — renombrar todos los DTOs:
    - `CampusDto.cs` → `UnidadAcademicaDto.cs`
    - `CampusResponseDto.cs` → `UnidadAcademicaResponseDto.cs` (si aplica)
    - Cualquier otro DTO de campus

12. **Seed files** (`infra/seed.sql`, `infra/seed-final.sql`):
    - Cambiar `INSERT INTO campus` → `INSERT INTO unidad_academica`
    - Cambiar `campus_id` → `unidad_academica_id` en FKs
    - Cambiar `campus_id_seq` → `unidad_academica_id_seq`

---

### PASO 2 — Mapear las 5 tablas sin backend

**Crear modelos:**

13. **`Data/Models/CustomizationUa.cs`** (nuevo)
    ```csharp
    public class CustomizationUa
    {
        public int Id { get; set; }
        public JsonElement Headers { get; set; }    // JSONB [{image, cta, text, description}]
        public JsonElement Faqs { get; set; }       // JSONB [{question, response}]
        public int UnidadAcademicaId { get; set; }
        public UnidadAcademica UnidadAcademica { get; set; } = null!;
    }
    ```

14. **`Data/Models/WorkshopUa.cs`** (nuevo)
    ```csharp
    public class WorkshopUa
    {
        public int Id { get; set; }
        public string? Tubnail { get; set; }
        public string? CoverPhoto { get; set; }
        public string? Title { get; set; }
        public string? Description { get; set; }
        public string? ContentMd { get; set; }
        public int UnidadAcademicaId { get; set; }
        public UnidadAcademica UnidadAcademica { get; set; } = null!;
    }
    ```

15. **`Data/Models/StaffWorkshopUa.cs`** (nuevo)
    ```csharp
    public class StaffWorkshopUa
    {
        public int Id { get; set; }
        public string? Portrait { get; set; }
        public string? Name { get; set; }
        public string? Shift { get; set; }
        public string? Expertise { get; set; }
        public int UnidadAcademicaId { get; set; }
        public UnidadAcademica UnidadAcademica { get; set; } = null!;
    }
    ```

16. **`Data/Models/CourseUa.cs`** (nuevo)
    ```csharp
    public class CourseUa
    {
        public int Id { get; set; }
        public string? CoverPhoto { get; set; }
        public string? Title { get; set; }
        public string? DescriptionMd { get; set; }
        public JsonElement? Content { get; set; }     // JSONB
        public string? ContentMd { get; set; }
        public JsonElement? Resources { get; set; }   // JSONB
        public JsonElement? Cta { get; set; }         // JSONB
        public int UnidadAcademicaId { get; set; }
        public UnidadAcademica UnidadAcademica { get; set; } = null!;
    }
    ```

17. **`Data/Models/ExtraCurricularActivity.cs`** (nuevo)
    ```csharp
    public class ExtraCurricularActivity
    {
        public int Id { get; set; }
        public string? Name { get; set; }
        public string? Type { get; set; }  // "course" | "taller"
        public string? CoverPhoto { get; set; }
        public string? Flayer { get; set; }
        public string? Description { get; set; }
        public int UnidadAcademicaId { get; set; }
        public UnidadAcademica UnidadAcademica { get; set; } = null!;
    }
    ```

**Actualizar AppDbContext:**

18. **`Data/AppDbContext.cs`** — agregar DbSets y configuraciones:
    ```csharp
    public DbSet<CustomizationUa> CustomizationUas => Set<CustomizationUa>();
    public DbSet<WorkshopUa> WorkshopsUa => Set<WorkshopUa>();
    public DbSet<StaffWorkshopUa> StaffWorkshopsUa => Set<StaffWorkshopUa>();
    public DbSet<CourseUa> CoursesUa => Set<CourseUa>();
    public DbSet<ExtraCurricularActivity> ExtraCurricularActivities => Set<ExtraCurricularActivity>();
    ```
    + Métodos de configuración para cada tabla

**Actualizar modelo `UnidadAcademica` con navegaciones:**
19. Agregar colecciones de navegación en `UnidadAcademica.cs`:
    ```csharp
    public ICollection<CustomizationUa> Customizations { get; set; } = new List<CustomizationUa>();
    public ICollection<WorkshopUa> Workshops { get; set; } = new List<WorkshopUa>();
    public ICollection<StaffWorkshopUa> StaffWorkshops { get; set; } = new List<StaffWorkshopUa>();
    public ICollection<CourseUa> Courses { get; set; } = new List<CourseUa>();
    public ICollection<ExtraCurricularActivity> ExtraCurricularActivities { get; set; } = new List<ExtraCurricularActivity>();
    ```

---

### PASO 3 — Crear tabla `users` (sin migración EF)

20. **`infra/create-users-table.sql`** (nuevo archivo)
    ```sql
    CREATE TABLE users (
        id            SERIAL PRIMARY KEY,
        google_id     VARCHAR(30) UNIQUE NOT NULL,
        email         VARCHAR(255) NOT NULL,
        display_name  VARCHAR(150),
        avatar_url    VARCHAR(500),
        role          VARCHAR(20) NOT NULL DEFAULT 'Public',
        created_at    TIMESTAMPTZ NOT NULL DEFAULT NOW(),
        updated_at    TIMESTAMPTZ NOT NULL DEFAULT NOW()
    );
    CREATE INDEX idx_users_google_id ON users (google_id);
    CREATE INDEX idx_users_email ON users (email);
    ```

21. **`Data/Models/User.cs`** (nuevo)
    ```csharp
    public class User
    {
        public int Id { get; set; }
        public string GoogleId { get; set; } = string.Empty;
        public string Email { get; set; } = string.Empty;
        public string? DisplayName { get; set; }
        public string? AvatarUrl { get; set; }
        public string Role { get; set; } = "Public";
        public DateTime CreatedAt { get; set; }
        public DateTime UpdatedAt { get; set; }
    }
    ```

22. **`Data/AppDbContext.cs`** — agregar DbSet (sin fluent config por ahora):
    ```csharp
    public DbSet<User> Users => Set<User>();
    ```

---

### PASO 4 — Auditoría de datos

23. Revisar `AGENTS.md` rules vs. esquema actual:
    - Enums como VARCHAR: `modality`, `type` (courses), `type` (documents) — ¿cumple?
    - JSONB en `customization_ua`, `courses_ua` — ¿correcto según regla 3?
    - Tabla `users` con `role` como VARCHAR(20) — ¿debería ser ENUM?

24. Revisar seeds por consistencia:
    - `seed.sql` y `seed-final.sql` usan `campus` como nombre de tabla → cambiar a `unidad_academica`
    - Verificar que los IDs de campus coincidan entre seeds

25. Verificar tipos de datos:
    - `VARCHAR(50)` para `workshops_ua.title` y `workshops_ua.description` — ¿suficiente?
    - `VARCHAR(50)` para `extra_curricular_activities.type` — ¿suficiente para ENUM?

---

### PASO 5 — Validación

26. Verificar que el backend compila sin errores:
    ```bash
    cd backedn/tesj-core && dotnet build
    ```

27. Verificar que los seeds son consistentes con el esquema:
    - Todos los `INSERT INTO campus` → `INSERT INTO unidad_academica`
    - Todos los `campus_id` en FKs → `unidad_academica_id`

28. Verificar que `AppDbContext` refleja las 16 tablas (11 originales + 5 nuevas + users)

---

## Archivos a crear/modificar

| # | Archivo | Acción |
|---|---------|--------|
| 1 | `Data/Models/Campus.cs` → `UnidadAcademica.cs` | Renombrar clase + archivo |
| 2 | `Data/Models/CampusStudyProgram.cs` → `UnidadAcademicaStudyProgram.cs` | Renombrar + actualizar FKs |
| 3 | `Data/Models/AdministrativeStaff.cs` | Actualizar FK y navegación |
| 4 | `Data/Models/Researcher.cs` | Actualizar FK y navegación |
| 5 | `Data/Models/CustomizationUa.cs` | **Crear** |
| 6 | `Data/Models/WorkshopUa.cs` | **Crear** |
| 7 | `Data/Models/StaffWorkshopUa.cs` | **Crear** |
| 8 | `Data/Models/CourseUa.cs` | **Crear** |
| 9 | `Data/Models/ExtraCurricularActivity.cs` | **Crear** |
| 10 | `Data/Models/User.cs` | **Crear** |
| 11 | `Data/AppDbContext.cs` | Actualizar todos los DbSets + configs |
| 12 | `Endpoints/Campus/` → `Endpoints/UnidadAcademica/` | Renombrar directorio + archivos |
| 13 | `infra/create-users-table.sql` | **Crear** |
| 14 | `infra/seed.sql` | Actualizar nombres de tablas |
| 15 | `infra/seed-final.sql` | Actualizar nombres de tablas |

---

## Estimación de esfuerzo

| Paso | Archivos | Complejidad |
|------|----------|-------------|
| 1. Renombrar Campus → UnidadAcademica | ~12 archivos | Alta (muchos archivos, cascade) |
| 2. Mapear 5 tablas nuevas | 7 archivos (5 modelos + DbContext + nav) | Media |
| 3. Crear users | 3 archivos (SQL + modelo + DbContext) | Baja |
| 4. Auditoría de datos | Revisión, sin código | Baja |
| 5. Validación | Build + revisión | Baja |
| **Total** | **~20 archivos** | **Alta** |
