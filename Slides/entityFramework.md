---
marp: true
theme: default
paginate: true
header: 'Entity Framework Core i .NET'
footer: 'C# & .NET | Lektion 05'

---

# Entity Framework Core i .NET

## Objekt-Relationel Mapping (ORM), DbContext, Migrations, CRUD & Best Practices

---

## Indholdsfortegnelse

1. **Hvad er Entity Framework Core (EF Core)?**
2. **Hvorfor bruge en ORM? (ORM vs. Rå SQL)**
3. **EF Core Arkitektur & Hovedkomponenter**
4. **Setup & NuGet-pakker**
5. **Konfiguration & DI i `Program.cs`**
6. **Modellering af Entiteter (Entities)**
7. **Relational Modellering (1-til-1, 1-til-Mange, Mange-til-Mange)**
8. **Data Annotations vs. Fluent API**
9. **Fluent API i Praksis (`OnModelCreating`)**
10. **EF Core Migrations & CLI Kommandoer**

---

## Indholdsfortegnelse (fortsat)

11. **CRUD Operationer (Create, Read, Update, Delete)**
12. **Asynkrone EF Core Metoder**
13. **Indlæsning af Relaterede Data (`Include`, `ThenInclude`)**
14. **N+1 Problem & Projektion (`Select`)**
15. **Change Tracking & Entity States**
16. **Performance Optimering med `AsNoTracking()`**
17. **Data Seeding**
18. **EF Core i Web APIs (DTOs & Scoped Lifetime)**
19. **Samlet Praktisk Eksempel**
20. **Best Practices & Opsummering**

---

## 1. Hvad er Entity Framework Core?

- **Entity Framework Core (EF Core)** er et moderne, lightweight, extensible og cross-platform **Object-Relational Mapper (ORM)** til .NET.
- **Hvad gør en ORM?**
  - Bygger bro mellem den **objektorienterede verden** i C# (klasser, objekter, lister) og den **relationelle verden** i databasen (tabeller, rækker, fremmednøgler).
- **Hovedfunktioner**:
  - Konverterer LINQ-forespørgsler i C# til optimeret **SQL**.
  - Sporer ændringer på in-memory objekter (**Change Tracking**).
  - Synkroniserer databasedesign med C# klasser via **Migrations**.
  - Understøtter mange databaser: SQL Server, PostgreSQL, SQLite, MySQL, Oracle, InMemory m.fl.

---

<style scoped>section{font-size:22px;}</style>

## 2. Hvorfor bruge en ORM?

### Rå SQL (ADO.NET) vs. EF Core (ORM)

| Egenskab | Rå SQL (ADO.NET / Dapper) | EF Core (ORM) |
| :--- | :--- | :--- |
| **Typesikkerhed** | Ingen i SQL-strenge (fejl opdages ved runtime) | Fuld typesikkerhed i C# ved kompilering |
| **Kode-mængde** | Meget boilerplate (forbindelser, command-objekter, mappers) | Minimal (LINQ & metoder som `Add`, `SaveChanges`) |
| **Database-uafhængighed** | Lav (SQL-syntaks varierer efter DB) | Høj (samme LINQ-kode uanset DB provider) |
| **Performance** | Maksimal kontrol og hastighed | Yderst god (ofte tæt på rå SQL med korrekt brug) |
| **Migrations** | Manuelle SQL-scripts | Automatiske C# migrations |

> **Konklusion**: EF Core øger udviklingshastigheden markant og reducerer risikoen for SQL injection og runtime-fejl.

---

## 3. EF Core Arkitektur & Hovedkomponenter

```
+-------------------------------------------------------------+
|                     C# Applikation                          |
|  (Controllers / Services / LINQ Queries / C# Entiteter)     |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                         DbContext                           |
|   + Change Tracker        + Model Builder (Fluent API)      |
|   + DbSet<Student>        + DbSet<Course>                   |
+------------------------------+------------------------------+
                               |
                               v
+-------------------------------------------------------------+
|                     Database Provider                       |
|   (e.g., Microsoft.EntityFrameworkCore.SqlServer / Sqlite)  |
+------------------------------+------------------------------+
                               | (Genererer SQL)
                               v
+-------------------------------------------------------------+
|                     Relationsdatabase                       |
|                 (SQL Server / SQLite / PostgreSQL)          |
+-------------------------------------------------------------+
```

---

## 4. Setup & NuGet-pakker

For at komme i gang skal følgende NuGet-pakker installeres i dit projekt:

### 1. Database Provider (Vælg den relevante DB)
- `Microsoft.EntityFrameworkCore.SqlServer` (til SQL Server)
- `Microsoft.EntityFrameworkCore.Sqlite` (til SQLite)
- `Npgsql.EntityFrameworkCore.PostgreSQL` (til PostgreSQL)

### 2. Værktøjer (Til Migrations via terminal/CLI)
- `Microsoft.EntityFrameworkCore.Tools` (Package Manager Console i Visual Studio)
- `Microsoft.EntityFrameworkCore.Design` (Nødvendig for `dotnet ef` CLI)

```bash
# Terminal kommandoer til installering:
dotnet add package Microsoft.EntityFrameworkCore.Sqlite
dotnet add package Microsoft.EntityFrameworkCore.Design
```

---

## 5. Konfiguration & DI i `Program.cs`

### Step 1: Definer `DbContext`

```csharp
using Microsoft.EntityFrameworkCore;

public class AppDbContext : DbContext
{
    public AppDbContext(DbContextOptions<AppDbContext> options) : base(options) { }

    public DbSet<Product> Products => Set<Product>();
    public DbSet<Category> Categories => Set<Category>();
}
```

---

### Step 2: Registrer i `Program.cs`

```csharp
var builder = WebApplication.CreateBuilder(args);

// Hent connection string fra appsettings.json
var connectionString = builder.Configuration.GetConnectionString("DefaultConnection") 
                       ?? "Data Source=app.db";

// Registrer AppDbContext som Scoped service i DI containeren
builder.Services.AddDbContext<AppDbContext>(options =>
    options.UseSqlite(connectionString));
```

---

## 6. Modellering af Entiteter (Entities)

En **entitet** er en C# klasse, der repræsenterer en tabel i databasen.

```csharp
public class Product
{
    // Konvention: Egenskaber opkaldt 'Id' eller '<Klassenavn>Id' bliver automatisk Primary Key
    public int Id { get; set; }
    
    public string Name { get; set; } = string.Empty;
    
    public decimal Price { get; set; }
    
    public DateTime CreatedAt { get; set; } = DateTime.UtcNow;
    
    // Foreign Key konvention: CategoryId refererer til Category entitetens Id
    public int CategoryId { get; set; }
    
    // Navigation property (Reference til den relaterede entitet)
    public Category? Category { get; set; }
}
```

---

## 7. Relational Modellering

### 1-til-Mange (One-to-Many) - Standard Mønster

En `Category` har mange `Product`er; et `Product` tilhører én `Category`.

```csharp
public class Category
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Collection Navigation Property
    public List<Product> Products { get; set; } = new();
}

```

---

```csharp
public class Product
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;

    // Foreign Key & Single Navigation Property
    public int CategoryId { get; set; }
    public Category Category { get; set; } = null!;
}
```

---

## 7. Relational Modellering (fortsat)

### Mange-til-Mange (Many-to-Many)

EF Core 5.0+ opretter automatisk en implicit join-tabel i databasen!

```csharp
public class Student
{
    public int Id { get; set; }
    public string Name { get; set; } = string.Empty;
    public List<Course> Courses { get; set; } = new();
}
```

---

```csharp
public class Course
{
    public int Id { get; set; }
    public string Title { get; set; } = string.Empty;
    public List<Student> Students { get; set; } = new();
}
```

> **Note**: Hvis join-tabellen skal indeholde ekstra felter (f.eks. `EnrollmentDate`), bør man oprette en eksplicit join-entitet (`StudentCourse`).

---

## 8. Data Annotations vs. Fluent API

Der er **to måder** at konfigurere entiteter udover EF Core konventioner:

### 1. Data Annotations (Attributter i entitetsklassen)
```csharp
using System.ComponentModel.DataAnnotations;
using System.ComponentModel.DataAnnotations.Schema;

public class Customer
{
    [Key] // Eksplicit Primary Key
    public int Id { get; set; }

    [Required] // NOT NULL i database
    [MaxLength(100)] // NVARCHAR(100) / VARCHAR(100)
    public string Name { get; set; } = string.Empty;

    [Column(TypeName = "decimal(18,2)")]
    public decimal CreditLimit { get; set; }
}
```

---

## 9. Fluent API i Praksis (`OnModelCreating`)

**Fluent API** foretrækkes ofte i større applikationer, da det adskiller domænemodellen fra databaseregler.

```csharp
public class AppDbContext : DbContext
{
    public DbSet<Customer> Customers => Set<Customer>();

```

---

```csharp
    protected override void OnModelCreating(ModelBuilder modelBuilder)
    {
        base.OnModelCreating(modelBuilder);

        modelBuilder.Entity<Customer>(entity =>
        {
            entity.HasKey(c => c.Id);
            entity.Property(c => c.Name)
                  .IsRequired()
                  .HasMaxLength(100);
            
            entity.Property(c => c.CreditLimit)
                  .HasPrecision(18, 2);

            entity.HasIndex(c => c.Name); // Opret index på Name
        });
    }
}
```

---

## 10. EF Core Migrations

**Migrations** sporer ændringer i dine C# entiteter og opdaterer databaseskemaet kontrolleret og versionsstyret.

### Workflow med Migrations

```
1. Ret C# Entiteter/DbContext ──> 2. dotnet ef migrations add <Navn> ──> 3. dotnet ef database update
```

### De 3 Vigtigste CLI Kommandoer:

```bash
# 1. Opret en ny migration ud fra nuværende kode-ændringer
dotnet ef migrations add InitialCreate

# 2. Anvend manglende migrations på databasen (opretter/opdaterer DB)
dotnet ef database update

# 3. Fjern den seneste migration (kun hvis den IKKE er anvendt på DB endnu)
dotnet ef migrations remove
```

---

## 11. CRUD Operationer i EF Core

### Create (Opret)
```csharp
var newProduct = new Product { Name = "Gaming Laptop", Price = 9999.95m, CategoryId = 1 };

context.Products.Add(newProduct);
await context.SaveChangesAsync(); // Persisterer ændring til DB
```

### Read (Læs)
```csharp
// Hent alle produkter
List<Product> products = await context.Products.ToListAsync();

// Hent enkelt produkt via Id
Product? product = await context.Products.FindAsync(id);

// Filtrer med LINQ
Product? cheapProduct = await context.Products
    .FirstOrDefaultAsync(p => p.Price < 100);
```

---

## 11. CRUD Operationer (fortsat)

### Update (Opdater)
EF Core sporer automatisk ændringer på hentede entiteter!

```csharp
var product = await context.Products.FindAsync(1);
if (product != null)
{
    product.Price = 8999.95m; // Ændring spores af Change Tracker
    await context.SaveChangesAsync(); // Genererer UPDATE SQL sætning
}
```

### Delete (Slet)
```csharp
var product = await context.Products.FindAsync(1);
if (product != null)
{
    context.Products.Remove(product);
    await context.SaveChangesAsync(); // Genererer DELETE SQL sætning
}
```

---

<style scoped>section{font-size:22px;}</style>

## 12. Asynkrone EF Core Metoder

Brug **altid** de asynkrone metoder i webapplikationer for at undgå at blokere tråde under I/O-operationer!

| Synkron Metode | Asynkron Metode (Anbefalet) |
| :--- | :--- |
| `context.SaveChanges()` | `await context.SaveChangesAsync()` |
| `query.ToList()` | `await query.ToListAsync()` |
| `query.FirstOrDefault()` | `await query.FirstOrDefaultAsync()` |
| `query.Count()` | `await query.CountAsync()` |
| `query.Any()` | `await query.AnyAsync()` |
| `context.Products.Find(id)` | `await context.Products.FindAsync(id)` |

> **Vigtigt**: Asynkrone LINQ-utvidelsesmetoder som `ToListAsync()` og `FirstOrDefaultAsync()` findes i namespace **`Microsoft.EntityFrameworkCore`**.

---

## 13. Indlæsning af Relaterede Data

Som standard indlæser EF Core **ikke** relaterede entiteter (fx indlæses `Category` ikke automatisk, når du henter et `Product`).

### 1. Eager Loading (`Include` & `ThenInclude`)
Henter relateret data i samme SQL query via `JOIN`.

```csharp
var products = await context.Products
    .Include(p => p.Category) // Henter Product.Category
    .ToListAsync();

var categories = await context.Categories
    .Include(c => c.Products)
        .ThenInclude(p => p.OrderItems) // Henter Products og derefter deres OrderItems
    .ToListAsync();
```

---

## 14. N+1 Problem & Projektion

### N+1 Forespørgsels-fælden (Undgå dette!)
Hvis du henter 100 produkter og derefter i en løkke tilgår `product.Category.Name` uden `Include()`, laves der 1 initial query + 100 SQL queries til databasen!

### Projektion via `Select` (Bedste Performance)
Ved at projicere direkte til DTOs hentes **kun** de nødvendige kolonner fra databasen:

```csharp
var productDtos = await context.Products
    .Where(p => p.Price > 100)
    .Select(p => new ProductDto
    {
        Id = p.Id,
        Name = p.Name,
        CategoryName = p.Category.Name // EF Core laver automatisk den nødvendige JOIN!
    })
    .ToListAsync();
```

---

## 15. Change Tracking & Entity States

EF Cores **Change Tracker** holder øje med alle entiteter hentet via en `DbContext`.

| EntityState | Beskrivelse | Genereret SQL ved `SaveChangesAsync()` |
| :--- | :--- | :--- |
| `Detached` | Entiteten spores ikke af `DbContext` | Ingen |
| `Unchanged` | Entiteten er uændret siden hentning | Ingen |
| `Added` | Entiteten er ny (via `.Add()`) | `INSERT INTO ...` |
| `Modified` | En eller flere egenskaber er ændret | `UPDATE ... SET ...` |
| `Deleted` | Entiteten er markeret til sletning (via `.Remove()`) | `DELETE FROM ...` |

```csharp
var state = context.Entry(product).State; // F.eks. EntityState.Modified
```

---

## 16. Performance Optimering med `AsNoTracking()`

Når du henter data, der **kun skal læses** (f.eks. i et GET endpoint i et API), er Change Tracking spild af RAM og CPU-kraft.

### Slå Change Tracking fra med `AsNoTracking()`:

```csharp
// 100% Read-only forespørgsel - Hurtigere og bruger mindre hukommelse!
var products = await context.Products
    .AsNoTracking()
    .Include(p => p.Category)
    .Where(p => p.Price < 500)
    .ToListAsync();
```

> **Rule of Thumb**:
> - GET endpoints / Read-only forespørgsler -> Brug `AsNoTracking()`
> - Mutationer (Update / Delete) -> Undlad `AsNoTracking()`

---

## 17. Data Seeding

Data Seeding bruges til at indsætte indledende stamdata (f.eks. standard kategorier eller roller) i databasen.

### Seeding i `OnModelCreating` (Fluent API):

```csharp
protected override void OnModelCreating(ModelBuilder modelBuilder)
{
    modelBuilder.Entity<Category>().HasData(
        new Category { Id = 1, Name = "Elektronik" },
        new Category { Id = 2, Name = "Bøger" },
        new Category { Id = 3, Name = "Tøj" }
    );
}
```

> **Note**: Når du anvender `HasData`, skal primærnøglen (`Id`) angives eksplicit. Seeding-data indbygges derefter direkte i din næste **Migration**.

---

## 18. EF Core i Web APIs (DTOs & Lifetime)

### Lifetimes i Dependency Injection
- `DbContext` registreres automatisk som **`Scoped`**.
- Det betyder, at der oprettes én instans af `DbContext` per HTTP request, som automatisk ryddes op (`Dispose`) til sidst.

### Cirkulære Referencer & DTOs
Retur **aldrig** EF Core entiteter direkte fra dine API Controllers!

```csharp
// BAD: Product har reference til Category, og Category har en liste af Products -> JSON Infinite Loop!
public ActionResult<Product> GetProduct(int id) => context.Products.Include(p => p.Category)...;

// GOOD: Map altid til en Data Transfer Object (DTO) klasse eller Record
public async Task<ActionResult<ProductDto>> GetProduct(int id) { ... }
```

---

## 19. Samlet Praktisk Eksempel

### Full CRUD API Controller med EF Core & DTOs

```csharp
[ApiController]
[Route("api/[controller]")]
public class ProductsController : ControllerBase
{
    private readonly AppDbContext _context;

    public ProductsController(AppDbContext context)
    {
        _context = context;
    }

```

---

```csharp
    [HttpGet]
    public async Task<ActionResult<IEnumerable<ProductDto>>> GetProducts()
    {
        return await _context.Products
            .AsNoTracking()
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
            .ToListAsync();
    }
```

---

## 19. Samlet Praktisk Eksempel (fortsat)

```csharp
    [HttpGet("{id:int}")]
    public async Task<ActionResult<ProductDto>> GetProduct(int id)
    {
        var product = await _context.Products
            .AsNoTracking()
            .Where(p => p.Id == id)
            .Select(p => new ProductDto(p.Id, p.Name, p.Price, p.Category.Name))
            .FirstOrDefaultAsync();

        if (product == null) return NotFound();

        return Ok(product);
    }

```

---

```csharp
    [HttpPost]
    public async Task<ActionResult<ProductDto>> CreateProduct(CreateProductDto dto)
    {
        var product = new Product { Name = dto.Name, Price = dto.Price, CategoryId = dto.CategoryId };
        
        _context.Products.Add(product);
        await _context.SaveChangesAsync();

        return CreatedAtAction(nameof(GetProduct), new { id = product.Id }, product);
    }
}
```

---

## 20. Best Practices & Opsummering

1. **Brug altid asynkrone metoder** (`ToListAsync()`, `SaveChangesAsync()`).
2. **Brug `AsNoTracking()`** på alle skrivebeskyttede (read-only) LINQ forespørgsler.
3. **Brug DTOs / Projektioner (`Select`)** til retursvar i APIs i stedet for rå entiteter (undgår N+1 og JSON cirkulære referencer).
4. **Respekter Scoped Lifetime**: Inicer aldrig `DbContext` i en `Singleton` service uden et `IServiceScopeFactory`.
5. **Brug Migrations** til styring og versionskontrol af databaseskemaet.
6. **Fluent API over Data Annotations** ved kompleks modellering for renere domæneklasser.
7. **Hold `DbContext` slank**: Placer forretningslogik i services, ikke direkte i `DbContext`.

---

# Spørgsmål & Diskussion

### Tak for i dag! 🚀
