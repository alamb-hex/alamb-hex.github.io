---
layout: post
title: "Building True Offline-First PWAs: Architecture Patterns That Actually Work"
date: 2026-01-21
categories: [development, pwa, architecture]
tags: [pwa, offline-first, indexeddb, service-worker, dexie, sync, architecture]
author: Aaron Lamb
description: "Real-world patterns for building Progressive Web Apps that truly work offline, not just claim to. Learn Dexie.js schema versioning, exponential backoff sync, service worker authentication, and client-side operations that enable genuine offline-first experiences."
---

# Building True Offline-First PWAs: Architecture Patterns That Actually Work

I built a PWA for field sales tracking as a proof-of-concept. The requirements seemed straightforward: track store visits, capture product photos, generate reports. Standard CRUD app stuff.

Then I started field testing.

The app broke constantly. WiFi in warehouses is terrible. Cellular data drops when moving between locations. Users would capture data, see a success message, then hours later discover their work was gone. Upload failures looked like successes. Syncs timed out mid-request. The database corrupted after partial writes.

**The core problem: I built a web app that cached assets, not an offline-first app.**

There's a massive difference. "Works offline sometimes" is not the same as "offline-first." Most PWAs bolt caching onto a network-dependent architecture. It breaks the moment conditions get real.

I had to rebuild from scratch with one principle: **assume the network doesn't exist, then add sync as a bonus when it shows up.**

**Note on this project:** This was a proof-of-concept that ultimately didn't move to production. The client decided not to proceed with the project. That said, the lessons learned about offline-first architecture are broadly applicable, and the patterns work. Some security trade-offs (like long-lived device tokens) were acceptable for a POC but would need more hardening for a production system handling sensitive data.

## Why Most PWAs Fail at Offline-First: The Cascade of Breaking

Here's what I learned the hard way.

**Problem 1: The UI lies to users.**

User captures data offline. The app shows a success message. Looks saved. Eight hours later, they realize it's gone. The save worked locally, but the sync failed silently. No error. No warning. Their work just disappeared.

This happened constantly in early testing. I thought handling the IndexedDB save was enough. Wrong. The sync is where everything breaks, and users never see it happen.

**Problem 2: Authentication stops working.**

You need to be authenticated to call the API. But you need to call the API to get authenticated. Session tokens expire after 30 minutes. Refresh tokens require network calls. OAuth redirects are impossible offline.

I initially used JWT tokens with 1-hour expiration. Field workers would go offline for 2-3 hours. When connectivity returned, their tokens were expired. They'd get logged out mid-sync, losing their upload progress. Brutal user experience.

**Problem 3: Conflict resolution is harder than you think.**

Two devices work offline. Both modify the same record. Which one wins when they sync?

Last-write-wins sounds simple. It causes data loss. User A updates field X, User B updates field Y, last sync deletes one of those changes.

Conflict resolution UIs sound sophisticated. Users don't understand them. They just want their data to be there.

**Problem 4: Partial failures corrupt everything.**

Upload starts. Five images succeed. Network drops. Sixth image fails. Now what?

The user sees "5 of 6 uploaded" but doesn't know which one failed. Retry uploads all six, causing duplicates. Or worse, the local database marks everything as synced, leaving one image orphaned forever.

Most teams compromise at this point. "Requires internet connection" disclaimer. Offline mode is read-only. Elaborate conflict UIs that confuse users.

That's not offline-first. That's giving up.

## Dexie.js Schema Versioning: Never Break Offline Users

I tried using IndexedDB directly. Terrible mistake.

The API is callback hell. Schema migrations require manually tracking version numbers. One mistake corrupts the entire database. Users lose everything.

Dexie.js saved me weeks of debugging. Clean Promise-based API, automatic migrations, better error handling.

But here's what I didn't understand initially: **you can never do destructive schema changes in an offline-first app.**

Think about it. A user goes offline for a week. They're running version 1 of your schema. Meanwhile, you ship version 3 which removes a field they're actively using. When they sync, chaos.

The solution: only add, never remove.

**Additive-only schema migrations**

```typescript
// IndexedDB setup with Dexie for offline-first storage
import Dexie, { type EntityTable } from 'dexie';

const db = new Dexie('AppDB') as Dexie & {
  sessions: EntityTable<Session, 'id'>;
  products: EntityTable<Product, 'id'>;
  syncQueue: EntityTable<SyncQueue, 'id'>;
  deviceToken: EntityTable<DeviceToken, 'id'>;
  stockItems: EntityTable<StockItem, 'id'>;
};

// Version 1: Original schema
db.version(1).stores({
  sessions: 'id, storeName, storeLocation, repName, visitDate, createdAt, status, synced',
  products: 'id, sessionId, productName, createdAt, synced',
});

// Version 2: Add sync queue and product sync fields
db.version(2).stores({
  sessions: 'id, storeName, storeLocation, repName, visitDate, createdAt, status, synced',
  products: 'id, sessionId, productName, createdAt, synced, syncStatus, lastSyncAttempt',
  syncQueue: 'id, productId, sessionId, status, createdAt',
});

// Version 3: Add device token for API auth
db.version(3).stores({
  sessions: 'id, storeName, storeLocation, repName, visitDate, createdAt, status, synced',
  products: 'id, sessionId, productName, createdAt, synced, syncStatus, lastSyncAttempt',
  syncQueue: 'id, productId, sessionId, status, createdAt',
  deviceToken: 'id',
});

// Version 4: Add shift start/end times to sessions
db.version(4).stores({
  sessions: 'id, storeName, storeLocation, repName, visitDate, shiftStart, shiftEnd, createdAt, status, synced',
  products: 'id, sessionId, productName, createdAt, synced, syncStatus, lastSyncAttempt',
  syncQueue: 'id, productId, sessionId, status, createdAt',
  deviceToken: 'id',
});

// Version 5: Add stock items for inventory tracking
db.version(5).stores({
  sessions: 'id, storeName, storeLocation, repName, visitDate, shiftStart, shiftEnd, createdAt, status, synced',
  products: 'id, sessionId, productName, createdAt, synced, syncStatus, lastSyncAttempt',
  syncQueue: 'id, productId, sessionId, status, createdAt',
  deviceToken: 'id',
  stockItems: 'id, sessionId, stockName, createdAt',
});
```

**Why this works:**

Version 1 creates the base tables. Version 2 adds sync tracking fields. Version 3 adds device tokens. Version 4 adds timestamps. Version 5 adds inventory.

Nothing ever gets removed. Nothing ever breaks.

When a user on version 1 opens the app a week later, Dexie runs migrations 2, 3, 4, and 5 sequentially. Their old data stays intact, just gets new fields with sensible defaults.

I tried destructive migrations early on. Added a field in version 2, removed it in version 3 thinking I didn't need it. Bad idea. Users who went offline between versions 2 and 3 had data that couldn't sync. The server rejected it. I spent a day writing migration logic to handle the missing field.

Never again. **Now I only add, never remove.**

New fields get defaults. New tables start empty. Old clients ignore new fields they don't understand. New clients fill in missing fields from old data. Everything stays compatible.

**Server-side compatibility:**

The server needs to handle clients running different schema versions. When data syncs, the client sends its version number. The server fills in missing fields with defaults. Simple pattern because additive migrations mean old data is just a subset of new data.

```typescript
// Helper function to create a product with schema-aware defaults
export async function createProduct(product: Product): Promise<string> {
  // Set default syncStatus to 'pending' if not provided
  // Older schema versions don't have this field
  const productWithDefaults = {
    ...product,
    syncStatus: product.syncStatus || 'pending' as const,
  };
  await db.products.add(productWithDefaults);
  return product.id;
}
```

This pattern means the app can evolve its data model without breaking offline users. When you need to add functionality, you add tables and fields. When you need to change behavior, you add status flags and feature toggles. The schema grows, but it never breaks backward compatibility.

## Exponential Backoff Sync: Stop Killing Batteries

My first sync implementation checked for connectivity every 10 seconds. Simple. Reliable. And it killed the battery in 4 hours.

Field workers complained immediately. They'd start their shift at 8 AM, battery would be dead by noon. The constant network polling drained power even when the requests failed instantly.

I tried the opposite: manual sync button. Users had to remember to tap "Sync" when they saw connectivity. They forgot. Data sat in the queue for hours. When they finally remembered to sync, they'd have 50 items backed up and no idea which ones mattered.

**The solution: exponential backoff.**

Failed sync? Wait 1 second, retry.
Failed again? Wait 2 seconds, retry.
Failed again? Wait 4, then 8, then 16, max 30 seconds.

If the network is truly down, you stop hammering it. If it's intermittent, you catch brief windows of connectivity quickly.

```typescript
// Sync queue processing with exponential backoff
const MAX_RETRIES = 5;
const BASE_BACKOFF_MS = 1000; // 1 second
const MAX_BACKOFF_MS = 30000; // 30 seconds

/**
 * Calculate exponential backoff delay
 * 1s, 2s, 4s, 8s, 16s, max 30s
 */
function getBackoffDelay(retryCount: number): number {
  const delay = BASE_BACKOFF_MS * Math.pow(2, retryCount);
  return Math.min(delay, MAX_BACKOFF_MS);
}

/**
 * Check if item should be retried based on backoff timing
 */
function shouldRetryNow(item: SyncQueue): boolean {
  if (!item.lastSyncAttempt) {
    return true; // No previous attempt, process now
  }

  const backoffDelay = getBackoffDelay(item.retryCount);
  const lastAttemptTime = new Date(item.lastSyncAttempt).getTime();
  const now = Date.now();
  const timeSinceLastAttempt = now - lastAttemptTime;

  return timeSinceLastAttempt >= backoffDelay;
}
```

Battery life improved dramatically. Users could work full 8-hour shifts without charging.

**Track sync state explicitly:**

Every item in the queue has a status: pending, syncing, or failed. Prevents duplicate uploads. Shows users exactly what's happening.

```typescript
/**
 * Process a single sync queue item
 */
async function processSyncItem(item: SyncQueue): Promise<boolean> {
  try {
    // Check if we should retry now (respecting backoff)
    if (!shouldRetryNow(item)) {
      return false; // Skip this item for now
    }

    // Check if exceeded max retries
    if (item.retryCount >= MAX_RETRIES) {
      console.warn(`Max retries exceeded for queue item ${item.id}`);
      await updateSyncQueueItem(item.id, {
        status: 'failed',
        lastSyncAttempt: new Date().toISOString(),
        uploadError: 'Max retries exceeded',
      });
      await updateProduct(item.productId, {
        syncStatus: 'failed',
        uploadError: 'Max retries exceeded',
      });
      return false;
    }

    // Update status to syncing
    await updateSyncQueueItem(item.id, {
      status: 'syncing',
      lastSyncAttempt: new Date().toISOString(),
    });

    // Upload image
    const cloudUrl = await uploadImage(
      item.imageData,
      item.productId,
      item.sessionId
    );

    // Update product with cloud URL and mark as synced
    await updateProduct(item.productId, {
      imageUrl: cloudUrl,
      syncStatus: 'synced',
      synced: true,
      lastSyncAttempt: new Date().toISOString(),
      uploadError: undefined,
    });

    // Remove from sync queue
    await removeSyncQueueItem(item.id);

    return true;
  } catch (error) {
    // Update sync queue item with failure
    await updateSyncQueueItem(item.id, {
      status: 'failed',
      retryCount: Math.min(item.retryCount + 1, MAX_RETRIES),
      lastSyncAttempt: new Date().toISOString(),
      uploadError: error instanceof Error ? error.message : 'Unknown error',
    });

    return false;
  }
}
```

**What happens after days offline:**

The app doesn't try to upload 100 items simultaneously when connectivity returns. It processes them one by one, respecting the backoff delays. If one item fails, the others keep going. Much easier to debug.

Bonus: exponential backoff naturally handles API rate limiting. If the server starts returning 429 errors, the delays automatically throttle the client.

## Service Worker Authentication: The Catch-22

Traditional auth breaks offline: you need credentials to call the API, but you need the API to get credentials.

Session tokens expire after 30 minutes. Users go offline for 3 hours. Tokens are expired when they reconnect. Auth fails, sync fails, users lose data.

JWT refresh tokens don't help. Refreshing requires an API call. Can't make API calls offline.

OAuth redirects are impossible without connectivity.

**Device tokens solve this:**

Generate a long-lived token on first app launch. Store it in IndexedDB. Include it in every API request. The server tracks which tokens belong to which users and can revoke them remotely if needed.

Not as secure as short-lived sessions, but offline-first requires trade-offs. The alternative is an app that can't function offline at all.

```typescript
// Device token generation and storage
const DEVICE_TOKEN_ID = 'device-token';

/**
 * Get or create a device token for API authentication.
 * The token is generated once per device/browser and stored in IndexedDB.
 */
export async function getOrCreateDeviceToken(): Promise<string> {
  const existing = await db.deviceToken.get(DEVICE_TOKEN_ID);

  if (existing) {
    return existing.token;
  }

  // Generate a new token
  const token = `pt_${crypto.randomUUID().replace(/-/g, '')}`;
  const deviceToken: DeviceToken = {
    id: DEVICE_TOKEN_ID,
    token,
    createdAt: new Date().toISOString(),
  };

  await db.deviceToken.add(deviceToken);
  return token;
}
```

**Service workers need the token too:**

Service workers run background sync. They need auth credentials. But they can't access localStorage or cookies. Only IndexedDB works in both contexts.

```javascript
// Service worker authentication with IndexedDB
const DB_NAME = 'AppDB';
const TOKEN_STORE = 'deviceToken';
const TOKEN_ID = 'device-token';

// Get device token from IndexedDB in service worker context
async function getDeviceToken() {
  return new Promise((resolve, reject) => {
    const request = indexedDB.open(DB_NAME);

    request.onerror = () => {
      console.error('[Service Worker] Failed to open IndexedDB');
      resolve(null);
    };

    request.onsuccess = (event) => {
      const db = event.target.result;

      // Check if the store exists
      if (!db.objectStoreNames.contains(TOKEN_STORE)) {
        console.warn('[Service Worker] Token store not found');
        db.close();
        resolve(null);
        return;
      }

      const transaction = db.transaction(TOKEN_STORE, 'readonly');
      const store = transaction.objectStore(TOKEN_STORE);
      const getRequest = store.get(TOKEN_ID);

      getRequest.onsuccess = () => {
        db.close();
        if (getRequest.result && getRequest.result.token) {
          resolve(getRequest.result.token);
        } else {
          resolve(null);
        }
      };

      getRequest.onerror = () => {
        db.close();
        resolve(null);
      };
    };
  });
}

// Background sync using device token
self.addEventListener('sync', (event) => {
  if (event.tag === 'sync-images') {
    event.waitUntil(
      syncImages().then(() => {
        // Notify clients that sync completed
        return self.clients.matchAll().then((clients) => {
          clients.forEach((client) => {
            client.postMessage({ type: 'SYNC_COMPLETE', success: true });
          });
        });
      })
    );
  }
});

// Process sync queue with authentication
async function syncImages() {
  const token = await getDeviceToken();

  if (!token) {
    console.warn('[Service Worker] No device token available');
    return { success: false, error: 'No device token' };
  }

  const response = await fetch('/api/sync', {
    method: 'POST',
    headers: {
      'Content-Type': 'application/json',
      'Authorization': `Bearer ${token}`,
    },
  });

  if (!response.ok) {
    throw new Error(`Sync failed with status: ${response.status}`);
  }

  return response.json();
}
```

**Security trade-offs:**

Device tokens are less secure. Compromised token = permanent access until revoked.

But the alternative is an app that locks users out when offline. That's worse.

For the POC, this trade-off was acceptable. For production systems handling sensitive data, you'd want additional hardening:
- Server-side token revocation when devices are lost
- Device fingerprinting to detect suspicious usage
- Re-authenticate periodically when online (but never block offline work)
- Rate limiting and anomaly detection server-side
- Additional encryption layers for sensitive data in IndexedDB

Offline-first means graceful degradation: full security online, reduced security offline, never complete lockout.

## Client-Side Operations: The Server Can't Help You

Traditional web apps offload heavy work to the server. Image processing, document generation, complex calculations - all server-side.

Offline-first flips this completely. **If it requires a server, it doesn't work offline.**

**Image compression:**

Users capture 20-30 product photos per shift. 3MB each. That's 60-90MB per day over cellular connections.

Initial version: upload raw photos, compress server-side. Problem: users couldn't save photos offline. They'd capture images, lose connectivity, photos never uploaded.

Solution: compress client-side before saving to IndexedDB.

```typescript
// Client-side image compression
import imageCompression from 'browser-image-compression';

const compressionOptions = {
  maxSizeMB: 0.5, // Max 500KB
  maxWidthOrHeight: 800, // Max dimension
  useWebWorker: true,
  fileType: 'image/jpeg' as const,
  initialQuality: 0.8,
};

export async function compressImage(file: File): Promise<File> {
  try {
    const compressedFile = await imageCompression(file, compressionOptions);
    return compressedFile;
  } catch (error) {
    console.error('Image compression failed:', error);
    // Return original if compression fails
    return file;
  }
}

export async function fileToBase64(file: File): Promise<string> {
  return new Promise((resolve, reject) => {
    const reader = new FileReader();
    reader.readAsDataURL(file);
    reader.onload = () => resolve(reader.result as string);
    reader.onerror = (error) => reject(error);
  });
}
```

Result: 3MB photo becomes 400KB. 80-90% smaller. Sync time drops from 30 seconds to 3 seconds. Battery life improves. Data costs drop by 85%.

Users don't notice quality loss. 800px is plenty for product photos viewed on phones.

**Document generation:**

Field workers need to generate reports from collected data at end of shift.

Traditional approach: send data to server, server generates Word doc, user downloads.

This breaks offline. Can't generate report without connectivity.

Solution: generate documents client-side. Browser creates the Word doc using docx.js. User downloads immediately, no network required.

```typescript
// Client-side Word document generation
import { Document, Paragraph, TextRun, Packer } from 'docx';

export async function generateReport(session: SessionWithProducts): Promise<Blob> {
  const doc = new Document({
    sections: [{
      properties: {},
      children: [
        // Title
        new Paragraph({
          text: 'FIELD REPORT',
          heading: HeadingLevel.HEADING_1,
          alignment: AlignmentType.CENTER,
        }),

        // Session info
        new Paragraph({
          children: [
            new TextRun({ text: 'Location: ', bold: true }),
            new TextRun(session.storeLocation),
          ],
        }),
        new Paragraph({
          children: [
            new TextRun({ text: 'Date: ', bold: true }),
            new TextRun(format(new Date(session.visitDate), 'MMMM d, yyyy')),
          ],
        }),

        // Products data
        ...session.products.map((product, index) =>
          new Paragraph({
            children: [
              new TextRun({ text: `${index + 1}. ${product.productName}`, bold: true }),
            ],
          })
        ),
      ],
    }],
  });

  return Packer.toBlob(doc);
}
```

Users work their entire shift offline. Capture data, compress images, generate reports, download them. Zero network required.

**Why client-side is the only option:**

Anything that requires the server doesn't work offline. Period.

Trade-offs: larger bundle size, more complex client code. But modern browsers handle this well. WebWorkers prevent UI blocking. Performance is good enough.

## Testing: DevTools Lies About Real Network Conditions

Chrome DevTools has an "offline" checkbox. It's barely useful.

Real networks are messier:
- Intermittent: works 30 seconds, drops 2 minutes, comes back
- Slow: requests take 60+ seconds then timeout
- Partial: some requests succeed, others fail randomly
- Rate limited: server rejects too many requests

**What actually works:**

Create custom network profiles in DevTools. "Warehouse WiFi": 100ms latency, 500Kbps, 5% packet loss. "Moving vehicle": 500ms latency, drops every 30 seconds.

More important: field test with real devices in real conditions. App works great on simulated 3G, then breaks on actual rural 3G. The simulation doesn't capture everything.

**Service worker debugging:**

Service workers are invisible. Errors don't surface in console. You need extensive logging.

Log every cache hit, cache miss, network request, sync event. Send logs to analytics in production. You can't debug offline issues without visibility into what's happening on user devices.

**IndexedDB inspection:**

Build admin tools to inspect sync queues remotely. When a user reports issues, you need to see what's stuck in their queue. DevTools Application tab works for local testing, but you need production visibility.

## What I Learned: Mistakes and Corrections

**What worked:**

Device tokens from day one. Didn't retrofit them later. Saved months.

Sync queue as first-class data structure. Not an afterthought. Made failures visible and debuggable.

Additive-only migrations. Never broke offline users.

Client-side image compression. Sync completion jumped from 60% to 95%. Data usage dropped 80%.

**Mistakes I made:**

Mistake: Sync every 60 seconds automatically.
Result: Batteries died in 4 hours.
Fix: Exponential backoff + manual sync button for urgent cases.

Mistake: Last-write-wins conflict resolution.
Result: Users lost data when multiple devices synced.
Fix: Keep full version history, let users resolve conflicts when needed (but make auto-resolution good enough that manual is rare).

Mistake: Put too much logic in service worker.
Result: Hit size limits, couldn't access needed APIs.
Fix: Service worker does minimal work (cache, sync coordination). Main app handles business logic.

**What to skip:**

Don't build elaborate conflict resolution UIs. Users won't use them. Last-write-wins + metadata is usually enough.

Don't optimize bundle size early. Get offline functionality working first.

**What's critical:**

Invest in logging and observability. Offline-first apps fail in ways you can't reproduce locally. You need visibility into production devices.

IndexedDB queries are slower than SQL. Use indexed fields for frequent queries. Do complex filtering client-side.

Service workers add complexity. But for offline-first, the trade-off is worth it.

## When Offline-First Is Worth the Complexity

Don't build offline-first unless you actually need it. It's complex. More code. Larger bundles. Harder testing. Careful data modeling.

**You need it if:**

Users work in unreliable connectivity. Field workers. Warehouses. Rural areas. International travel. Remote locations.

Users have unreliable devices. Older smartphones. Budget tablets. Limited data plans.

Users perform critical workflows that can't be interrupted. Emergency responders. Field technicians. Sales reps capturing orders.

**When you don't need it:**

Users mostly have good connectivity. Occasional offline is acceptable.

Users work on desktops with reliable office WiFi.

For these cases: cache static assets aggressively, use optimistic UI updates, handle errors gracefully. Don't build full offline-first.

**What's next:**

Background Sync API is maturing. Periodic Background Sync will enable better automatic sync.

Better conflict resolution (CRDT-style operational transforms) is possible. But the complexity often isn't worth it for simple data models.

Offline-first is a commitment. It shapes every architectural decision. But for users who need it, it's the difference between productive work and constant frustration.

This POC ran successfully during testing. Field workers tested it for several weeks. Zero data loss during that period. Sync completion hit 95%+. Battery lasted full shifts.

The client decided not to move forward with the project for business reasons unrelated to the technical implementation. But the architectural patterns work, and the lessons apply to any offline-first scenario.

Worth the complexity if you need it.

---

**About the Author:** Aaron Lamb is the founder of Hexaxia Technologies, specializing in cybersecurity consulting, infrastructure engineering, and AI product development.
