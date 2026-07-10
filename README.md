<div align="center">

# Binify

**Smart waste-bin monitoring — from bare-metal sensor to cloud dashboard.**

An end-to-end IoT system that measures how *full* and how *smelly* waste containers are, buffers the readings on a local gateway, syncs them to the cloud, and surfaces them to organizations through a web app with charts and threshold-based alerts.

![Zig](https://img.shields.io/badge/firmware-Zig%200.14-F7A41D)
![TypeScript](https://img.shields.io/badge/cloud-TypeScript-3178C6)
![React](https://img.shields.io/badge/web-React%2019-61DAFB)
![tRPC](https://img.shields.io/badge/api-tRPC%2011-398CCB)
![Postgres](https://img.shields.io/badge/db-Postgres%20%2B%20TimescaleDB-336791)

</div>

---

## Overview

**Binify** is a university team project built around a single, concrete IoT problem: knowing when a waste container needs attention *before* someone walks over to check it. A device based on the **Raspberry Pi Pico W** sits inside a bin and periodically measures:

- **Fullness** — via an ultrasonic distance sensor (distance from the lid to the top of the waste).
- **Odor intensity** ("smelliness") — via an air-quality sensor read as a gas concentration in ppm.

Those readings travel through a small distributed pipeline — **firmware → local hub → cloud API → web dashboard** — where they are stored as time-series data, visualized per device, and turned into **alerts** (e.g. a Telegram message when a bin crosses a fullness threshold).

The project was intentionally split into an **embedded** part and a **cloud** part so the two halves could evolve independently while communicating over plain HTTP with a shared, typed contract.

## Features

- **Battery-of-sensors firmware** in Zig running on the Pico W — reads an ultrasonic distance sensor and an analog air-quality sensor, then posts JSON measurements over Wi-Fi.
- **Offline-tolerant gateway ("hub")** — a Node.js service that buffers incoming measurements, persists the buffer to disk so nothing is lost across restarts, and syncs to the cloud when connectivity is available.
- **Typed end-to-end API** — the backend and frontend share types through **tRPC**, so the client cannot call an endpoint or read a field that does not exist.
- **Time-series storage** — measurements are stored in **PostgreSQL / TimescaleDB** as a hypertable, well suited to high-frequency sensor data.
- **Multi-tenant model** — organizations own activated bins; users are members with **Admin** or **Viewer** roles; devices are claimed via activation codes.
- **Authentication** via **Microsoft OAuth** (Passport) with JWT sessions.
- **Alerting** — configurable alert sources with fullness thresholds and optional repeat intervals; alerts are delivered through a **Telegram bot**, and a history of sent alerts is browsable in the UI.
- **Rich dashboard** — device detail pages with interactive charts (Recharts), date-range filtering, pagination, and organization management, built on shadcn/ui + Tailwind.
- **CI** — GitHub Actions type-checks and lints the web, API, and hub packages on every pull request.

## Tech stack

| Layer | Technologies |
|-------|--------------|
| **Firmware** | Zig 0.14, Raspberry Pi Pico W, pico-sdk, lwIP (TCP/IP), ARM GNU toolchain, CMake |
| **Hub (gateway)** | Node.js 22, TypeScript, Express, tRPC client, on-disk buffer persistence |
| **API (cloud)** | Node.js 22, TypeScript, Express, tRPC 11, Prisma 6, Zod, Passport (Microsoft OAuth), JWT, Pino |
| **Database** | PostgreSQL 17 + TimescaleDB (hypertables) |
| **Web** | React 19, Vite 6, TypeScript, tRPC + TanStack Query, TanStack Router, shadcn/ui, Radix UI, Tailwind CSS, Recharts |
| **Tooling** | npm workspaces (monorepo), Biome (lint/format), Docker Compose, GitHub Actions |

## Architecture

```
┌──────────────┐    HTTP/JSON    ┌──────────────┐    tRPC/HTTP    ┌──────────────┐
│   Firmware   │ ──────────────▶ │     Hub      │ ──────────────▶ │  Cloud API   │
│  Pico W (Zig)│   over Wi-Fi    │ (Node/Express│   on sync tick  │ (Node/tRPC/  │
│              │                 │  + buffer)   │                 │  Prisma)     │
│ • distance   │                 │              │                 │      │       │
│ • air quality│                 │ persists to  │                 │      ▼       │
└──────────────┘                 │ disk offline │                 │  Postgres/   │
                                 └──────────────┘                 │ TimescaleDB  │
                                                                  └──────┬───────┘
                                     ┌──────────────┐   tRPC             │
                                     │   Web app    │ ◀──────────────────┘
                                     │ (React/Vite) │   typed queries
                                     └──────────────┘
                                                                  Telegram bot ◀── alerts
```

- **Firmware** samples the sensors, formats a small JSON body (`deviceId` + measurement), and pushes it to the hub over a raw lwIP TCP connection.
- **The hub** decouples the device from the cloud: it accepts measurements locally, buffers and persists them, and periodically flushes to the cloud API. If the cloud is unreachable, readings survive a restart and are retried.
- **The API** validates and stores measurements, exposes a tRPC router (`accounts`, `organizations`, `bins`, `alerts`), runs background jobs (fullness-alert evaluation, Telegram updates), and enforces auth and org/role permissions.
- **The web app** consumes the same tRPC router with full type inference, rendering per-device charts, alert history, and organization/admin management.

## Project structure

```
binify/
├── firmware/                 # Zig firmware for the Raspberry Pi Pico W
│   ├── src/
│   │   ├── main.zig          # entry point: Wi-Fi + sensor loop
│   │   ├── distance_sensor.zig
│   │   ├── air_sensor.zig
│   │   ├── http_client.zig
│   │   └── http_request_builder.zig
│   ├── build.zig             # Zig build → firmware.uf2
│   └── CMakeLists.txt
└── cloud/                    # npm-workspace monorepo (api / web / hub)
    ├── api/                  # tRPC + Prisma backend
    │   ├── prisma/schema.prisma
    │   └── src/{router,jobs,auth,core,libs}
    ├── web/                  # React + Vite dashboard
    │   └── src/{pages,components,libs}
    ├── hub/                  # local buffering gateway
    │   └── src/{DataBuffer,BufferPersistanceAdapter,DataSync,...}
    └── docker-compose.yaml   # Postgres/TimescaleDB for local dev
```

## Getting started

### Cloud (API + web + hub)

Requirements: **Node.js 22** and **Docker** (for the database).

```bash
cd cloud

# 1) Start Postgres/TimescaleDB
docker compose up -d

# 2) Install all workspaces (once)
npm install

# 3) Apply database migrations
cd api && npx prisma migrate deploy && cd ..

# 4) Run the backend and frontend (in separate terminals)
cd api && npm run dev     # tRPC API
cd web && npm run dev     # dashboard on http://localhost:3000
```

The local **hub** gateway can be run on its own when working with a real device:

```bash
cd cloud/hub && npm run dev
```

> Configuration (API URL, OAuth keys, database URL, Telegram token, …) is supplied through environment variables; local defaults live in per-package `.env` files.

Useful scripts (per workspace): `npm run check` (type-check + lint), `npm run lint:fix` (Biome autofix), and in `api`, `npx prisma studio` to browse the database.

### Firmware

Requirements: **Zig 0.14**, the **pico-sdk** (`PICO_SDK_PATH` env var), the **ARM GNU toolchain**, CMake (3.5–4), Python 3, and a C toolchain. Full setup notes are in [`firmware/README.md`](firmware/README.md).

```bash
cd firmware
zig build
# → produces zig-out/firmware.uf2, flashable to the Pico W
```

## Tests & CI

There is no unit-test suite; correctness is guarded primarily by **static typing** end-to-end (TypeScript + tRPC + Zod + Prisma) and by CI. The GitHub Actions workflow ([`.github/workflows/cloud-check.yml`](.github/workflows/cloud-check.yml)) runs `npm run check` — TypeScript type-checking, Biome linting, and Prisma validation — for the **web**, **api**, and **hub** packages on every pull request.

## My role

This was a **university team project** developed by a relatively large team (**9 members**), which made task distribution challenging given the limited scope of the work. My contribution was correspondingly **focused**:

- **Frontend development** — implementation of two application pages plus several smaller UI and behavior adjustments.
- **Project documentation.**

The system as a whole (firmware, hub, API, database design, alerting) reflects the combined work of the team.

## Notes / status

- Built as coursework; feature-complete for its assignment scope rather than a production deployment.
- AI tools were used during development in a limited capacity.
- The README language is English for accessibility; the project itself was developed in a Czech university setting.
