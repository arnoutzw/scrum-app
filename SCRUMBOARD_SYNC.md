# Scrumboard Database Sync

This document explains how the scrumboard application synchronizes data between the browser client and the cloud database.

## Architecture Overview

The app uses a **local-first, cloud-second** architecture. Every change is written to the browser's `localStorage` immediately for fast UI responsiveness, then asynchronously pushed to **Google Cloud Firestore** for persistence and cross-device sync. A real-time Firestore listener keeps all connected clients in sync.

```
 ┌──────────────┐       save()        ┌──────────────┐  debounced (500ms)  ┌─────────────────┐
 │   UI State   │ ───────────────────► │ localStorage │ ──────────────────► │    Firestore    │
 │  (in-memory) │                      │  (DB_KEY)    │                     │ kanban-boards/  │
 └──────┬───────┘                      └──────────────┘                     │    shared       │
        │                                                                   └────────┬────────┘
        │              onSnapshot (real-time listener)                               │
        ◄───────────────────────────────────────────────────────────────────────────────┘
```

## Database Technology

**Primary database:** Google Firebase Cloud Firestore

- Firestore project: `arnout-s-homelab`
- Main collection: `kanban-boards`
- Main document: `shared` (stores the entire board state as a single document)
- Additional collections: `retro-nicknames` (user nicknames), `retro-presence` (active user tracking)

Initialization (`index.html:117-127`):
```javascript
firebase.initializeApp({ /* config */ });
const db = firebase.firestore();
const FIRESTORE_DOC = db.collection('kanban-boards').doc('shared');
```

**Secondary storage:** Browser `localStorage` (key: `diy-kanban-data`)

## Data Model

### Global State

The entire application state is stored as a single JSON object:

```
{
  projects: Project[],
  activeProjectId: string,
  view: 'board' | 'backlog' | 'deps' | 'retro' | 'settings'
}
```

### Project

```
{
  id: string,
  name: string,
  columns: Column[],
  labels: Label[],
  members: string[],
  cards: Card[],
  retro: { boards: RetroBoard[] },
  createdAt: number
}
```

### Card

```
{
  id: string,
  title: string,
  description: string,
  priority: 'low' | 'medium' | 'high' | 'critical',
  labels: string[],
  materials: string,
  estimatedTime: string,
  dueDate: string,
  subtasks: { title: string, done: boolean }[],
  dependencies: string[],        // IDs of cards this card depends on
  assignee: string,
  columnId: string,
  order: number,                  // position within its column
  createdAt: number,
  type?: 'story' | 'feature' | 'epic',
  storyPoints?: string | number,
  parentId?: string              // parent epic ID
}
```

### Column

```
{
  id: string,
  name: string,
  wipLimit: number       // 0 means unlimited
}
```

### Label

```
{
  name: string,
  color: 'amber' | 'blue' | 'cyan' | 'purple' | 'red' | 'green' | 'zinc'
}
```

## Write Path (Client to Database)

Every mutation in the app follows this call chain:

1. **Modify in-memory state** -- The operation (e.g. `addCard`, `updateCard`, `moveCard`) mutates the global `state` object directly.
2. **`persist()`** (`index.html:267-270`) -- Calls `save(state)` then `render()`.
3. **`save(data)`** (`index.html:212-215`) -- Writes to `localStorage` synchronously, then calls `saveToFirestore(data)`.
4. **`saveToFirestore(data)`** (`index.html:130-136`) -- Debounced by **500 ms**. Serializes the full state via `JSON.parse(JSON.stringify(data))` and writes it to the Firestore document with `.set()`.

```javascript
function saveToFirestore(data) {
  clearTimeout(_fsWriteTimer);
  _fsWriteTimer = setTimeout(() => {
    FIRESTORE_DOC.set(JSON.parse(JSON.stringify(data)))
      .catch(err => console.warn('Firestore write failed:', err));
  }, 500);
}
```

Key characteristics:
- **Full-document replacement:** Every write replaces the entire Firestore document. There are no field-level or sub-document updates.
- **500 ms debounce:** Rapid consecutive changes (e.g. drag-and-drop reordering) are batched into a single write.
- **Fire-and-forget:** Write failures are logged but do not block the UI. Local state remains intact.

## Read Path (Database to Client)

### Startup

1. `initData()` (`index.html:230-247`) loads state from `localStorage`. If nothing is stored, a default project is created and saved.
2. `migrateData()` (`index.html:5615-5625`) runs schema migration, ensuring all cards have `dependencies` and `assignee` fields and all projects have `members` and `retro`.
3. A one-time `FIRESTORE_DOC.get()` check (`index.html:5637-5641`) pushes local state to Firestore if the remote document does not yet exist.

### Real-time Listener

A Firestore `onSnapshot` listener (`index.html:5627-5634`) keeps the client in sync with remote changes:

```javascript
FIRESTORE_DOC.onSnapshot(doc => {
  if (doc.exists && !doc.metadata.hasPendingWrites) {
    const data = migrateData(doc.data());
    state = data;
    localStorage.setItem(DB_KEY, JSON.stringify(data));
    render();
  }
}, err => console.warn('Firestore listener error:', err));
```

When a remote change arrives:
1. The snapshot is checked for `doc.metadata.hasPendingWrites` -- if `true`, the change originated from this client and is skipped to avoid overwriting local state with a stale echo.
2. The data is migrated for schema compatibility.
3. The in-memory `state` is replaced, `localStorage` is updated, and the UI is re-rendered.

## Conflict Resolution

The application uses a **last-write-wins (LWW)** strategy at the document level:

- Since the entire state is written as one Firestore document, the most recent `.set()` call always overwrites the previous value.
- The `hasPendingWrites` check prevents a client from overwriting its own local changes with a stale snapshot echo from Firestore.
- There is no field-level merging or operational transform -- if two clients modify different cards simultaneously, the last writer's full state replaces the other's.

## Offline Support

### Service Worker (`sw.js`)

A service worker provides offline capability for static assets:

- **Install:** Caches `index.html`, `manifest.json`, Tailwind CSS CDN, and Google Fonts.
- **Fetch:** Uses a cache-first strategy -- serves from cache if available, falls back to network, and caches successful responses. If both cache and network fail, serves the cached `index.html` as a fallback.

### localStorage as Offline Store

- All state changes are written to `localStorage` immediately, regardless of network availability.
- When offline, Firestore writes will silently fail (caught with `.catch()`), but the app continues to function from local data.
- On reconnection, the Firestore SDK's built-in offline persistence may replay queued writes, and the `onSnapshot` listener re-establishes the real-time connection.

## Retro Presence Tracking

The retrospective view has a separate sync mechanism for tracking active users:

**Collection:** `retro-presence`

- **Heartbeat** (`index.html:3430-3435`): Every 20 seconds, each client writes a document with `{ nickname, boardId, lastSeen: Date.now() }`.
- **Listener** (`index.html:3437-3449`): An `onSnapshot` query filtered by `boardId` collects all documents and shows users whose `lastSeen` is within the last 60 seconds.
- **Cleanup** (`index.html:3454-3459`): On page unload or board switch, the client's presence document is deleted and listeners are unsubscribed.

## CRUD Operations Summary

All CRUD operations follow the same pattern: mutate `state` in-memory, then call `persist()`.

| Operation | Function | Location |
|-----------|----------|----------|
| Create card | `addCard()` | `index.html:508` |
| Update card | `updateCard()` | `index.html:531` |
| Delete card | `deleteCard()` | `index.html:537` |
| Move card | `moveCard()` | `index.html:549` |
| Add subtask | `addSubtask()` | `index.html:570` |
| Remove subtask | `removeSubtask()` | `index.html:576` |
| Toggle subtask | `toggleSubtask()` | `index.html:561` |
| Add dependency | `addDependency()` | `index.html:583` |
| Remove dependency | `removeDependency()` | `index.html:598` |
| Create project | `createProject()` | `index.html:217` |
| Export data | `exportData()` | JSON file download |
| Import data | `importData()` | JSON file upload |

Notable behaviors:
- **Deleting a card** automatically removes it from all other cards' `dependencies` arrays (`index.html:540-543`).
- **Adding a dependency** checks for circular chains via `wouldCreateCycle()` and blocks the operation if one is detected (`index.html:585-588`).
- **Removing a column** moves its cards to the backlog rather than deleting them.

## Key Sync Functions Reference

| Function | Location | Purpose |
|----------|----------|---------|
| `save(data)` | `index.html:212` | Write to localStorage + trigger Firestore save |
| `persist()` | `index.html:267` | Save state and re-render the UI |
| `saveToFirestore(data)` | `index.html:130` | Debounced full-document write to Firestore |
| `load()` | `index.html:208` | Read state from localStorage |
| `initData()` | `index.html:230` | Bootstrap state on app startup |
| `migrateData(data)` | `index.html:5615` | Ensure schema compatibility across versions |
