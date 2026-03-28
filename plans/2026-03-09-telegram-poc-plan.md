# Telegram PoC Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Standalone demo app — run it, enter Telegram API keys, login via MTProto or Bot API, browse chats, send messages.

**Architecture:** Go HTTP server serves React SPA + REST API. Two Telegram clients (gotd/td for MTProto, go-telegram/bot for Bot API) managed in-memory. Smart routing redirects user through setup → login → chats automatically.

**Tech Stack:** Go 1.24+, gotd/td (MTProto), go-telegram/bot (Bot API), React 19, React Router 7, Vite, TypeScript

---

## Task 1: Project Scaffold

**Files:**
- Create: `poc/telegram-demo/go.mod`
- Create: `poc/telegram-demo/main.go`
- Create: `poc/telegram-demo/Makefile`
- Create: `poc/telegram-demo/.gitignore`

**Step 1: Create directory structure**

```bash
mkdir -p poc/telegram-demo/internal/telegram
mkdir -p poc/telegram-demo/web
```

**Step 2: Initialize Go module**

```bash
cd poc/telegram-demo
go mod init github.com/messenger-sf/telegram-poc
```

**Step 3: Create main.go (minimal — just prints "starting")**

```go
// poc/telegram-demo/main.go
package main

import (
	"fmt"
	"os"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}

func run() error {
	fmt.Println("Telegram PoC starting...")
	return nil
}
```

**Step 4: Create Makefile**

```makefile
# poc/telegram-demo/Makefile
.PHONY: run build-web dev

PORT ?= 8080

# Build React + run Go server
run: build-web
	@echo "Starting server on http://localhost:$(PORT)"
	go run . -port $(PORT)

# Build React production bundle
build-web:
	cd web && npm install && npm run build

# Dev mode: just Go server (assumes web already built)
dev:
	go run . -port $(PORT)
```

**Step 5: Create .gitignore**

```gitignore
# poc/telegram-demo/.gitignore
web/node_modules/
web/dist/
```

**Step 6: Verify it compiles**

Run: `cd poc/telegram-demo && go build .`
Expected: Success, no errors

**Step 7: Commit**

```bash
git add poc/telegram-demo/
git commit -m "feat(poc): scaffold telegram demo project"
```

---

## Task 2: React App Scaffold

**Files:**
- Create: `poc/telegram-demo/web/package.json`
- Create: `poc/telegram-demo/web/tsconfig.json`
- Create: `poc/telegram-demo/web/vite.config.ts`
- Create: `poc/telegram-demo/web/index.html`
- Create: `poc/telegram-demo/web/src/main.tsx`
- Create: `poc/telegram-demo/web/src/App.tsx`
- Create: `poc/telegram-demo/web/src/api.ts`
- Create: `poc/telegram-demo/web/src/app.css`
- Create: `poc/telegram-demo/web/src/pages/Setup.tsx`
- Create: `poc/telegram-demo/web/src/pages/Login.tsx`
- Create: `poc/telegram-demo/web/src/pages/Chats.tsx`
- Create: `poc/telegram-demo/web/src/pages/Chat.tsx`
- Create: `poc/telegram-demo/web/src/components/Nav.tsx`
- Create: `poc/telegram-demo/web/src/components/AuthGuard.tsx`

**Step 1: Initialize Vite + React project**

```bash
cd poc/telegram-demo/web
npm create vite@latest . -- --template react-ts
```

If it asks to overwrite, say yes (directory is empty).

**Step 2: Install React Router**

```bash
cd poc/telegram-demo/web
npm install react-router
```

**Step 3: Create API helper**

```typescript
// poc/telegram-demo/web/src/api.ts
const BASE = "/api";

export async function apiFetch<T>(
  path: string,
  opts?: RequestInit
): Promise<T> {
  const res = await fetch(`${BASE}${path}`, {
    headers: { "Content-Type": "application/json", ...opts?.headers },
    ...opts,
  });
  if (!res.ok) {
    const body = await res.json().catch(() => ({ error: res.statusText }));
    throw new Error(body.error || res.statusText);
  }
  return res.json();
}

// Types
export interface AppStatus {
  stage: "needs_config" | "needs_auth" | "ready";
  mtproto: { configured: boolean; connected: boolean; user?: { id: number; first_name: string; username: string } };
  bot: { configured: boolean; connected: boolean; username?: string };
}

export interface Chat {
  id: string;
  name: string;
  type: string; // "user" | "group" | "channel"
  last_message: string;
  source: "mtproto" | "bot";
}

export interface Message {
  id: number;
  sender: string;
  text: string;
  date: string;
  outgoing: boolean;
}
```

**Step 4: Create Nav component**

```tsx
// poc/telegram-demo/web/src/components/Nav.tsx
import { NavLink } from "react-router";
import { AppStatus } from "../api";

export function Nav({ status }: { status: AppStatus | null }) {
  const stage = status?.stage ?? "needs_config";
  const canLogin = stage !== "needs_config";
  const canChat = stage === "ready";

  return (
    <nav className="nav">
      <div className="nav-brand">Telegram PoC</div>
      <div className="nav-links">
        <NavLink to="/setup" className={({ isActive }) => isActive ? "active" : ""}>
          Setup
          <span className={`dot ${status?.mtproto.configured || status?.bot.configured ? "green" : "red"}`} />
        </NavLink>
        <NavLink
          to="/login"
          className={({ isActive }) => `${isActive ? "active" : ""} ${!canLogin ? "disabled" : ""}`}
          onClick={(e) => !canLogin && e.preventDefault()}
        >
          Login
          <span className={`dot ${status?.mtproto.connected ? "green" : canLogin ? "yellow" : "red"}`} />
        </NavLink>
        <NavLink
          to="/chats"
          className={({ isActive }) => `${isActive ? "active" : ""} ${!canChat ? "disabled" : ""}`}
          onClick={(e) => !canChat && e.preventDefault()}
        >
          Chats
          <span className={`dot ${canChat ? "green" : "red"}`} />
        </NavLink>
      </div>
    </nav>
  );
}
```

**Step 5: Create AuthGuard component**

```tsx
// poc/telegram-demo/web/src/components/AuthGuard.tsx
import { useEffect, useState } from "react";
import { Outlet, useNavigate, useLocation } from "react-router";
import { apiFetch, AppStatus } from "../api";
import { Nav } from "./Nav";

const STAGE_ROUTES: Record<string, string> = {
  needs_config: "/setup",
  needs_auth: "/login",
  ready: "/chats",
};

export function AuthGuard() {
  const [status, setStatus] = useState<AppStatus | null>(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const navigate = useNavigate();
  const location = useLocation();

  const refreshStatus = async () => {
    try {
      const s = await apiFetch<AppStatus>("/status");
      setStatus(s);
      setError(null);
      return s;
    } catch (e: any) {
      setError(e.message);
      return null;
    } finally {
      setLoading(false);
    }
  };

  useEffect(() => {
    refreshStatus().then((s) => {
      if (s && location.pathname === "/") {
        navigate(STAGE_ROUTES[s.stage] ?? "/setup", { replace: true });
      }
    });
  }, []);

  // Redirect on stage change
  useEffect(() => {
    if (!status) return;
    const target = STAGE_ROUTES[status.stage];
    if (target && location.pathname === "/") {
      navigate(target, { replace: true });
    }
  }, [status?.stage]);

  if (loading) return <div className="loading">Loading...</div>;
  if (error) return <div className="error">API Error: {error}. Is the server running?</div>;

  return (
    <>
      <Nav status={status} />
      <main className="content">
        <Outlet context={{ status, refreshStatus }} />
      </main>
    </>
  );
}

// Hook for child pages
import { useOutletContext } from "react-router";

export function useAppContext() {
  return useOutletContext<{ status: AppStatus; refreshStatus: () => Promise<AppStatus | null> }>();
}
```

**Step 6: Create placeholder pages**

```tsx
// poc/telegram-demo/web/src/pages/Setup.tsx
import { useState } from "react";
import { useNavigate } from "react-router";
import { apiFetch } from "../api";
import { useAppContext } from "../components/AuthGuard";

export function Setup() {
  const { status, refreshStatus } = useAppContext();
  const navigate = useNavigate();
  const [apiId, setApiId] = useState("");
  const [apiHash, setApiHash] = useState("");
  const [botToken, setBotToken] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [saving, setSaving] = useState(false);

  const handleSubmit = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setSaving(true);
    try {
      await apiFetch("/config", {
        method: "POST",
        body: JSON.stringify({
          api_id: apiId ? parseInt(apiId, 10) : 0,
          api_hash: apiHash,
          bot_token: botToken,
        }),
      });
      const s = await refreshStatus();
      if (s?.stage === "needs_auth") navigate("/login");
      else if (s?.stage === "ready") navigate("/chats");
    } catch (e: any) {
      setError(e.message);
    } finally {
      setSaving(false);
    }
  };

  return (
    <div className="page">
      <h1>Setup</h1>
      <p>Enter your Telegram credentials. MTProto requires API ID + Hash. Bot API requires a Bot Token.</p>
      {error && <div className="error">{error}</div>}
      <form onSubmit={handleSubmit}>
        <fieldset>
          <legend>MTProto (User Account)</legend>
          <label>
            API ID
            <input type="number" value={apiId} onChange={(e) => setApiId(e.target.value)} placeholder="12345678" />
          </label>
          <label>
            API Hash
            <input type="text" value={apiHash} onChange={(e) => setApiHash(e.target.value)} placeholder="abcdef1234567890..." />
          </label>
        </fieldset>
        <fieldset>
          <legend>Bot API (Optional)</legend>
          <label>
            Bot Token
            <input type="text" value={botToken} onChange={(e) => setBotToken(e.target.value)} placeholder="123456:ABC-DEF1234..." />
          </label>
        </fieldset>
        <button type="submit" disabled={saving || (!apiId && !botToken)}>
          {saving ? "Connecting..." : "Connect"}
        </button>
      </form>
      <div className="status-box">
        <h3>Status</h3>
        <p>MTProto: {status?.mtproto.configured ? "Configured" : "Not configured"}</p>
        <p>Bot API: {status?.bot.configured ? "Configured" : "Not configured"}</p>
      </div>
    </div>
  );
}
```

```tsx
// poc/telegram-demo/web/src/pages/Login.tsx
import { useState } from "react";
import { useNavigate } from "react-router";
import { apiFetch } from "../api";
import { useAppContext } from "../components/AuthGuard";

type Step = "phone" | "code" | "2fa" | "done";

export function Login() {
  const { status, refreshStatus } = useAppContext();
  const navigate = useNavigate();
  const [step, setStep] = useState<Step>("phone");
  const [phone, setPhone] = useState("");
  const [phoneCodeHash, setPhoneCodeHash] = useState("");
  const [code, setCode] = useState("");
  const [password, setPassword] = useState("");
  const [error, setError] = useState<string | null>(null);
  const [loading, setLoading] = useState(false);

  // Already logged in?
  if (status?.mtproto.connected && step !== "done") {
    return (
      <div className="page">
        <h1>Login</h1>
        <div className="success">
          Logged in as {status.mtproto.user?.first_name} (@{status.mtproto.user?.username})
        </div>
        <button onClick={() => navigate("/chats")}>Go to Chats</button>
      </div>
    );
  }

  const handlePhone = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      const res = await apiFetch<{ phone_code_hash: string }>("/auth/phone", {
        method: "POST",
        body: JSON.stringify({ phone }),
      });
      setPhoneCodeHash(res.phone_code_hash);
      setStep("code");
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  const handleCode = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      const res = await apiFetch<{ status: string }>("/auth/code", {
        method: "POST",
        body: JSON.stringify({ phone, phone_code_hash: phoneCodeHash, code }),
      });
      if (res.status === "2fa_required") {
        setStep("2fa");
      } else {
        setStep("done");
        await refreshStatus();
        navigate("/chats");
      }
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  const handle2FA = async (e: React.FormEvent) => {
    e.preventDefault();
    setError(null);
    setLoading(true);
    try {
      await apiFetch("/auth/2fa", {
        method: "POST",
        body: JSON.stringify({ password }),
      });
      setStep("done");
      await refreshStatus();
      navigate("/chats");
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="page">
      <h1>Login — MTProto</h1>
      {error && <div className="error">{error}</div>}

      {step === "phone" && (
        <form onSubmit={handlePhone}>
          <label>
            Phone Number (international format)
            <input type="tel" value={phone} onChange={(e) => setPhone(e.target.value)} placeholder="+1234567890" required />
          </label>
          <button type="submit" disabled={loading}>{loading ? "Sending..." : "Send Code"}</button>
        </form>
      )}

      {step === "code" && (
        <form onSubmit={handleCode}>
          <p>Code sent to {phone}</p>
          <label>
            Verification Code
            <input type="text" value={code} onChange={(e) => setCode(e.target.value)} placeholder="12345" required autoFocus />
          </label>
          <button type="submit" disabled={loading}>{loading ? "Verifying..." : "Verify Code"}</button>
        </form>
      )}

      {step === "2fa" && (
        <form onSubmit={handle2FA}>
          <p>Two-factor authentication required</p>
          <label>
            2FA Password
            <input type="password" value={password} onChange={(e) => setPassword(e.target.value)} required autoFocus />
          </label>
          <button type="submit" disabled={loading}>{loading ? "Verifying..." : "Submit"}</button>
        </form>
      )}

      {step === "done" && (
        <div className="success">Login successful! Redirecting...</div>
      )}
    </div>
  );
}
```

```tsx
// poc/telegram-demo/web/src/pages/Chats.tsx
import { useEffect, useState } from "react";
import { useNavigate } from "react-router";
import { apiFetch, Chat } from "../api";
import { useAppContext } from "../components/AuthGuard";

export function Chats() {
  const { status } = useAppContext();
  const navigate = useNavigate();
  const [chats, setChats] = useState<Chat[]>([]);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState<string | null>(null);
  const [filter, setFilter] = useState<"all" | "mtproto" | "bot">("all");

  useEffect(() => {
    loadChats();
  }, [filter]);

  const loadChats = async () => {
    setLoading(true);
    try {
      const data = await apiFetch<Chat[]>(`/chats?source=${filter}`);
      setChats(data);
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  return (
    <div className="page">
      <h1>Chats</h1>
      <div className="filter-bar">
        <button className={filter === "all" ? "active" : ""} onClick={() => setFilter("all")}>All</button>
        {status?.mtproto.connected && (
          <button className={filter === "mtproto" ? "active" : ""} onClick={() => setFilter("mtproto")}>MTProto</button>
        )}
        {status?.bot.connected && (
          <button className={filter === "bot" ? "active" : ""} onClick={() => setFilter("bot")}>Bot</button>
        )}
      </div>
      {error && <div className="error">{error}</div>}
      {loading ? (
        <div className="loading">Loading chats...</div>
      ) : chats.length === 0 ? (
        <div className="empty">No chats found</div>
      ) : (
        <ul className="chat-list">
          {chats.map((chat) => (
            <li key={`${chat.source}-${chat.id}`} onClick={() => navigate(`/chats/${chat.id}?source=${chat.source}`)}>
              <div className="chat-avatar">{chat.name[0]?.toUpperCase()}</div>
              <div className="chat-info">
                <div className="chat-name">{chat.name}</div>
                <div className="chat-preview">{chat.last_message}</div>
              </div>
              <span className={`badge ${chat.source}`}>{chat.source}</span>
            </li>
          ))}
        </ul>
      )}
    </div>
  );
}
```

```tsx
// poc/telegram-demo/web/src/pages/Chat.tsx
import { useEffect, useState, useRef } from "react";
import { useParams, useSearchParams } from "react-router";
import { apiFetch, Message } from "../api";
import { useAppContext } from "../components/AuthGuard";

export function Chat() {
  const { id } = useParams();
  const [searchParams] = useSearchParams();
  const source = searchParams.get("source") || "mtproto";
  const { status } = useAppContext();
  const [messages, setMessages] = useState<Message[]>([]);
  const [text, setText] = useState("");
  const [loading, setLoading] = useState(true);
  const [sending, setSending] = useState(false);
  const [error, setError] = useState<string | null>(null);
  const bottomRef = useRef<HTMLDivElement>(null);

  useEffect(() => {
    loadMessages();
  }, [id, source]);

  useEffect(() => {
    bottomRef.current?.scrollIntoView({ behavior: "smooth" });
  }, [messages]);

  const loadMessages = async () => {
    setLoading(true);
    try {
      const data = await apiFetch<Message[]>(`/chats/${id}/messages?source=${source}&limit=50`);
      setMessages(data);
    } catch (e: any) {
      setError(e.message);
    } finally {
      setLoading(false);
    }
  };

  const sendMessage = async (e: React.FormEvent) => {
    e.preventDefault();
    if (!text.trim()) return;
    setSending(true);
    try {
      await apiFetch("/send", {
        method: "POST",
        body: JSON.stringify({ chat_id: id, text: text.trim(), source }),
      });
      setText("");
      await loadMessages();
    } catch (e: any) {
      setError(e.message);
    } finally {
      setSending(false);
    }
  };

  return (
    <div className="page chat-page">
      <div className="chat-header">
        <h2>Chat {id}</h2>
        <span className={`badge ${source}`}>{source}</span>
      </div>
      {error && <div className="error">{error}</div>}
      <div className="messages">
        {loading ? (
          <div className="loading">Loading messages...</div>
        ) : messages.length === 0 ? (
          <div className="empty">No messages yet</div>
        ) : (
          messages.map((msg) => (
            <div key={msg.id} className={`message ${msg.outgoing ? "outgoing" : "incoming"}`}>
              <div className="message-sender">{msg.sender}</div>
              <div className="message-text">{msg.text}</div>
              <div className="message-date">{new Date(msg.date).toLocaleTimeString()}</div>
            </div>
          ))
        )}
        <div ref={bottomRef} />
      </div>
      <form className="send-bar" onSubmit={sendMessage}>
        <input
          type="text"
          value={text}
          onChange={(e) => setText(e.target.value)}
          placeholder="Type a message..."
          disabled={sending}
          autoFocus
        />
        <button type="submit" disabled={sending || !text.trim()}>
          {sending ? "..." : "Send"}
        </button>
      </form>
    </div>
  );
}
```

**Step 7: Create App.tsx with router**

```tsx
// poc/telegram-demo/web/src/App.tsx
import { createBrowserRouter, RouterProvider, Navigate } from "react-router";
import { AuthGuard } from "./components/AuthGuard";
import { Setup } from "./pages/Setup";
import { Login } from "./pages/Login";
import { Chats } from "./pages/Chats";
import { Chat } from "./pages/Chat";

const router = createBrowserRouter([
  {
    path: "/",
    Component: AuthGuard,
    children: [
      { index: true, element: <Navigate to="/setup" replace /> },
      { path: "setup", Component: Setup },
      { path: "login", Component: Login },
      { path: "chats", Component: Chats },
      { path: "chats/:id", Component: Chat },
    ],
  },
]);

export default function App() {
  return <RouterProvider router={router} />;
}
```

**Step 8: Create main.tsx**

```tsx
// poc/telegram-demo/web/src/main.tsx
import React from "react";
import ReactDOM from "react-dom/client";
import App from "./App";
import "./app.css";

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <App />
  </React.StrictMode>
);
```

**Step 9: Create app.css (minimal clean styling)**

```css
/* poc/telegram-demo/web/src/app.css */
* { margin: 0; padding: 0; box-sizing: border-box; }

body {
  font-family: -apple-system, BlinkMacSystemFont, "Segoe UI", Roboto, sans-serif;
  background: #f5f5f5;
  color: #333;
}

/* Nav */
.nav {
  display: flex;
  align-items: center;
  gap: 2rem;
  padding: 0.75rem 1.5rem;
  background: #fff;
  border-bottom: 1px solid #ddd;
  position: sticky;
  top: 0;
  z-index: 10;
}
.nav-brand { font-weight: 700; font-size: 1.1rem; }
.nav-links { display: flex; gap: 1rem; }
.nav-links a {
  display: flex;
  align-items: center;
  gap: 0.4rem;
  padding: 0.4rem 0.8rem;
  border-radius: 6px;
  text-decoration: none;
  color: #555;
  font-weight: 500;
  transition: background 0.15s;
}
.nav-links a:hover { background: #f0f0f0; }
.nav-links a.active { background: #e8f0fe; color: #1a73e8; }
.nav-links a.disabled { opacity: 0.4; cursor: not-allowed; }

.dot {
  width: 8px;
  height: 8px;
  border-radius: 50%;
  display: inline-block;
}
.dot.green { background: #34a853; }
.dot.yellow { background: #fbbc04; }
.dot.red { background: #ea4335; }

/* Content */
.content { max-width: 800px; margin: 2rem auto; padding: 0 1rem; }
.page h1 { margin-bottom: 1rem; }

/* Forms */
form { display: flex; flex-direction: column; gap: 0.75rem; max-width: 400px; }
fieldset { border: 1px solid #ddd; border-radius: 8px; padding: 1rem; }
legend { font-weight: 600; padding: 0 0.5rem; }
label { display: flex; flex-direction: column; gap: 0.25rem; font-size: 0.9rem; font-weight: 500; }
input {
  padding: 0.5rem;
  border: 1px solid #ccc;
  border-radius: 6px;
  font-size: 1rem;
}
input:focus { outline: none; border-color: #1a73e8; box-shadow: 0 0 0 2px rgba(26,115,232,0.2); }
button {
  padding: 0.6rem 1.2rem;
  border: none;
  border-radius: 6px;
  background: #1a73e8;
  color: #fff;
  font-size: 0.95rem;
  font-weight: 500;
  cursor: pointer;
  transition: background 0.15s;
}
button:hover { background: #1557b0; }
button:disabled { opacity: 0.5; cursor: not-allowed; }

/* Status */
.status-box { margin-top: 1.5rem; padding: 1rem; background: #fff; border-radius: 8px; border: 1px solid #ddd; }
.status-box h3 { margin-bottom: 0.5rem; }
.error { padding: 0.75rem; background: #fce8e6; color: #c5221f; border-radius: 6px; margin-bottom: 0.75rem; }
.success { padding: 0.75rem; background: #e6f4ea; color: #137333; border-radius: 6px; margin-bottom: 0.75rem; }
.loading { padding: 2rem; text-align: center; color: #666; }
.empty { padding: 2rem; text-align: center; color: #999; }

/* Filter bar */
.filter-bar { display: flex; gap: 0.5rem; margin-bottom: 1rem; }
.filter-bar button { background: #e0e0e0; color: #333; padding: 0.4rem 0.8rem; font-size: 0.85rem; }
.filter-bar button.active { background: #1a73e8; color: #fff; }

/* Chat list */
.chat-list { list-style: none; display: flex; flex-direction: column; gap: 2px; }
.chat-list li {
  display: flex;
  align-items: center;
  gap: 0.75rem;
  padding: 0.75rem;
  background: #fff;
  border-radius: 8px;
  cursor: pointer;
  transition: background 0.15s;
}
.chat-list li:hover { background: #e8f0fe; }
.chat-avatar {
  width: 40px;
  height: 40px;
  border-radius: 50%;
  background: #1a73e8;
  color: #fff;
  display: flex;
  align-items: center;
  justify-content: center;
  font-weight: 600;
  flex-shrink: 0;
}
.chat-info { flex: 1; min-width: 0; }
.chat-name { font-weight: 600; }
.chat-preview { font-size: 0.85rem; color: #666; white-space: nowrap; overflow: hidden; text-overflow: ellipsis; }
.badge {
  font-size: 0.7rem;
  padding: 0.15rem 0.4rem;
  border-radius: 4px;
  font-weight: 600;
  text-transform: uppercase;
}
.badge.mtproto { background: #e8f0fe; color: #1a73e8; }
.badge.bot { background: #fef7e0; color: #e37400; }

/* Chat view */
.chat-page { display: flex; flex-direction: column; height: calc(100vh - 60px - 4rem); }
.chat-header { display: flex; align-items: center; gap: 0.75rem; padding-bottom: 0.75rem; border-bottom: 1px solid #ddd; }
.messages { flex: 1; overflow-y: auto; padding: 1rem 0; display: flex; flex-direction: column; gap: 0.5rem; }
.message { max-width: 70%; padding: 0.5rem 0.75rem; border-radius: 12px; }
.message.incoming { background: #fff; align-self: flex-start; border: 1px solid #ddd; }
.message.outgoing { background: #1a73e8; color: #fff; align-self: flex-end; }
.message-sender { font-size: 0.75rem; font-weight: 600; margin-bottom: 0.15rem; }
.message.outgoing .message-sender { color: rgba(255,255,255,0.8); }
.message-text { font-size: 0.95rem; word-break: break-word; }
.message-date { font-size: 0.7rem; opacity: 0.6; margin-top: 0.15rem; text-align: right; }
.send-bar { display: flex; flex-direction: row; gap: 0.5rem; padding-top: 0.75rem; border-top: 1px solid #ddd; max-width: 100%; }
.send-bar input { flex: 1; }
```

**Step 10: Update vite.config.ts for API proxy in dev mode**

```typescript
// poc/telegram-demo/web/vite.config.ts
import { defineConfig } from "vite";
import react from "@vitejs/plugin-react";

export default defineConfig({
  plugins: [react()],
  server: {
    proxy: {
      "/api": "http://localhost:8080",
    },
  },
  build: {
    outDir: "dist",
  },
});
```

**Step 11: Build and verify**

```bash
cd poc/telegram-demo/web
npm install && npm run build
```

Expected: `dist/` directory created with `index.html` and assets.

**Step 12: Commit**

```bash
git add poc/telegram-demo/web/
git commit -m "feat(poc): scaffold React app with router, pages, and styling"
```

---

## Task 3: Go HTTP Server + Static File Serving

**Files:**
- Create: `poc/telegram-demo/internal/server.go`
- Modify: `poc/telegram-demo/main.go`

**Step 1: Create server.go**

```go
// poc/telegram-demo/internal/server.go
package internal

import (
	"encoding/json"
	"io/fs"
	"log/slog"
	"net/http"
	"strings"
)

type Server struct {
	mux   *http.ServeMux
	state *AppState
}

func NewServer(state *AppState, webFS fs.FS) *Server {
	s := &Server{
		mux:   http.NewServeMux(),
		state: state,
	}
	s.routes(webFS)
	return s
}

func (s *Server) ServeHTTP(w http.ResponseWriter, r *http.Request) {
	s.mux.ServeHTTP(w, r)
}

func (s *Server) routes(webFS fs.FS) {
	// API routes
	s.mux.HandleFunc("GET /api/status", s.handleStatus)
	s.mux.HandleFunc("POST /api/config", s.handleConfig)
	s.mux.HandleFunc("POST /api/auth/phone", s.handleAuthPhone)
	s.mux.HandleFunc("POST /api/auth/code", s.handleAuthCode)
	s.mux.HandleFunc("POST /api/auth/2fa", s.handleAuth2FA)
	s.mux.HandleFunc("GET /api/chats", s.handleChats)
	s.mux.HandleFunc("GET /api/chats/{id}/messages", s.handleMessages)
	s.mux.HandleFunc("POST /api/send", s.handleSend)

	// Static files — serve React SPA
	if webFS != nil {
		fileServer := http.FileServer(http.FS(webFS))
		s.mux.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
			// Try to serve the file directly
			path := strings.TrimPrefix(r.URL.Path, "/")
			if path == "" {
				path = "index.html"
			}
			// Check if file exists
			if f, err := webFS.Open(path); err == nil {
				f.Close()
				fileServer.ServeHTTP(w, r)
				return
			}
			// SPA fallback: serve index.html for all non-file routes
			r.URL.Path = "/"
			fileServer.ServeHTTP(w, r)
		})
	}
}

// Stub handlers — implemented in later tasks

func (s *Server) handleStatus(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusOK, s.state.Status())
}

func (s *Server) handleConfig(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleAuthPhone(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleAuthCode(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleAuth2FA(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleChats(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleMessages(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

func (s *Server) handleSend(w http.ResponseWriter, r *http.Request) {
	writeJSON(w, http.StatusNotImplemented, map[string]string{"error": "not implemented"})
}

// Helpers

func writeJSON(w http.ResponseWriter, status int, v any) {
	w.Header().Set("Content-Type", "application/json")
	w.WriteHeader(status)
	if err := json.NewEncoder(w).Encode(v); err != nil {
		slog.Error("failed to write JSON response", "error", err)
	}
}

func readJSON(r *http.Request, v any) error {
	defer r.Body.Close()
	return json.NewDecoder(r.Body).Decode(v)
}
```

**Step 2: Create state.go**

```go
// poc/telegram-demo/internal/state.go
package internal

import "sync"

type AppConfig struct {
	APIID    int    `json:"api_id"`
	APIHash  string `json:"api_hash"`
	BotToken string `json:"bot_token"`
}

type StatusResponse struct {
	Stage   string         `json:"stage"`
	MTProto MTProtoStatus  `json:"mtproto"`
	Bot     BotStatus      `json:"bot"`
}

type MTProtoStatus struct {
	Configured bool        `json:"configured"`
	Connected  bool        `json:"connected"`
	User       *UserInfo   `json:"user,omitempty"`
}

type BotStatus struct {
	Configured bool   `json:"configured"`
	Connected  bool   `json:"connected"`
	Username   string `json:"username,omitempty"`
}

type UserInfo struct {
	ID        int64  `json:"id"`
	FirstName string `json:"first_name"`
	Username  string `json:"username"`
}

type AppState struct {
	mu     sync.RWMutex
	Config *AppConfig

	// MTProto state
	mtprotoConnected bool
	mtprotoUser      *UserInfo

	// Bot state
	botConnected bool
	botUsername   string
}

func NewAppState() *AppState {
	return &AppState{}
}

func (s *AppState) Status() StatusResponse {
	s.mu.RLock()
	defer s.mu.RUnlock()

	stage := "needs_config"
	if s.Config != nil && (s.Config.APIID != 0 || s.Config.BotToken != "") {
		stage = "needs_auth"
		// If MTProto is connected or only bot is configured and connected
		if s.mtprotoConnected || (s.Config.APIID == 0 && s.botConnected) {
			stage = "ready"
		}
	}

	resp := StatusResponse{
		Stage: stage,
		MTProto: MTProtoStatus{
			Configured: s.Config != nil && s.Config.APIID != 0,
			Connected:  s.mtprotoConnected,
			User:       s.mtprotoUser,
		},
		Bot: BotStatus{
			Configured: s.Config != nil && s.Config.BotToken != "",
			Connected:  s.botConnected,
			Username:   s.botUsername,
		},
	}
	return resp
}
```

**Step 3: Update main.go to wire server**

```go
// poc/telegram-demo/main.go
package main

import (
	"context"
	"flag"
	"fmt"
	"io/fs"
	"log/slog"
	"net/http"
	"os"
	"os/signal"
	"syscall"
	"time"

	"github.com/messenger-sf/telegram-poc/internal"
)

func main() {
	if err := run(); err != nil {
		fmt.Fprintf(os.Stderr, "error: %v\n", err)
		os.Exit(1)
	}
}

func run() error {
	port := flag.Int("port", 8080, "HTTP server port")
	flag.Parse()

	state := internal.NewAppState()

	// Try to load React build from web/dist
	var webFS fs.FS
	if info, err := os.Stat("web/dist"); err == nil && info.IsDir() {
		webFS = os.DirFS("web/dist")
		slog.Info("serving React build from web/dist")
	} else {
		slog.Warn("web/dist not found, API-only mode")
	}

	srv := &http.Server{
		Addr:         fmt.Sprintf(":%d", *port),
		Handler:      internal.NewServer(state, webFS),
		ReadTimeout:  15 * time.Second,
		WriteTimeout: 15 * time.Second,
		IdleTimeout:  60 * time.Second,
	}

	// Graceful shutdown
	ctx, stop := signal.NotifyContext(context.Background(), syscall.SIGINT, syscall.SIGTERM)
	defer stop()

	go func() {
		slog.Info("server starting", "addr", srv.Addr)
		if err := srv.ListenAndServe(); err != nil && err != http.ErrServerClosed {
			slog.Error("server error", "error", err)
			os.Exit(1)
		}
	}()

	<-ctx.Done()
	slog.Info("shutting down...")

	shutdownCtx, cancel := context.WithTimeout(context.Background(), 5*time.Second)
	defer cancel()
	return srv.Shutdown(shutdownCtx)
}
```

**Step 4: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

Expected: Success

**Step 5: Commit**

```bash
git add poc/telegram-demo/main.go poc/telegram-demo/internal/
git commit -m "feat(poc): Go HTTP server with SPA serving and API stubs"
```

---

## Task 4: Config Endpoint

**Files:**
- Modify: `poc/telegram-demo/internal/server.go` (handleConfig)
- Modify: `poc/telegram-demo/internal/state.go` (SetConfig)

**Step 1: Add SetConfig to state**

Add to `state.go`:

```go
func (s *AppState) SetConfig(cfg AppConfig) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.Config = &cfg
	// Reset connection state when config changes
	s.mtprotoConnected = false
	s.mtprotoUser = nil
	s.botConnected = false
	s.botUsername = ""
}
```

**Step 2: Implement handleConfig**

Replace the stub in `server.go`:

```go
func (s *Server) handleConfig(w http.ResponseWriter, r *http.Request) {
	var req AppConfig
	if err := readJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	if req.APIID == 0 && req.BotToken == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "provide api_id+api_hash or bot_token"})
		return
	}
	s.state.SetConfig(req)
	slog.Info("config updated", "api_id", req.APIID, "has_bot_token", req.BotToken != "")
	writeJSON(w, http.StatusOK, s.state.Status())
}
```

**Step 3: Test manually**

```bash
cd poc/telegram-demo && go run . &
curl -s -X POST http://localhost:8080/api/config \
  -H 'Content-Type: application/json' \
  -d '{"api_id":12345,"api_hash":"abc","bot_token":""}' | jq .
kill %1
```

Expected: `{"stage":"needs_auth","mtproto":{"configured":true,"connected":false},"bot":{"configured":false,"connected":false}}`

**Step 4: Commit**

```bash
git add poc/telegram-demo/internal/
git commit -m "feat(poc): implement config endpoint"
```

---

## Task 5: MTProto Client Wrapper

**Files:**
- Create: `poc/telegram-demo/internal/telegram/mtproto.go`

**Step 1: Add gotd/td dependency**

```bash
cd poc/telegram-demo
go get github.com/gotd/td@latest
```

**Step 2: Create MTProto wrapper**

```go
// poc/telegram-demo/internal/telegram/mtproto.go
package telegram

import (
	"context"
	"fmt"
	"log/slog"
	"sync"

	"github.com/gotd/td/telegram"
	"github.com/gotd/td/telegram/auth"
	"github.com/gotd/td/telegram/message"
	"github.com/gotd/td/telegram/query"
	"github.com/gotd/td/tg"
)

type MTProtoClient struct {
	mu     sync.Mutex
	client *telegram.Client
	api    *tg.Client
	cancel context.CancelFunc

	// Auth state
	AuthPhone     string
	PhoneCodeHash string
	Authenticated bool
	User          *UserInfo

	// Channel for auth code (web → gotd)
	codeCh    chan string
	// Channel for 2FA password (web → gotd)
	passwordCh chan string
	// Channel for auth result (gotd → web)
	authResultCh chan authResult
}

type UserInfo struct {
	ID        int64  `json:"id"`
	FirstName string `json:"first_name"`
	Username  string `json:"username"`
}

type authResult struct {
	Need2FA bool
	User    *UserInfo
	Err     error
}

func NewMTProtoClient(apiID int, apiHash string) *MTProtoClient {
	client := telegram.NewClient(apiID, apiHash, telegram.Options{})

	return &MTProtoClient{
		client:       client,
		codeCh:       make(chan string, 1),
		passwordCh:   make(chan string, 1),
		authResultCh: make(chan authResult, 1),
	}
}

// Start connects to Telegram servers. Must be called before auth.
func (m *MTProtoClient) Start(ctx context.Context) error {
	ctx, cancel := context.WithCancel(ctx)
	m.cancel = cancel

	errCh := make(chan error, 1)
	go func() {
		errCh <- m.client.Run(ctx, func(ctx context.Context) error {
			m.mu.Lock()
			m.api = m.client.API()
			m.mu.Unlock()

			slog.Info("MTProto client connected to Telegram")
			// Block until context is cancelled
			<-ctx.Done()
			return ctx.Err()
		})
	}()

	// Wait a moment for connection to establish
	// In production, use a ready channel; for PoC this is fine
	select {
	case err := <-errCh:
		return fmt.Errorf("client stopped: %w", err)
	case <-ctx.Done():
		return ctx.Err()
	default:
		// Give it a moment to connect
		return nil
	}
}

// SendCode initiates phone auth
func (m *MTProtoClient) SendCode(ctx context.Context, phone string) (string, error) {
	m.mu.Lock()
	defer m.mu.Unlock()

	m.AuthPhone = phone

	// We need to run auth inside client.Run context, so we use a goroutine approach
	// For PoC: use the API directly
	sentCode, err := m.client.Auth().SendCode(ctx, phone, auth.SendCodeOptions{})
	if err != nil {
		return "", fmt.Errorf("send code: %w", err)
	}

	switch c := sentCode.(type) {
	case *tg.AuthSentCode:
		m.PhoneCodeHash = c.PhoneCodeHash
		return c.PhoneCodeHash, nil
	default:
		return "", fmt.Errorf("unexpected sent code type: %T", sentCode)
	}
}

// VerifyCode checks the OTP code
func (m *MTProtoClient) VerifyCode(ctx context.Context, phone, phoneCodeHash, code string) (need2FA bool, err error) {
	m.mu.Lock()
	defer m.mu.Unlock()

	result, err := m.client.Auth().SignIn(ctx, phone, code, phoneCodeHash)
	if err != nil {
		// Check if 2FA is required
		if auth.IsKeyUnregistered(err) {
			return false, fmt.Errorf("phone not registered")
		}
		// gotd returns specific error for 2FA
		if tg.IsCodeError(err, "SESSION_PASSWORD_NEEDED") {
			return true, nil
		}
		return false, fmt.Errorf("sign in: %w", err)
	}

	m.setUser(result)
	return false, nil
}

// Verify2FA checks the two-factor auth password
func (m *MTProtoClient) Verify2FA(ctx context.Context, password string) error {
	m.mu.Lock()
	defer m.mu.Unlock()

	// Get current password info
	pwd, err := m.api.AccountGetPassword(ctx)
	if err != nil {
		return fmt.Errorf("get password: %w", err)
	}

	// Compute SRP answer
	srpAnswer, err := auth.PasswordHash([]byte(password), pwd.CurrentAlgo)
	if err != nil {
		return fmt.Errorf("password hash: %w", err)
	}

	srpResult, err := auth.CheckPassword(pwd, srpAnswer)
	if err != nil {
		return fmt.Errorf("check password: %w", err)
	}

	authResult, err := m.api.AuthCheckPassword(ctx, srpResult)
	if err != nil {
		return fmt.Errorf("auth check password: %w", err)
	}

	switch r := authResult.(type) {
	case *tg.AuthAuthorization:
		if u, ok := r.User.(*tg.User); ok {
			m.Authenticated = true
			m.User = &UserInfo{
				ID:        u.ID,
				FirstName: u.FirstName,
				Username:  u.Username,
			}
		}
	}
	return nil
}

func (m *MTProtoClient) setUser(authResult *tg.AuthAuthorization) {
	if u, ok := authResult.User.(*tg.User); ok {
		m.Authenticated = true
		m.User = &UserInfo{
			ID:        u.ID,
			FirstName: u.FirstName,
			Username:  u.Username,
		}
	}
}

// GetDialogs fetches recent chat list
func (m *MTProtoClient) GetDialogs(ctx context.Context) ([]ChatInfo, error) {
	m.mu.Lock()
	api := m.api
	m.mu.Unlock()

	if api == nil {
		return nil, fmt.Errorf("not connected")
	}

	var chats []ChatInfo

	dialogs := query.GetDialogs(api)
	err := dialogs.ForEach(ctx, func(ctx context.Context, elem tg.DialogsClass) error {
		switch d := elem.(type) {
		case *tg.Dialogs:
			for i, dialog := range d.Dialogs {
				ci := dialogToChat(dialog, d.Users, d.Chats, d.Messages, i)
				if ci != nil {
					chats = append(chats, *ci)
				}
			}
		case *tg.DialogsSlice:
			for i, dialog := range d.Dialogs {
				ci := dialogToChat(dialog, d.Users, d.Chats, d.Messages, i)
				if ci != nil {
					chats = append(chats, *ci)
				}
			}
		}
		return nil
	})

	return chats, err
}

// GetMessages fetches message history for a chat
func (m *MTProtoClient) GetMessages(ctx context.Context, peerID int64, limit int) ([]MessageInfo, error) {
	m.mu.Lock()
	api := m.api
	m.mu.Unlock()

	if api == nil {
		return nil, fmt.Errorf("not connected")
	}

	// Resolve peer
	peer := &tg.InputPeerUser{UserID: peerID}

	history, err := api.MessagesGetHistory(ctx, &tg.MessagesGetHistoryRequest{
		Peer:  peer,
		Limit: limit,
	})
	if err != nil {
		return nil, fmt.Errorf("get history: %w", err)
	}

	var messages []MessageInfo
	switch h := history.(type) {
	case *tg.MessagesMessages:
		messages = msgsToInfo(h.Messages)
	case *tg.MessagesMessagesSlice:
		messages = msgsToInfo(h.Messages)
	case *tg.MessagesChannelMessages:
		messages = msgsToInfo(h.Messages)
	}

	return messages, nil
}

// SendMessage sends a text message
func (m *MTProtoClient) SendMessage(ctx context.Context, peerID int64, text string) (int, error) {
	m.mu.Lock()
	api := m.api
	m.mu.Unlock()

	if api == nil {
		return 0, fmt.Errorf("not connected")
	}

	sender := message.NewSender(api)
	updates, err := sender.To(&tg.InputPeerUser{UserID: peerID}).Text(ctx, text)
	if err != nil {
		return 0, fmt.Errorf("send message: %w", err)
	}

	_ = updates
	return 0, nil
}

// Stop disconnects the client
func (m *MTProtoClient) Stop() {
	if m.cancel != nil {
		m.cancel()
	}
}

// Helper types

type ChatInfo struct {
	ID          string `json:"id"`
	Name        string `json:"name"`
	Type        string `json:"type"`
	LastMessage string `json:"last_message"`
	Source      string `json:"source"`
}

type MessageInfo struct {
	ID       int    `json:"id"`
	Sender   string `json:"sender"`
	Text     string `json:"text"`
	Date     string `json:"date"`
	Outgoing bool   `json:"outgoing"`
}

func dialogToChat(dialog tg.DialogClass, users []tg.UserClass, chatClasses []tg.ChatClass, msgs []tg.MessageClass, idx int) *ChatInfo {
	d, ok := dialog.(*tg.Dialog)
	if !ok {
		return nil
	}

	ci := &ChatInfo{Source: "mtproto"}

	switch p := d.Peer.(type) {
	case *tg.PeerUser:
		ci.ID = fmt.Sprintf("%d", p.UserID)
		ci.Type = "user"
		for _, u := range users {
			if user, ok := u.(*tg.User); ok && user.ID == p.UserID {
				ci.Name = user.FirstName
				if user.LastName != "" {
					ci.Name += " " + user.LastName
				}
				if ci.Name == "" {
					ci.Name = user.Username
				}
				break
			}
		}
	case *tg.PeerChat:
		ci.ID = fmt.Sprintf("%d", p.ChatID)
		ci.Type = "group"
		for _, c := range chatClasses {
			if chat, ok := c.(*tg.Chat); ok && chat.ID == p.ChatID {
				ci.Name = chat.Title
				break
			}
		}
	case *tg.PeerChannel:
		ci.ID = fmt.Sprintf("%d", p.ChannelID)
		ci.Type = "channel"
		for _, c := range chatClasses {
			if ch, ok := c.(*tg.Channel); ok && ch.ID == p.ChannelID {
				ci.Name = ch.Title
				break
			}
		}
	}

	// Get last message text
	if idx < len(msgs) {
		if msg, ok := msgs[idx].(* tg.Message); ok {
			ci.LastMessage = msg.Message
			if len(ci.LastMessage) > 50 {
				ci.LastMessage = ci.LastMessage[:50] + "..."
			}
		}
	}

	if ci.Name == "" {
		ci.Name = "Chat " + ci.ID
	}

	return ci
}

func msgsToInfo(msgs []tg.MessageClass) []MessageInfo {
	var result []MessageInfo
	for _, m := range msgs {
		if msg, ok := m.(*tg.Message); ok {
			mi := MessageInfo{
				ID:       msg.ID,
				Text:     msg.Message,
				Date:     fmt.Sprintf("%d", msg.Date),
				Outgoing: msg.Out,
			}
			if msg.FromID != nil {
				if p, ok := msg.FromID.(*tg.PeerUser); ok {
					mi.Sender = fmt.Sprintf("%d", p.UserID)
				}
			}
			result = append(result, mi)
		}
	}
	return result
}
```

**Step 3: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

**Step 4: Commit**

```bash
git add poc/telegram-demo/
git commit -m "feat(poc): MTProto client wrapper with gotd/td"
```

---

## Task 6: Bot API Client Wrapper

**Files:**
- Create: `poc/telegram-demo/internal/telegram/botapi.go`

**Step 1: Add go-telegram/bot dependency**

```bash
cd poc/telegram-demo
go get github.com/go-telegram/bot@latest
```

**Step 2: Create Bot API wrapper**

```go
// poc/telegram-demo/internal/telegram/botapi.go
package telegram

import (
	"context"
	"fmt"
	"log/slog"
	"sync"

	"github.com/go-telegram/bot"
	"github.com/go-telegram/bot/models"
)

type BotAPIClient struct {
	mu       sync.Mutex
	bot      *bot.Bot
	cancel   context.CancelFunc
	Username string

	// Track chats we've seen from updates
	knownChats map[int64]ChatInfo
}

func NewBotAPIClient(token string) (*BotAPIClient, error) {
	b := &BotAPIClient{
		knownChats: make(map[int64]ChatInfo),
	}

	opts := []bot.Option{
		bot.WithDefaultHandler(b.handleUpdate),
	}

	tgBot, err := bot.New(token, opts...)
	if err != nil {
		return nil, fmt.Errorf("create bot: %w", err)
	}

	b.bot = tgBot
	return b, nil
}

// Start begins long-polling for updates
func (b *BotAPIClient) Start(ctx context.Context) error {
	ctx, cancel := context.WithCancel(ctx)
	b.cancel = cancel

	// Get bot info
	me, err := b.bot.GetMe(ctx)
	if err != nil {
		cancel()
		return fmt.Errorf("get me: %w", err)
	}
	b.Username = me.Username
	slog.Info("Bot API connected", "username", me.Username)

	// Start polling in background
	go b.bot.Start(ctx)

	return nil
}

func (b *BotAPIClient) handleUpdate(ctx context.Context, tgBot *bot.Bot, update *models.Update) {
	if update.Message == nil {
		return
	}

	msg := update.Message
	chatID := msg.Chat.ID

	b.mu.Lock()
	b.knownChats[chatID] = ChatInfo{
		ID:          fmt.Sprintf("%d", chatID),
		Name:        chatName(msg.Chat),
		Type:        string(msg.Chat.Type),
		LastMessage: truncate(msg.Text, 50),
		Source:      "bot",
	}
	b.mu.Unlock()

	slog.Debug("bot received message", "chat_id", chatID, "text", msg.Text)
}

// GetChats returns all chats the bot has seen
func (b *BotAPIClient) GetChats() []ChatInfo {
	b.mu.Lock()
	defer b.mu.Unlock()

	chats := make([]ChatInfo, 0, len(b.knownChats))
	for _, c := range b.knownChats {
		chats = append(chats, c)
	}
	return chats
}

// SendMessage sends a text message via Bot API
func (b *BotAPIClient) SendMessage(ctx context.Context, chatID int64, text string) (int, error) {
	msg, err := b.bot.SendMessage(ctx, &bot.SendMessageParams{
		ChatID: chatID,
		Text:   text,
	})
	if err != nil {
		return 0, fmt.Errorf("send message: %w", err)
	}
	return msg.ID, nil
}

// Stop disconnects the bot
func (b *BotAPIClient) Stop() {
	if b.cancel != nil {
		b.cancel()
	}
}

func chatName(chat models.Chat) string {
	if chat.Title != "" {
		return chat.Title
	}
	name := chat.FirstName
	if chat.LastName != "" {
		name += " " + chat.LastName
	}
	if name == "" {
		name = chat.Username
	}
	if name == "" {
		name = fmt.Sprintf("Chat %d", chat.ID)
	}
	return name
}

func truncate(s string, n int) string {
	if len(s) <= n {
		return s
	}
	return s[:n] + "..."
}
```

**Step 3: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

**Step 4: Commit**

```bash
git add poc/telegram-demo/
git commit -m "feat(poc): Bot API client wrapper with go-telegram/bot"
```

---

## Task 7: Wire Telegram Clients into State + Config Handler

**Files:**
- Modify: `poc/telegram-demo/internal/state.go`
- Modify: `poc/telegram-demo/internal/server.go`

**Step 1: Add Telegram client fields to AppState**

Update `state.go` to import and hold telegram clients:

```go
// Add to state.go imports
import (
	"context"
	"log/slog"
	"sync"

	tgclient "github.com/messenger-sf/telegram-poc/internal/telegram"
)

// Add fields to AppState struct
type AppState struct {
	mu     sync.RWMutex
	Config *AppConfig

	MTProto *tgclient.MTProtoClient
	BotAPI  *tgclient.BotAPIClient

	mtprotoConnected bool
	mtprotoUser      *UserInfo
	botConnected     bool
	botUsername       string
}

// Update SetConfig to initialize clients
func (s *AppState) SetConfig(ctx context.Context, cfg AppConfig) error {
	s.mu.Lock()
	defer s.mu.Unlock()

	// Stop existing clients
	if s.MTProto != nil {
		s.MTProto.Stop()
	}
	if s.BotAPI != nil {
		s.BotAPI.Stop()
	}

	s.Config = &cfg
	s.mtprotoConnected = false
	s.mtprotoUser = nil
	s.botConnected = false
	s.botUsername = ""

	// Initialize MTProto if configured
	if cfg.APIID != 0 && cfg.APIHash != "" {
		s.MTProto = tgclient.NewMTProtoClient(cfg.APIID, cfg.APIHash)
		if err := s.MTProto.Start(ctx); err != nil {
			slog.Error("MTProto start failed", "error", err)
			// Non-fatal: user can still use Bot API
		}
	}

	// Initialize Bot API if configured
	if cfg.BotToken != "" {
		botClient, err := tgclient.NewBotAPIClient(cfg.BotToken)
		if err != nil {
			slog.Error("Bot API init failed", "error", err)
		} else {
			if err := botClient.Start(ctx); err != nil {
				slog.Error("Bot API start failed", "error", err)
			} else {
				s.BotAPI = botClient
				s.botConnected = true
				s.botUsername = botClient.Username
			}
		}
	}

	return nil
}

// SetMTProtoAuth marks MTProto as authenticated
func (s *AppState) SetMTProtoAuth(user *UserInfo) {
	s.mu.Lock()
	defer s.mu.Unlock()
	s.mtprotoConnected = true
	s.mtprotoUser = user
}
```

**Step 2: Update handleConfig in server.go**

```go
func (s *Server) handleConfig(w http.ResponseWriter, r *http.Request) {
	var req AppConfig
	if err := readJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid JSON"})
		return
	}
	if req.APIID == 0 && req.BotToken == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "provide api_id+api_hash or bot_token"})
		return
	}
	if err := s.state.SetConfig(r.Context(), req); err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": err.Error()})
		return
	}
	slog.Info("config updated", "api_id", req.APIID, "has_bot_token", req.BotToken != "")
	writeJSON(w, http.StatusOK, s.state.Status())
}
```

**Step 3: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

**Step 4: Commit**

```bash
git add poc/telegram-demo/internal/
git commit -m "feat(poc): wire Telegram clients into state and config handler"
```

---

## Task 8: Auth Endpoints

**Files:**
- Modify: `poc/telegram-demo/internal/server.go`

**Step 1: Implement auth handlers**

Replace the auth stubs in `server.go`:

```go
func (s *Server) handleAuthPhone(w http.ResponseWriter, r *http.Request) {
	s.state.mu.RLock()
	mtproto := s.state.MTProto
	s.state.mu.RUnlock()

	if mtproto == nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "MTProto not configured"})
		return
	}

	var req struct {
		Phone string `json:"phone"`
	}
	if err := readJSON(r, &req); err != nil || req.Phone == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "phone required"})
		return
	}

	hash, err := mtproto.SendCode(r.Context(), req.Phone)
	if err != nil {
		writeJSON(w, http.StatusInternalServerError, map[string]string{"error": err.Error()})
		return
	}

	writeJSON(w, http.StatusOK, map[string]string{"phone_code_hash": hash})
}

func (s *Server) handleAuthCode(w http.ResponseWriter, r *http.Request) {
	s.state.mu.RLock()
	mtproto := s.state.MTProto
	s.state.mu.RUnlock()

	if mtproto == nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "MTProto not configured"})
		return
	}

	var req struct {
		Phone         string `json:"phone"`
		PhoneCodeHash string `json:"phone_code_hash"`
		Code          string `json:"code"`
	}
	if err := readJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid request"})
		return
	}

	need2FA, err := mtproto.VerifyCode(r.Context(), req.Phone, req.PhoneCodeHash, req.Code)
	if err != nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": err.Error()})
		return
	}

	if need2FA {
		writeJSON(w, http.StatusOK, map[string]string{"status": "2fa_required"})
		return
	}

	// Auth successful
	if mtproto.User != nil {
		s.state.SetMTProtoAuth(&UserInfo{
			ID:        mtproto.User.ID,
			FirstName: mtproto.User.FirstName,
			Username:  mtproto.User.Username,
		})
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}

func (s *Server) handleAuth2FA(w http.ResponseWriter, r *http.Request) {
	s.state.mu.RLock()
	mtproto := s.state.MTProto
	s.state.mu.RUnlock()

	if mtproto == nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "MTProto not configured"})
		return
	}

	var req struct {
		Password string `json:"password"`
	}
	if err := readJSON(r, &req); err != nil || req.Password == "" {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "password required"})
		return
	}

	if err := mtproto.Verify2FA(r.Context(), req.Password); err != nil {
		writeJSON(w, http.StatusUnauthorized, map[string]string{"error": err.Error()})
		return
	}

	if mtproto.User != nil {
		s.state.SetMTProtoAuth(&UserInfo{
			ID:        mtproto.User.ID,
			FirstName: mtproto.User.FirstName,
			Username:  mtproto.User.Username,
		})
	}

	writeJSON(w, http.StatusOK, map[string]string{"status": "ok"})
}
```

**Step 2: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

**Step 3: Commit**

```bash
git add poc/telegram-demo/internal/
git commit -m "feat(poc): implement MTProto auth endpoints (phone, code, 2FA)"
```

---

## Task 9: Chat List + Messages + Send Endpoints

**Files:**
- Modify: `poc/telegram-demo/internal/server.go`

**Step 1: Implement chat, message, and send handlers**

Replace the remaining stubs in `server.go`:

```go
func (s *Server) handleChats(w http.ResponseWriter, r *http.Request) {
	source := r.URL.Query().Get("source")
	if source == "" {
		source = "all"
	}

	s.state.mu.RLock()
	mtproto := s.state.MTProto
	botapi := s.state.BotAPI
	mtprotoConnected := s.state.mtprotoConnected
	botConnected := s.state.botConnected
	s.state.mu.RUnlock()

	var allChats []map[string]string

	// MTProto chats
	if (source == "all" || source == "mtproto") && mtprotoConnected && mtproto != nil {
		chats, err := mtproto.GetDialogs(r.Context())
		if err != nil {
			slog.Error("failed to get MTProto dialogs", "error", err)
		} else {
			for _, c := range chats {
				allChats = append(allChats, map[string]string{
					"id": c.ID, "name": c.Name, "type": c.Type,
					"last_message": c.LastMessage, "source": c.Source,
				})
			}
		}
	}

	// Bot API chats
	if (source == "all" || source == "bot") && botConnected && botapi != nil {
		for _, c := range botapi.GetChats() {
			allChats = append(allChats, map[string]string{
				"id": c.ID, "name": c.Name, "type": c.Type,
				"last_message": c.LastMessage, "source": c.Source,
			})
		}
	}

	if allChats == nil {
		allChats = []map[string]string{}
	}
	writeJSON(w, http.StatusOK, allChats)
}

func (s *Server) handleMessages(w http.ResponseWriter, r *http.Request) {
	chatID := r.PathValue("id")
	source := r.URL.Query().Get("source")

	s.state.mu.RLock()
	mtproto := s.state.MTProto
	s.state.mu.RUnlock()

	if source == "mtproto" && mtproto != nil {
		peerID, err := strconv.ParseInt(chatID, 10, 64)
		if err != nil {
			writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid chat ID"})
			return
		}

		msgs, err := mtproto.GetMessages(r.Context(), peerID, 50)
		if err != nil {
			writeJSON(w, http.StatusInternalServerError, map[string]string{"error": err.Error()})
			return
		}
		if msgs == nil {
			msgs = []telegram.MessageInfo{}
		}
		writeJSON(w, http.StatusOK, msgs)
		return
	}

	// Bot API doesn't store message history — return empty
	writeJSON(w, http.StatusOK, []any{})
}

func (s *Server) handleSend(w http.ResponseWriter, r *http.Request) {
	var req struct {
		ChatID string `json:"chat_id"`
		Text   string `json:"text"`
		Source string `json:"source"`
	}
	if err := readJSON(r, &req); err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid request"})
		return
	}

	peerID, err := strconv.ParseInt(req.ChatID, 10, 64)
	if err != nil {
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "invalid chat_id"})
		return
	}

	s.state.mu.RLock()
	mtproto := s.state.MTProto
	botapi := s.state.BotAPI
	s.state.mu.RUnlock()

	switch req.Source {
	case "mtproto":
		if mtproto == nil {
			writeJSON(w, http.StatusBadRequest, map[string]string{"error": "MTProto not connected"})
			return
		}
		msgID, err := mtproto.SendMessage(r.Context(), peerID, req.Text)
		if err != nil {
			writeJSON(w, http.StatusInternalServerError, map[string]string{"error": err.Error()})
			return
		}
		writeJSON(w, http.StatusOK, map[string]int{"message_id": msgID})

	case "bot":
		if botapi == nil {
			writeJSON(w, http.StatusBadRequest, map[string]string{"error": "Bot API not connected"})
			return
		}
		msgID, err := botapi.SendMessage(r.Context(), peerID, req.Text)
		if err != nil {
			writeJSON(w, http.StatusInternalServerError, map[string]string{"error": err.Error()})
			return
		}
		writeJSON(w, http.StatusOK, map[string]int{"message_id": msgID})

	default:
		writeJSON(w, http.StatusBadRequest, map[string]string{"error": "source must be 'mtproto' or 'bot'"})
	}
}
```

**Step 2: Add missing imports to server.go**

```go
import (
	"strconv"

	telegram "github.com/messenger-sf/telegram-poc/internal/telegram"
)
```

**Step 3: Verify compilation**

```bash
cd poc/telegram-demo && go build .
```

**Step 4: Commit**

```bash
git add poc/telegram-demo/internal/
git commit -m "feat(poc): implement chat list, messages, and send endpoints"
```

---

## Task 10: Integration Test — End-to-End Smoke Test

**Files:**
- Create: `poc/telegram-demo/internal/server_test.go`

**Step 1: Write a test for the status → config → status flow**

```go
// poc/telegram-demo/internal/server_test.go
package internal

import (
	"bytes"
	"encoding/json"
	"net/http"
	"net/http/httptest"
	"testing"
)

func TestStatusFlow(t *testing.T) {
	state := NewAppState()
	srv := NewServer(state, nil)

	// 1. Initial status should be needs_config
	rec := httptest.NewRecorder()
	req := httptest.NewRequest("GET", "/api/status", nil)
	srv.ServeHTTP(rec, req)

	if rec.Code != 200 {
		t.Fatalf("expected 200, got %d", rec.Code)
	}

	var status StatusResponse
	json.NewDecoder(rec.Body).Decode(&status)

	if status.Stage != "needs_config" {
		t.Fatalf("expected needs_config, got %s", status.Stage)
	}

	// 2. Config with invalid request should fail
	rec = httptest.NewRecorder()
	body := bytes.NewBufferString(`{"api_id":0,"api_hash":"","bot_token":""}`)
	req = httptest.NewRequest("POST", "/api/config", body)
	req.Header.Set("Content-Type", "application/json")
	srv.ServeHTTP(rec, req)

	if rec.Code != 400 {
		t.Fatalf("expected 400, got %d", rec.Code)
	}
}
```

**Step 2: Run the test**

```bash
cd poc/telegram-demo && go test ./internal/ -v
```

Expected: PASS

**Step 3: Commit**

```bash
git add poc/telegram-demo/internal/server_test.go
git commit -m "test(poc): add status flow smoke test"
```

---

## Task 11: Final Polish — Build Script + README

**Files:**
- Modify: `poc/telegram-demo/Makefile`
- Create: `poc/telegram-demo/README.md`

**Step 1: Update Makefile with clean target**

```makefile
# poc/telegram-demo/Makefile
.PHONY: run build-web dev clean test

PORT ?= 8080

run: build-web
	@echo "Starting server on http://localhost:$(PORT)"
	go run . -port $(PORT)

build-web:
	cd web && npm install --silent && npm run build

dev:
	go run . -port $(PORT)

test:
	go test ./... -v

clean:
	rm -rf web/node_modules web/dist
	go clean
```

**Step 2: Create README**

```markdown
# Telegram PoC

Standalone demo: connect to Telegram via MTProto (user account) and Bot API, browse chats, send messages.

## Prerequisites

- Go 1.24+
- Node.js 18+
- Telegram API credentials (https://my.telegram.org)
- Bot token from @BotFather (optional)

## Run

```bash
make run
```

Opens at http://localhost:8080

## How it works

1. **Setup** — Enter API ID, API Hash, and optionally a Bot Token
2. **Login** — Authenticate via phone number + OTP + optional 2FA
3. **Chats** — Browse your recent conversations from both protocols
4. **Chat** — View messages and send replies

All state is in-memory. Restart = start fresh.
```

**Step 3: Full end-to-end test**

```bash
cd poc/telegram-demo
make clean
make run
# In browser: open http://localhost:8080
# Should auto-redirect to /setup
```

**Step 4: Commit**

```bash
git add poc/telegram-demo/
git commit -m "feat(poc): final polish — Makefile, README, complete PoC"
```

---

## Summary

| Task | What | Key Files |
|------|------|-----------|
| 1 | Project scaffold | `go.mod`, `main.go`, `Makefile` |
| 2 | React app (all pages + routing) | `web/src/**/*.tsx`, `app.css` |
| 3 | Go HTTP server + SPA serving | `internal/server.go`, `internal/state.go` |
| 4 | Config endpoint | `server.go`, `state.go` |
| 5 | MTProto wrapper (gotd/td) | `internal/telegram/mtproto.go` |
| 6 | Bot API wrapper (go-telegram/bot) | `internal/telegram/botapi.go` |
| 7 | Wire clients into state | `state.go`, `server.go` |
| 8 | Auth endpoints | `server.go` |
| 9 | Chat + messages + send endpoints | `server.go` |
| 10 | Smoke test | `internal/server_test.go` |
| 11 | Polish + README | `Makefile`, `README.md` |

**Total: 11 tasks, ~15 files, fully standalone in `poc/telegram-demo/`**
