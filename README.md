# personal-work
# Perpetua — Personal AI System Blueprint

## Architecture Summary (V1)

Perpetua V1 is a **single Next.js 15 monolith** (App Router) that includes both UI and backend logic. Supabase provides auth, Postgres, and pgvector for durable user data and long-term memory. LangGraph runs inside backend routes/server actions for multi-step reasoning, while LangMem powers memory extraction/retrieval workflows. The design stays intentionally minimal: one deploy target (Vercel), one primary database (Supabase), and modular internal boundaries (`app`, `lib`, `tools`) that can later be split without rework.

---

## STAGE 1 — FRONTEND FOUNDATION

### Objective

Build a calm, premium, dark UI foundation that is production-credible and testable without backend dependencies:
- Chat interface with streaming-ready message area
- Memory panel (timeline-style placeholder list)
- Task panel (structured task list + quick add)
- Skeleton loading states
- Stable app layout/navigation

No business logic yet; this stage focuses on UX shell, interaction patterns, and component contracts for Stage 2.

### File Structure

```txt
.
├─ app/
│  ├─ (workspace)/
│  │  ├─ layout.tsx
│  │  ├─ page.tsx                        # chat-first workspace
│  │  ├─ loading.tsx                     # route-level skeleton
│  │  └─ components/
│  │     ├─ shell/
│  │     │  ├─ app-shell.tsx
│  │     │  ├─ top-nav.tsx
│  │     │  └─ side-rail.tsx
│  │     ├─ chat/
│  │     │  ├─ chat-thread.tsx
│  │     │  ├─ chat-input.tsx
│  │     │  ├─ message-bubble.tsx
│  │     │  └─ streaming-cursor.tsx
│  │     ├─ memory/
│  │     │  ├─ memory-panel.tsx
│  │     │  ├─ memory-item.tsx
│  │     │  └─ memory-timeline.tsx
│  │     ├─ tasks/
│  │     │  ├─ task-panel.tsx
│  │     │  ├─ task-item.tsx
│  │     │  └─ quick-add-task.tsx
│  │     └─ ui/
│  │        ├─ skeleton.tsx
│  │        ├─ icon-button.tsx
│  │        └─ section-card.tsx
│  ├─ globals.css
│  └─ layout.tsx
├─ lib/
│  ├─ types/
│  │  ├─ chat.ts
│  │  ├─ memory.ts
│  │  └─ task.ts
│  └─ mock/
│     ├─ chat-seed.ts
│     ├─ memory-seed.ts
│     └─ task-seed.ts
└─ README.md
```

### Data Models (Frontend Contracts for Stage 1)

```ts
// lib/types/chat.ts
export type ChatMessage = {
  id: string;
  role: 'user' | 'assistant' | 'system';
  content: string;
  createdAt: string; // ISO
  status?: 'streaming' | 'complete' | 'error';
};

// lib/types/memory.ts
export type MemoryEntry = {
  id: string;
  kind: 'insight' | 'preference' | 'event' | 'goal';
  summary: string;
  source: 'chat' | 'manual' | 'system';
  relevance?: number;
  createdAt: string;
};

// lib/types/task.ts
export type TaskItem = {
  id: string;
  title: string;
  status: 'todo' | 'in_progress' | 'done';
  priority: 'low' | 'medium' | 'high';
  dueAt?: string;
  createdAt: string;
};
```

### API Routes (Stage 1 Scope)

Stage 1 uses mock/local state, but define route contracts now for smooth Stage 2 handoff:

- `POST /api/chat/stream`
  - input: `{ message: string, conversationId?: string }`
  - output: text/event-stream chunks
- `GET /api/memory`
  - output: `{ items: MemoryEntry[] }`
- `GET /api/tasks`
  - output: `{ items: TaskItem[] }`
- `POST /api/tasks`
  - input: `{ title: string }`
  - output: `{ item: TaskItem }`

> In Stage 1 these can remain unimplemented or mocked via local adapters; UI should be ready for them.

### UI Components

#### 1) Workspace Shell
- **Top nav**: product name (“Perpetua”), workspace switch placeholder, profile menu placeholder
- **Three-column desktop layout**:
  - Left: Memory panel
  - Center: Chat thread + input
  - Right: Task panel
- **Responsive behavior**:
  - Mobile: chat primary, memory/task panels as bottom sheets or tab toggles

#### 2) Chat Interface
- Message bubbles with strict typography hierarchy
- Sticky input composer at bottom
- Streaming placeholder behavior:
  - Assistant message appears instantly with animated cursor
  - Text appends token-by-token (simulated for now)
- Empty state prompt: “What should we think through today?”

#### 3) Memory Panel
- Timeline-like list grouped by relative day (“Today”, “Yesterday”, “Earlier”)
- Entry chips for type (`insight`, `goal`, etc.)
- “No memories yet” empty state

#### 4) Task Panel
- Task groups by status (Todo / In Progress / Done)
- Quick add input + Enter to add local item
- Minimal actions: mark done, reopen

#### 5) Loading / Skeletons
- Route-level `loading.tsx`
- Inline skeletons for chat messages, memory rows, and tasks
- No spinners except for very short inline states; prefer skeleton shimmer

### Step-by-Step Implementation

1. **Initialize Next.js 15 app shell**
   - App Router + TypeScript + ESLint + Tailwind
   - Set dark theme tokens in `globals.css` (black/charcoal/neutral text only)

2. **Create layout primitives**
   - Build `app-shell.tsx`, `top-nav.tsx`, `side-rail.tsx`
   - Add responsive grid with fixed-height viewport behavior

3. **Implement typed frontend models**
   - Add `ChatMessage`, `MemoryEntry`, `TaskItem` types under `lib/types`
   - Add seed data under `lib/mock` for deterministic UI tests

4. **Build chat components**
   - `chat-thread.tsx`, `message-bubble.tsx`, `chat-input.tsx`
   - Add local “simulate stream” utility (interval appending tokens)

5. **Build memory panel components**
   - Timeline rendering + kind badges
   - Add empty and loading states

6. **Build task panel components**
   - Render grouped tasks
   - Implement local quick-add and status toggles

7. **Add skeleton system**
   - Reusable `skeleton.tsx`
   - Route-level and component-level loading states

8. **Accessibility + UX pass**
   - Keyboard-first interactions (Enter to send/add, Shift+Enter new line)
   - Focus-visible styles, contrast checks, reduced motion support

9. **Stage QA pass**
   - Validate responsive layouts (mobile, tablet, desktop)
   - Ensure no visual gimmicks: no gradients, no glow, no unnecessary animation

### Acceptance Criteria (Stage 1 Complete When)

- [ ] User can open app and see a stable three-panel workspace on desktop.
- [ ] Chat supports local send + simulated streaming assistant response.
- [ ] Memory panel shows seeded timeline entries and proper empty state.
- [ ] Task panel supports quick-add and status changes locally.
- [ ] Loading skeletons appear during route/component loading.
- [ ] UI follows strict minimal dark design (no gradients/glows).
- [ ] Components are typed and ready to wire to real APIs in Stage 2.
- [ ] Stage can be demoed end-to-end with no backend dependency.

---
