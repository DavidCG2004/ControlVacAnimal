<picture>
  <img src="https://img.shields.io/badge/Flutter-02569B?style=for-the-badge&logo=flutter&logoColor=white" alt="Flutter">
  <img src="https://img.shields.io/badge/Supabase-3FCF8E?style=for-the-badge&logo=supabase&logoColor=white" alt="Supabase">
  <img src="https://img.shields.io/badge/Drift-0175C2?style=for-the-badge&logo=sqlite&logoColor=white" alt="Drift">
  <img src="https://img.shields.io/badge/License-Proprietary-red?style=for-the-badge" alt="License">
</picture>
---

# App — ControlVacAnimal

**Sistema de gestión de campañas de vacunación animal** diseñado para operar en territorio con conectividad intermitente. Permite registrar, consultar y sincronizar vacunaciones de perros y gatos en sectores urbanos, con captura de fotografía, geolocalización y soporte offline completo.

---

## Tabla de contenidos

- [Descripción general](#descripción-general)
- [Roles y permisos](#roles-y-permisos)
- [Capturas de pantalla](#capturas-de-pantalla)
- [Arquitectura](#arquitectura)
  - [Clean Architecture + BLoC](#clean-architecture--bloc)
  - [Flujo offline-first](#flujo-offline-first)
  - [Sincronización automática](#sincronización-automática)
- [Tech stack](#tech-stack)
- [Estructura del proyecto](#estructura-del-proyecto)
- [Modelo de datos](#modelo-de-datos)
  - [Supabase (PostgreSQL)](#supabase-postgresql)
  - [SQLite local (Drift)](#sqlite-local-drift)
- [Instalación y configuración](#instalación-y-configuración)
- [Uso](#uso)
- [Licencia](#licencia)

---

## Descripción general

VacunApp es una aplicación móvil desarrollada en Flutter que permite a los municipios gestionar campañas de vacunación animal directamente en terreno. Los vacunadores pueden registrar cada aplicación con fotografía, coordenadas GPS y datos del propietario, todo **sin necesidad de conexión a internet**. Cuando el dispositivo recupera conectividad, la sincronización con Supabase ocurre de forma automática y transparente.

El proyecto sigue los principios de **Clean Architecture** con **BLoC** para la gestión de estado, **Drift** (SQLite) para persistencia local offline, y **Supabase** como backend cloud (autenticación, base de datos PostgreSQL, almacenamiento de fotos y políticas de seguridad RLS).

---

## Roles y permisos

| Rol | Abreviatura | Descripción | Alcance |
|-----|-------------|-------------|---------|
| **Coordinador de Campaña** | CC | Administrador general del sistema | Gestión de sectores, CRUD de coordinadores de brigada, dashboard global con KPIs y gráficos por sector/vacunador |
| **Coordinador de Brigada** | CB | Supervisa la operación en sectores asignados | Dashboard de su brigada, gestión de vacunadores, visualización de registros de sus sectores |
| **Vacunador** | V | Ejecuta las vacunaciones en terreno | Registro de vacunaciones con foto + GPS, consulta de sus propios registros y sectores asignados |

Cada rol tiene políticas de seguridad **Row-Level Security (RLS)** en Supabase que restringen el acceso a nivel de fila según el usuario autenticado.

---
### Módulo de autenticación
- **Login** — Ingreso con correo electrónico y contraseña
- **Cambio de contraseña** — Obligatorio en primer inicio de sesión
- **Recuperación de contraseña** — Envío de enlace por correo

### Dashboard (CC / CB)
- **KPIs** — Total de vacunaciones, cobertura estimada, distribución por especie
- **Gráficos de barras** — Vacunaciones por sector y por vacunador (con gradiente azul)
- **Tarjetas de acceso rápido** — Navegación a sectores, usuarios y registros

### Gestión de sectores
- **Lista de sectores** — 23 sectores de Quito precargados
- **Formulario** — Creación y edición con nombre, descripción, ciudad y coordenadas
- **Detalle** — Información del sector con lista de vacunaciones asociadas

### Registro de vacunación
- **Formulario** — Captura de datos del propietario (cédula ecuatoriana con validación módulo 10), datos de la mascota, vacuna aplicada, fotografía (cámara o galería) y geolocalización automática
- **Detalle** — Visualización completa con foto, datos, mapa y estado de sincronización
- **Indicador de sync** — Chip visual que muestra "Sincronizado", "Pendiente" o "Sin sync"

### Gestión de usuarios
- **Lista** — Usuarios filtrables por rol
- **Creación** — Registro de coordinadores de brigada y vacunadores con validación de cédula ecuatoriana
- **Asignación** — Asignación de usuarios a sectores
- **Detalle** — Información del usuario con sectores asignados

---

## Arquitectura

### Clean Architecture + BLoC

```
┌──────────────────────────────────────────────────────────────┐
│                   PRESENTATION LAYER                         │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Pages /    │  │   Cubits /   │  │  Shared Widgets   │  │
│  │   Screens    │──│   Blocs      │──│  (photo_picker,   │  │
│  │              │  │              │  │   custom_button…)  │  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│                     DOMAIN LAYER                             │
│  ┌──────────────┐  ┌──────────────┐  ┌───────────────────┐  │
│  │   Entities   │  │   UseCases   │  │  Repository       │  │
│  │  (Equatable) │──│   (dartz     │──│  Interfaces       │  │
│  │              │  │    Either)   │  │                   │  │
│  └──────────────┘  └──────────────┘  └───────────────────┘  │
├──────────────────────────────────────────────────────────────┤
│                       DATA LAYER                             │
│  ┌─────────────────────┐    ┌────────────────────────────┐  │
│  │ Remote DataSources  │    │  Local DataSources         │  │
│  │ (Supabase REST API) │    │  (Drift DAOs)              │  │
│  ├─────────────────────┤    ├────────────────────────────┤  │
│  │ • AuthRemoteDS      │    │  • VaccinationDao          │  │
│  │ • SectorRemoteDS    │    │  • SectorDao               │  │
│  │ • VaccinationRemote │    │  • UserSectorDao           │  │
│  │ • UserMgmtRemoteDS  │    │  • SyncQueueDao            │  │
│  └─────────────────────┘    └────────────────────────────┘  │
│           │                            │                     │
│           ▼                            ▼                     │
│  ┌─────────────────┐      ┌────────────────────────────┐    │
│  │    Supabase     │      │  Drift (SQLite local DB)   │    │
│  │  • Auth         │      │  ┌──────────────────────┐  │    │
│  │  • PostgreSQL   │      │  │  vacunapp_db.sqlite  │  │    │
│  │  • Storage      │      │  │  • local_vaccinations│  │    │
│  │  • RLS Policies │      │  │  • sync_queue        │  │    │
│  └─────────────────┘      │  │  • local_sectors     │  │    │
│                           │  │  • local_user_sectors│  │    │
│                           │  └──────────────────────┘  │    │
│                           └────────────────────────────┘    │
└──────────────────────────────────────────────────────────────┘
```

Cada capa tiene responsabilidades claras y depende únicamente de la capa inferior a través de abstracciones (interfaces de repositorio). Las dependencias se inyectan mediante **GetIt** (`injection_container.dart`).

### Flujo offline-first

El módulo de vacunaciones es el núcleo offline de la aplicación:

```
┌──────────────┐     ┌──────────────────┐     ┌─────────────────┐
│  Usuario     │     │  RepositoryImpl  │     │  NetworkInfo    │
│  (UI/Cubit)  │────▶│  (Vaccination)   │────▶│  (connectivity) │
└──────────────┘     └──────────────────┘     └─────────────────┘
                             │                         │
                    ┌────────┴────────┐         ┌──────┴──────┐
                    ▼                 ▼         ▼             ▼
              ┌──────────┐    ┌──────────┐  Online        Offline
              │  Remoto  │    │  Local   │    │              │
              │ Supabase │    │  Drift   │    ▼              ▼
              └──────────┘    └──────────┘  ┌────────┐  ┌──────────┐
                                            │Operar  │  │Guardar   │
                                            │en      │  │en SQLite │
                                            │Supabase│  │+ encolar │
                                            └────────┘  └──────────┘
```

**Online**: La operación se ejecuta directamente en Supabase y luego se cachea localmente como `synced`.

**Offline**: La operación se guarda en la base SQLite local con estado `pending` y se encola en `sync_queue` con el payload completo (`operation`: `insert`/`update`/`delete`). Las fotos se guardan en el sistema de archivos del dispositivo (`localPhotoPath`).

### Sincronización automática

El `SyncService` es el motor de sincronización:

```
  ┌──────────────┐      ┌───────────────────┐      ┌──────────────┐
  │ App inicia   │─────▶│ _initialSync()    │─────▶│ syncPending  │
  └──────────────┘      └───────────────────┘      │   Items()    │
                                                   └──────┬───────┘
  ┌──────────────────┐      ┌───────────────────┐           │
  │ Conectividad     │─────▶│ onConnectivity    │───────────▶│ FIFO
  │ offline → online │      │   .listen()       │            │ cola
  └──────────────────┘      └───────────────────┘            ▼
                                                   ┌──────────────────┐
                                                   │ Procesar cada    │
                                                   │ item de la cola  │
                                                   └────────┬─────────┘
                                            ┌─────────────────┼────────────────┐
                                            ▼                 ▼                ▼
                                      ┌──────────┐    ┌──────────┐    ┌──────────┐
                                      │ INSERT   │    │ UPDATE   │    │ DELETE   │
                                      │ create() │    │ update() │    │ delete() │
                                      └──────────┘    └──────────┘    └──────────┘
                                            │                │               │
                                            ▼                ▼               ▼
                                      ┌─────────────────────────────────────────┐
                                      │  Marcar como synced (borrar de cola)    │
                                      └─────────────────────────────────────────┘
```

- Se activa al iniciar la app y al detectar transición offline → online
- Procesa la cola en orden FIFO
- Respeta un máximo de 3 reintentos por operación
- Incluye migración de IDs locales a IDs de Supabase cuando es necesario

---

## Tech stack

| Capa | Tecnología | Propósito |
|------|-----------|-----------|
| **Lenguaje** | Dart 3.12+ | SDK del lenguaje |
| **Framework** | Flutter 3.x | UI multiplataforma |
| **Estado** | flutter_bloc 8.x + equatable | Manejo de estado con BLoC/Cubit |
| **Router** | go_router 14.x | Navegación declarativa con protección por rol |
| **Backend** | Supabase 2.x | Autenticación, PostgreSQL, Storage, RLS |
| **Base local** | Drift 2.22 + sqlite3_flutter_libs | SQLite offline con DAOs tipados |
| **DI** | get_it 8.x | Inyección de dependencias |
| **Functional** | dartz 0.10 | Either para manejo de errores |
| **Fotos** | image_picker + cached_network_image + shimmer | Captura y visualización de imágenes |
| **GPS** | geolocator 13.x | Coordenadas geográficas |
| **Red** | connectivity_plus 6.x | Detección de conectividad |
| **Gráficos** | fl_chart 0.69 | Dashboard con KPIs y barras |
| **Tipografía** | google_fonts (Space Grotesk + Inter) | UI moderna |
| **Storage seguro** | flutter_secure_storage | Caché de sesión |
| **Generación** | build_runner + drift_dev | Código generado para Drift |
| **Testing** | bloc_test + mocktail | Pruebas unitarias y de widgets |

---

## Estructura del proyecto

```
lib/
├── main.dart                          # Punto de entrada: inicializa Supabase, DI y SyncService
├── app.dart                           # MaterialApp.router con MultiBlocProvider
├── injection_container.dart           # GetIt — 50+ dependencias registradas
│
├── config/
│   └── router/
│       └── app_router.dart            # GoRouter: 18 rutas con redirect por AuthBloc
│
├── core/
│   ├── constants/
│   │   ├── app_constants.dart         # Límites de sync, valores por defecto
│   │   ├── supabase_constants.dart    # URL, anon key, nombres de tablas/buckets
│   │   └── route_constants.dart       # Nombres de rutas
│   ├── enums/
│   │   ├── user_role.dart             # coordinadorCampana, coordinadorBrigada, vacunador
│   │   ├── sync_status.dart           # pending, synced, failed
│   │   ├── pet_type.dart              # perro, gato
│   │   └── pet_sex.dart               # macho, hembra
│   ├── errors/
│   │   ├── exceptions.dart            # Server, Cache, Auth, Location, Permission
│   │   └── failures.dart              # Failures equivalentes para dartz Either
│   ├── network/
│   │   └── network_info.dart          # NetworkInfoImpl con connectivity_plus
│   ├── theme/
│   │   ├── app_colors.dart            # Paleta institucional (azul #0F4C81)
│   │   ├── app_text_styles.dart       # Space Grotesk + Inter, 11sp–36sp
│   │   └── app_theme.dart             # Material 3 ThemeData completo
│   ├── usecases/
│   │   └── usecase.dart               # UseCase<Type, Params> base
│   ├── utils/
│   │   ├── input_validators.dart      # Cédula Ecuador, teléfono, email, password
│   │   ├── date_formatter.dart        # dd/MM/yyyy, HH:mm, timeAgo
│   │   └── location_helper.dart       # getCurrentPosition con permisos
│   └── widgets/
│       ├── loading_widget.dart
│       ├── error_widget.dart
│       ├── empty_widget.dart
│       ├── custom_text_field.dart
│       ├── custom_button.dart
│       ├── photo_picker_widget.dart   # Bottom sheet: cámara o galería
│       ├── role_badge.dart
│       └── pet_avatar.dart
│
├── database/
│   ├── app_database.dart              # AppDatabase (Drift), schema v2
│   ├── tables/
│   │   ├── local_vaccinations_table.dart   # Registros offline con syncStatus
│   │   ├── local_sectors_table.dart        # Cache de sectores
│   │   ├── local_users_table.dart          # Cache de usuarios
│   │   ├── local_user_sectors_table.dart   # Asignaciones usuario↔sector
│   │   └── sync_queue_table.dart           # Cola de operaciones pendientes
│   └── daos/
│       ├── vaccination_dao.dart            # CRUD + markAsSynced/Failed
│       ├── sector_dao.dart                 # CRUD + watch streams
│       ├── user_dao.dart                   # CRUD por rol
│       ├── user_sector_dao.dart            # Asignaciones batch
│       └── sync_queue_dao.dart             # Enqueue, markAs*, getPending
│
└── features/
    ├── auth/
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   ├── auth_remote_datasource.dart   # Supabase Auth
    │   │   │   └── auth_local_datasource.dart    # FlutterSecureStorage
    │   │   ├── models/user_model.dart
    │   │   └── repositories/auth_repository_impl.dart
    │   ├── domain/
    │   │   ├── entities/user_entity.dart
    │   │   ├── repositories/auth_repository.dart
    │   │   └── usecases/                         # 6 use cases
    │   └── presentation/
    │       ├── bloc/                              # AuthBloc + 8 estados
    │       └── pages/                             # Login, ChangePassword, ForgotPassword
    │
    ├── sectors/
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   ├── sector_remote_datasource.dart  # Supabase CRUD
    │   │   │   └── sector_local_datasource.dart   # Drift cache
    │   │   ├── models/sector_model.dart
    │   │   └── repositories/sector_repository_impl.dart
    │   ├── domain/
    │   │   ├── entities/sector_entity.dart
    │   │   ├── repositories/sector_repository.dart
    │   │   └── usecases/                         # 4 use cases
    │   └── presentation/
    │       ├── bloc/sector_cubit.dart
    │       └── pages/                             # List, Form, Detail
    │
    ├── vaccination/
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   └── vaccination_remote_datasource.dart  # Supabase CRUD + Storage
    │   │   ├── models/vaccination_record_model.dart
    │   │   └── repositories/vaccination_repository_impl.dart  # Offline-first core
    │   ├── domain/
    │   │   ├── entities/vaccination_record_entity.dart
    │   │   ├── repositories/vaccination_repository.dart
    │   │   └── usecases/                              # 7 use cases
    │   └── presentation/
    │       ├── bloc/                                   # VaccinationCubit + 7 estados
    │       └── pages/                                  # New form, Detail, Lists (V + CB)
    │
    ├── users_management/
    │   ├── data/
    │   │   ├── datasources/
    │   │   │   └── user_mgmt_remote_datasource.dart    # Supabase Auth admin + profiles
    │   │   ├── models/managed_user_model.dart
    │   │   └── repositories/user_mgmt_repository_impl.dart
    │   ├── domain/
    │   │   ├── entities/managed_user_entity.dart
    │   │   ├── repositories/user_mgmt_repository.dart
    │   │   └── usecases/                              # 8 use cases
    │   └── presentation/
    │       ├── bloc/user_mgmt_cubit.dart
    │       └── pages/                                  # List, Create, Assign, Detail
    │
    ├── dashboard/
    │   └── presentation/
    │       └── pages/
    │           ├── home_page.dart               # Menú basado en rol
    │           ├── cc_dashboard_page.dart       # KPIs + gráficos (CC)
    │           ├── cb_dashboard_page.dart       # KPIs + gráficos (CB)
    │           └── dashboard_widgets.dart       # kpiCard, barChart, sectionHeader
    │
    └── sync/
        └── sync_service.dart                   # Motor de sincronización offline→online
```

---

## Modelo de datos

### Supabase (PostgreSQL)

| Tabla | Columnas clave |
|-------|---------------|
| **profiles** | `id` (FK auth.users), `cedula` (UK), `nombres`, `apellidos`, `telefono`, `correo` (UK), `role` (CHECK), `must_change_password`, `created_by` |
| **sectors** | `id` (UUID PK), `nombre` (UK), `descripcion`, `ciudad`, `activo`, `latitud`, `longitud`, `created_by` |
| **user_sector_assignments** | `id` (UUID PK), `user_id` (FK profiles), `sector_id` (FK sectors), `assigned_by`, `activo` — UNIQUE(user_id, sector_id) |
| **vaccination_records** | `id` (UUID PK), `vaccinator_id`, `sector_id`, datos del propietario y mascota, `foto_url`, `latitud`, `longitud`, `fecha_hora` |

Todas las tablas tienen **Row-Level Security (RLS)** habilitada con políticas por rol.

### SQLite local (Drift)

| Tabla | Propósito |
|-------|-----------|
| **local_vaccinations** | Registros de vacunación con `syncStatus` (`pending`/`synced`/`failed`) y `localPhotoPath` para fotos offline |
| **sync_queue** | Cola FIFO de operaciones pendientes: `targetTable`, `recordId`, `operation` (`insert`/`update`/`delete`), `payload` (JSON), `status`, `attempts` |
| **local_sectors** | Cache de sectores con `syncStatus` |
| **local_users** | Cache de usuarios |
| **local_user_sectors** | Cache de asignaciones usuario↔sector |

---

## Instalación y configuración

### Requisitos

- Flutter SDK ^3.12.1
- Dart SDK ^3.12.1
- Proyecto en Supabase (plan free suficiente)
- Editor: VS Code o Android Studio

### Pasos

```bash
# 1. Clonar el repositorio
git clone https://github.com/tu-organizacion/control_vac_animal.git
cd control_vac_animal

# 2. Instalar dependencias
flutter pub get

# 3. Generar código de Drift (build_runner)
dart run build_runner build --delete-conflicting-outputs

# 4. Configurar Supabase
#    Editar lib/core/constants/supabase_constants.dart con tu URL y anon key
#    Ejecutar los scripts DDL del archivo Plan_de_Implementacion.md
#    (creación de tablas, funciones RLS, políticas, seed data y bucket Storage)

# 5. Ejecutar en modo desarrollo
flutter run
```

### Configuración de Supabase

1. Crear un proyecto en [supabase.com](https://supabase.com)
2. Ir a **SQL Editor** y ejecutar los scripts del `Plan_de_Implementacion.md` en orden:
   - Helper function `get_user_role()`
   - Tablas: `profiles`, `sectors`, `user_sector_assignments`, `vaccination_records`
   - Políticas RLS por rol
   - Seed data (23 sectores de Quito)
   - Bucket de Storage `vaccination-photos`
3. Copiar la URL del proyecto y la `anon key` a `lib/core/constants/supabase_constants.dart`

---

## Uso

### Credenciales por defecto

El primer usuario (Coordinador de Campaña) debe crearse directamente en Supabase Authentication. Los usuarios creados desde la app reciben la contraseña por defecto configurada en `app_constants.dart`.

### Flujo típico

1. **Coordinador de Campaña**: Crea sectores y registra coordinadores de brigada
2. **Coordinador de Brigada**: Asigna vacunadores a sectores y monitorea vacunaciones
3. **Vacunador**: Registra vacunaciones en terreno (incluso sin conexión)

---

## Licencia

VacunApp © 2026 — Todos los derechos reservados.

Este proyecto es de uso institucional. No está permitida su redistribución, modificación o uso comercial sin autorización expresa.
