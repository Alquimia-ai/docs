# Technical Blueprint
## Alquimia Argos - Frontend Architecture

**Version**: 2.0
**Date**: February 2026
**Target Framework**: Next.js 15 (App Router)

---

## Table of Contents
1. [Architecture Overview](#architecture-overview)
2. [Technology Stack](#technology-stack)
3. [Folder Structure](#folder-structure)
4. [Atomic Design System](#atomic-design-system)
5. [Server vs Client Components](#server-vs-client-components)
6. [Authentication Architecture](#authentication-architecture)
7. [Video Streaming Architecture](#video-streaming-architecture)
8. [Real-Time Events (SSE) Architecture](#real-time-events-sse-architecture)
9. [State Management Strategy](#state-management-strategy)
10. [BFF API Integration](#bff-api-integration)
11. [Routing Strategy](#routing-strategy)
12. [Styling Guidelines](#styling-guidelines)
13. [Error Handling](#error-handling)
14. [Performance Optimization](#performance-optimization)
15. [Security Considerations](#security-considerations)

---

## Architecture Overview

Alquimia Argos follows ** Next.js App Router architecture** with clear separation between server and client concerns. The application is structured as a single-page dashboard with real-time data synchronization.

### High-Level Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                     Alquimia Argos Frontend                  │
│                      (Next.js 15 App)                       │
│                                                             │
│  ┌──────────────┐  ┌──────────────┐  ┌─────────────────┐ │
│  │   Server     │  │   Client     │  │   Middleware    │ │
│  │  Components  │  │  Components  │  │  (Auth Guard)   │ │
│  └──────────────┘  └──────────────┘  └─────────────────┘ │
│         │                 │                    │          │
│         └─────────────────┴────────────────────┘          │
│                           │                               │
└───────────────────────────┼───────────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │      BFF Layer          │
              │  (REST + SSE + HLS)     │
              └─────────────────────────┘
                            │
                            ▼
              ┌─────────────────────────┐
              │   Argos Video Engine    │
              │   (AI Processing)       │
              └─────────────────────────┘
```

### Data Flow

```
User Action → Client Component → Server Action/API → BFF → Engine
                     ↓
            State Update (Zustand/SWR)
                     ↓
            UI Re-render
```

**SSE Flow**:
```
Engine → BFF → SSE Stream → EventSource (Client) → Zustand Store → UI
```

**Video Flow**:
```
Camera → Engine → BFF (HLS Proxy) → Video.js Player → Browser
```

---

## Technology Stack

### Core Framework

- **Next.js 15.1**: React framework with App Router
- **React 19**: UI library with Server Components
- **TypeScript 5.3+**: Type safety
- **pnpm 9+**: Package manager

### UI Layer

- **shadcn/ui**: Component library (Radix UI primitives)
- **Tailwind CSS 4**: Utility-first CSS framework
- **Lucide React**: Icon system
- **Radix UI**: Headless component primitives

### Video Streaming

- **Video.js 8**: Video player framework
- **@videojs/http-streaming**: HLS/DASH support
- **videojs-contrib-quality-levels**: Quality switching

### Authentication

- **Token-based authentication (static bearer token)**: Simple token validation against BFF

### State Management

- **Zustand 4**: Client-side state (events, UI state)
- **SWR 2**: Server state caching (camera configs)
- **React Context**: Cross-cutting concerns (theme)

### Forms & Validation

- **React Hook Form 7**: Form state management
- **Zod 3**: Schema validation
- **@hookform/resolvers**: RHF + Zod integration

### Data Fetching

- **Server Actions**: Mutations (camera CRUD, notifications)
- **fetch API**: Server-side data fetching
- **EventSource API**: SSE connections
- **TanStack Query (optional)**: Developers may use TanStack Query instead of (or alongside) the default fetch + server actions approach if they prefer its caching, invalidation, and loading/error handling for BFF-backed data.

### Development Tools

- **ESLint 9**: Linting
- **Prettier 3**: Code formatting
- **TypeScript ESLint**: TypeScript linting
- **Husky**: Git hooks (optional)

### Future Testing Stack (v2)

- **Vitest**: Unit testing
- **React Testing Library**: Component testing
- **Playwright**: E2E testing

---

## Folder Structure (this is an estimate, theres place for modification e.g api routes placement among other stuff)

```
apps/web/
├── public/
│   ├── fonts/                    # Custom fonts (if any)
│   ├── images/                   # Static images
│   └── favicon.ico
├── src/
│   ├── app/                      # Next.js App Router
│   │   ├── (auth)/              # Auth layout group (unauthenticated)
│   │   │   ├── layout.tsx       # Minimal layout for auth pages
│   │   │   └── login/
│   │   │       └── page.tsx     # Login page (token entry)
│   │   ├── (dashboard)/         # Dashboard layout group (authenticated)
│   │   │   ├── layout.tsx       # Dashboard shell (navbar, auth check)
│   │   │   ├── page.tsx         # Main dashboard (video grid + events)
│   │   │   └── cameras/         # Camera management page
│   │   │       └── page.tsx     # Camera CRUD + prompt config
│   │   ├── api/                 # API routes (if needed for SSE fallback)
│   │   ├── layout.tsx           # Root layout (global providers)
│   │   ├── not-found.tsx        # 404 page
│   │   └── error.tsx            # Error boundary
│   ├── components/
│   │   ├── atoms/               # Basic UI elements
│   │   │   ├── Button.tsx
│   │   │   ├── Badge.tsx
│   │   │   ├── Input.tsx
│   │   │   ├── Select.tsx
│   │   │   ├── Switch.tsx
│   │   │   └── Skeleton.tsx
│   │   ├── molecules/           # Composite components
│   │   │   ├── CameraCard.tsx       # Single camera display
│   │   │   ├── EventCard.tsx        # Single event display
│   │   │   ├── FilterToggle.tsx     # Normal/Anomaly filter
│   │   │   ├── RecIndicator.tsx     # Animated REC badge
│   │   │   ├── EventOverlay.tsx     # Event banner on camera
│   │   │   └── LoadingSpinner.tsx
│   │   ├── organisms/           # Complex components
│   │   │   ├── VideoGrid.tsx        # Multi-camera grid layout
│   │   │   ├── VideoPlayer.tsx      # Video.js wrapper
│   │   │   ├── EventSidebar.tsx     # Real-time event feed
│   │   │   ├── Navbar.tsx           # Top navigation bar
│   │   │   ├── CameraConfigForm.tsx # Camera create/edit form
│   │   │   ├── CameraConfigList.tsx # Camera list with management
│   │   │   └── UserMenu.tsx         # User profile dropdown
│   │   └── templates/           # Page-level layouts
│   │       ├── DashboardLayout.tsx  # Dashboard shell
│   │       └── AuthLayout.tsx       # Auth page shell
│   ├── lib/
│   │   ├── api/                 # BFF API client
│   │   │   ├── client.ts        # Fetch wrapper with auth
│   │   │   ├── actions.ts       # Server actions for mutations
│   │   │   ├── queries.ts       # Server-side data fetching
│   │   │   └── endpoints.ts     # API endpoint constants
│   │   ├── auth/                # Authentication
│   │   │   ├── token.ts         # Token management (get/set/clear)
│   │   │   └── session.ts       # Session utilities
│   │   ├── hooks/               # Custom React hooks
│   │   │   ├── useEventStream.ts    # SSE connection hook
│   │   │   ├── useCameraStatus.ts   # Camera status polling
│   │   │   ├── useVideoPlayer.ts    # Video.js lifecycle management
│   │   │   └── useMediaQuery.ts     # Responsive helpers
│   │   ├── stores/              # Zustand stores
│   │   │   ├── eventStore.ts    # Event state management
│   │   │   ├── uiStore.ts       # UI state (filters, modals)
│   │   │   └── cameraStore.ts   # Camera state (audio, focus)
│   │   ├── types/               # TypeScript type definitions
│   │   │   ├── api.ts           # BFF API types
│   │   │   ├── events.ts        # Event types
│   │   │   ├── cameras.ts       # Camera types
│   │   │   └── index.ts         # Type exports
│   │   ├── utils/               # Utility functions
│   │   │   ├── cn.ts            # Class name utility (clsx + tailwind-merge)
│   │   │   ├── formatTimestamp.ts   # Date/time formatting
│   │   │   ├── getCameraIcon.ts     # Status icon helper
│   │   │   └── reconnect.ts         # Exponential backoff logic
│   │   └── validations/         # Zod schemas
│   │       ├── cameraSchemas.ts
│   │       └── eventSchemas.ts
│   ├── config/                  # Application configuration
│   │   ├── site.ts              # Site metadata
│   │   └── constants.ts         # App constants (grid sizes, timeouts)
│   ├── styles/
│   │   └── globals.css          # Global styles + Tailwind imports
│   └── middleware.ts            # Auth middleware (token check)
├── docs/
│   ├── PRD.md
│   ├── TECHNICAL_BLUEPRINT.md
│   ├── BFF_API_CONTRACT.md
│   ├── COMPONENT_SPEC.md
│   ├── SETUP_GUIDE.md
│   ├── UI_REFERENCE.md
│   └── diagrams/
│       ├── architecture.mmd
│       ├── data-flow.mmd
│       ├── auth-flow.mmd
│       └── event-flow.mmd
├── .env.example
├── .env.local                   # Local environment (git ignored)
├── .eslintrc.json
├── .prettierrc
├── components.json              # shadcn/ui config
├── next.config.js
├── tailwind.config.ts
├── tsconfig.json
├── package.json
├── pnpm-lock.yaml
├── CLAUDE.md                    # Claude Code guidance
└── README.md
```

The `lib/api/` layer (e.g. `queries.ts`, server actions) is a suggested default; **TanStack Query** may be used instead if the developer considers it a better fit for caching, refetching, or client-side data flow.

### Folder Naming Conventions

- **Lowercase with hyphens**: `video-grid`, `event-card`
- **PascalCase for components**: `VideoGrid.tsx`, `EventCard.tsx`
- **camelCase for utilities/hooks**: `useEventStream.ts`, `formatTimestamp.ts`
- **Kebab-case for route folders**: `/cameras`, `/login`

---

## Atomic Design System

Components are organized following **Atomic Design methodology** for maintainability and reusability.

### Atoms (Basic Building Blocks)

- **Purpose**: Fundamental UI elements, often shadcn/ui components
- **Examples**: Button, Input, Badge, Select, Switch, Skeleton
- **Rules**:
  - No business logic
  - Highly reusable across the app
  - Accept props for customization
  - No external data fetching

**Example: Button.tsx**

```typescript
// src/components/atoms/Button.tsx
import * as React from 'react'
import { Slot } from '@radix-ui/react-slot'
import { cva, type VariantProps } from 'class-variance-authority'
import { cn } from '@/lib/utils/cn'

const buttonVariants = cva(
  'inline-flex items-center justify-center rounded-md text-sm font-medium transition-colors',
  {
    variants: {
      variant: {
        default: 'bg-red-600 text-white hover:bg-red-700',
        outline: 'border border-gray-700 hover:bg-gray-800',
        ghost: 'hover:bg-gray-800'
      },
      size: {
        default: 'h-10 px-4 py-2',
        sm: 'h-9 px-3',
        lg: 'h-11 px-8'
      }
    },
    defaultVariants: {
      variant: 'default',
      size: 'default'
    }
  }
)

export interface ButtonProps
  extends React.ButtonHTMLAttributes<HTMLButtonElement>,
    VariantProps<typeof buttonVariants> {
  asChild?: boolean
}

const Button = React.forwardRef<HTMLButtonElement, ButtonProps>(
  ({ className, variant, size, asChild = false, ...props }, ref) => {
    const Comp = asChild ? Slot : 'button'
    return (
      <Comp
        className={cn(buttonVariants({ variant, size, className }))}
        ref={ref}
        {...props}
      />
    )
  }
)
Button.displayName = 'Button'

export { Button, buttonVariants }
```

### Molecules (Simple Combinations)
- **Purpose**: Combinations of atoms forming simple functional units
- **Examples**: CameraCard, EventCard, FilterToggle, RecIndicator
- **Rules**:
  - Compose atoms with minimal logic
  - Accept data via props
  - May contain local UI state (hover, expanded)
  - No external data fetching

**Example: EventCard.tsx**
```typescript
// src/components/molecules/EventCard.tsx
'use client'

import { Badge } from '@/components/atoms/Badge'
import { Button } from '@/components/atoms/Button'
import { AlertTriangle, CheckCircle } from 'lucide-react'
import { formatTimestamp } from '@/lib/utils/formatTimestamp'
import type { DetectedEvent } from '@/lib/types/events'

interface EventCardProps {
  event: DetectedEvent
  onNotify?: (eventId: string) => void
}

export function EventCard({ event, onNotify }: EventCardProps) {
  const isAnomaly = event.type === 'anomaly'
  const Icon = isAnomaly ? AlertTriangle : CheckCircle
  const iconColor = isAnomaly ? 'text-red-500' : 'text-green-500'

  return (
    <div className="bg-gray-900 border border-gray-800 rounded-lg p-4 hover:border-gray-700 transition-colors">
      <div className="flex items-start gap-3">
        <Icon className={`w-5 h-5 mt-0.5 ${iconColor}`} />
        <div className="flex-1 space-y-2">
          <div className="text-xs text-gray-500">
            {event.cameraId} • {event.location}
          </div>
          <div className="text-sm font-semibold text-white">
            {event.description}
          </div>
          <div className="flex items-center justify-between">
            <span className="text-xs text-gray-500">
              {formatTimestamp(event.timestamp)}
            </span>
            {isAnomaly && !event.notified && onNotify && (
              <Button
                size="sm"
                variant="default"
                onClick={() => onNotify(event.id)}
              >
                Notify
              </Button>
            )}
            {event.notified && (
              <Badge variant="outline">Notified</Badge>
            )}
          </div>
        </div>
      </div>
    </div>
  )
}
```

### Organisms (Complex Components)
- **Purpose**: Complex UI sections with business logic and data management
- **Examples**: VideoGrid, EventSidebar, Navbar, CameraConfigForm
- **Rules**:
  - Orchestrate molecules and atoms
  - May fetch/mutate data
  - Manage complex local state
  - Handle user interactions

**Example: VideoGrid.tsx**
```typescript
// src/components/organisms/VideoGrid.tsx
'use client'

import { CameraCard } from '@/components/molecules/CameraCard'
import { Skeleton } from '@/components/atoms/Skeleton'
import { useCameras } from '@/lib/hooks/useCameras'
import { useEventStore } from '@/lib/stores/eventStore'

export function VideoGrid() {
  const { cameras, isLoading } = useCameras()
  const recentEvents = useEventStore((state) => state.recentEventsByCamera)

  if (isLoading) {
    return <VideoGridSkeleton />
  }

  const gridCols = cameras.length <= 6 ? 'grid-cols-3' : 'grid-cols-4'

  return (
    <div className={`grid ${gridCols} gap-4 p-6`}>
      {cameras.map((camera) => (
        <CameraCard
          key={camera.id}
          camera={camera}
          recentEvent={recentEvents[camera.id]}
        />
      ))}
    </div>
  )
}

function VideoGridSkeleton() {
  return (
    <div className="grid grid-cols-3 gap-4 p-6">
      {Array.from({ length: 6 }).map((_, i) => (
        <Skeleton key={i} className="aspect-video rounded-lg" />
      ))}
    </div>
  )
}
```

### Templates (Page Layouts)
- **Purpose**: Page-level composition of organisms
- **Examples**: DashboardLayout, AuthLayout
- **Rules**:
  - Define page structure
  - Compose organisms and layout elements
  - Manage page-level state coordination
  - Handle routing and navigation

---

## Server vs Client Components

Next.js 15 App Router defaults to **Server Components** for performance and security. Use Client Components only when necessary.

### Server Components (Default)
**Use for**:
- Static content (navbar, footer)
- Initial data fetching (camera list)
- SEO-critical content
- Secure operations (API calls with secrets)

**Advantages**:
- Zero JavaScript sent to client
- Direct database/API access
- Automatic code splitting
- Better SEO

**Example**:
```typescript
// src/app/(dashboard)/cameras/page.tsx (Server Component)
import { getCameras } from '@/lib/api/queries'
import { CameraConfigList } from '@/components/organisms/CameraConfigList'

export default async function CamerasPage() {
  const cameras = await getCameras()

  return (
    <div className="p-6">
      <h1>Camera Management</h1>
      <CameraConfigList initialCameras={cameras} />
    </div>
  )
}
```

### Client Components (`'use client'`)
**Use for**:
- Interactive elements (buttons, forms)
- Browser APIs (EventSource, sessionStorage)
- React hooks (useState, useEffect)
- Event listeners
- Video players
- Real-time subscriptions

**Examples**:
- VideoPlayer (Video.js integration)
- EventSidebar (SSE connection)
- CameraConfigForm (form state)

**Example**:
```typescript
// src/components/organisms/EventSidebar.tsx (Client Component)
'use client'

import { useEventStream } from '@/lib/hooks/useEventStream'
import { useEventStore } from '@/lib/stores/eventStore'
import { EventCard } from '@/components/molecules/EventCard'
import { FilterToggle } from '@/components/molecules/FilterToggle'

export function EventSidebar() {
  useEventStream() // Establishes SSE connection
  const events = useEventStore((state) => state.filteredEvents)
  const filter = useEventStore((state) => state.filter)
  const setFilter = useEventStore((state) => state.setFilter)

  return (
    <div className="w-[30%] bg-gray-950 border-l border-gray-800 flex flex-col">
      <div className="p-4 border-b border-gray-800">
        <h2 className="text-lg font-semibold mb-4">Event Registry</h2>
        <FilterToggle value={filter} onChange={setFilter} />
      </div>
      <div className="flex-1 overflow-y-auto p-4 space-y-3">
        {events.map((event) => (
          <EventCard key={event.id} event={event} />
        ))}
      </div>
    </div>
  )
}
```

### Composition Pattern
Server Components can import Client Components, but not vice versa. Pass server-fetched data as props to Client Components.

```typescript
// Server Component wrapping Client Component
import { ClientVideoPlayer } from '@/components/organisms/VideoPlayer' // 'use client'

export default async function DashboardPage() {
  const cameras = await fetchCameras() // Server-side fetch

  return (
    <div>
      {cameras.map((camera) => (
        <ClientVideoPlayer
          key={camera.id}
          cameraId={camera.id}
          streamUrl={`/api/cameras/${camera.id}/stream`}
        />
      ))}
    </div>
  )
}
```

---

## Authentication Architecture

### Overview

Alquimia Argos uses **simple token-based authentication**. The user enters a static bearer token on the login page. The token is stored in session storage and sent as a `Bearer` header with all API requests. There is no OAuth flow, no Keycloak, and no session management framework.

### Authentication Flow

1. **User visits `/dashboard`** -> Middleware checks for token cookie
2. **No token** -> Redirect to `/login`
3. **User enters token on login page** -> Token sent to BFF for validation (test request: `GET /api/cameras`)
4. **BFF responds 200** -> Token is valid, store in session storage + cookie, redirect to `/dashboard`
5. **BFF responds 401/403** -> Show error on login page
6. **Subsequent requests** -> Token included as `Authorization: Bearer <token>` header
7. **Logout** -> Clear token from session storage and cookie, redirect to `/login`

### Token Management

**File: `src/lib/auth/token.ts`**
```typescript
// src/lib/auth/token.ts

export function getToken(): string | null {
  if (typeof window !== 'undefined') {
    return sessionStorage.getItem('auth_token')
  }
  return null
}

export function setToken(token: string): void {
  sessionStorage.setItem('auth_token', token)
  // Also set as cookie for middleware access
  document.cookie = `auth_token=${token}; path=/; SameSite=Lax`
}

export function clearToken(): void {
  sessionStorage.removeItem('auth_token')
  document.cookie = 'auth_token=; path=/; expires=Thu, 01 Jan 1970 00:00:00 GMT'
}
```

### Login Action

```typescript
// src/lib/api/actions.ts
'use server'

import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'

export async function loginWithToken(token: string) {
  const BFF_URL = process.env.BFF_API_URL || 'http://localhost:8000'

  // Validate token by making a test request to the BFF
  const res = await fetch(`${BFF_URL}/api/cameras`, {
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  })

  if (!res.ok) {
    return { success: false, error: 'Invalid token' }
  }

  // Store token in cookie (accessible by middleware)
  const cookieStore = await cookies()
  cookieStore.set('auth_token', token, {
    path: '/',
    sameSite: 'lax',
    httpOnly: true
  })

  redirect('/dashboard')
}

export async function logout() {
  const cookieStore = await cookies()
  cookieStore.delete('auth_token')
  redirect('/login')
}
```

### Middleware Protection

**File: `src/middleware.ts`**
```typescript
import { NextResponse } from 'next/server'
import type { NextRequest } from 'next/server'

export function middleware(request: NextRequest) {
  const token = request.cookies.get('auth_token')?.value
  const isOnDashboard = request.nextUrl.pathname.startsWith('/dashboard')
  const isOnLogin = request.nextUrl.pathname === '/login'

  if (isOnDashboard && !token) {
    return NextResponse.redirect(new URL('/login', request.url))
  }

  if (isOnLogin && token) {
    return NextResponse.redirect(new URL('/dashboard', request.url))
  }

  return NextResponse.next()
}

export const config = {
  matcher: ['/((?!api|_next/static|_next/image|favicon.ico).*)']
}
```

### API Client with Token

**File: `src/lib/api/client.ts`**
```typescript
import { cookies } from 'next/headers'

const BFF_URL = process.env.BFF_API_URL || 'http://localhost:8000'

// Server-side fetcher (used in Server Components and Server Actions)
export async function fetcher<T>(url: string): Promise<T> {
  const cookieStore = await cookies()
  const token = cookieStore.get('auth_token')?.value

  const res = await fetch(`${BFF_URL}${url}`, {
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` })
    }
  })

  if (!res.ok) {
    throw new Error(`API error: ${res.status}`)
  }

  return res.json()
}

// Client-side fetcher (used in SWR hooks)
export function clientFetcher<T>(url: string): Promise<T> {
  const token = sessionStorage.getItem('auth_token')

  return fetch(`${process.env.NEXT_PUBLIC_BFF_URL}${url}`, {
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` })
    }
  }).then((res) => {
    if (!res.ok) throw new Error(`API error: ${res.status}`)
    return res.json()
  })
}

export async function mutate<T>(
  url: string,
  method: 'POST' | 'PUT' | 'DELETE',
  body?: unknown
): Promise<T> {
  const cookieStore = await cookies()
  const token = cookieStore.get('auth_token')?.value

  const res = await fetch(`${BFF_URL}${url}`, {
    method,
    headers: {
      'Content-Type': 'application/json',
      ...(token && { Authorization: `Bearer ${token}` })
    },
    ...(body && { body: JSON.stringify(body) })
  })

  if (!res.ok) {
    throw new Error(`API error: ${res.status}`)
  }

  return res.json()
}
```

---

## Video Streaming Architecture

### Overview

The frontend never connects directly to camera URLs or the Argos engine. All video streams are proxied through the BFF layer. The frontend requests stream URLs via `/api/cameras/:id/stream`, and Video.js connects to the BFF-provided HLS endpoint.

### Video Flow

```
Frontend                    BFF                     Engine
   │                         │                        │
   │  GET /api/cameras/:id   │                        │
   │  ───────────────────>   │                        │
   │  { streamUrl: ... }     │                        │
   │  <───────────────────   │                        │
   │                         │                        │
   │  Video.js connects to   │                        │
   │  BFF HLS proxy endpoint │                        │
   │  ───────────────────>   │  Proxies to engine    │
   │                         │  ───────────────────> │
   │  <── HLS segments ───   │  <── HLS segments ── │
   │                         │                        │
```

### Video.js Integration

**Component: `VideoPlayer.tsx`**
```typescript
// src/components/organisms/VideoPlayer.tsx
'use client'

import { useEffect, useRef } from 'react'
import videojs from 'video.js'
import 'video.js/dist/video-js.css'
import type { Camera } from '@/lib/types/cameras'

interface VideoPlayerProps {
  camera: Camera
}

const BFF_URL = process.env.NEXT_PUBLIC_BFF_URL || 'http://localhost:8000'

export function VideoPlayer({ camera }: VideoPlayerProps) {
  const videoRef = useRef<HTMLVideoElement>(null)
  const playerRef = useRef<ReturnType<typeof videojs> | null>(null)

  useEffect(() => {
    if (!videoRef.current) return

    // Stream URL points to BFF proxy, NOT directly to engine
    const streamUrl = `/api/cameras/${camera.id}/stream`

    // Initialize Video.js player
    const player = videojs(videoRef.current, {
      autoplay: true,
      muted: true, // Required for autoplay
      controls: true,
      preload: 'auto',
      fluid: true,
      sources: [
        {
          src: `${BFF_URL}${streamUrl}`,
          type: 'application/x-mpegURL' // HLS
        }
      ]
    })

    playerRef.current = player

    // Error handling
    player.on('error', () => {
      console.error('Video.js error:', player.error())
      // Show offline state
    })

    // Cleanup
    return () => {
      if (playerRef.current) {
        playerRef.current.dispose()
        playerRef.current = null
      }
    }
  }, [camera.id])

  return (
    <div data-vjs-player>
      <video ref={videoRef} className="video-js vjs-default-skin" />
    </div>
  )
}
```

### HLS Stream Handling
- **Protocol**: HLS (HTTP Live Streaming)
- **Format**: `.m3u8` playlist with `.ts` segments
- **BFF Role**: Proxy HLS streams from engine to frontend; frontend never sees raw camera URLs
- **Latency Target**: <2 seconds

### Error Recovery
- Auto-reconnect on stream failure (exponential backoff: 1s, 2s, 4s, 8s, max 30s)
- Display offline state after 3 failed reconnection attempts
- Manual retry button for user-initiated reconnection

---

## Real-Time Events (SSE) Architecture

### EventSource Connection

**Hook: `useEventStream.ts`**
```typescript
// src/lib/hooks/useEventStream.ts
'use client'

import { useEffect } from 'react'
import { useEventStore } from '@/lib/stores/eventStore'
import { getToken } from '@/lib/auth/token'

const BFF_URL = process.env.NEXT_PUBLIC_BFF_URL || 'http://localhost:8000'

export function useEventStream() {
  const addEvent = useEventStore((state) => state.addEvent)

  useEffect(() => {
    const token = getToken()
    if (!token) return

    const eventSource = new EventSource(
      `${BFF_URL}/api/events/stream`,
      {
        withCredentials: true
      }
    )

    eventSource.onmessage = (event) => {
      const data = JSON.parse(event.data)
      addEvent(data)
    }

    eventSource.onerror = (error) => {
      console.error('SSE error:', error)
      eventSource.close()
      // Implement reconnection logic with exponential backoff
    }

    return () => {
      eventSource.close()
    }
  }, [addEvent])
}
```

### Event Store (Zustand)

**Store: `eventStore.ts`**
```typescript
// src/lib/stores/eventStore.ts
import { create } from 'zustand'
import type { DetectedEvent } from '@/lib/types/events'

type EventFilter = 'all' | 'normal' | 'anomaly'

interface EventStore {
  events: DetectedEvent[]
  filter: EventFilter
  addEvent: (event: DetectedEvent) => void
  setFilter: (filter: EventFilter) => void
  filteredEvents: DetectedEvent[]
  recentEventsByCamera: Record<string, DetectedEvent>
}

export const useEventStore = create<EventStore>((set, get) => ({
  events: [],
  filter: 'all',

  addEvent: (event) =>
    set((state) => ({
      events: [event, ...state.events].slice(0, 100) // Keep last 100
    })),

  setFilter: (filter) => set({ filter }),

  get filteredEvents() {
    const { events, filter } = get()
    if (filter === 'all') return events
    return events.filter((e) => e.type === filter)
  },

  get recentEventsByCamera() {
    const events = get().events
    return events.reduce((acc, event) => {
      if (!acc[event.cameraId]) {
        acc[event.cameraId] = event
      }
      return acc
    }, {} as Record<string, DetectedEvent>)
  }
}))
```

---

## State Management Strategy

### State Categories

1. **Server State** (SWR)
   - Camera list and details
   - Camera configuration (prompts)

2. **Client State** (Zustand)
   - Real-time events
   - UI state (filters, modals, selected camera)
   - Audio state (muted cameras)

3. **Form State** (React Hook Form)
   - Camera configuration forms
   - Validation errors

4. **URL State** (searchParams)
   - Active camera filter
   - Management tab

### SWR for Server State

```typescript
// src/lib/hooks/useCameras.ts
import useSWR from 'swr'
import { clientFetcher } from '@/lib/api/client'
import type { CameraListResponse } from '@/lib/types/cameras'

export function useCameras() {
  const { data, error, isLoading, mutate } = useSWR<CameraListResponse>(
    '/api/cameras',
    clientFetcher,
    {
      refreshInterval: 30000, // Poll every 30s
      revalidateOnFocus: true
    }
  )

  return {
    cameras: data?.cameras ?? [],
    isLoading,
    error,
    refresh: mutate
  }
}
```

```typescript
// src/lib/hooks/useCamera.ts
import useSWR from 'swr'
import { clientFetcher } from '@/lib/api/client'
import type { Camera } from '@/lib/types/cameras'

export function useCamera(id: string) {
  const { data, error, isLoading, mutate } = useSWR<Camera>(
    id ? `/api/cameras/${id}` : null,
    clientFetcher,
    {
      revalidateOnFocus: true
    }
  )

  return {
    camera: data ?? null,
    isLoading,
    error,
    refresh: mutate
  }
}
```

---

## BFF API Integration

### API Client

See [Authentication Architecture > API Client with Token](#api-client-with-token) for the full `client.ts` implementation.

### Endpoints

| Method | Endpoint | Description |
|--------|----------|-------------|
| `GET` | `/api/cameras` | List all cameras |
| `GET` | `/api/cameras/:id` | Get camera details |
| `POST` | `/api/cameras` | Create a new camera |
| `PUT` | `/api/cameras/:id` | Update a camera |
| `DELETE` | `/api/cameras/:id` | Delete a camera |
| `GET` | `/api/cameras/:id/stream` | HLS stream proxy |
| `GET` | `/api/events/stream` | SSE endpoint for real-time events |

### Server Actions

**File: `src/lib/api/actions.ts`**
```typescript
'use server'

import { mutate } from './client'
import { revalidatePath } from 'next/cache'
import { cookies } from 'next/headers'
import { redirect } from 'next/navigation'
import type { CameraCreateRequest, CameraUpdateRequest } from '@/lib/types/cameras'

// --- Authentication ---

export async function loginWithToken(token: string) {
  const BFF_URL = process.env.BFF_API_URL || 'http://localhost:8000'

  const res = await fetch(`${BFF_URL}/api/cameras`, {
    headers: {
      Authorization: `Bearer ${token}`,
      'Content-Type': 'application/json'
    }
  })

  if (!res.ok) {
    return { success: false, error: 'Invalid token' }
  }

  const cookieStore = await cookies()
  cookieStore.set('auth_token', token, {
    path: '/',
    sameSite: 'lax',
    httpOnly: true
  })

  redirect('/dashboard')
}

export async function logout() {
  const cookieStore = await cookies()
  cookieStore.delete('auth_token')
  redirect('/login')
}

// --- Camera CRUD ---

export async function createCamera(data: CameraCreateRequest) {
  await mutate('/api/cameras', 'POST', data)
  revalidatePath('/dashboard/cameras')
  return { success: true }
}

export async function updateCamera(id: string, data: CameraUpdateRequest) {
  await mutate(`/api/cameras/${id}`, 'PUT', data)
  revalidatePath('/dashboard/cameras')
  return { success: true }
}

export async function deleteCamera(id: string) {
  await mutate(`/api/cameras/${id}`, 'DELETE')
  revalidatePath('/dashboard/cameras')
  return { success: true }
}

```

### Types

**File: `src/lib/types/cameras.ts`**
```typescript
export interface Camera {
  id: string
  name: string
  description: string
  observation_prompt: string
  decisions_prompt: string
  status: 'online' | 'offline' | 'error'
  lastSeen: string
  createdAt: string
  updatedAt: string
}

export interface CameraListResponse {
  cameras: Camera[]
}

export interface CameraCreateRequest {
  name: string
  description: string
  observation_prompt: string
  decisions_prompt: string
}

export interface CameraUpdateRequest {
  name?: string
  description?: string
  observation_prompt?: string
  decisions_prompt?: string
}
```

---

## Routing Strategy

### Route Groups
- **(auth)**: Unauthenticated layout (login)
- **(dashboard)**: Authenticated layout (main app)

### Protected Routes
- `/dashboard/*` -> Requires authentication (token in cookie)
- `/login` -> Public (redirects to dashboard if token present)

### Dynamic Routes (Future)
- `/dashboard/camera/[id]` -> Single camera view
- `/dashboard/events/[id]` -> Event details

---

## Styling Guidelines

### Tailwind CSS Conventions
- Use **utility classes** directly in components
- Extract common patterns to **component variants** (CVA)
- Avoid `@apply` in CSS files (use components instead)

### Color System 
```typescript
// tailwind.config.ts
export default {
  theme: {
    extend: {
      colors: {
        background: '#0a0a0a',
        card: '#1a1a1a',
        border: '#2a2a2a',
        primary: '#ff4444',
        success: '#44ff44',
        muted: '#999999'
      }
    }
  }
}
```

### Responsive Design
- **Mobile-first**: Not in v1 (desktop-only)
- **Breakpoints**: `sm:`, `md:`, `lg:`, `xl:`, `2xl:`
- **Grid**: Use CSS Grid for camera layout

---

## Error Handling

### Error Boundaries
```typescript
// src/app/error.tsx
'use client'

export default function Error({
  error,
  reset
}: {
  error: Error
  reset: () => void
}) {
  return (
    <div className="flex flex-col items-center justify-center min-h-screen">
      <h2>Something went wrong!</h2>
      <button onClick={reset}>Try again</button>
    </div>
  )
}
```

### API Error Handling
- Display toast notifications for API errors
- Retry logic for transient failures
- Fallback UI for failed data fetches

---

## Performance Optimization

### Strategies
- **Code Splitting**: Automatic via Next.js dynamic imports
- **Image Optimization**: Use `next/image` for static assets
- **Bundle Analysis**: `pnpm build` + `@next/bundle-analyzer`
- **Lazy Loading**: Dynamic import for heavy components (Video.js)
- **Memoization**: `React.memo` for expensive renders
- **Virtual Scrolling**: For event list (if >1000 events)

---

## Security Considerations

### XSS Prevention
- React escapes output by default
- Use `dangerouslySetInnerHTML` only with sanitized content
- CSP headers in Next.js config

### CSRF Protection
- SameSite cookies (`Lax`)
- Server Actions include built-in CSRF protection

### Token Security
- **httpOnly cookie** for token (server-side access by middleware and Server Actions)
- **Session storage** for client-side token access (cleared on tab close)
- **HTTPS only** in production

### Input Validation
- Zod schemas for all form inputs
- Server-side validation in Server Actions
- Sanitize configuration inputs

---

## Environment Variables

```bash
# App Configuration
NODE_ENV=development
NEXT_PUBLIC_APP_URL=http://localhost:3000

# BFF API
NEXT_PUBLIC_BFF_URL=http://localhost:8000
BFF_API_URL=http://localhost:8000
```

**Note**: `NEXT_PUBLIC_*` variables are exposed to the browser. `BFF_API_URL` is server-only and used in Server Actions and Server Components.

---

## Diagrams

Refer to the `docs/diagrams/` folder for visual representations:
- `architecture.mmd`: System architecture
- `data-flow.mmd`: Data flow between layers
- `auth-flow.mmd`: Authentication sequence
- `event-flow.mmd`: SSE event lifecycle

---

## Next Steps

1. Review this blueprint with the development team
2. Set up development environment (see SETUP_GUIDE.md)
3. Initialize Next.js project with specified dependencies
4. Implement token-based authentication flow first (blocking)
5. Build component library (atoms -> molecules -> organisms)
6. Integrate video streaming and SSE
7. Implement camera management (CRUD)
8. Testing and optimization

---

**Document Maintained By**: Alquimia Engineering Team
**Last Updated**: February 2026
