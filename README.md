<div align="center">

# 🛰️ SafeTrack Pulse

**Real-time safety, convoy telemetry & off-grid navigation for riders and families.**

[![Platform](https://img.shields.io/badge/platform-iOS%20%7C%20Android-10b981)](#)
[![Stack](https://img.shields.io/badge/stack-Node.js%20%7C%20MongoDB%20%7C%20Redis-131b2e)](#)
[![Realtime](https://img.shields.io/badge/realtime-Socket.io-ef4444)](#)
[![License](https://img.shields.io/badge/license-MIT-f59e0b)](#license)
[![Status](https://img.shields.io/badge/status-Phase%201%20MVP-10b981)](#roadmap)

</div>

---

SafeTrack Pulse bridges everyday family reassurance with tactical, expedition-grade telemetry. It moves seamlessly between an uncluttered **Family Circle** map for daily peace of mind and a rugged **Convoy Cockpit** supporting up to **50 riders**, with autonomous crash detection, satellite emergency fallback, and zero-connectivity offline vector maps.

> **Repository:** `https://github.com/HarishVerma-UI/safe-track-pulse`

## Table of Contents

- [Why SafeTrack Pulse](#why-safetrack-pulse)
- [Core Features](#core-features)
- [Tech Stack](#tech-stack)
- [System Architecture](#system-architecture)
- [Geospatial Data Models](#geospatial-data-models)
- [Real-Time GPS Streaming Pipeline](#real-time-gps-streaming-pipeline)
- [Emergency SOS Engine](#emergency-sos-engine)
- [REST API Reference](#rest-api-reference)
- [Socket.io Event Contracts](#socketio-event-contracts)
- [MongoDB Indexing Strategy](#mongodb-indexing-strategy)
- [Code Examples](#code-examples)
- [Getting Started](#getting-started)
- [Project Structure](#project-structure)
- [Performance Targets](#performance-targets)
- [Subscription Tiers](#subscription-tiers)
- [Roadmap](#roadmap)
- [Contributing](#contributing)
- [License](#license)

---

## Why SafeTrack Pulse

Riders and families are stuck between two bad options:

- **Consumer tracking apps** (Life360, Google Maps) drain battery on all-day rides, fail when cell reception drops in remote terrain, and have no telemetry-driven convoy mechanics.
- **Dedicated tactical GPS devices** (Garmin inReach) are expensive, complex, and poorly integrated with real-time family/social features.

SafeTrack Pulse delivers both: a **calm daily map** for caregivers and a **high-frequency convoy cockpit** for expedition leaders — on the phone riders already carry.

### Who it's for

| Persona | Role | Primary Needs |
| :--- | :--- | :--- |
| **Aarav** | Expedition Leader / Tour Captain | Visibility into up to 50 riders, straggler alerts, offline topo routes, autonomous SOS dispatch |
| **Priya** | Family Caregiver | Calm peace of mind, arrival notifications, battery/motion status, unambiguous crash alerts |
| **Solo Commuter** | Weekend Explorer | Silent background crash detection, rapid 112 calling, automated Medical ID sharing |

---

## Core Features

- **Dual-Mode Map** — switch between *Family Circle* (calm view) and *Convoy Cockpit* (expedition HUD).
- **50-Rider Convoy Telemetry** — formation roles (Lead Navigator, Sweep, Medical Officer, Tail), straggler/drift alarms, hazard broadcasts.
- **Autonomous Crash Detection** — 50 Hz tri-axis sensor fusion, `>4.5G` impact threshold, 15-second critical cancel screen that bypasses Silent/DND.
- **Emergency SOS Dispatch** — auto-transmits coordinates, impact speed, nearest landmark, and digital Medical ID; direct 112/911 dialer.
- **Offline Vector Maps** — draggable bounding-box selector with live `km²` + download-size estimates; pre-packaged expedition corridors (Manali–Leh, Spiti Loop).
- **POI Intelligence** — fuel, two-wheeler garages, 24/7 dhabas, oxygen posts, water filtration.
- **Secure by Default** — OTP + biometric auth, Apple/Google SSO, end-to-end encrypted telemetry packets.

---

## Tech Stack

| Layer | Technology |
| :--- | :--- |
| **Mobile / Frontend** | React Native + TypeScript, Tailwind (NativeWind), Mapbox GL / Leaflet |
| **Realtime Transport** | Socket.io over WebSockets |
| **Backend** | Node.js + Fastify (or Express) |
| **Datastore** | MongoDB (GeoJSON `2dsphere`) |
| **Cache / Pub-Sub** | Redis (geospatial + pub/sub) |
| **Async Jobs** | BullMQ (Redis-backed queues) |
| **Auth & Push** | Firebase Auth + Firebase Cloud Messaging (FCM) |
| **Payments** | Razorpay / Stripe (annual subscription webhooks) |

---

## System Architecture

```
          ┌───────────────────────────── Mobile Client (React Native) ─────────────────────────────┐
          │  watchPosition (battery-aware)  ·  Socket.io client  ·  Mapbox GL  ·  FCM receiver      │
          └───────────────────────────────────────────┬──────────────────────────────────────────┘
                                                        │  WSS (location:ping / sos:initiate)
                                                        ▼
          ┌───────────────────────────── API Gateway (Node.js / Fastify) ─────────────────────────┐
          │   REST (/api/v1/*)        Socket.io Gateway (circle rooms)        Auth Middleware (JWT) │
          └──────────┬───────────────────────┬────────────────────────────────────────┬───────────┘
                     │                        │                                        │
                     ▼                        ▼                                        ▼
          ┌──────────────────┐      ┌──────────────────┐                     ┌──────────────────┐
          │      Redis       │      │     MongoDB      │                     │     BullMQ       │
          │  GEO cache +     │◄────►│  2dsphere geo    │                     │  SOS / Push /    │
          │  pub/sub fanout  │      │  persistence     │                     │  SMS fallback    │
          └──────────────────┘      └──────────────────┘                     └────────┬─────────┘
                                                                                        │
                                                                          ┌─────────────┴─────────────┐
                                                                          ▼                           ▼
                                                                    FCM Push                   SMS Gateway
```

**Write path (hot):** Clients stream `location:ping` every 3–5s → Redis (`GEOADD` + ephemeral hash) → Socket.io fan-out to circle rooms. A batch worker flushes the latest fix per user to MongoDB on a fixed interval, decoupling high-frequency pings from durable writes.

**Read path:** Nearby POI / member queries hit MongoDB `2dsphere` indexes. Live positions are served from Redis for sub-second latency.

---

## Geospatial Data Models

TypeScript + Mongoose schemas. All location fields use **GeoJSON `Point`** (`[lng, lat]` order) for native `2dsphere` support.

### User & Circle

```typescript
// src/models/user.model.ts
import { Schema, model, Types } from 'mongoose';

export type CircleRole = 'owner' | 'lead_rider' | 'sweep_rider' | 'medical_officer' | 'member';

const geoPointSchema = new Schema(
  {
    type: { type: String, enum: ['Point'], default: 'Point' },
    coordinates: { type: [Number], required: true }, // [lng, lat]
  },
  { _id: false },
);

const userSchema = new Schema(
  {
    firebaseUid: { type: String, required: true, unique: true, index: true },
    phone: { type: String, required: true, unique: true },
    displayName: { type: String, required: true },
    medicalId: {
      bloodType: String,
      conditions: [String],
      emergencyContacts: [{ name: String, phone: String }],
    },
    location: { type: geoPointSchema, required: false },
    batteryPct: { type: Number, min: 0, max: 100 },
    speedKmh: { type: Number, default: 0 },
    lastPingAt: { type: Date },
    fcmTokens: [String],
    subscription: {
      tier: { type: String, enum: ['free', 'elite'], default: 'free' },
      renewsAt: Date,
      providerRef: String, // Razorpay/Stripe subscription id
    },
  },
  { timestamps: true },
);

userSchema.index({ location: '2dsphere' });

export const User = model('User', userSchema);
```

```typescript
// src/models/circle.model.ts
import { Schema, model, Types } from 'mongoose';

const memberSchema = new Schema(
  {
    userId: { type: Types.ObjectId, ref: 'User', required: true },
    role: {
      type: String,
      enum: ['owner', 'lead_rider', 'sweep_rider', 'medical_officer', 'member'],
      default: 'member',
    },
    joinedAt: { type: Date, default: Date.now },
  },
  { _id: false },
);

const circleSchema = new Schema(
  {
    name: { type: String, required: true },
    type: { type: String, enum: ['family', 'convoy'], required: true },
    joinCode: { type: String, required: true, unique: true, index: true },
    ownerId: { type: Types.ObjectId, ref: 'User', required: true },
    members: { type: [memberSchema], default: [] },
    maxMembers: { type: Number, default: 4 }, // 4 family · 50 convoy
    stragglerThresholdM: { type: Number, default: 1500 }, // drift alarm radius
    routePath: {
      type: { type: String, enum: ['LineString'], default: 'LineString' },
      coordinates: { type: [[Number]], default: [] }, // [[lng, lat], ...]
    },
  },
  { timestamps: true },
);

export const Circle = model('Circle', circleSchema);
```

### Place / POI

```typescript
// src/models/poi.model.ts
import { Schema, model, Types } from 'mongoose';

export type PoiCategory =
  | 'petrol_pump' | 'restaurant' | 'mechanic' | 'hospital'
  | 'dhaba' | 'oxygen_post' | 'water_point' | 'sos_beacon';

const poiSchema = new Schema(
  {
    name: { type: String, required: true },
    category: { type: String, required: true },
    location: {
      type: { type: String, enum: ['Point'], default: 'Point' },
      coordinates: { type: [Number], required: true }, // [lng, lat]
    },
    circleId: { type: Types.ObjectId, ref: 'Circle', default: null }, // null = public
    metadata: { type: Schema.Types.Mixed }, // fuel grades, 24/7, elevation, etc.
    verified: { type: Boolean, default: false },
  },
  { timestamps: true },
);

poiSchema.index({ location: '2dsphere' });
poiSchema.index({ category: 1, circleId: 1 });

export const Poi = model('Poi', poiSchema);
```

### SOS Incident

```typescript
// src/models/sos.model.ts
import { Schema, model, Types } from 'mongoose';

const sosSchema = new Schema(
  {
    userId: { type: Types.ObjectId, ref: 'User', required: true, index: true },
    circleId: { type: Types.ObjectId, ref: 'Circle', required: true },
    triggerType: { type: String, enum: ['manual', 'crash_auto'], required: true },
    dispatchedAt: { type: Date, default: Date.now },
    location: {
      type: { type: String, enum: ['Point'], default: 'Point' },
      coordinates: { type: [Number], required: true }, // [lng, lat]
    },
    impactSpeedKmh: Number,
    gForce: Number,
    nearestLandmark: String,
    status: {
      type: String,
      enum: ['active', 'acknowledged', 'resolved', 'false_alarm'],
      default: 'active',
      index: true,
    },
    contactLogs: [
      {
        channel: { type: String, enum: ['push', 'sms', 'call', 'satellite'] },
        target: String,
        sentAt: Date,
        ok: Boolean,
      },
    ],
  },
  { timestamps: true },
);

sosSchema.index({ location: '2dsphere' });

export const SosIncident = model('SosIncident', sosSchema);
```

---

## Real-Time GPS Streaming Pipeline

### The throughput problem

50 riders × a ping every 3–5s ≈ **10–17 writes/sec per convoy**. Writing every fix straight to MongoDB creates needless IO and index churn. The fix: **Redis absorbs the hot path, MongoDB stores durable checkpoints.**

```
location:ping ──► Redis GEOADD (convoy:{id}:geo)      ◄── live reads (nearby, map)
             └──► Redis HSET  (convoy:{id}:live:{uid}) ◄── battery/speed/ts snapshot
             └──► Socket.io room fan-out (instant)
                        │
                  (every 10s batch)
                        ▼
             MongoDB bulkWrite(latest-per-user)        ◄── history / audit / recovery
```

- **Redis `GEOADD`** keeps live positions queryable with `GEOSEARCH` in O(log N).
- **Debounced batch persistence** flushes only the *latest* fix per user per interval via `bulkWrite`, collapsing dozens of pings into one upsert.
- **Pub/Sub** lets horizontally-scaled Socket.io nodes share room events (via `@socket.io/redis-adapter`).

### Straggler / drift algorithm (Haversine)

Distance of each rider from the Lead Rider (or nearest point on the route polyline). If it exceeds `stragglerThresholdM`, emit `convoy:straggler_warning`.

```typescript
// src/utils/geo.ts
const R = 6_371_000; // Earth radius (m)
const rad = (d: number) => (d * Math.PI) / 180;

/** Great-circle distance between two [lng, lat] points, in metres. */
export function haversineMeters(a: [number, number], b: [number, number]): number {
  const [lng1, lat1] = a;
  const [lng2, lat2] = b;
  const dLat = rad(lat2 - lat1);
  const dLng = rad(lng2 - lng1);
  const s =
    Math.sin(dLat / 2) ** 2 +
    Math.cos(rad(lat1)) * Math.cos(rad(lat2)) * Math.sin(dLng / 2) ** 2;
  return 2 * R * Math.asin(Math.sqrt(s));
}

/** Perpendicular-ish distance from a point to a LineString route (min over segments). */
export function distanceToPath(point: [number, number], path: [number, number][]): number {
  if (path.length === 0) return Infinity;
  if (path.length === 1) return haversineMeters(point, path[0]);
  let min = Infinity;
  for (let i = 0; i < path.length - 1; i++) {
    // Approximate: distance to the nearer segment endpoint (cheap, good enough at tile scale).
    min = Math.min(min, haversineMeters(point, path[i]), haversineMeters(point, path[i + 1]));
  }
  return min;
}
```

---

## Emergency SOS Engine

When a crash is detected (or SOS is pressed and not cancelled within 15s):

1. **Broadcast** — `sos:broadcast` fires to every active member of the rider's circle room instantly over WebSockets.
2. **Persist** — an `SosIncident` document is written with coordinates, impact speed, and G-force.
3. **Fallback queue** — a **BullMQ** job dispatches high-priority FCM pushes and builds SMS payloads with a tappable Google Maps link:
   `https://maps.google.com/?q=<lat>,<lng>`
4. **Satellite fallback** *(Phase 3)* — if zero cellular signal, the packet is queued for Iridium / Apple Satellite relay.

```typescript
// src/services/sos.service.ts
import { sosQueue } from '../queues/sos.queue';
import { SosIncident } from '../models/sos.model';
import type { Server } from 'socket.io';

interface SosPayload {
  userId: string;
  circleId: string;
  triggerType: 'manual' | 'crash_auto';
  coordinates: [number, number]; // [lng, lat]
  impactSpeedKmh?: number;
  gForce?: number;
}

export async function handleSos(io: Server, p: SosPayload) {
  const incident = await SosIncident.create({
    userId: p.userId,
    circleId: p.circleId,
    triggerType: p.triggerType,
    location: { type: 'Point', coordinates: p.coordinates },
    impactSpeedKmh: p.impactSpeedKmh,
    gForce: p.gForce,
    status: 'active',
  });

  // 1 — instant realtime broadcast to the circle
  io.to(`circle:${p.circleId}`).emit('sos:broadcast', {
    incidentId: incident.id,
    userId: p.userId,
    coordinates: p.coordinates,
    mapUrl: `https://maps.google.com/?q=${p.coordinates[1]},${p.coordinates[0]}`,
    triggeredAt: incident.dispatchedAt,
  });

  // 2 — async push + SMS fallback (retriable, high priority)
  await sosQueue.add(
    'dispatch',
    { incidentId: incident.id },
    { priority: 1, attempts: 5, backoff: { type: 'exponential', delay: 2000 } },
  );

  return incident;
}
```

---

## REST API Reference

Base URL: `/api/v1` · Auth: `Authorization: Bearer <Firebase ID token>`

| Method | Endpoint | Description |
| :--- | :--- | :--- |
| `POST` | `/circles` | Create a family or convoy circle |
| `POST` | `/circles/:id/join` | Join via invite code |
| `GET` | `/circles/:id/members` | List members + live snapshots |
| `GET` | `/pois/nearby?lat=&lng=&category=&radius=` | Geospatial POI search (radius in metres) |
| `POST` | `/sos/trigger` | Manual / auto emergency dispatch |
| `PATCH` | `/sos/:id/resolve` | Mark incident resolved / false alarm |
| `POST` | `/subscriptions/verify` | Razorpay / Stripe webhook handler |

**Example — nearby POIs**

```http
GET /api/v1/pois/nearby?lat=34.1526&lng=77.5770&category=petrol_pump&radius=10000
```

```json
{
  "count": 2,
  "results": [
    { "id": "...", "name": "IOCL Karu", "category": "petrol_pump", "distanceM": 3420,
      "location": { "type": "Point", "coordinates": [77.58, 34.16] } }
  ]
}
```

**Example — create circle**

```json
POST /api/v1/circles
{ "name": "Ladakh Rally 2026", "type": "convoy", "maxMembers": 50, "stragglerThresholdM": 1500 }
```

---

## Socket.io Event Contracts

### Client → Server

| Event | Payload | Purpose |
| :--- | :--- | :--- |
| `circle:join` | `{ circleId }` | Join the circle's realtime room |
| `location:ping` | `{ circleId, coordinates, speedKmh, batteryPct }` | High-frequency GPS fix |
| `sos:initiate` | `{ circleId, coordinates, triggerType, gForce? }` | Emergency trigger |
| `convoy:alert` | `{ circleId, type, message }` | Hazard broadcast (e.g. "Landslide Ahead") |

### Server → Client

| Event | Payload | Purpose |
| :--- | :--- | :--- |
| `circle:member_updated` | `{ userId, coordinates, speedKmh, batteryPct, ts }` | Live member position |
| `sos:broadcast` | `{ incidentId, userId, coordinates, mapUrl, triggeredAt }` | SOS alert to circle |
| `convoy:straggler_warning` | `{ userId, distanceM, thresholdM }` | Rider drifted past threshold |

---

## MongoDB Indexing Strategy

Geospatial queries scoped to a circle and recency need compound indexes so the planner avoids collection scans.

```javascript
// Live member lookups: "who in this circle, most recently seen"
db.users.createIndex({ location: '2dsphere' });
db.users.createIndex({ 'subscription.tier': 1 });

// POIs: geo + category + privacy scope
db.pois.createIndex({ location: '2dsphere' });
db.pois.createIndex({ category: 1, circleId: 1 }); // filter before geo-sort

// Circles: instant join-by-code
db.circles.createIndex({ joinCode: 1 }, { unique: true });

// SOS: active incidents by circle, newest first
db.sosincidents.createIndex({ status: 1, circleId: 1, dispatchedAt: -1 });
db.sosincidents.createIndex({ location: '2dsphere' });
```

> **Ordering rule:** in `2dsphere` compound indexes, place equality-filter fields (`category`, `circleId`) *before* the geo field when you filter then sort by distance; MongoDB narrows the candidate set first, then runs the geo stage over far fewer docs.

---

## Code Examples

### Express + Socket.io gateway (live circle rooms)

```typescript
// src/gateway/socket.gateway.ts
import { Server } from 'socket.io';
import type { Server as HttpServer } from 'http';
import { createAdapter } from '@socket.io/redis-adapter';
import { pub, sub, redis } from '../lib/redis';
import { verifyFirebaseToken } from '../lib/auth';
import { handleSos } from '../services/sos.service';
import { haversineMeters } from '../utils/geo';

export function attachGateway(httpServer: HttpServer) {
  const io = new Server(httpServer, { cors: { origin: '*' } });
  io.adapter(createAdapter(pub, sub)); // multi-node fan-out

  io.use(async (socket, next) => {
    try {
      socket.data.user = await verifyFirebaseToken(socket.handshake.auth.token);
      next();
    } catch {
      next(new Error('unauthorized'));
    }
  });

  io.on('connection', (socket) => {
    const uid = socket.data.user.uid;

    socket.on('circle:join', ({ circleId }) => socket.join(`circle:${circleId}`));

    socket.on('location:ping', async ({ circleId, coordinates, speedKmh, batteryPct }) => {
      // hot path: Redis only
      await redis.geoadd(`convoy:${circleId}:geo`, coordinates[0], coordinates[1], uid);
      await redis.hset(`convoy:${circleId}:live:${uid}`, {
        lng: coordinates[0], lat: coordinates[1], speedKmh, batteryPct, ts: Date.now(),
      });

      socket.to(`circle:${circleId}`).emit('circle:member_updated', {
        userId: uid, coordinates, speedKmh, batteryPct, ts: Date.now(),
      });

      // straggler check against lead rider
      const lead = await redis.hgetall(`convoy:${circleId}:lead`);
      if (lead?.lng) {
        const gap = haversineMeters(coordinates, [+lead.lng, +lead.lat]);
        if (gap > 1500) {
          io.to(`circle:${circleId}`).emit('convoy:straggler_warning', {
            userId: uid, distanceM: Math.round(gap), thresholdM: 1500,
          });
        }
      }
    });

    socket.on('convoy:alert', ({ circleId, type, message }) =>
      socket.to(`circle:${circleId}`).emit('convoy:alert', { type, message, from: uid }));

    socket.on('sos:initiate', (p) =>
      handleSos(io, { ...p, userId: uid }));
  });

  return io;
}
```

### Batch persistence worker

```typescript
// src/workers/persist.worker.ts  (runs every 10s)
import { redis } from '../lib/redis';
import { User } from '../models/user.model';

export async function flushLiveToMongo(circleId: string) {
  const keys = await redis.keys(`convoy:${circleId}:live:*`);
  if (!keys.length) return;

  const ops = [];
  for (const key of keys) {
    const uid = key.split(':').pop();
    const h = await redis.hgetall(key);
    if (!h.lng) continue;
    ops.push({
      updateOne: {
        filter: { firebaseUid: uid },
        update: {
          $set: {
            location: { type: 'Point', coordinates: [+h.lng, +h.lat] },
            speedKmh: +h.speedKmh, batteryPct: +h.batteryPct, lastPingAt: new Date(+h.ts),
          },
        },
      },
    });
  }
  if (ops.length) await User.bulkWrite(ops, { ordered: false });
}
```

### Client-side geolocation hook (battery-efficient)

```typescript
// mobile/src/hooks/useLiveLocation.ts
import { useEffect, useRef, useState } from 'react';
import Geolocation from 'react-native-geolocation-service';
import { socket } from '../lib/socket';

export function useLiveLocation(circleId: string, batteryPct: number) {
  const [error, setError] = useState<string | null>(null);
  const watchId = useRef<number | null>(null);

  useEffect(() => {
    if (!circleId) return;

    watchId.current = Geolocation.watchPosition(
      (pos) => {
        setError(null);
        socket.emit('location:ping', {
          circleId,
          coordinates: [pos.coords.longitude, pos.coords.latitude], // [lng, lat]
          speedKmh: Math.round((pos.coords.speed ?? 0) * 3.6),
          batteryPct,
        });
      },
      (err) => setError(err.message),
      {
        enableHighAccuracy: true,
        distanceFilter: 15,      // only emit after ~15m of movement — saves battery & writes
        interval: 4000,          // Android: target 4s cadence
        fastestInterval: 3000,
        showsBackgroundLocationIndicator: true,
      },
    );

    return () => {
      if (watchId.current != null) Geolocation.clearWatch(watchId.current);
    };
  }, [circleId, batteryPct]);

  return { error };
}
```

---

## Getting Started

### Prerequisites

- Node.js ≥ 20, npm or pnpm
- MongoDB ≥ 6 (replica set recommended for change streams)
- Redis ≥ 7
- Firebase project (Auth + Cloud Messaging)

### Installation

```bash
git clone https://github.com/HarishVerma-UI/safe-track-pulse.git
cd safe-track-pulse

# Backend
cd server
cp .env.example .env        # fill in Mongo / Redis / Firebase creds
npm install
npm run dev

# Mobile
cd ../mobile
npm install
npx pod-install ios         # iOS only
npm run ios                 # or: npm run android
```

### Environment variables

```ini
# server/.env
PORT=8080
MONGO_URI=mongodb://localhost:27017/safetrack
REDIS_URL=redis://localhost:6379
FIREBASE_PROJECT_ID=safetrack-pulse
FIREBASE_CLIENT_EMAIL=...
FIREBASE_PRIVATE_KEY=...
RAZORPAY_KEY_ID=...
RAZORPAY_KEY_SECRET=...
SMS_PROVIDER_KEY=...
```

---

## Project Structure

```
safe-track-pulse/
├── server/
│   ├── src/
│   │   ├── models/          # Mongoose schemas (user, circle, poi, sos)
│   │   ├── gateway/         # Socket.io gateway (live circle rooms)
│   │   ├── services/        # SOS engine, subscription, POI search
│   │   ├── workers/         # BullMQ consumers + batch persistence
│   │   ├── queues/          # BullMQ queue definitions
│   │   ├── routes/          # REST /api/v1 controllers
│   │   ├── lib/             # redis, firebase auth, db connection
│   │   └── utils/           # geo (haversine), landmarks
│   └── package.json
├── mobile/
│   ├── src/
│   │   ├── screens/         # FamilyMap, ConvoyCockpit, CrashModal, Offline, Auth
│   │   ├── hooks/           # useLiveLocation, useCrashDetection
│   │   ├── components/      # tactical UI (cards, HUD, SOS button)
│   │   └── lib/             # socket client, mapbox config
│   └── package.json
└── README.md
```

---

## Performance Targets

| Metric | Target | Rationale |
| :--- | :--- | :--- |
| GPS latency (cellular) | `< 800 ms` | Real-time formation tracking prevents pileups |
| Battery drain | `< 3.8% / hour` | 8–10 hour all-day remote rides |
| Crash detection latency | `< 250 ms` | Immediate 15s rescue countdown |
| Offline vector render | `60 FPS` @ 1:25k | Readable while handlebar-mounted |
| Core bundle size | `< 45 MB` | Downloads over constrained 3G edges |
| Cold start | `< 1.2 s` | Instant response in emergencies |

---

## Subscription Tiers

| | **Convoy Free** | **Convoy Elite** — $4.99/mo · $39.99/yr |
| :--- | :--- | :--- |
| Members | Up to 3 family | 50-rider convoy mode |
| Map tracking | Standard | Formation telemetry + sweep alerts |
| Location history | 7 days | Extended |
| Offline downloads | — | Unlimited custom bounding-box vectors |
| Crash detection | — | Autonomous + satellite relay fallback |
| Trial | — | 14-day free trial, annual savings badge |

---

## Roadmap

- [x] **Phase 1 — MVP:** Core UI, dark-mode design system, family tracking map, crash modal, convoy HUD, auth/profile, offline bounding-box manager.
- [ ] **Phase 2 — Telemetry Integration:** BLE pairing for OBD-II sensors and helmet intercoms (Sena, Cardo).
- [ ] **Phase 3 — Satellite Mesh:** Iridium / Apple Satellite Emergency SOS for off-grid dispatch when zero cellular signal is detected.

---

## Design System

- **Surface:** `#0b1326` (night navy) · containers `#131b2e` / `#1b2438`
- **Brand primary (emerald):** `#10b981` · **Critical:** `#ef4444` / `#f87171` · **Caution:** `#f59e0b`
- **Typography:** Inter with `tabular-nums` for telemetry counters
- **Roundness:** 8px (`ROUND_EIGHT`) · **Haptics:** dual-rumble on SOS, countdown ticks, boundary resize

---

## Contributing

1. Fork the repo and create a feature branch: `git checkout -b feature/my-feature`
2. Commit with conventional messages: `feat(convoy): add sweep rider role`
3. Run `npm test` and `npm run lint`
4. Open a pull request against `main`

---

## License

Distributed under the **MIT License**. See [`LICENSE`](LICENSE) for details.

---

<div align="center">
<sub>SafeTrack Pulse — <em>everyday reassurance, expedition-grade safety.</em></sub>
</div>
