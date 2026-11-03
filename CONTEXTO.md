# Contexto del proyecto: migración de APIQuarkus a .NET 8

Registro de lo trabajado y de las decisiones tomadas. Se actualiza al cerrar cada fase.

## Objetivo

Reescribir en **ASP.NET Core 8 (C# 12)** la API de Proyectos y Comentarios hecha originalmente en Quarkus,
para practicar el stack .NET: Web API con controladores, Entity Framework Core sobre SQL Server,
inyección de dependencias, validación, tarea programada en segundo plano y pruebas de integración con xUnit.

- Código original (Quarkus 3.10, Java, MySQL): https://github.com/jairocodes/APIQuarkus
- Este repositorio (.NET 8): https://github.com/jairocodes/ApiNet8

## Entorno y decisiones

| Tema | Decisión |
|---|---|
| IDE | **VS Code** en lugar de Visual Studio, con las extensiones C# Dev Kit y REST Client (Thunder Client se eliminó) |
| SDK | .NET 8, mínimo 8.0.424, fijado con [global.json](global.json) (`rollForward: latestFeature`). Instalado: 8.0.427 |
| Base de datos | **SQL Server 2022 en Docker** (el original usaba MySQL) |
| Secretos | Cadena de conexión en `dotnet user-secrets` en desarrollo y en variables de entorno en producción. Nunca en el repositorio. |
| Ubicación | La solución vive en la raíz del repositorio |
| Forma de trabajo | Desarrollo por fases: cada paso se documenta y se revisa antes de empezar la siguiente fase |
| Autoría | Exclusiva de Jairo Castillo. El hook [.githooks/commit-msg](.githooks/commit-msg) rechaza trailers de coautoría y autores no autorizados |
| Soporte de .NET 8 | Termina el 10-nov-2026. Se mantiene porque la vacante objetivo lo pide; la migración a .NET 10 queda como tema para la entrevista. |

## Estructura actual

```
Proyectos.sln
global.json
.vscode/            Extensiones recomendadas, configuración, depuración y tareas de VS Code
.githooks/          Hooks de git versionados (control de autoría)
Proyectos.Api/      Web API (plantilla webapi --use-controllers, net8.0)
Proyectos.Tests/    xUnit, con referencia a Proyectos.Api
```

## Preparar el equipo (VS Code + .NET 8)

1. **SDK de .NET 8**: `winget install Microsoft.DotNet.SDK.8`. Verificar con `dotnet --list-sdks` (debe aparecer 8.0.4xx).
2. **Extensiones de VS Code**: al abrir la carpeta, VS Code ofrece instalar las recomendadas de [.vscode/extensions.json](.vscode/extensions.json):
   - `ms-dotnettools.csdevkit` (C# Dev Kit: explorador de soluciones, pruebas y depuración; instala también `ms-dotnettools.csharp`)
   - `humao.rest-client` (ejecuta los archivos `.http`)
   - `ms-azuretools.vscode-docker` (contenedor de SQL Server)
3. **Docker Desktop** abierto para SQL Server 2022 (a partir de la Fase 1).
4. **Hooks de git**, una sola vez después de clonar: `git config core.hooksPath .githooks`.

Estructura objetivo de `Proyectos.Api`: `Controllers/`, `Data/`, `Domain/`, `Dtos/`, `Services/`, `Background/`.

## Estado por fase

| Fase | Contenido | Estado |
|---|---|---|
| 0 | Entorno, solución, proyectos de API y pruebas | **Completa**. `dotnet build` sin errores ni advertencias, `dotnet test` 1/1, Swagger responde en `http://localhost:5025/swagger` |
| 1 | Dominio y EF Core, SQL Server en Docker, migraciones | Pendiente de aprobación |
| 2 | Controladores, DTOs, servicios, manejo de errores | Pendiente |
| 3 | Reporte diario en `BackgroundService` y correo | Pendiente |
| 4 | Pruebas de integración (WebApplicationFactory) | Pendiente |
| 5 | Depuración, archivos `.http`, logs de SQL, publicación | Pendiente |

### Propuesta para la Fase 1 (pendiente de aprobación)

1. Migrar el **dominio real** del repositorio Quarkus (no las entidades simplificadas de la guía de estudio), con `required`, tipos anulables y relaciones configuradas en el `DbContext`.
2. Instalar `Microsoft.EntityFrameworkCore.SqlServer` y `Microsoft.EntityFrameworkCore.Design` (8.0.*), más la herramienta `dotnet-ef`.
3. Levantar SQL Server 2022 en Docker (Docker Desktop debe estar abierto), guardar la cadena de conexión en user-secrets y aplicar la migración `Inicial`.
4. Eliminar `WeatherForecast.cs` y `WeatherForecastController.cs` de la plantilla.

## Lo que hay que migrar (análisis de APIQuarkus)

### Entidades

| Entidad Java | Tabla | Campos (columna) |
|---|---|---|
| `Admin` | `administradores` | `admin_id` (PK), `nombre`, `email` → `correo_electronico` |
| `Project` | `proyectos` | `proyecto_id` (PK), `urlRepo` → `url_repositorio`, `urlImage` → `url_imagen`, `description` → `descripcion`, `nameProject` → `nombre`, `fecha` → `fecha_proyecto`, `admin` → FK `admin_id` |
| `Comentario` | `comentarios` | `comentario_id` (PK), `comentario` (`@NotBlank`), `fecha` → `fecha_comentario` (por defecto: ahora), `nombreUser` → `nombre_usuario` (`@NotBlank`), `emailUser` → `email_usuario` (`@NotBlank @Email`), `project` → FK `id_project` (`@NotNull`) |

### Endpoints

| Método y ruta | Comportamiento original |
|---|---|
| `GET /Project` | Lista todos los proyectos |
| `GET /Project/{id}` | Un proyecto (si no existe devuelve vacío, no 404) |
| `GET /Comentario/{id}` | Un comentario |
| `GET /Comentario?page=N` | Paginado de 10, responde `{ data, page, totalPages }` |
| `POST /Comentario/save` | Valida, guarda, envía correo de agradecimiento al autor y agrega una línea al CSV diario `<fecha>_comentarios.csv`. Responde texto `"Comentario guardado"` |

### Procesos y configuración

- **Tarea programada** `@Scheduled(cron = "0 0 22 * * ?")`: cada día a las 22:00 envía al administrador el CSV del día como adjunto, si existe.
- **Manejo de errores** (`ApiExceptionMapper`): `PersistenceException` → 400, `UncheckedIOException` → 500 "No se pudo generar el reporte", resto → 500. Responde `{ "message": "..." }`.
- **Configuración por variables de entorno:** `DB_*`, `MAIL_HOST/PORT/USERNAME/PASSWORD/FROM` (SMTP de Resend, puerto 465 SSL), `ADMIN_MAIL_RECIPIENT`, `REPORTS_OUTPUT_DIR`, `HTTP_PORT` (8081).

### Mejoras detectadas para aplicar en la migración

- Las entidades se exponen directamente en la API: usar DTOs de entrada y salida.
- Un `id` inexistente no devuelve 404.
- El CSV no escapa comas ni comillas en el texto del comentario.
- El correo se envía de forma bloqueante dentro de la petición, y un fallo de SMTP rompe la respuesta aunque el comentario ya se haya guardado.
- Los servicios usan las clases concretas en lugar de sus interfaces (`ComentarioService` en vez de `IComentarioService`).
- `POST` debería responder 201 con el recurso creado, no un texto plano.

## Equivalencias Quarkus → ASP.NET Core

| Quarkus | ASP.NET Core |
|---|---|
| `@Path`, `@GET`, `@POST` | `[Route]`, `[HttpGet]`, `[HttpPost]` |
| `@Inject` | Inyección por constructor (constructores primarios de C# 12) |
| `@ApplicationScoped` | `AddScoped` / `AddSingleton` según el caso |
| Hibernate / Panache | Entity Framework Core |
| `application.properties` | `appsettings.json` + user-secrets + variables de entorno |
| `@ConfigProperty` | `IOptions<T>` |
| Bean Validation | DataAnnotations + `[ApiController]` (400 automático) |
| `ExceptionMapper` | `IExceptionHandler` + `ProblemDetails` |
| `@Scheduled` | `BackgroundService` |
| `quarkus-mailer` | MailKit |
| `quarkus dev` | `dotnet watch` |
| JUnit + RestAssured | xUnit + `WebApplicationFactory` |

## Comandos útiles

```powershell
dotnet build                                     # compilar la solución
dotnet test                                      # ejecutar las pruebas
dotnet run --project Proyectos.Api --launch-profile http   # API en http://localhost:5025
dotnet watch --project Proyectos.Api             # recarga en caliente
```

En VS Code: **F5** para depurar (C# Dev Kit usa `launchSettings.json`) y la pestaña *Testing* para el explorador de pruebas.
