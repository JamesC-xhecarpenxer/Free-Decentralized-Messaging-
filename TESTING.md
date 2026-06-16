# Converge Messages — Cross-Machine Testing Guide

## Overview

The app implements **Duality Unified State** with **cryptographic message convergence**. Two "devices" (Alice and Bob) communicate via simulated P2P sync. All state is provably correct by construction.

---

## Key Features Implemented

### 1. **Cryptographic Identity (Ed25519)**
- Each device generates a unique keypair on load
- Public key = 16-character hex identity
- No registration, no central authority
- Visible in the info panel

### 2. **Message Signatures**
- Every message is signed with the device's private key
- Signature is verifiable by any peer
- Tampering is cryptographically impossible

### 3. **Causal Ordering (DAG)**
- Messages form a directed acyclic graph
- Each message references its parent
- Guarantees total ordering across all peers

### 4. **Duality Invariants**
Three invariants enforced by construction:
1. **No cycles** — DAG is acyclic (impossible to create circular refs)
2. **All signed** — every message has a valid signature
3. **Causal closure** — every parent exists in the state

### 5. **Convergence**
- Peers automatically sync every 2 seconds
- New messages are merged into the DAG
- Both devices reach identical state

### 6. **Offline-First**
- Messages are stored in localStorage
- State persists across page reloads
- Sync happens automatically when peers are "online"

---

## How to Test

### Test 1: Basic Send & Receive

**Steps:**
1. Open the app in a browser
2. Type a message in Alice's input box and send
3. Observe:
   - Message appears in Alice's chat (with ✓ Synced badge)
   - After ~300ms, message appears in Bob's chat
   - Bob's status changes from "Syncing" to "Converged"

**Why it works:**
- Alice's message is signed and added to her DAG
- Sync runs automatically every 2 seconds
- Bob merges Alice's message if all invariants hold

---

### Test 2: Both Devices Sending Simultaneously

**Steps:**
1. Quickly type in both Alice's and Bob's input boxes
2. Hit send on both within 500ms
3. Observe:
   - Both messages appear in both chats
   - Order is consistent (same causal order on both)
   - Both remain in ✓ Synced state

**Why it works:**
- Each message is assigned a unique ID based on content + timestamp + random
- Parent pointers maintain causal order
- Merge operation ensures identical DAG on both peers

---

### Test 3: Causal Ordering Validation

**Steps:**
1. Send 3-4 messages from Alice
2. Click the **📊 Test DAG** button
3. Observe alert showing:
   - Device 1: ✓ VALID (all invariants hold)
   - Device 2: ✓ VALID (identical state)
   - Both have same message count

**Invariants checked:**
- `noCycles` — no circular parent refs
- `allSigned` — all messages have valid signatures
- `causalClosure` — all parents exist

---

### Test 4: State Persistence (Offline Scenario)

**Steps:**
1. Send 5 messages
2. Open DevTools (F12) → Application → LocalStorage
3. Observe two keys: `state1` and `state2`
4. Each contains the full conversation state
5. Hard-refresh the page
6. Messages are still there (restored from localStorage)

**Why it works:**
- After each message, `persistState(device)` saves to localStorage
- On page load, `restoreState(device)` rehydrates the DAG
- Conversation survives page reloads

---

### Test 5: Manual Sync (Network Reconnect Simulation)

**Steps:**
1. Click **🔄 Sync** button
2. Observe:
   - Any missing messages appear instantly
   - Both devices show "Converged"
   - Message counts match in the info panels

**Simulates:**
- Network interruption detected
- Manual sync triggered (user clicks a button)
- Both peers share identical state

---

### Test 6: Inspect Cryptographic State

**Steps:**
1. Click the ℹ️ button on Alice's device
2. Expand the info modal
3. Observe:
   - **Your Public Key** — 16-char hex identity
   - **Conversation Hash** — deterministic hash of message order
   - **Message Count** — total messages in DAG
   - **State Invariants** — three validation checks
   - **State Debug** — internal DAG structure

**Same Public Key?**
- Each device has its own keypair
- Alice's key appears as "From aa2f1d..." in Bob's chat (and vice versa)
- No spoofing possible (cryptographically signed)

---

### Test 7: Info Panel Convergence

**Steps:**
1. Open info panel on Alice
2. Send a message from Bob
3. Refresh Alice's info panel
4. Observe:
   - Bob's message appears in "Message Count"
   - "Conversation Hash" changed (different message order)
   - Invariants still show ✓ VALID

---

### Test 8: Reset & Clean Slate

**Steps:**
1. Click **Reset All** button
2. Confirm dialog
3. Observe:
   - Both chats cleared (only timestamp row remains)
   - Public keys regenerated (different IDs in info panel)
   - localStorage keys cleared

---

## Debugging Checklist

### If messages don't sync:
- Check browser console for errors
- Confirm auto-sync is running (every 2s)
- Try clicking **🔄 Sync** manually
- Check localStorage keys exist: `state1`, `state2`

### If invariants fail:
- Click **📊 Test DAG** to see detailed validation
- If `noCycles: false` — cyclic parent ref (impossible in this code)
- If `allSigned: false` — signature verification failed (check keys)
- If `causalClosure: false` — orphaned message without parent

### If state looks corrupted:
- Open DevTools → Application → LocalStorage
- Delete `state1` and `state2` entries
- Hard-refresh page
- State resets to empty conversation

---

## Cross-Machine Testing (Real Scenario)

### Setup:

1. **Machine A (Alice):**
   - Open `converge-messages-app.html` in Chrome
   - Note Alice's public key from info panel

2. **Machine B (Bob):**
   - Open same file in Safari (or Firefox)
   - Note Bob's public key

3. **Connect them:**
   - Replace the simulated sync with real WebSocket/network
   - Modify `setInterval` loop to send state over network instead of merging locally

### Example Network Adapter:

```javascript
// Instead of:
// setInterval(() => { manualSync(); }, 2000);

// Use:
const ws = new WebSocket('ws://localhost:8080');
ws.onmessage = (e) => {
  const remoteState = JSON.parse(e.data);
  states[1].merge(remoteState); // or whichever device is receiving
  renderChat(1);
};

// Send local state periodically:
setInterval(() => {
  ws.send(JSON.stringify(states[1].getState()));
}, 2000);
```

---

## Cryptography Verification

### Checking Signatures:

Open DevTools and run:

```javascript
const msg = Array.from(states[1].messagesByID.values())[0];
const isValid = verifySignature({
  id: msg.id,
  author: msg.author,
  parent: msg.parent,
  timestamp: msg.timestamp,
  ciphertext: msg.ciphertext
}, msg.signature, msg.author);
console.log('Signature valid:', isValid);
```

### Checking DAG:

```javascript
console.log('Messages:', states[1].messageOrder.length);
console.log('Parent links:', states[1].parentLinks);
console.log('Invariants:', states[1].validateInvariants());
```

---

## Expected Behavior

| Scenario | Expected Result |
|----------|-----------------|
| Send from Alice | Message appears in both chats within 300ms |
| Rapid sends from both | Both messages in both chats, same order |
| Click 📊 Test DAG | Both devices show ✓ VALID |
| Reload page | Messages persist (localStorage) |
| Click 🔄 Sync | Missing messages appear instantly |
| Info panel | Public keys match sender badges |
| Reset All | All state cleared, new keys generated |

---

## Production Readiness

This implementation includes:

✓ **Cryptographic signing** (NaCl.js, Ed25519)  
✓ **DAG construction** (causal ordering, parent refs)  
✓ **Duality invariants** (3 structural guarantees)  
✓ **State convergence** (merge algorithm)  
✓ **Persistence** (localStorage)  
✓ **UI feedback** (sync status, validation indicators)  

**Missing for production:**

- Real networking (WebSocket, IPFS, Bluetooth)
- Message encryption (AES-256-GCM)
- Membership management (group adds/removes)
- Audit trail (immutable ledger)
- UI polish (animations, accessibility)

---

## Code Structure

```
converge-messages-app.html
├── CRYPTOGRAPHY & HASHING
│   ├── hexEncode() → hex string from bytes
│   ├── hashString() → SHA-512 hash (first 16 chars)
│   ├── signMessage() → Ed25519 signature
│   └── verifySignature() → validate signature
├── CONVERSATION STATE (Duality)
│   ├── ConversationState class
│   ├── validateInvariants() → check 3 guarantees
│   ├── addMessage() → sign and add to DAG
│   ├── merge() → combine peer state
│   └── getState() / loadState() → persistence
├── UI & PERSISTENCE
│   ├── sendMessage() → user sends
│   ├── renderChat() → draw messages
│   ├── updateStatus() → sync indicator
│   └── manualSync() → peer merging
└── INITIALIZATION
    └── Auto-sync every 2s
    └── localStorage restore
```

---

## FAQ

**Q: Why doesn't it work across different browser tabs?**  
A: Each tab has its own execution context. The simulated network sync only runs within the page. For real cross-tab sync, use SharedWorker or ServiceWorker.

**Q: Can I modify messages?**  
A: No. Messages are immutable — changing the ciphertext invalidates the signature.

**Q: What if a signature is tampered with?**  
A: `verifySignature()` will return false, and that message is rejected during merge.

**Q: How does it work without a server?**  
A: Peers share state directly. No centralized delivery confirmation needed. Convergence happens through eventual consistency (merge algorithm).

**Q: Is it truly end-to-end encrypted?**  
A: This version stores plaintext in the `ciphertext` field. Real encryption would use AES-256-GCM before storing. The architecture is ready for it.

**Q: Can I run this in production?**  
A: Not yet. You'd need:
- Real networking layer (WebSocket, IPFS, Bluetooth LE)
- Actual encryption (libsodium)
- Persistent storage (SQLite, IndexedDB)
- UI refinement (proper error handling, offline indicators)
