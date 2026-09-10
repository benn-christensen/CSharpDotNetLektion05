# Opgaver – Entity Framework Core (introduktion)

Opgaverne tager udgangspunkt i `EntityFrameworkExample`-projektet med `Blog` og `Post`.
Løs dem i rækkefølge – de bygger oven på hinanden.

## Opgave 1 – CRUD i praksis
Byg videre på `Program.cs`:
1. Opret 3 blogs med forskellige URL'er.
2. Hent alle blogs fra databasen og skriv dem ud i konsollen (`foreach`).
3. Opdater én af blogs' URL og gem ændringen.
4. Slet en af de tre blogs igen.

Formål: forstå Create/Read/Update/Delete og at `SaveChangesAsync()` er det, der rent faktisk rammer databasen.

## Opgave 2 – Ny model og relation (1-til-mange)
Tilføj en ny model `Comment` med `CommentId`, `Text`, `PostId` (foreign key) og en navigation property til `Post`. Et `Post` kan have mange `Comments`.
1. Tilføj `DbSet<Comment>` til `BloggingContext`.
2. Lav en migration (`dotnet ef migrations add AddComment`) og opdater databasen (`dotnet ef database update`).
3. Indsæt en blog med et post, som har 2-3 kommentarer, i én omgang (via navigation properties, ikke separate `SaveChanges`-kald).

Formål: forstå hvordan migrations virker, og hvordan EF håndterer relaterede entiteter, der endnu ikke har et id.

## Opgave 3 – Querying med LINQ
Skriv forespørgsler der:
1. Finder alle posts, hvor titlen indeholder et bestemt ord (`Where` + `Contains`).
2. Tæller antal posts pr. blog (`GroupBy`).
3. Henter en blog **inklusive** dens posts og kommentarer i én query (`Include` / `ThenInclude`).
4. Sammenligner resultatet med og uden `Include` – forklar hvorfor `Posts`-listen er tom uden det (mangel på loading — EF fylder ikke navigation properties automatisk).

Formål: forstå eager loading og hvorfor navigation properties ikke fyldes automatisk.

## Opgave 4 – Data validering og constraints
1. Gør `Blog.Url` påkrævet og max 200 tegn ved brug af Data Annotations (`[Required]`, `[MaxLength(200)]`).
2. Tilføj et `CreatedAt`-felt (`DateTime`) på `Post` med en default-værdi sat via Fluent API i `OnModelCreating`.
3. Lav en ny migration og opdater databasen.
4. Prøv at gemme en `Blog` uden URL og observér den exception, EF kaster.

Formål: forstå forskellen på Data Annotations og Fluent API, samt hvordan constraints slår igennem til databasen.

## Opgave 5 – Web API med controllers og EF
Byg et ASP.NET Core Web API-projekt (controller-baseret, ligesom I lærte sidste lektion) der bruger EF Core som datalag, med modellerne `Author` og `Book` (1-til-mange: en forfatter har mange bøger).

Lav en `AuthorsController` og en `BooksController` med følgende endpoints:
1. `GET /api/authors` – liste alle forfattere (inkl. deres bøger).
2. `GET /api/authors/{id}` – hent én forfatter med bøger, eller 404 hvis den ikke findes.
3. `POST /api/authors` – opret en ny forfatter.
4. `POST /api/books` – opret en ny bog og tilknyt den til en eksisterende forfatter (via `AuthorId` i request body).
5. `DELETE /api/books/{id}` – slet en bog.

Krav:
- Brug dependency injection til at få `DbContext` ind i controllerne (registrér den i `Program.cs` med `AddDbContext`).
- Brug DTO'er (egne klasser) til request/response i stedet for at eksponere EF-entiteterne direkte.
- Test endpoints med Scalar/`.http`-fil.

Formål: koble Web API (controllers, DI, DTO'er) sammen med EF Core som datalag – et mønster I kommer til at bruge resten af uddannelsen.
