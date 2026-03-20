# Product Requirements Document (PRD)

## Alquimia Argos - Real-Time Video Surveillance Dashboard

**Version**: 2.0
**Date**: February 2026
**Status**: Draft for External Development

---

## Executive Summary

**Alquimia Argos** is a web-based real-time video surveillance dashboard that displays multi-camera feeds and AI-detected events from the Argos video processing engine. The application provides security operators with a unified interface to monitor multiple camera streams, receive real-time event notifications, and configure cameras and their detection prompts.

**Target Users**: Security operators, system administrators
**Platform**: Web application (desktop-first, Chrome/Firefox/Safari)
**Core Value**: Unified real-time surveillance monitoring with AI-powered event detection

---

## Product Overview

### Context

Alquimia Argos is the frontend component of the ecosystem, which processes video streams through an AI engine to detect events and anomalies. The application communicates exclusively with a Backend-For-Frontend (BFF) layer that interfaces with the video processing engine.

### Goals

- Provide real-time monitoring of 8-16 surveillance cameras simultaneously
- Display AI-detected events
- Enable operators to configure cameras and their detection/decision prompts
- Deliver an intuitive, professional surveillance interface
- Support secure access via simple token-based authentication

### Reference Design

The UI/UX should match the provided POC (SentinelAI) with a dark, bold surveillance aesthetic.

---

## User Personas

### Primary Persona: Security Operator

**Role**: Security Operations Center (SOC) Operator
**Goals**:

- Monitor multiple camera feeds simultaneously
- Quickly identify and respond to anomalies
- Understand event context (location, camera, severity)
- Maintain situational awareness across all cameras

**Pain Points**:

- Difficulty tracking events across multiple monitors
- Delayed notification of critical events
- Lack of context in generic security alerts

**Usage Pattern**: 8-12 hour shifts, continuous monitoring

---

### Secondary Persona: System Administrator

**Role**: Security Systems Administrator
**Goals**:

- Configure cameras and their observation/decision prompts
- Optimize false positive rates
- Maintain system uptime and performance

**Pain Points**:

- Complex configuration interfaces
- Difficulty tuning detection sensitivity
- Lack of visibility into system performance

**Usage Pattern**: Periodic configuration updates, troubleshooting

---

## User Stories

### Epic 1: Multi-Camera Video Monitoring

**US-001: View Multiple Camera Feeds**
*As a security operator, I want to view all active camera feeds in a grid layout, so I can monitor multiple locations simultaneously.*

**Acceptance Criteria**:

- Display 6 cameras in 3x2 grid layout (default)
- Support up to 16 cameras in 4x4 grid layout
- Each camera card shows:
  - Red "REC" indicator with animated dot
  - Camera ID (CAM-01, CAM-02, etc.)
  - Location name and sub-location
  - Live video feed or OFFLINE status
  - Audio mute/unmute control
- Grid maintains 16:9 aspect ratio per camera
- Video streams served via BFF proxy endpoint `/api/cameras/:id/stream` 

**US-002: Handle Camera Offline State**
*As a security operator, I want to clearly see when a camera is offline, so I know which areas lack coverage.*

**Acceptance Criteria**:

- Display crossed camera icon for offline cameras
- Show "CAMERA OFFLINE" text in camera card
- Gray out camera card to indicate inactive state
- Maintain camera card position in grid (don't collapse)
- Show last seen timestamp (if available)

**US-003: Control Camera Audio**
*As a security operator, I want to mute/unmute individual camera audio, so I can focus on audio from specific locations.*

**Acceptance Criteria**:

- Audio muted by default (browser autoplay policy compliance)
- Mute/unmute icon visible on each camera card
- Audio state persists during session
- Visual feedback on mute state change
- Only one camera audio active at a time (optional enhancement)

---

### Epic 2: Real-Time Event Notifications

**US-004: View Real-Time Events**
*As a security operator, I want to see AI-detected events as they occur, so I can respond immediately to incidents.*

**Acceptance Criteria**:

- Display events in right sidebar
- Show "Real-time" indicator with green dot
- Auto-scroll to latest events
- Display event cards with:
  - Severity icon (green checkmark for Normal, red triangle for Anomaly)
  - Camera ID and location
  - Event description
  - Timestamp (formatted as "HH:MM:SS")
  - Confidence level (if available)
- Event display latency <500ms from BFF emission
- Connection auto-reconnects if interrupted

**US-005: Filter Events by Severity**
*As a security operator, I want to filter events by Normal/Anomaly, so I can focus on critical incidents.*

**Acceptance Criteria**:

- Filter toggle buttons: "Normal" / "Anomaly"
- Active filter highlighted
- Filter state persists during session
- Event list updates immediately on filter change
- Show count of hidden events (optional)

**US-006: Associate Events with Cameras**
*As a security operator, I want to see which camera triggered an event, so I can quickly locate the incident visually.*

**Acceptance Criteria**:

- Event overlay appears on camera card when event occurs
- Overlay shows alert icon + event description
- Overlay semi-transparent to allow video viewing
- Overlay auto-dismisses after 5 seconds (configurable)
- Click event card to highlight associated camera

**US-007: Manually Notify on Critical Events**
*As a security operator, I want to manually trigger notifications for anomaly events, so I can escalate incidents to response teams.*

**Acceptance Criteria**:

- "Notify" button visible on Anomaly event cards
- Button styled in red for visibility
- Click triggers notification action (visual confirmation)
- Button disables after notification sent
- Show "Notified" status on event card

---

### Epic 3: Camera Configuration

**US-008: Configure Camera**
*As a system administrator, I want to add or edit a camera with its full configuration (connection, frame, observation, decision settings), so that each camera is fully configured in a single form.*

**Acceptance Criteria**:

- Access camera management via settings icon in navbar
- Camera list page showing all configured cameras
- Single form per camera with the following field groups:
  - **General**:
    - `name` (text input, required): Human-readable camera name
    - `description` (text input, required): Camera location or purpose
  - **Connection**:
    - `connection.stream_url` (text input, required): RTSP URL or local device index
    - `connection.protocol` (select: `rtsp` / `local`, default: `rtsp`)
    - `connection.frame_interval` (number input, default: `1.0`): Seconds between frame captures
  - **Frame**:
    - `frame.target_width` (number input, default: `1280`)
    - `frame.target_height` (number input, default: `720`)
    - `frame.jpeg_quality` (number input, default: `85`, range: 1-100)
  - **Observation** (VLM):
    - `observation.model` (text/select, default: `openai:gpt-4o`): Model in `provider:model` format
    - `observation.system_prompt` (textarea, required): Instruction sent to VLM alongside each frame
  - **Decision** (LLM):
    - `decision.model` (text/select, default: `openai:gpt-4o`): Model in `provider:model` format
    - `decision.system_prompt` (textarea, required): Instruction sent to LLM alongside stories
    - `decision.json_schema` (JSON editor or textarea, optional): Custom JSON Schema for structured output
  - **Credentials** (optional, collapsible):
    - `credentials.username` (text input, sensitive)
    - `credentials.password` (password input, sensitive)
- Use sensible defaults for technical fields so operators only need to fill in essential fields (name, stream_url, prompts)
- Form validates all fields before submission (Zod schema validation)
- Adding a camera sends POST to `/api/cameras`
- Editing a camera sends PUT to `/api/cameras/:id`
- Shows success feedback (toast notification) on save
- Shows error feedback with field-level validation messages on failure
- Changes propagate to engine within 30 seconds


---

### Epic 4: Authentication & Security

**US-010: Authenticate**
*As a user, I want to enter a pre-configured access token to authenticate, so I can securely access the dashboard.*

**Acceptance Criteria**:

- Login page with a token input field
- User enters a pre-configured access token
- Token is validated against the BFF (e.g., `POST /api/auth/validate` or a test request to a protected endpoint)
- On success: token stored in sessionStorage (or localStorage), user redirected to dashboard
- On failure: clear error message displayed ("Invalid token")
- Token sent as `Authorization: Bearer <token>` header on all subsequent API requests
- Logout clears stored token and redirects to login page
- No OAuth flow, no Keycloak redirect, no external identity provider

**US-011: Protect Dashboard Routes**
*As a system owner, I want all dashboard routes protected by authentication, so unauthorized users cannot access camera feeds.*

**Acceptance Criteria**:

- Unauthenticated users redirected to login
- Middleware validates presence of stored token on every request
- Invalid or missing tokens trigger re-authentication
- Authorization error shows user-friendly message

---

## Functional Requirements

### FR-001: Multi-Camera Video Grid Display

**Priority**: P0 (Must Have)
**Description**: Display 6-16 camera feeds in responsive grid layout with HLS streaming via BFF proxy

**Requirements**:

- Support HLS video protocol via Video.js library
- 3x2 grid for 6 cameras, 4x4 for 16 cameras
- 16:9 aspect ratio per camera card
- Auto-reconnect on stream failure (exponential backoff)
- Graceful degradation for offline cameras
- Muted autoplay (browser policy compliance)
- **BFF proxy model**: Video.js connects to `/api/cameras/:id/stream` on the BFF; the frontend never knows or accesses camera URLs directly. The BFF proxies the stream from the underlying camera/engine.

---

### FR-002: Real-Time Event Notification System

**Priority**: P0 (Must Have)
**Description**: Display AI-detected events via Server-Sent Events (SSE) with <500ms latency

**Requirements**:

- SSE connection to BFF endpoint `/api/events/stream`
- Event buffering during temporary disconnections
- Auto-reconnect with exponential backoff (max 30s)
- Event classification: Normal (green) / Anomaly (red)
- Event card displays: camera ID, location, description, timestamp
- Scrollable event list with auto-scroll to latest
- Filter events by Normal/Anomaly
- Visual distinction between severity levels

---

### FR-003: Event-Camera Association

**Priority**: P0 (Must Have)
**Description**: Visual association between events and camera feeds

**Requirements**:

- Event overlay on camera card when event triggered
- Overlay shows alert icon + event description
- Semi-transparent overlay (allow video viewing)
- Auto-dismiss after 5 seconds
- Manual dismiss on click
- Click event card to highlight associated camera
- Animation on event occurrence (subtle pulse)

---

### FR-004: Camera Configuration Management

**Priority**: P1 (Should Have)
**Description**: Interface for managing cameras using the engine's nested CameraConfig structure

**Requirements**:

- **Camera list page**: Displays all cameras with their status, name, and description
- **Add camera form**: Single form with grouped fields:
  - General: name, description
  - Connection: stream_url, protocol (select), frame_interval (number)
  - Frame: target_width, target_height, jpeg_quality (number inputs with defaults)
  - Observation: model (text/select), system_prompt (textarea)
  - Decision: model (text/select), system_prompt (textarea), json_schema (JSON editor, optional)
  - Credentials: username, password (optional, sensitive fields)
- **Edit camera form**: Same fields as add form, pre-populated with existing camera data
- **Delete camera**: Confirmation dialog, then `DELETE /api/cameras/:id` API call
- Sensible defaults for technical fields (protocol: `rtsp`, frame_interval: `1.0`, target_width: `1280`, target_height: `720`, jpeg_quality: `85`, model: `openai:gpt-4o`)
- Per-field validation using Zod schemas
- Save/Cancel actions with confirmation
- Visual feedback on save success/failure (toast notifications)
- Changes propagate to engine within 30 seconds

---

### FR-005: Authentication & Authorization

**Priority**: P0 (Must Have)
**Description**: Secure access via simple bearer token authentication

**Requirements**:

- Login page with a token input form
- Token stored in sessionStorage (or cookie) after successful validation
- Token sent as `Authorization: Bearer <token>` header on all BFF API requests
- Middleware checks for a valid token on all protected routes
- Unauthenticated users redirected to `/login`
- Logout clears stored token and redirects to login
- Session timeout: configurable (default 8 hours inactivity)
- Middleware-based route protection

---

## Non-Functional Requirements

### Performance (these are all estimates, TBD)

- **Video Latency**: <2 seconds from live feed
- **Event Display Latency**: <500ms from BFF SSE emission
- **Page Load Time**: <3 seconds on broadband connection
- **Time to Interactive**: <5 seconds
- **Frame Rate**: 24-30 FPS per camera feed (dependent on stream)
- **Concurrent Users**: Support 10+ simultaneous operators

### Scalability

- **Camera Support**: 8-16 cameras simultaneously
- **Event Throughput**: Handle 100+ events/minute across all cameras
- **Session Duration**: Support 12+ hour continuous monitoring sessions
- **Memory Usage**: <2GB browser memory for 16 cameras

### Security

- **Authentication**: Bearer token-based authentication
- **Token Storage**: sessionStorage or secure cookie
- **HTTPS**: Enforce TLS 1.2+ in production
- **XSS Protection**: Content Security Policy headers
- **CSRF Protection**: SameSite cookie attribute (if using cookies for token)
- **API Authentication**: Bearer token on all BFF requests
- **No Secrets in Client**: All sensitive config in server-side env vars
- **Token Validation**: BFF validates token on each request

### Browser Support

- **Chrome**: Latest 2 versions (primary target)
- **Firefox**: Latest 2 versions
- **Safari**: Latest 2 versions
- **Edge**: Latest 2 versions (Chromium-based)
- **Mobile**: Not supported in v1 (future enhancement)

### Reliability

- **Uptime**: 99.5% availability target
- **Error Recovery**: Auto-reconnect on SSE/video stream failure
- **Graceful Degradation**: Show offline state for failed cameras
- **Error Messages**: User-friendly error descriptions
- **Logging**: Client-side error logging to BFF
- **Session Persistence**: Survive page refresh

### Accessibility

- **WCAG 2.1 Level AA**: Minimum compliance target (v2)
- **Keyboard Navigation**: Full keyboard support for controls
- **Color Contrast**: 4.5:1 minimum for text
- **Screen Readers**: Semantic HTML structure
- **Focus Indicators**: Visible focus states

### Maintainability

- **TypeScript**: Strict mode, no `any` types
- **Code Style**: ESLint + Prettier enforcement
- **Component Structure**: Atomic design methodology
- **Documentation**: JSDoc comments on complex functions
- **Type Safety**: Full TypeScript interfaces for all APIs
- **Testing**:
  - **Unit tests** (Vitest): utility functions, Zod schemas, Zustand store logic
  - **Component tests** (React Testing Library): all interactive molecules and organisms (EventCard, CameraCard, CameraConfigForm, VideoPlayer, EventSidebar)
  - **E2E tests** (Playwright): authentication flow, camera CRUD, SSE event display

---

## User Interface Requirements

### Language

- **All UI text in English**

### Layout

- **Desktop-First**: Primary target resolution 1920x1080
- **Dark Theme**: Surveillance aesthetic with dark backgrounds
- **Dashboard Structure**:
  - **Left Section (70% width)**: Camera grid view
  - **Right Sidebar (30% width)**: Real-time event feed
  - **Top Navigation Bar**: Logo, time/date, notifications, settings, user profile

### Visual Design

#### Color Palette (TBD)

- **Background**: #0a0a0a (near black)
- **Card Background**: #1a1a1a (dark gray)
- **Card Border**: #2a2a2a (subtle border)
- **Primary Accent**: #ff4444 (red for alerts, REC indicator)
- **Success/Normal**: #44ff44 (green for normal events)
- **Text Primary**: #ffffff (white)
- **Text Secondary**: #999999 (gray)
- **Overlay**: rgba(0, 0, 0, 0.7) (semi-transparent black)

#### Typography

- **Primary Font**: System sans-serif (Inter, -apple-system, BlinkMacSystemFont)
- **Monospace**: Monospace for camera IDs (JetBrains Mono, Courier New)
- **Heading Size**: 24px (dashboard title), 18px (section headers)
- **Body Size**: 14px (event descriptions), 12px (timestamps, labels)
- **Line Height**: 1.5 for body text

#### Camera Card Elements

1. **Top-Left**: Red "REC" badge with animated pulsing dot
2. **Top-Right**: Camera ID label (e.g., "CAM-01") in monospace
3. **Center**: Video feed (streamed via BFF proxy `/api/cameras/:id/stream`) or OFFLINE state with crossed camera icon
4. **Bottom Section**: Location name + sub-location (e.g., "Main Entrance - North Wing")
5. **Audio Control**: Mute/unmute icon (bottom-right corner)
6. **Event Overlay**: Semi-transparent banner with alert icon + event text (appears on event trigger)

#### Event Feed Elements

1. **Header**: "Event Registry" title + "Real-time" indicator (green dot)
2. **Filter Toggle**: "Normal" / "Anomaly" buttons (toggle style)
3. **Event Cards**:
   - **Severity Icon**: Green checkmark (Normal) / Red triangle (Anomaly)
   - **Camera ID + Location**: Gray text, smaller font
   - **Event Description**: White text, larger font, bold
   - **Timestamp**: Gray text, right-aligned
   - **Notify Button**: Red button for Anomaly events only
4. **Empty State**: "No events detected" message with icon
5. **Auto-Scroll**: Smooth scroll to latest event

#### Camera Management UI

- **Camera List Page**: Table or card grid showing all configured cameras with:
  - Camera name and description
  - Status indicator (online/offline)
  - Edit and Delete action buttons
  - "Add Camera" button
- **Camera Form** (used for both add and edit):
  - **General section**:
    - `name` (text input): Camera display name
    - `description` (text input): Location or purpose description
  - **Connection section**:
    - `connection.stream_url` (text input): RTSP URL or local device index
    - `connection.protocol` (select: rtsp/local): Connection protocol
    - `connection.frame_interval` (number input): Seconds between frame captures
  - **Frame section** (collapsible, defaults provided):
    - `frame.target_width` (number input): Frame width in pixels
    - `frame.target_height` (number input): Frame height in pixels
    - `frame.jpeg_quality` (number input, range slider): JPEG quality 1-100
  - **Observation section**:
    - `observation.model` (text/select): VLM model in provider:model format
    - `observation.system_prompt` (textarea): What to observe/detect — supports multi-line prompt text
  - **Decision section**:
    - `decision.model` (text/select): LLM model in provider:model format
    - `decision.system_prompt` (textarea): How to classify/interpret detections — supports multi-line prompt text
    - `decision.json_schema` (JSON editor or textarea, optional): Custom output schema
  - **Credentials section** (collapsible, optional):
    - `credentials.username` (text input): Camera auth username
    - `credentials.password` (password input): Camera auth password
  - Real-time field validation with Zod schemas
  - Save and Cancel buttons at bottom of form
- **Delete Confirmation**: Modal dialog confirming camera deletion before executing `DELETE /api/cameras/:id`

#### Navigation Bar

- **Left**: App logo + "Alquimia Argos" title
- **Center**: Current time/date display (updates every second)
- **Right**: Notification bell (badge count), settings icon, user profile/logout

### Responsive Behavior (v1)

- **Primary Target**: 1920x1080 desktop
- **Minimum Resolution**: 1366x768
- **Mobile**: Not supported in v1 (future enhancement)
- **Grid Adaptation**: Scale camera cards to fit viewport (maintain aspect ratio)

### Animations & Interactions

- **REC Indicator**: Pulsing animation (fade in/out, 1s interval)
- **Event Arrival**: Slide-in animation for new event cards
- **Event Overlay**: Fade-in on event, fade-out after 5s
- **Button Hover**: Subtle color shift + shadow
- **Loading States**: Skeleton loaders for camera feeds
- **Toast Notifications**: Top-right corner for system messages (success/error)

---

## Technology Stack & Dependencies

### Core

- **Next.js 15** (App Router)
- **React 19** (Server & Client Components)
- **TypeScript 5.3+** (strict mode)

### UI & Styling

- **shadcn/ui** (Radix UI primitives)
- **Tailwind CSS 4**
- **Lucide React** (icons)
- **class-variance-authority (CVA)** (component variants)

### Video Streaming

- **Video.js 8** (HLS player, connects to BFF proxy endpoints)

### State Management & Data Fetching

- **Zustand 4** (client-side state: events, UI)
- **SWR 2** (server state caching: cameras, configs)
- **React Hook Form 7** (form state management)
- **Zod 3** (schema validation)

### Authentication

- Simple bearer token mechanism (no external auth library required)
- Token stored in sessionStorage and sent via `Authorization` header

---

## Success Metrics (TBD)

### User Engagement

- **Session Duration**: Average 8+ hours (shift length)
- **Active Monitoring Time**: >90% of session (minimal idle time)
- **Event Response Time**: <10 seconds from event display to operator action

### System Performance

- **Video Latency**: <2 seconds (target met in 95% of sessions)
- **Event Latency**: <500ms (target met in 99% of events)
- **System Uptime**: 99.5% availability
- **Error Rate**: <1% of page loads result in errors

### Business Impact

- **Incident Response Time**: 20% reduction vs. previous system
- **False Positive Rate**: <10% of total events
- **Missed Incident Rate**: <1% (high-confidence events not detected)

---

## Acceptance Criteria (Overall)

The Alquimia Argos v1 application will be considered **complete and ready for deployment** when:

1. All P0 functional requirements are implemented and verified
2. All user stories have passing acceptance criteria
3. Non-functional requirements meet specified targets (performance, security)
4. UI matches POC design aesthetic with English translations
5. All API integrations with BFF are functional and type-safe
6. Authentication flow works end-to-end with bearer token validation
7. Video streaming supports 16 concurrent cameras via BFF proxy with <2s latency
8. SSE event stream operates reliably with auto-reconnection
9. Camera management (add/edit/delete) saves and applies settings correctly
10. Code passes TypeScript strict mode compilation with no errors
11. ESLint + Prettier pass with zero violations
12. Application runs without console errors in supported browsers
13. Deployment guide enables production deployment
14. Handoff documentation is complete and accurate

---

## Glossary

- **Argos**: Video processing engine
- **BFF**: Backend-For-Frontend - API layer between frontend and engine
- **SSE**: Server-Sent Events - HTTP streaming protocol for real-time updates
- **HLS**: HTTP Live Streaming - Adaptive bitrate streaming protocol for video
- **Bearer Token**: Pre-configured access token used for API authentication
- **shadcn/ui**: Component library built on Radix UI and Tailwind CSS
- **Atomic Design**: Component organization methodology (atoms -> molecules -> organisms -> templates)
- **SOC**: Security Operations Center
- **observation**: Configuration block for the VLM (Vision-Language Model) stage — includes `model`, `kwargs`, and `system_prompt`
- **decision**: Configuration block for the LLM decision stage — includes `model`, `kwargs`, `system_prompt`, and `json_schema`
- **Event**: AI-detected occurrence in video stream
- **Anomaly**: Event classified as abnormal or requiring attention
- **Normal**: Event classified as expected behavior
- **Camera Card**: UI component displaying single camera feed
- **Event Card**: UI component displaying single event in feed
- **BFF Proxy**: Pattern where the frontend accesses camera streams through the BFF rather than directly

---

## Document Control

**Author**: Alquimia Product Team
**Reviewers**: Engineering Lead, Security Architect, UX Designer
**Approval**: Product Owner
**Next Review**: Upon completion of v1 development

**Change Log**:

- v2.0 (Feb 2026): Simplified authentication (bearer token replaces Keycloak/OAuth), merged camera configuration into single form (US-008 absorbs US-009), clarified BFF proxy model for video streaming, updated tech stack (removed NextAuth.js)
- v1.0 (Feb 2026): Initial PRD for outsourcing package
