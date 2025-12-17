# A Browser Based Game Boy Advance Emulator

This project is a Game Boy Advance emulator that works in any modern browser without plugins.

It began as a re-skin of the [gbajs2](https://github.com/andychase/gbajs2) fork by andychase, but now supports the [mGBA wasm](https://github.com/thenick775/mgba/tree/feature/wasm) core through the use of emscripten.


## Overview

At a high level, GBAJS3 gives you:

- A **React/TypeScript single-page app** that embeds the mGBA core compiled to WebAssembly and provides a touch-friendly UI.
- A **Go “auth & storage” service** that handles login, tokens, and managing ROM/save files for users in a PostgreSQL-backed store.
- A **Go-based admin interface** built with GoAdmin for managing users and data in the database.
- A **PostgreSQL database** (with initialization scripts) and a **Dockerized Nginx frontend** that serves the SPA over HTTPS and proxies API traffic.

Everything is wired together via Docker Compose so you can stand up the entire stack locally with a handful of commands.

---

## Features

### Front-End Emulator

Located in: `gbajs3/`

The browser client is a full-screen, responsive emulator UI with:

- **mGBA WASM core** via `@thenick775/mgba-wasm`.
- **Keyboard and on-screen controls**:
  - Desktop keyboards with configurable bindings.
  - Virtual D-pad and buttons for touch devices.
- **Configurable emulator settings**:
  - Controls for performance/audio/video exposed through a settings modal (e.g., sample rate choices, buffer sizes).
- **Modal-driven UX**:
  - Pre-game actions (e.g., load saves before starting a ROM).
  - Dedicated dialogs for loading ROMs, loading saves, cheats, emulator settings, file system, import/export, and legal notices.
- **Error handling and toasts**:
  - Central error boundary and toast notifications for API failures or emulator issues.
- **Responsive layout**:
  - Works on desktop, tablets, and phones with a layout provider + virtual controls.

### ROM & Save Management

The system is built around the idea that ROMs and saves can live both **locally in the browser** and **remotely on the server**:

- **List & load ROMs from the backend**:
  - Uses React Query hooks (`useListRoms`, `useLoadRom`, etc.) to talk to `/api/rom/*` endpoints.
- **Upload/download saves**:
  - Upload save files to the backend (`/api/save/upload`) and fetch them back (`/api/save/download`).
- **Local file loading**:
  - Load a ROM directly from the local filesystem using file picker components.
- **Virtual file system views**:
  - UI for exploring the emulator’s internal file tree (ROMs, saves, autosaves, screenshots, cheats, patches, etc.).
- **Cheat management**:
  - A dedicated cheats modal for adding, enabling/disabling, and removing cheat entries.

### Authentication & API

Located in: `auth/`

A standalone Go service provides all auth and storage endpoints:

- **JWT-based authentication**:
  - Endpoints for logging in (`/api/account/login`) and refreshing access tokens (`/api/tokens/refresh`).
- **User persistence in PostgreSQL**:
  - GORM models for users, tokens, and per-user storage directories.
- **ROM & save APIs**:
  - `/api/rom/list`, `/api/rom/upload`, `/api/rom/download`
  - `/api/save/list`, `/api/save/upload`, `/api/save/download`
- **CORS and gzip**:
  - CORS middleware for browser clients.
  - Optional gzip compression for responses.
- **Swagger/OpenAPI docs**:
  - Generated documentation stored in `auth/docs/` so you can inspect request/response shapes while developing.

### Admin Dashboard

Located in: `admin/`

The admin service gives you a management interface for the backing databases:

- Built with **GoAdmin** and Gorilla Mux.
- Connects to the same PostgreSQL instance to provide admin tables and dashboards.
- Serves an admin landing page at `/admin` using `html/welcome.tmpl`.
- Terminated over HTTPS with local certs in the Docker setup (configurable via environment variables).

This is intended for maintaining users, inspecting data, and eventually adding application-specific dashboards.

### PWA & Offline Support

The front-end is set up as a **Progressive Web App**:

- Uses `vite-plugin-pwa` to generate a PWA manifest and service worker.
- Can be **installed** on supported devices as a standalone app icon.
- A **“COOP/COEP through service worker”** mode (`with-coi-serviceworker`) is available for advanced browser isolation and WebAssembly performance scenarios.
- Static assets (including the emulator core and UI) are cached, enabling offline or spotty-network gameplay once ROMs are available locally.

---


## Getting Started

- Local builds require [docker](https://www.docker.com)

  - today, the project is only compatible with [docker compose v2](https://docs.docker.com/compose/releases/migrate/) and above

- Run the bootstrap script and follow the interactive prompts:

  ```
  ./bin/boostrap.sh;
  ```

- The bootstrap script will do the following:

  - copy env files from the examples in all directories
  - create default local directories
  - generate a local test ssl certificate pair with prompting from OpenSSL (if installed)

- This script will also generate a top level `.env` file of the following format, merging all service specific env files and additional docker config:

  ```
  # gbajs3
  ROM_PATH=./<local server rom path>/
  SAVE_PATH=./<local server save path>/
  CLIENT_HOST=https://<your client location>
  CERT_DIR=./<path to your certificate root directory>
  CERT_LOC=./<path to your certificate>
  KEY_LOC=./<path to your key>

  # admin
  ADMIN_APP_ID=<your unique alphanumeric app id>

  # postgres
  PG_DB_HOST=<database host>
  PG_DB_USER=<database user>
  PG_DB_PASSWORD=<database user password>
  GBAJS_DB_NAME=<your gbajs3 database name, default `gbajs3`>
  ADMIN_DB_NAME=<your goadmin database name, default `goadmin`>
  PG_DB_PORT=<postgres db port, default 5432>
  PG_SSL_MODE=<pg ssl mode>
  PG_DATA_LOCATION=./<path to postgres persistent bind mount point>

  # compose for merging yaml definitions
  COMPOSE_FILE_SEPARATOR=:
  COMPOSE_FILE=<colon separated list of compose files to merge>
  SERVICE_POSTGRES=<relative path of postgres service>
  SERVICE_GBAJS3=<relative path of gbajs3 service>
  SERVICE_AUTH=<relative path of auth service>
  SERVICE_ADMIN=<relative path of admin service>
  ```

  Leaving all default values in place will work for local development.

- If your developing on a mac, you will need to share the database bind mount location(s) manually, and ensure they have the correct permissions

  These settings are located in `Settings -> Resources -> File Sharing`

- Build and run the docker containers using compose:

  ```
  docker compose up;
  ```

- Build and run the docker containers using swarm:

  ```
  # swarm does not build images by default
  docker compose build;
  # add all images to the stack except shepherd for local dev
  docker stack deploy -c docker-compose.swarm.yaml -c ./auth/docker-compose.yaml -c ./admin/docker-compose.yaml -c ./postgres/docker-compose.yaml -c ./gbajs3/docker-compose.yaml gbajs3;
  ```

- Once docker has created the containers, the web server will be available at https://localhost

- The Admin UI can be found at https://localhost/admin

  - The default password for all admin users is `admin`, **please log in to the admin portal and change the default passwords immediately**

- Golang api swagger UI can be found at https://localhost/api/documentation/

- To run each service individually, with or without docker, see the nested READMEs under each top level service directory

## Architecture

High-level components:

```text
+--------------------+        +---------------------+
| Browser (React SPA)| <----> | gba-webserver       |
|  - Emulator UI     |  HTTPS |  (Nginx + TLS)      |
+--------------------+        |  - serves SPA       |
                              |  - proxies /api/... |
                              +----------+----------+
                                         |
                                         |
                                         v
                         +---------------+---------------+
                         |                               |
               +---------+---------+           +---------+---------+
               | gba-auth-server  |           | gba-admin-server  |
               |  (Go)            |           |  (Go + GoAdmin)   |
               |  - JWT auth      |           |  - Admin UI       |
               |  - ROM/save API  |           |  - DB mgmt        |
               +---------+--------+           +---------+---------+
                         \                         /
                          \                       /
                           v                     v
                              +-----------------+
                              |  PostgreSQL DB  |
                              |  (gbajs3, admin)|
                              +-----------------+

GBA-JSSCRIPT/
├── .env.example              # Combined env for docker compose
├── docker-compose.yaml       # Orchestration glue for all services
├── docker-compose.swarm.yaml # Swarm deployment variant
├── LICENSE
├── README.md                 # (can be replaced by this file)
│
├── gbajs3/                   # Front-end SPA + Nginx image
│   ├── Dockerfile
│   ├── docker/
│   │   ├── nginx.conf.template
│   │   └── fail2ban/...
│   ├── src/
│   │   ├── App.tsx
│   │   ├── components/...
│   │   ├── context/...
│   │   ├── emulator/mgba/...
│   │   ├── hooks/...
│   │   └── service-worker/
│   ├── vite.config.ts
│   └── tsconfig.json
│
├── auth/                     # Auth + ROM/save API (Go)
│   ├── Dockerfile
│   ├── .env.example
│   ├── main.go
│   ├── routes.go
│   ├── auth_handlers.go
│   ├── db.go
│   ├── models.go
│   └── docs/swagger.*        # Swagger/OpenAPI description
│
├── admin/                    # Admin dashboard (Go + GoAdmin)
│   ├── Dockerfile
│   ├── .env.example
│   ├── main.go
│   ├── conf/
│   ├── models/
│   ├── pages/
│   ├── tables/
│   └── html/welcome.tmpl
│
├── postgres/                 # Postgres container setup
│   ├── Dockerfile
│   ├── .env.example
│   ├── docker-compose.yaml
│   ├── init_admin_db.sql
│   └── init_gbajs3_db.sql
│
├── readme-graphics/          # Screenshots for documentation
│   ├── gbajs3-desktop-v5.png
│   ├── gbajs3-mobile-landscape-v2.png
│   ├── gbajs3-mobile-portrait-v5.png
│   └── admin-desktop.png
│
└── shepherd/                 # Optional container auto-updater config
    └── docker-compose.yaml
