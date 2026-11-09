# RaceDay

Race-event platform: a **REST API** (Part 2, extended with Azure Blob Storage) and an **ASP.NET Core MVC web app** (Part 3) that consumes it, both containerised with Docker.

```
Part2-API/RaceDay.Api    ASP.NET Core Web API  (EF Core + SQL Server, session cookie auth, Azure Blob Storage)
Part2-API/RaceDay.Tests  xUnit integration tests for the API
Part3-Web/RaceDay.Web    ASP.NET Core MVC front end (talks ONLY to the API over HTTP)
docker-compose.yml       API + web + SQL Server + Azurite in one command
```

## Run it with Docker

**Everything, one command** (web app + API + SQL Server + Azure Blob emulator):

```bash
docker compose up --build
```

Then open **http://localhost:8080** (web app) - API/Swagger is on http://localhost:5080/swagger.
The first start takes a minute while SQL Server boots; the API retries until the database is ready.

**Only the MVC container** (API already running on your machine at port 5080):

```bash
docker build -t raceday-web ./Part3-Web && docker run --rm -p 8080:8080 --add-host=host.docker.internal:host-gateway -e Api__BaseUrl=http://host.docker.internal:5080 --name raceday-web raceday-web
```

> If the container can't reach an API that listens on `localhost` only, start the API with `dotnet run --urls http://0.0.0.0:5080`.

## Run it locally (without Docker)

Prerequisites: .NET 8 SDK, SQL Server LocalDB (default connection string), and for banner uploads either
[Azurite](https://learn.microsoft.com/azure/storage/common/storage-use-azurite) (`azurite-blob`) or an Azure Storage account.

1. **API** - `Part2-API\START-API.bat` (or `dotnet run --project Part2-API/RaceDay.Api`) -> http://localhost:5080/swagger
   The database is created on first run; databases from Part 2 are upgraded automatically (new `Event.BannerBlobName` and `Enrolment.Status` columns).
2. **Web** - `START-WEB.bat` (or `dotnet run --project Part3-Web/RaceDay.Web`) -> http://localhost:5090
3. **Tests** - `Part2-API\RUN-TESTS.bat` (or `dotnet test RaceDay.sln`)

### Azure Blob Storage configuration (API only)

| Setting | Meaning |
|---|---|
| `AzureBlobStorage:ConnectionString` | Storage account connection string. Development default is `UseDevelopmentStorage=true` (Azurite). |
| `AzureBlobStorage:ContainerName` | Container for banners, default `event-banners` (created automatically, private). |

For real Azure, keep the secret out of source control: `dotnet user-secrets set "AzureBlobStorage:ConnectionString" "<connection string>" --project Part2-API/RaceDay.Api`
(or the environment variable `AzureBlobStorage__ConnectionString`). The MVC app has no Azure settings - it never talks to Azure.
If storage is not configured the rest of the app works; uploading a banner shows a friendly "storage unavailable" message.

## How the requirements are met

| Requirement | Where |
|---|---|
| MVC consumes the API, no DB access or duplicated business logic | `Infrastructure/RaceDayApi.cs` is the only data access; the csproj has no database/Azure packages |
| Register / log in via the API, session keeps user + role | `AccountController`; the API's session cookie is kept server-side in the MVC session and forwarded on every call |
| Role-specific interfaces and navigation | `_Layout.cshtml` shows different menus for guests, Participants and Organisers |
| Unauthenticated users blocked; logout redirects | `[SessionAuthorize]` filter -> login page (with return URL); role mismatch -> access-denied page; logout clears both sessions |
| Organiser: dashboard, events CRUD, categories, enrolments, results | `OrganiserController` + `Views/Organiser/*` |
| Participant: filterable home, event details, enrol, my enrolments, my results ("47th out of 312 finishers") | `HomeController`, `EventsController`, `EnrolmentsController`, `ResultsController` |
| Azure Blob Storage via the API only | API: `POST/GET/DELETE /api/events/{id}/banner` (`Services/BlobStorage.cs`). MVC uploads through the API and serves images through `/events/{id}/banner` |
| Docker | `Part3-Web/Dockerfile` (multi-stage, non-root), `Part2-API/Dockerfile`, `docker-compose.yml` |
| UI: branded, Bootstrap, colour-coded statuses, graceful errors, consistent | `wwwroot/css/site.css`; statuses use colour **and** icon **and** text (green Confirmed, amber Pending, red Cancelled); validation messages, friendly 400/403/404/500/503 pages, anti-forgery on every POST |

## API additions made for Part 3

| Endpoint | Purpose |
|---|---|
| `GET /api/events/mine` | Organiser dashboard: own events with enrolment/category counts |
| `POST /api/events/{id}/banner` (multipart `file`), `GET`, `DELETE` | Banner image in Azure Blob Storage (JPG/PNG/GIF/WebP, max 5 MB, type checked from the file bytes) |
| `GET /api/events/{id}/enrolments` | All enrolments for an event, with status and result |
| `PUT /api/enrolments/{id}/status` | Organiser sets Pending / Confirmed / Cancelled (new enrolments start as Pending) |
| `PUT /api/enrolments/{id}/result` | Organiser corrects a recorded result |
| `GET /api/users/me/results` | Participant race history incl. `totalFinishers` (finishers with results in the same category) |

`EventDto` gained `hasBanner`; `EnrolmentDto` gained `status`, `eventDate`, `eventLocation`, `hasResult`, `finishTime`, `finishPosition`. Existing endpoints are otherwise unchanged. `docs/RaceDay_Database_Script.sql` has the two new columns.

## Using the app

1. Register an **Organiser** account, create an event (add a banner), then add categories.
2. In another browser/private window register a **Participant**, open the event and enrol -> status **Pending**.
3. As the Organiser open *Enrolments* -> **Confirm**; after the event enter finish time (`hh:mm:ss`) and position under *Results* -> **Publish**.
4. The Participant sees it under *My results*.
## Project Structure
- **Part2-API/RaceDay.Api** — ASP.NET Core Web API with EF Core, SQL Server, session cookie auth, Azure Blob Storage
- **Part2-API/RaceDay.Tests** — xUnit integration tests for the API
- **Part3-Web/RaceDay.Web** — ASP.NET Core MVC frontend — communicates ONLY with the API
- **docker-compose.yml** — orchestrates API + Web + SQL Server + Azurite (Azure emulator)

## Key Features
- Role-based authentication (Organiser / Participant)
- Session management across pages
- Organiser: Dashboard, Events CRUD, Categories, Enrolments, Results Entry
- Participant: Event browsing, Enrolment, My Results, Profile
- Azure Blob Storage integration via API only
- Docker containerisation for full-stack deployment
## Setup & Installation

### Prerequisites
- .NET 8 SDK
- Docker Desktop
- SQL Server (or use the Docker compose setup below)

### Run Locally
1. Clone the repository
2. Update `appsettings.json` connection strings if needed
3. Build and run the API first
4. Build and run the MVC Web application
5. Navigate to `https://localhost:72xx` in your browser

### Run with Docker
```bash
docker compose up --build
## Authentication & Roles
- **Register / Login** — all users authenticate via the API
- **Organiser** — manages events, categories, enrolments, results, uploads banners
- **Participant** — browses events, enrols, views results, manages profile
- **Session** — stores user ID & role after login
- **Logout** — clears session and redirects to login
- **Unauthenticated users** — redirected away from protected pages
## Organiser Interface
- **Dashboard** — summary of events, total enrolments, upcoming dates
- **Events Management** — create/edit/delete events (name, description, date, location, distance, type)
- **Banner Upload** — upload event banner image stored in Azure Blob Storage
- **Categories** — add age and distance categories per event
- **Enrolments** — view participants per event, their category and status
- **Results Entry** — capture finish times and positions after events conclude
## Participant Interface
- **Home Page** — upcoming events with filtering options
- **Event Details** — full info and available categories
- **Enrolment** — select event + category → saved immediately
- **My Enrolments** — personal list with enrolment status
- **My Results** — race history: event name, date, category, finish time, position (e.g. 47th of 312)
- **Profile** — update personal info + upload profile picture
## Azure Blob Storage Integration
- MVC communicates **only** with the API — never directly with Azure
- API handles blob storage operations and returns URLs to MVC
- **Event banners** — uploaded by organisers → stored in Azure → displayed on event pages
- **Profile pictures** — uploaded by participants → stored in Azure → displayed on profile
- Local development uses **Azurite** (Azure Storage Emulator) via Docker
## MVC → API Communication
- All data operations go through the Part 2 REST API
- HTTP client configured in `Program.cs`
- JSON payloads exchanged between MVC and API
- Centralised error handling for API connection issues
- No direct database access from MVC project
## UI Design
- **RaceDay branding** — sporting, energetic visual theme
- **Bootstrap** — responsive layout across devices
- **Role-based navigation** — different menus for Organiser vs Participant
- **Status colours** — Green = Confirmed, Amber = Pending
- **Error handling** — graceful messages, no raw stack traces
- **Consistency** — matching fonts, colours, spacing across all pages
## Docker Configuration
- `Dockerfile` — ASP.NET Core MVC optimised image
- `docker-compose.yml` — orchestrates full stack:
  - SQL Server database
  - Azurite (Azure Blob emulator)
  - RaceDay API
  - RaceDay MVC Web
- Build command: `docker compose up --build`
- Container behaves identically to local development
## CI/CD — GitHub Actions
- Workflow automatically builds both API and MVC projects
- Triggers on push to `main` branch
- Uses .NET SDK setup action
- Restores dependencies → builds solution → reports status
- Green build = all checks passed ✅
## File Structure
## Dependencies
- **.NET 8 SDK** — runtime & framework
- **ASP.NET Core MVC** — web frontend
- **HttpClient** — API communication
- **Bootstrap 5** — responsive UI framework
- **Docker** — containerisation
- **Azure Blob Storage SDK** — used only in API project
## Testing
- API tested via Swagger UI
- Manual end-to-end testing:
  - Register as Organiser → create event → upload banner → verify display
  - Register as Participant → enrol → verify status → view results
- Container tested against local: same behaviour confirmed
## Notes & Limitations
- Azure Blob Storage key is for development only — use secrets in production
- HTTPS ports may differ — check console output
- Results entry restricted to organisers only
- Profile pictures resizing handled by display — not server-side
## Troubleshooting
- **API connection fails** → ensure Part 2 API is running before MVC
- **Docker permission errors** → ensure Docker Desktop is running
- **Blob images not loading** → check Azurite container is healthy
- **Login loops** → verify session cookie config in Program.cs
## Enrolment Status Codes
- ✅ **Confirmed** — green
- ⏳ **Pending** — amber
- ❌ **Cancelled** — red

## HTTP Status Codes
- `200 OK` — request successful
- `201 Created` — new record added
- `400 Bad Request` — invalid input
- `401 Unauthorized` — login required
- `403 Forbidden` — wrong role
- `404 Not Found` — record doesn't exist
## Future Improvements
- Server-side image resizing
- Email notifications on enrolment
- Pagination for event lists
- Advanced filtering by date range
- Admin role for user management
- Unit tests for MVC controllers
## Video Demonstration
Full walkthrough: [YouTube — click here]()
- Covers MVC structure & API integration
- Azure Blob Storage flow
- Docker build & run
- Organiser & Participant demos
## Submission Summary
- ✅ MVC app consuming Part 2 API (no direct DB access)
- ✅ Role-based auth & navigation
- ✅ Azure Blob via API only
- ✅ Dockerfile + docker-compose
- ✅ CI/CD GitHub Actions workflow
- ✅ 20 meaningful commits ✅
- ✅ YouTube demo link included above
