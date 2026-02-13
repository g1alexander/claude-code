# Platziflix - Proyecto Multi-plataforma

## Arquitectura del Sistema

Platziflix es una plataforma de cursos online con arquitectura multi-plataforma. El **Backend es la única fuente de verdad** y los 3 clientes (Web, Android, iOS) consumen la misma API REST de forma independiente.

```
┌──────────────┐   ┌──────────────────┐   ┌────────────────┐
│  Frontend     │   │  Android App     │   │  iOS App       │
│  Next.js 15   │   │  Kotlin/Compose  │   │  Swift/SwiftUI │
│  Port: 3000   │   │  MVVM + MVI      │   │  MVVM + Clean  │
└──────┬───────┘   └────────┬─────────┘   └───────┬────────┘
       │ fetch              │ Retrofit             │ URLSession
       └───────────────────►┼◄─────────────────────┘
                            ▼
              ┌──────────────────────────┐
              │  FastAPI (Port 8000)     │
              │  Service Layer Pattern   │
              │  SQLAlchemy 2.0 ORM      │
              └────────────┬─────────────┘
                           ▼
              ┌──────────────────────────┐
              │  PostgreSQL 15 (Port 5432)│
              │  Alembic migrations      │
              └──────────────────────────┘
              🐳 Docker Compose
```

## Estructura del Proyecto

```
claude-code/
├── CLAUDE.md
├── Backend/                          # API FastAPI + PostgreSQL
│   ├── app/
│   │   ├── main.py                   # FastAPI app, routes, DI
│   │   ├── core/config.py            # Pydantic Settings (env vars)
│   │   ├── db/
│   │   │   ├── base.py               # Engine, SessionLocal, get_db()
│   │   │   └── seed.py               # Datos de prueba (3 teachers, 3 courses, 6 lessons)
│   │   ├── models/
│   │   │   ├── base.py               # BaseModel (id, created_at, updated_at, deleted_at)
│   │   │   ├── course.py             # Course + average_rating/total_ratings properties
│   │   │   ├── teacher.py            # Teacher
│   │   │   ├── lesson.py             # Lesson (antes "Class")
│   │   │   ├── course_teacher.py     # Tabla asociativa M:N
│   │   │   └── course_rating.py      # CourseRating (1-5, soft delete, unique user+course)
│   │   ├── schemas/
│   │   │   └── rating.py             # RatingRequest, RatingResponse, RatingStatsResponse
│   │   ├── services/
│   │   │   └── course_service.py     # Toda la lógica de negocio (courses + ratings)
│   │   ├── alembic/
│   │   │   └── versions/             # Migraciones (initial schema + ratings)
│   │   └── tests/
│   │       ├── test_main.py          # Tests de endpoints y contratos
│   │       ├── test_rating_endpoints.py
│   │       ├── test_course_rating_service.py
│   │       └── test_rating_db_constraints.py
│   ├── docker-compose.yml            # Servicios: api + db
│   ├── Dockerfile                    # python:3.11-slim + uv
│   ├── Makefile                      # Comandos de desarrollo
│   └── pyproject.toml                # Dependencias (uv)
│
├── Frontend/                         # Next.js 15 App
│   ├── src/
│   │   ├── app/
│   │   │   ├── layout.tsx            # Root layout (Geist fonts, lang="es")
│   │   │   ├── page.tsx              # Home: fetch /courses, grid de cards
│   │   │   ├── course/[slug]/
│   │   │   │   ├── page.tsx          # Detalle: fetch /courses/{slug}
│   │   │   │   ├── error.tsx         # Error boundary
│   │   │   │   ├── loading.tsx       # Suspense fallback
│   │   │   │   └── not-found.tsx     # 404
│   │   │   └── classes/[class_id]/
│   │   │       ├── page.tsx          # Reproductor: fetch /classes/{id}
│   │   │       └── page.test.tsx
│   │   ├── components/
│   │   │   ├── Course/               # Card Netflix-style + StarRating
│   │   │   ├── CourseDetail/         # Detalle completo con teachers/lecciones
│   │   │   ├── VideoPlayer/          # HTML5 video player
│   │   │   └── StarRating/           # SVG stars, half-star support, readonly
│   │   ├── services/
│   │   │   └── ratingsApi.ts         # Cliente API ratings (CRUD + timeout + errors)
│   │   ├── types/
│   │   │   ├── index.ts              # Course, Class, CourseDetail, Progress, Quiz
│   │   │   └── rating.ts             # CourseRating, RatingRequest, RatingStats, ApiError
│   │   ├── styles/
│   │   │   ├── vars.scss             # Paleta colores (primary #ff2d2d), fn color()
│   │   │   └── reset.scss            # CSS reset
│   │   └── test/setup.ts             # Vitest + testing-library setup
│   ├── next.config.ts                # SCSS auto-import vars, sassOptions
│   ├── vitest.config.ts              # jsdom, path aliases
│   └── package.json                  # next 15.3.3, react 19, sass, vitest
│
└── Mobile/
    ├── PlatziFlixAndroid/            # Kotlin App
    │   └── app/src/main/java/com/espaciotiago/platziflixandroid/
    │       ├── MainActivity.kt       # Entry point, edge-to-edge
    │       ├── domain/
    │       │   ├── models/Course.kt  # Domain model
    │       │   └── repositories/CourseRepository.kt  # Interface
    │       ├── data/
    │       │   ├── entities/CourseDTO.kt    # Gson @SerializedName
    │       │   ├── mappers/CourseMapper.kt  # DTO → Domain
    │       │   ├── network/
    │       │   │   ├── ApiService.kt        # Retrofit interface
    │       │   │   └── NetworkModule.kt     # OkHttp config, base URL 10.0.2.2:8000
    │       │   └── repositories/
    │       │       ├── RemoteCourseRepository.kt  # API implementation
    │       │       └── MockCourseRepository.kt    # Dev/testing (1500ms delay, 10% fail)
    │       ├── presentation/courses/
    │       │   ├── viewmodel/CourseListViewModel.kt  # StateFlow + MVI events
    │       │   ├── state/CourseListUiState.kt        # UI state + sealed events
    │       │   ├── screen/CourseListScreen.kt        # LazyColumn + collapsing toolbar
    │       │   └── components/                       # CourseCard, ErrorMessage, LoadingIndicator
    │       ├── di/AppModule.kt       # Manual DI (toggle mock/real)
    │       └── ui/theme/             # Material3 dark/light + dynamic color
    │
    └── PlatziFlixiOS/                # Swift App
        └── PlatziFlixiOS/
            ├── PlatziFlixiOSApp.swift    # @main entry
            ├── ContentView.swift         # Root → CourseListView
            ├── Domain/
            │   ├── Models/               # Course, Teacher, Class (+ mock data)
            │   └── Repositories/CourseRepositoryProtocol.swift
            ├── Data/
            │   ├── Entities/             # CourseDTO, TeacherDTO, ClassDetailDTO (Codable)
            │   ├── Mapper/               # CourseMapper, TeacherMapper, ClassMapper
            │   └── Repositories/
            │       ├── RemoteCourseRepository.swift   # NetworkService implementation
            │       └── CourseAPIEndpoints.swift        # Endpoint enum, base URL localhost:8000
            ├── Services/
            │   ├── NetworkManager.swift   # Singleton, URLSession
            │   ├── NetworkService.swift   # Protocol
            │   ├── APIEndpoint.swift      # Protocol (baseURL, path, method)
            │   ├── HTTPMethod.swift       # GET, POST, PUT, DELETE, PATCH
            │   └── NetworkError.swift     # Enum con LocalizedError
            └── Presentation/
                ├── ViewModels/CourseListViewModel.swift  # @Published, search 300ms debounce
                └── Views/
                    ├── CourseListView.swift    # NavigationView, search, pull-to-refresh
                    ├── CourseCardView.swift    # AsyncImage, 16:9, accessibility
                    └── DesignSystem.swift      # Colors, spacing, typography, CardStyle
```

## Modelo de Datos

```
┌──────────┐    M:N (course_teachers)    ┌──────────────┐    1:N    ┌──────────┐
│ Teacher   │◄──────────────────────────►│    Course     │──────────►│  Lesson  │
│ - name    │                            │ - name        │           │ - name   │
│ - email   │                            │ - description │           │ - slug   │
│ (unique)  │                            │ - thumbnail   │           │ - video  │
└──────────┘                            │ - slug (unique)│          │ - course_id
                                         └───────┬───────┘          └──────────┘
                                                 │ 1:N
                                         ┌───────▼───────┐
                                         │ CourseRating   │
                                         │ - user_id      │
                                         │ - rating (1-5) │  CHECK constraint
                                         │ - deleted_at   │  Soft Delete
                                         └────────────────┘
                                         UNIQUE(course_id, user_id, deleted_at)

Todos los modelos heredan de BaseModel:
  - id (PK), created_at (auto), updated_at (auto), deleted_at (soft delete)
```

## API Endpoints

### Courses
| Método | Ruta | Descripción | Response |
|--------|------|-------------|----------|
| GET | `/` | Bienvenida | `{message}` |
| GET | `/health` | Health check + DB connectivity | `{status, database, courses_count}` |
| GET | `/courses` | Lista cursos con rating stats | `[{id, name, description, thumbnail, slug, average_rating, total_ratings}]` |
| GET | `/courses/{slug}` | Detalle con teachers + lessons | `{..., teacher_id[], classes[], rating_distribution}` |
| GET | `/classes/{class_id}` | Detalle de clase para reproductor | `{id, title, description, slug, video}` |

### Ratings (CRUD completo)
| Método | Ruta | Descripción | Status |
|--------|------|-------------|--------|
| POST | `/courses/{id}/ratings` | Crear rating (o update si existe) | 201 |
| GET | `/courses/{id}/ratings` | Listar ratings del curso | 200 |
| GET | `/courses/{id}/ratings/stats` | Stats: avg, total, distribución | 200 |
| GET | `/courses/{id}/ratings/user/{uid}` | Rating de un usuario | 200/204 |
| PUT | `/courses/{id}/ratings/{uid}` | Actualizar rating | 200 |
| DELETE | `/courses/{id}/ratings/{uid}` | Soft delete rating | 204 |

## Comandos de Desarrollo

### Backend (Docker obligatorio)
```bash
cd Backend
make start              # docker-compose up -d (api + db)
make stop               # docker-compose down
make restart            # docker-compose restart
make build              # docker-compose build
make logs               # docker-compose logs -f
make clean              # down -v --rmi all (elimina todo)
make migrate            # alembic upgrade head (dentro del container api)
make create-migration   # alembic autogenerate (pide mensaje)
make seed               # Poblar datos de prueba
make seed-fresh         # Limpiar + seed
```

### Frontend
```bash
cd Frontend
yarn dev          # Dev server con Turbopack (port 3000)
yarn build        # Build de producción
yarn start        # Servidor producción
yarn test         # Vitest
yarn lint         # ESLint
```

### Desarrollo completo
```bash
cd Backend && make start    # Primero el backend
cd Frontend && yarn dev     # Luego el frontend
```

## URLs del Sistema

- **Backend API**: http://localhost:8000
- **API Docs (Swagger)**: http://localhost:8000/docs
- **Frontend Web**: http://localhost:3000
- **Android Emulator → API**: http://10.0.2.2:8000
- **iOS Simulator → API**: http://localhost:8000

## Base de Datos

- **Motor**: PostgreSQL 15 (container Docker `db`)
- **Usuario**: platziflix_user
- **Password**: platziflix_password
- **Database**: platziflix_db
- **Puerto**: 5432
- **Migraciones**: `Backend/app/alembic/versions/`
  - `d18a08253457` - Schema inicial (teachers, courses, lessons, course_teachers)
  - `0e3a8766f785` - Tabla course_ratings

## Patrones de Arquitectura

### Backend (Python/FastAPI)
- **Service Layer Pattern**: `CourseService` centraliza toda la lógica de negocio
- **Dependency Injection**: `get_db()` → `get_course_service()` via `Depends()`
- **Soft Deletes**: `deleted_at` en todos los modelos para preservar histórico
- **Pydantic Schemas**: Validación de entrada/salida (rating 1-5, user_id > 0)
- **Archivos clave**: `main.py` (routes), `services/course_service.py` (lógica), `models/` (ORM)

### Frontend (TypeScript/Next.js)
- **Server Components**: Todas las páginas son async server components con fetch directo
- **App Router**: Rutas dinámicas `[slug]` y `[class_id]`, con error/loading/not-found
- **CSS Modules + SCSS**: Variables globales auto-importadas, `color()` function
- **No client state**: Sin useState/useEffect para data fetching, todo SSR con `cache: "no-store"`
- **ratingsApi.ts**: Cliente con timeout (10s), AbortController, clase ApiError custom

### Mobile Android (Kotlin)
- **MVVM + MVI**: ViewModel con StateFlow + sealed class para eventos UI
- **Clean Architecture**: domain/ (models, interfaces) → data/ (DTOs, mappers, repos) → presentation/
- **Retrofit + OkHttp**: ApiService interface, logging interceptor, 30s timeout
- **Coil**: AsyncImage para thumbnails con crossfade
- **Manual DI**: `AppModule` object con toggle `USE_MOCK_DATA`
- **Material3**: Dynamic color (Android 12+), dark/light themes

### Mobile iOS (Swift)
- **MVVM + Clean Architecture**: Misma separación domain/ → data/ → presentation/
- **SwiftUI + @Published**: ViewModel con @MainActor, Combine para debounce search (300ms)
- **URLSession nativo**: NetworkManager singleton, protocolos APIEndpoint + NetworkService
- **Codable + CodingKeys**: Mapeo snake_case automático en DTOs
- **Design System**: DesignSystem.swift con colores, spacing, tipografía, CardStyle modifier
- **Mock data**: Extensions en domain models con datos de preview

## Convenciones

### Naming
- **Python (Backend)**: `snake_case` para variables/funciones, `PascalCase` para clases
- **TypeScript (Frontend)**: `camelCase` para variables/funciones, `PascalCase` para componentes
- **Kotlin (Android)**: `camelCase` para variables/funciones, `PascalCase` para clases
- **Swift (iOS)**: `camelCase` para variables/funciones, `PascalCase` para tipos

### Estructura de archivos
- **Backend**: Un service por dominio, schemas separados de models, tests en `tests/`
- **Frontend**: Componente + `.module.scss` + `__test__/` por carpeta
- **Mobile**: Capa domain/ sin dependencias externas, data/ con DTOs y mappers separados

### Testing
- **Backend**: pytest + httpx TestClient, mocks del service layer
- **Frontend**: Vitest + React Testing Library + jsdom, test de componentes y server components
- **Android**: JUnit4 + kotlinx-coroutines-test, mocks de repository
- **iOS**: Sin tests aún

## Reglas Importantes

1. **Docker obligatorio** para el backend - siempre verificar que los containers estén corriendo
2. **Cualquier comando del Backend** debe ejecutarse dentro del container `api` (usar Makefile)
3. **TypeScript strict** en Frontend - no usar `any`
4. **Testing requerido** para nuevas funcionalidades
5. **Migraciones Alembic** para cualquier cambio de DB (nunca modificar schema manualmente)
6. **API REST es la única fuente de datos** - Frontend y Mobile NO acceden directamente a la DB
7. **Soft deletes** - usar `deleted_at` en lugar de DELETE real en la DB
8. **Server Components** en Frontend - preferir SSR sobre client components cuando sea posible
9. **Validación en Pydantic** (Backend) - nunca confiar en datos del cliente sin validar

## Funcionalidades Implementadas

- [x] Catálogo de cursos con grid estilo Netflix (Web + Mobile)
- [x] Detalle de cursos con profesores, lecciones, clases (Web)
- [x] Navegación por slug SEO-friendly (Web)
- [x] Reproductor de video HTML5 (Web)
- [x] Sistema de ratings CRUD completo (Backend + Frontend)
- [x] StarRating component con half-stars (Frontend)
- [x] Health checks API + DB (Backend)
- [x] Apps nativas Android + iOS con listado de cursos
- [x] Search/filtrado en apps móviles
- [x] Loading/Error/Empty states en todas las plataformas
- [x] Testing: Backend endpoints + services, Frontend components, Android ViewModel

## Pendiente

- [ ] Navegación a detalle de curso en Mobile (Android + iOS)
- [ ] Pantalla de profesores en Mobile
- [ ] Reproductor de video en Mobile
- [ ] Sistema de ratings en Mobile
- [ ] Tests unitarios en iOS
- [ ] Autenticación de usuarios (no existe modelo User aún)
- [ ] Sistema de progreso de clases
- [ ] Favoritos
- [ ] Quizzes (tipos definidos en Frontend pero no implementados)
