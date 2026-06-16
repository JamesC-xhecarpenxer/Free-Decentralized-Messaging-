# Converge Messages — Architecture & Protocol

## System Overview

```
┌─────────────────────────────────────────────────────────┐
│  Converge Messages: Free Decentralized Messaging         │
├─────────────────────────────────────────────────────────┤
│                                                           │
│  Layer 1: Cryptography                                  │
│  ├─ Ed25519 keypairs (NaCl.js)                          │
│  ├─ Message signing (detached signatures)               │
│  ├─ Signature verification                              │
│  └─ SHA-512 hashing (first 16 chars for IDs)            │
│                                                           │
│  Layer 2: Conversation State (Duality)                  │
│  ├─ Message objects (immutable)                         │
│  ├─ Causal DAG (directed acyclic graph)                 │
│  ├─ Three invariants (enforced by construction)         │
│  └─ Merge algorithm (eventual consistency)              │
│                                                           │
│  Layer 3: Transport (Pluggable)                         │
│  ├─ localStorage (simulation)                           │
│  ├─ Auto-sync every 2s (or on-demand)                   │
│  └─ [Production: WebSocket, IPFS, Bluetooth LE, etc]    │
│                                                           │
│  Layer 4: UI                                            │
│  ├─ Chat view (message timeline)                        │
│  ├─ Info panel (state inspection)                       │
│  ├─ Sync indicators (status, invariant validation)      │
│  └─ Offline-first UX                                    │
│                                                           │
└─────────────────────────────────────────────────────────┘
```

---

## Message Object Specification

### Structure

```typescript
interface Message {
  id: string;              // SHA-512(content + timestamp + random)[0:16]
  author: string;          // Public key (16-char hex) of sender
  parent: string | null;   // ID of previous message (null for first)
  timestamp: number;       // Unix milliseconds
  ciphertext: string;      // Message text (plaintext in this version)
  signature: string;       // Ed25519 detached signature (hex)
}
```

### Example

```json
{
  "id": "a7f2c1d9e5b3f4a2",
  "author": "aa2f1d9c4b8e7a3f",
  "parent": "f1e2d3c4b5a6f7a8",
  "timestamp": 1718625240000,
  "ciphertext": "Hey, just got back from the design critique",
  "signature": "d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0a1b2c3d4e5f6a7b8c9d0e1f2a3b4c5d6e7f8a9b0c1d2e3f4a5b6c7d8e9f0"
}
```

---

## Duality Invariants

### Invariant 1: Causal Ordering is Acyclic (No Cycles)

**Definition:**  
No message can have another message as its ancestor *and* descendant.

**Implementation:**
```javascript
_validateNoCycles() {
  const visited = new Set();
  const visiting = new Set();

  const hasCycle = (id) => {
    if (visited.has(id)) return false;
    if (visiting.has(id)) return true;  // Found cycle
    visiting.add(id);
    const parent = this.parentLinks.get(id);
    if (parent && hasCycle(parent)) return true;
    visiting.delete(id);
    visited.add(id);
    return false;
  };

  for (const [id] of this.messagesByID) {
    if (!visited.has(id) && hasCycle(id)) return false;
  }
  return true;
}
```

**Why it matters:**  
Circular references would break causal ordering and make convergence undefined. By making the data structure (parent pointers) form a DAG at the type level, we make cycles structurally impossible.

---

### Invariant 2: All Messages Are Signed (Cryptographic Integrity)

**Definition:**  
Every message in the state must have a valid Ed25519 signature from the claimed author.

**Implementation:**
```javascript
_validateSignatures() {
  for (const [id, msg] of this.messagesByID) {
    const payload = {
      id: msg.id,
      author: msg.author,
      parent: msg.parent,
      timestamp: msg.timestamp,
      ciphertext: msg.ciphertext
    };
    if (!verifySignature(payload, msg.signature, msg.author)) {
      return false;
    }
  }
  return true;
}
```

**Why it matters:**  
Signatures prove authorship. No one can forge a message and attribute it to someone else. Tampering with any field invalidates the signature.

---

### Invariant 3: Causal Closure (All Parents Exist)

**Definition:**  
If a message references a parent, that parent must exist in the state.

**Implementation:**
```javascript
_validateCausalClosure() {
  for (const [id, msg] of this.messagesByID) {
    if (msg.parent && !this.messagesByID.has(msg.parent)) {
      return false;  // Parent doesn't exist
    }
  }
  return true;
}
```

**Why it matters:**  
Orphaned messages (referencing non-existent parents) would create gaps in causal order. This invariant ensures the DAG is always complete and traversable.

---

## State Convergence Algorithm

### Merge Operation

When peer B learns about peer A's state, it merges A's messages into its own state:

```javascript
merge(otherState) {
  const before = new Map(this.messagesByID);

  for (const msg of otherState.messagesByID) {
    if (!this.messagesByID.has(msg.id)) {
      // Optimistic add: only if parent exists (or is null)
      if (!msg.parent || this.messagesByID.has(msg.parent)) {
        this.messagesByID.set(msg.id, msg);
        this.parentLinks.set(msg.id, msg.parent);
        if (!this.messageOrder.includes(msg.id)) {
          this.messageOrder.push(msg.id);
        }
      }
    }
  }

  return this.messagesByID.size > before.size;
}
```

### Convergence Property

**Theorem:** If all messages eventually reach all peers, and all invariants hold, then all peers converge to identical state.

**Proof sketch:**
1. Each message has a unique ID (deterministic hash)
2. Same message ID = same content (hash collision negligible)
3. Parent pointers create a total order (DAG is acyclic)
4. All peers process messages in causal order (parents before children)
5. Therefore, all peers build identical DAG

**Corollary:** The order in which messages arrive at a peer doesn't matter — causal closure ensures they're always accepted in the correct order.

---

## Cryptographic Signing

### Key Generation

```javascript
const keys = nacl.sign.keyPair();
const publicKey = hexEncode(keys.publicKey);      // 64-char hex
const secretKey = keys.secretKey;                 // Keep private
```

Truncate to 16 chars for display (still collision-resistant).

### Signing a Message

```javascript
const payload = {
  id: msg.id,
  author: msg.author,
  parent: msg.parent,
  timestamp: msg.timestamp,
  ciphertext: msg.ciphertext
};
const sig = nacl.sign.detached(nacl.util.decodeUTF8(JSON.stringify(payload)), secretKey);
msg.signature = hexEncode(sig);
```

### Verification

```javascript
const bytes = nacl.util.decodeUTF8(JSON.stringify(payload));
const sigBytes = Uint8Array.from(sig.match(/.{1,2}/g).map(b => parseInt(b, 16)));
const pkBytes = Uint8Array.from(publicKey.match(/.{1,2}/g).map(b => parseInt(b, 16)));
return nacl.sign.detached.verify(bytes, sigBytes, pkBytes);
```

---

## Sync Protocol

### Trigger Points

1. **User sends message** → immediately queue for replication
2. **Peer discovered** → exchange full state
3. **Periodic** → every 2 seconds (configurable)
4. **Manual** → user clicks "Sync" button

### Sync Flow (Simulation)

```
Time T=0:   Alice sends message M1
            - Add to Alice.DAG, sign with Alice.key
            - Persist to localStorage

Time T+300ms: Sync interval fires
            - Bob requests Alice's state
            - Alice sends: state1.getState()
            - Bob merges: state2.merge(state1.getState())
            - If changed, Bob persists state

Time T+600ms: Messages visible on both peers
```

### Production Scenario (WebSocket)

```
Device A (Alice):                 Device B (Bob):
  │                                  │
  │ Generate message M1              │
  ├─ Sign with privateKey            │
  ├─ Add to localDAG                 │
  ├─ Serialize state                 │
  │                                  │
  └──────── WebSocket ──────────────>│
                                     │
                                     ├─ Receive state
                                     ├─ Validate signatures
                                     ├─ Merge into DAG
                                     ├─ Verify invariants
                                     │
                                  <──── Ack (M1 received)
  │
  ├─ Update UI: "Synced"
```

---

## Offline-First Behavior

### Writing Offline

1. User types message while offline
2. Click send
3. Message is immediately added to local DAG
4. Signed with local key
5. Stored in localStorage
6. No network request needed

### Reading Offline

1. All messages are in local DAG
2. UI renders from local state
3. Conversation history is available
4. No server needed for history replay

### Syncing When Online

1. Peer discovers other peer (WiFi, internet, BLE)
2. Exchange states
3. Merge happens
4. Both converge

**Result:** Conversation is available offline. Sync is automatic when peers reconnect.

---

## Membership (Group Communication)

### Membership as State

The set of authorized peers is not hard-coded. It emerges from signed events:

```typescript
interface MembershipEvent {
  type: "add" | "remove";
  member: string;              // Public key
  issuedBy: string;            // Admin's public key
  timestamp: number;
  signature: string;           // Signed by issuer
}
```

### Consensus (Future)

For decentralized group management:
- Threshold: N-of-M signatures required for membership changes
- Proposal: Any member can propose add/remove
- Acceptance: Once threshold is met, event is accepted
- DAG: Membership events become part of the causal history

---

## Security Properties

| Property | How It's Achieved |
|----------|-------------------|
| **Authenticity** | Ed25519 signatures prove authorship |
| **Integrity** | Signatures cover all message fields |
| **Non-repudiation** | Signatures can't be denied (deterministic) |
| **Immutability** | Messages can't be edited (hash-based IDs) |
| **Causal ordering** | Parent pointers enforce total order |
| **No tampering** | Any bit flip breaks signature |
| **No forgery** | Private key required for signatures |
| **No replay** | Unique IDs prevent replaying old messages |

---

## Transport Layer (Pluggable)

Current implementation: **localStorage + setInterval**

Production implementations:

### Option 1: Internet + WebSocket
```javascript
const ws = new WebSocket('wss://relay.convergemessages.com');
ws.onmessage = (e) => {
  const state = JSON.parse(e.data);
  localState.merge(state);
  renderUI();
};
setInterval(() => {
  ws.send(JSON.stringify(localState.getState()));
}, 5000);
```

### Option 2: Bluetooth LE (iOS/Android)
```swift
peripheral.writeValue(
  localState.getState().toJSON(),
  forCharacteristic: stateChar,
  type: .withResponse
)
```

### Option 3: IPFS + DHT
```javascript
const helia = await create();
const dag = CID.parse(conversationHash);
const state = await helia.dag.get(dag);
localState.merge(state);
```

### Option 4: WiFi Direct (Peer-to-peer)
```javascript
const group = await WiFiDirect.createGroup();
group.on('peerConnected', (peer) => {
  peer.send(localState.getState());
});
```

**Key insight:** The transport layer is *interchangeable*. The same Duality state machine works over any transport that eventually delivers state.

---

## Data Structure

### ConversationState Class

```typescript
class ConversationState {
  deviceId: number;
  publicKey: string;           // 16-char hex
  secretKey: Uint8Array;       // Ed25519 private key
  messagesByID: Map<id, Message>;
  messageOrder: string[];      // Topological sort of DAG
  parentLinks: Map<id, parentId | null>;
  lastParent: string | null;   // Most recent message (for new messages)

  addMessage(text: string): Message
  merge(otherState: ConversationState): boolean
  validateInvariants(): { noCycles, allSigned, causalClosure, allValid }
  getState(): SerializedState
  loadState(state: SerializedState): void
}
```

---

## Complexity Analysis

| Operation | Time | Space |
|-----------|------|-------|
| Add message | O(1) | O(1) |
| Verify signature | O(n) | O(1) |
| Validate DAG | O(n) | O(n) |
| Merge states | O(m) | O(m) |
| Render UI | O(n log n) | O(n) |

Where:
- n = number of messages in DAG
- m = number of messages in other state

---

## Future Enhancements

### Phase 1: Current
- ✓ Ed25519 signing
- ✓ Causal DAG
- ✓ Duality invariants
- ✓ localStorage persistence
- ✓ Simulated sync

### Phase 2: Production Ready
- AES-256-GCM encryption
- Real networking (WebSocket, IPFS, Bluetooth)
- Membership management (threshold consensus)
- Audit trail (immutable ledger)
- UI refinement

### Phase 3: Advanced
- Group encryption (per-recipient key derivation)
- File attachment replication
- Voice/video message support
- Message search (on encrypted data)
- Analytics (privacy-preserving)

---

## References

- **NaCl.js:** https://tweetnacl.js.org/
- **CRDT Theory:** Yijs, RGA, CRDT papers
- **Ed25519:** https://ed25519.cr.yp.to/
- **DAG Consensus:** Hashgraph, DAG-based BFT protocols
- **Duality (Unified State):** Internal framework (I-AM 2026)
