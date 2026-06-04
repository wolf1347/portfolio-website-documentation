# Design Doc: Initial Frontend-Backend Connectivity

**Date:** 2026-06-03
**Status:** Draft
**Topic:** Establishing resilient communication between React + TS Frontend and ASP.NET Core Backend.

## 1. Goal
Establish a clean, reusable pattern for the frontend to communicate with the backend. This includes handling standardized states, demonstrating architectural "jitter," and ensuring robust cleanup to prevent unnecessary state updates on unmounted components or hung requests.

## 2. Architecture
- **Frontend:** React (TypeScript) using a custom `useApi` hook.
- **Backend:** ASP.NET Core (C#) Minimal API.
- **Protocol:** REST (JSON over HTTPS).
- **Security:** Environment-aware CORS policy and request timeouts.

## 3. State Machine
All API interactions will follow these standardized states (implemented as a `const` object):
- `IDLE`: Initial state, ready for input.
- `PENDING`: Request is in flight; UI shows processing.
- `SUCCESS`: Request completed successfully; data available.
- `ERROR`: Request failed; error details available.

## 4. Backend Design (ASP.NET Core)

### 4.1 Endpoint: `POST /api/ping`
- **Payload:** `{ "message": string }`
- **Behavior:**
    1. Log incoming message.
    2. **Jitter Simulation:** 
       `int jitterDelay = 1500 + Random.Shared.Next(0, 1000); await Task.Delay(jitterDelay);`
    3. Return `200 OK` with `{ "response": string, "timestamp": DateTime }`.

### 4.2 Middleware & Configuration
- **CORS Ordering:** 
  Middleware must be ordered: `UseRouting` -> `UseCors` -> `MapPost`.
- **Environment Awareness:**
  The allowed CORS origins will be loaded from configuration (`appsettings.json` / `appsettings.Development.json`) to allow easy transition from `localhost` to AWS production environments.

## 5. Frontend Design (React + TS)

### 5.1 `useApi` Hook
- **State Management:** Uses the standardized states defined in Section 3.
- **AbortController:** Every request will initialize an `AbortController`. The request will be cancelled automatically if:
    1. The component unmounts.
    2. A new request is triggered before the old one finishes.
- **Timeout & Timer Management:** 
    - A global timeout (default 10s) will be implemented via `setTimeout`.
    - The timer will be explicitly cleared using `clearTimeout` in the `.finally()` block to ensure no background timers persist after request completion.
- **Modern React 18 Patterns:**
    - Replaces legacy `isMounted` guards with strict `AbortError` handling. If a request is aborted, the catch block will identify the `AbortError` and suppress state updates, preventing leaks natively.
- **Error Categorization:**
    - **Infrastructure Errors:** Timeouts, network drops, or aborts (handled as system-level failures).
    - **Application Errors:** HTTP 4xx/5xx responses (handled as business-level failures).

### 5.2 Demo Component
- **UI:** A simple card with an input and a "Send" button.
- **Interactivity:**
    - Button disabled during `PENDING`.
    - Shows simulated "processing" time (jitter).
    - Displays "Request Timed Out" or "Network Error" clearly if `ERROR` occurs.

## 6. Success Criteria
1. Frontend successfully POSTs to Backend.
2. Backend responds after a variable jitter duration.
3. Frontend UI transitions correctly: `IDLE` -> `PENDING` -> `SUCCESS/ERROR`.
4. Navigating away during a request cancels the fetch and causes no console warnings.
5. Requests hanging beyond the timeout period trigger the `ERROR` state.
