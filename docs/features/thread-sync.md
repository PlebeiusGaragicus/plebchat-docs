# Thread Sync

Cross-device thread synchronization using Nostr keypair authentication.

## Overview

PlebChat allows users to sync their conversation threads across multiple devices. When in **Sync mode**, threads are stored on the LangGraph server with the user's pubkey in the metadata, enabling secure retrieval from any device where the user logs in with the same keypair.

## How It Works

### Thread Ownership

When a thread is created in Sync mode:
1. The frontend sends the user's hex pubkey to the backend
2. The backend stores the pubkey in the thread's metadata on LangGraph
3. This creates a permanent association between the user and their thread

```python
# Backend stores pubkey in thread metadata
thread_metadata = {
    "user_pubkey": "a1b2c3d4..."  # hex pubkey
}
await client.threads.create(metadata=thread_metadata)
```

### Syncing Threads

When the user clicks "Sync..." in the Privacy Mode popover:
1. Frontend creates a NIP-98 signed authorization header
2. Backend verifies the signature to extract the verified pubkey
3. Backend searches LangGraph for threads with matching `user_pubkey` metadata
4. Thread data (including messages) is returned to the frontend
5. Frontend imports threads into local storage

```
User → Frontend: Click "Sync..."
Frontend → Frontend: Sign NIP-98 event with keypair
Frontend → Backend: GET /api/threads/sync (Authorization: Nostr <event>)
Backend → Backend: Verify signature, extract pubkey
Backend → LangGraph: Search threads where user_pubkey = verified_pubkey
LangGraph → Backend: Thread list with messages
Backend → Frontend: SyncResponse
Frontend → Frontend: Import to localStorage
```

## NIP-98 Authentication

[NIP-98](https://github.com/nostr-protocol/nips/blob/master/98.md) defines HTTP authentication using signed Nostr events. This provides cryptographic proof that a request comes from the owner of a specific pubkey.

### How It Works

1. **Client creates event**: A kind 27235 event with:
   - `u` tag: The absolute URL of the request
   - `method` tag: The HTTP method (GET, POST, etc.)
   - Content: Empty string

2. **Client signs event**: Using their Nostr keypair (via plebtap)

3. **Client sends request**: With `Authorization: Nostr <base64-encoded-event>`

4. **Server verifies**:
   - Decodes the base64 event
   - Verifies the Schnorr signature
   - Checks the URL and method match
   - Checks the event is recent (not expired)
   - Extracts the verified pubkey

### Example Event

```json
{
  "id": "...",
  "pubkey": "a1b2c3d4e5f6...",
  "created_at": 1704067200,
  "kind": 27235,
  "tags": [
    ["u", "https://api.plebchat.com/api/threads/sync"],
    ["method", "GET"]
  ],
  "content": "",
  "sig": "..."
}
```

## Security Model

| Threat | Mitigation |
|--------|------------|
| **Thread ID guessing** | UUIDs are cryptographically random (2^122 possibilities) |
| **Pubkey spoofing** | NIP-98 signature verification ensures only the true key owner can query |
| **LangGraph API exposure** | API key is secret, only backend has access |
| **Cross-user thread access** | Metadata filtering by cryptographically verified pubkey |
| **Replay attacks** | NIP-98 events expire after 60 seconds |

## Privacy Mode Behavior

| Mode | Pubkey Stored | Syncable | Server Retention |
|------|---------------|----------|------------------|
| **Sync** | Yes | Yes | Permanent until deleted |
| **Paranoid** | No | No | Deleted after completion |
| **Incognito** | No | No | Memory only, best-effort deletion |

### Sync Mode
- Full server persistence
- Cross-device access via sync
- Full stream recovery on connection loss
- User pubkey stored for ownership

### Paranoid Mode
- Local storage only
- Server data deleted after stream completion
- No cross-device sync
- No pubkey stored (anonymous threads)

### Incognito Mode
- Memory only (no localStorage)
- Threads marked for garbage collection
- No recovery on page refresh
- Maximum privacy, minimum persistence

## API Endpoints

### GET /api/threads/sync

Retrieve all threads owned by the authenticated user.

**Headers:**
- `Authorization: Nostr <base64-encoded-nip98-event>`

**Response:**
```json
{
  "success": true,
  "threads": [
    {
      "thread_id": "uuid",
      "langgraph_thread_id": "uuid",
      "agent_id": "chat",
      "title": "First 50 chars of first message...",
      "created_at": "2024-01-01T00:00:00Z",
      "updated_at": "2024-01-01T00:01:00Z",
      "messages": [
        {
          "id": "msg-1",
          "type": "human",
          "content": "Hello!",
          "is_complete": true
        },
        {
          "id": "msg-2",
          "type": "ai",
          "content": "Hi there! How can I help?",
          "is_complete": true
        }
      ]
    }
  ],
  "total_count": 1
}
```

**Error Response:**
```json
{
  "success": false,
  "threads": [],
  "total_count": 0,
  "error": "Authentication failed: Invalid event signature"
}
```

## Usage

### From the UI

1. Ensure you're in **Sync mode** (blue cloud icon in navbar)
2. Click the privacy mode button to open the popover
3. Click "Sync..." link (only visible in Sync mode when logged in)
4. Click "Start Sync" in the modal
5. Wait for sync to complete
6. New threads will appear in your sidebar

### Programmatically

```typescript
import { syncThreadsFromServer } from '$lib/stores/sync.svelte.js';

const result = await syncThreadsFromServer();
if (result.success) {
  console.log(`Synced ${result.syncedCount} threads`);
  console.log(`${result.newCount} new, ${result.updatedCount} updated`);
} else {
  console.error(`Sync failed: ${result.error}`);
}
```

## Troubleshooting

### "Please log in to sync threads"
You must be logged in with plebtap to sync threads. Your keypair is required for authentication.

### "Thread sync is only available in Sync mode"
Switch to Sync mode using the privacy mode popover. Paranoid and Incognito modes don't store pubkey metadata.

### "Authentication failed: Event expired"
The NIP-98 event has a 60-second expiry. Ensure your system clock is accurate.

### "Authentication failed: Invalid event signature"
The signature verification failed. This could indicate:
- A corrupted request
- Mismatched signing key
- Client-side signing error

### No threads found
Threads from other devices must have been created in Sync mode with the same keypair. Threads created in Paranoid or Incognito mode cannot be synced.
