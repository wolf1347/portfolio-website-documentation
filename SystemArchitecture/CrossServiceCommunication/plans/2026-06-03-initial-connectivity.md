# Initial Frontend-Backend Connectivity Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Establish a resilient communication channel between the React frontend and ASP.NET Core backend with standardized state management and robust error handling.

**Architecture:** A reusable React hook (`useApi`) using `AbortController` and `setTimeout` for request management, connecting to a Minimal API endpoint with simulated jitter delay.

**Tech Stack:** React, TypeScript, ASP.NET Core 10, C#.

---

### Task 1: Backend Configuration & CORS

**Files:**
- Modify: `portfolio-website-backend/portfolio-website-backend/Program.cs`
- Modify: `portfolio-website-backend/portfolio-website-backend/appsettings.Development.json`

- [ ] **Step 1: Add CORS origin to configuration**

```json
{
  "Logging": {
    "LogLevel": {
      "Default": "Information",
      "Microsoft.AspNetCore": "Warning"
    }
  },
  "AllowedOrigins": "http://localhost:5173"
}
```

- [ ] **Step 2: Implement CORS and the Ping endpoint**

```csharp
namespace portfolio_website_backend;

public class Program
{
    public static void Main(string[] args)
    {
        var builder = WebApplication.CreateBuilder(args);

        // Add services
        builder.Services.AddAuthorization();
        builder.Services.AddOpenApi();

        // 1. Define CORS policy from config
        var allowedOrigins = builder.Configuration.GetValue<string>("AllowedOrigins")?.Split(';') ?? Array.Empty<string>();
        builder.Services.AddCors(options =>
        {
            options.AddPolicy("DefaultPolicy", policy =>
            {
                policy.WithOrigins(allowedOrigins)
                      .AllowAnyMethod()
                      .AllowAnyHeader();
            });
        });

        var app = builder.Build();

        if (app.Environment.IsDevelopment())
        {
            app.MapOpenApi();
        }

        app.UseHttpsRedirection();
        
        // 2. Middleware Ordering: UseRouting -> UseCors -> UseAuthorization
        app.UseRouting();
        app.UseCors("DefaultPolicy");
        app.UseAuthorization();

        // 3. Define the Ping endpoint with jitter
        app.MapPost("/api/ping", async (PingRequest request) =>
        {
            // Simulate processing with jitter (1.5s - 2.5s)
            int jitterDelay = 1500 + Random.Shared.Next(0, 1000);
            await Task.Delay(jitterDelay);

            return Results.Ok(new PingResponse(
                $"Echo: {request.Message}",
                DateTime.UtcNow
            ));
        });

        app.Run();
    }
}

public record PingRequest(string Message);
public record PingResponse(string Reply, DateTime Timestamp);
```

- [ ] **Step 3: Verify Backend Build**

Run: `dotnet build portfolio-website-backend/portfolio-website-backend.sln`
Expected: Build succeeded.

---

### Task 2: Frontend API Types & State Definition

**Files:**
- Create: `portfolio-website-frontend/src/types/api.ts`

- [ ] **Step 1: Define API states and response types**

```typescript
export type ApiStatus = 'IDLE' | 'PENDING' | 'SUCCESS' | 'ERROR';

export interface ApiResponse {
  reply: string;
  timestamp: string;
}

export interface ApiState<T> {
  data: T | null;
  status: ApiStatus;
  error: string | null;
}
```

---

### Task 3: Implement useApi Hook

**Files:**
- Create: `portfolio-website-frontend/src/hooks/useApi.ts`

- [ ] **Step 1: Implement the useApi hook with AbortController, Timeout, and Unmount Cleanup**

```typescript
import { useState, useCallback, useRef, useEffect } from 'react';
import { ApiStatus, ApiState } from '../types/api';

const DEFAULT_TIMEOUT = 10000;

export function useApi<TRequest, TResponse>(url: string) {
  const [state, setState] = useState<ApiState<TResponse>>({
    data: null,
    status: 'IDLE',
    error: null,
  });

  const abortControllerRef = useRef<AbortController | null>(null);

  // Cleanup on unmount
  useEffect(() => {
    return () => {
      if (abortControllerRef.current) {
        abortControllerRef.current.abort();
      }
    };
  }, []);

  const trigger = useCallback(async (payload: TRequest) => {
    // 1. Cancel previous request if still pending
    if (abortControllerRef.current) {
      abortControllerRef.current.abort();
    }

    const controller = new AbortController();
    abortControllerRef.current = controller;

    setState({ data: null, status: 'PENDING', error: null });

    let didTimeout = false;
    const timeoutId = setTimeout(() => {
      didTimeout = true;
      controller.abort();
    }, DEFAULT_TIMEOUT);

    try {
      const response = await fetch(url, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify(payload),
        signal: controller.signal,
      });

      if (!response.ok) {
        throw new Error(`Application Error: ${response.status} ${response.statusText}`);
      }

      const data = await response.json() as TResponse;
      setState({ data, status: 'SUCCESS', error: null });
    } catch (err: any) {
      if (err.name === 'AbortError') {
        // Only update state if it was a timeout; otherwise (unmount) stay silent
        if (didTimeout) {
          setState({
            data: null,
            status: 'ERROR',
            error: 'Request Timed Out',
          });
        }
        return;
      }
      setState({
        data: null,
        status: 'ERROR',
        error: err.message || 'Unknown Network Error',
      });
    } finally {
      clearTimeout(timeoutId);
      if (abortControllerRef.current === controller) {
        abortControllerRef.current = null;
      }
    }
  }, [url]);

  return { ...state, trigger };
}
```

---

### Task 4: Create Connectivity Demo Component

**Files:**
- Create: `portfolio-website-frontend/src/components/ConnectivityDemo.tsx`
- Modify: `portfolio-website-frontend/src/App.tsx`

- [ ] **Step 1: Build the Demo UI**

```tsx
import { useState } from 'react';
import { useApi } from '../hooks/useApi';
import { ApiResponse } from '../types/api';

export function ConnectivityDemo() {
  const [message, setMessage] = useState('');
  const { data, status, error, trigger } = useApi<any, ApiResponse>('https://localhost:7073/api/ping');

  const handleSend = () => {
    if (message.trim()) {
      trigger({ message });
    }
  };

  return (
    <div style={{ padding: '20px', border: '1px solid #ccc', borderRadius: '8px', maxWidth: '400px' }}>
      <h2>API Connectivity Test</h2>
      <input
        type="text"
        value={message}
        onChange={(e) => setMessage(e.target.value)}
        placeholder="Type a message..."
        disabled={status === 'PENDING'}
      />
      <button onClick={handleSend} disabled={status === 'PENDING' || !message.trim()}>
        {status === 'PENDING' ? 'Processing...' : 'Send to Backend'}
      </button>

      <div style={{ marginTop: '20px' }}>
        <strong>Status:</strong> {status}
        {status === 'SUCCESS' && data && (
          <div style={{ color: 'green', marginTop: '10px' }}>
            <p><strong>Reply:</strong> {data.reply}</p>
            <p><small>Timestamp: {new Date(data.timestamp).toLocaleString()}</small></p>
          </div>
        )}
        {status === 'ERROR' && (
          <div style={{ color: 'red', marginTop: '10px' }}>
            <p><strong>Error:</strong> {error}</p>
          </div>
        )}
      </div>
    </div>
  );
}
```

- [ ] **Step 2: Update App.tsx to include the demo**

```tsx
import { ConnectivityDemo } from './components/ConnectivityDemo'

function App() {
  return (
    <div style={{ padding: '40px' }}>
      <h1>Portfolio Website</h1>
      <p>Clean slate for the React + TypeScript frontend.</p>
      <hr />
      <ConnectivityDemo />
    </div>
  )
}

export default App
```

---

### Task 5: End-to-End Verification

- [ ] **Step 1: Start Backend**
Run: `dotnet run --project portfolio-website-backend/portfolio-website-backend/portfolio-website-backend.csproj`
Expected: Backend starts on `https://localhost:7073`.

- [ ] **Step 2: Start Frontend**
Run: `npm run dev` in `portfolio-website-frontend`
Expected: Frontend starts on `http://localhost:5173`.

- [ ] **Step 3: Test Communication**
Action: Type "Hello" in the UI and click "Send".
Expected: Button shows "Processing...", status becomes `PENDING`. After ~2s, status becomes `SUCCESS` and "Echo: Hello" appears.

- [ ] **Step 4: Test Abort on Unmount**
Action: Trigger a request and then refresh the page or simulate unmount.
Expected: Check Browser Network tab; the pending request should show "Canceled".
