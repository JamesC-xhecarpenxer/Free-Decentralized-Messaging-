# Converge Messages — Quick Start

## Open the App

1. Download `converge-messages-app.html`
2. Open in any modern browser
3. You'll see two "devices" side-by-side (Alice and Bob)

## Send a Message

1. Type in Alice's input box
2. Press Enter or click →
3. Message appears in Alice's chat with ✓ Synced badge
4. After ~300ms, message appears in Bob's chat too

## Try These Interactions

### Conversation Info
- Click **ℹ️** button to see:
  - Your public key (cryptographic identity)
  - Conversation hash
  - Message count
  - State validation (3 invariants)

### State Validation
- Click **📊 Test DAG** button
- Alert shows if both devices are valid
- Both should say ✓ VALID

### Manual Sync
- Click **🔄 Sync** to force peer sync
- Useful if auto-sync (every 2s) didn't catch changes

### Network Simulation
- Click **🔌 Disconnect** to see status
- Messages still work — they'll sync when you reconnect

### Reset
- Click **Reset All** to start fresh
- Both devices get new keypairs

---

## What's Happening Under the Hood

### Each message:
1. Gets a unique ID (hash of content + timestamp)
2. References previous message (parent)
3. Is signed with your private key
4. Syncs to the other peer automatically

### Both devices:
- Have identical conversation state
- Can verify each other's signatures
- Maintain a causal DAG (directed acyclic graph)
- Never need a server

### Three Invariants (always true):
1. ✓ No cycles in parent refs
2. ✓ All messages are signed
3. ✓ All parents exist

---

## Testing Offline

### Scenario: Reload Page
1. Send 5 messages
2. Hard-refresh (Ctrl+R or Cmd+R)
3. Messages are still there
4. State is restored from localStorage

### Scenario: Rapid Sends
1. Type in both boxes quickly
2. Send from both within 500ms
3. Both messages appear in both chats
4. Order is consistent

---

## Open DevTools for Debug

Press **F12** (or right-click → Inspect):

### Check localStorage:
```
Application → LocalStorage → state1, state2
```
Both contain the full conversation as JSON.

### Check a message signature:
```javascript
const msg = Array.from(states[1].messagesByID.values())[0];
console.log('Message:', msg);
console.log('Is valid?', verifySignature({...}, msg.signature, msg.author));
```

### See the DAG:
```javascript
console.log('Parent links:', states[1].parentLinks);
console.log('Message order:', states[1].messageOrder);
console.log('Invariants:', states[1].validateInvariants());
```

---

## Key Concepts

| Concept | What It Does |
|---------|------------|
| **Public Key** | Your cryptographic identity (16-char hex) |
| **Signature** | Proof that you created a message (Ed25519) |
| **Parent** | The message you're replying to (creates causal order) |
| **DAG** | Directed acyclic graph of all messages |
| **Convergence** | Both peers have identical state |
| **Invariant** | A guarantee enforced by construction |

---

## Real-World Use

This prototype is a **single-browser simulation**. In production:

- **Device A & B** would be two actual phones/computers
- **Network** would be WiFi, Bluetooth LE, or internet
- **Messages** would be encrypted (AES-256-GCM)
- **Storage** would be encrypted + replicated (IPFS, SCMP)

But the **core protocol** (signatures, DAG, convergence) would be identical.

---

## Troubleshooting

**Messages not syncing?**
- Auto-sync runs every 2s. Wait a moment.
- Click **🔄 Sync** to force it.
- Check console (F12) for errors.

**Invariants failing?**
- Should never happen. If it does, it's a bug in the code.
- Click **Reset All** and start fresh.

**Confused about public keys?**
- Each device has its own keypair.
- Alice's key appears as sender badge in Bob's chat.
- Same key every time until you reset.

**Can't find localStorage?**
- Open DevTools → Application → LocalStorage
- Look for `state1` and `state2`
- Each is a JSON dump of that device's conversation state

---

## What to Show Someone

1. **Send a message** → appears in both chats
2. **Open info panel** → show cryptographic identity + key
3. **Click Test DAG** → both devices valid
4. **Reload page** → message persists
5. **Open localStorage** → show state is stored locally, not on a server

Then say: _"No phone company. No messaging company. No accounts. No cloud. Just people communicating."_
