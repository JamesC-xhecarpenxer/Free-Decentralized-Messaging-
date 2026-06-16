# Real Networking — From Simulation to Production

The current app simulates sync via localStorage and setInterval. This guide shows how to adapt it for real peer-to-peer communication.

---

## Current Architecture (Simulation)

```
┌──────────────┐         ┌──────────────┐
│  Alice (Tab) │         │  Bob (Tab)   │
│              │         │              │
│  states[1]   │         │  states[2]   │
└──────┬───────┘         └────┬─────────┘
       │                      │
       └──────────────────────┘
              localStorage
            (shared storage)
```

**How it works:**
1. Alice's message → added to `states[1]`
2. Persisted to localStorage
3. setInterval every 2s → reads from localStorage
4. Merge into `states[2]`
5. Render Bob's view

**Limitation:** Only works in same browser on same machine.

---

## Real Network Architecture

```
┌──────────────┐                    ┌──────────────┐
│  Alice (iOS) │                    │  Bob (Mac)   │
│              │                    │              │
│  states[1]   │                    │  states[2]   │
└──────┬───────┘                    └────┬─────────┘
       │                                 │
       │     Bluetooth LE / WiFi         │
       │   / Internet / IPFS / Tor       │
       │                                 │
       └─────────── Network ─────────────┘
```

---

## Option 1: WebSocket (Internet)

### Server Code (Node.js + Express)

```javascript
// server.js
const WebSocket = require('ws');
const http = require('http');
const express = require('express');

const app = express();
const server = http.createServer(app);
const wss = new WebSocket.Server({ server });

const peers = new Map(); // device_id -> WebSocket

wss.on('connection', (ws) => {
  let deviceId = null;

  ws.on('message', (data) => {
    const msg = JSON.parse(data);

    if (msg.type === 'register') {
      deviceId = msg.deviceId;
      peers.set(deviceId, ws);
      console.log(`Device ${deviceId} connected`);
    }

    if (msg.type === 'state') {
      // Broadcast state to all other peers
      for (const [otherId, otherWs] of peers) {
        if (otherId !== deviceId) {
          otherWs.send(JSON.stringify({
            type: 'state',
            from: deviceId,
            state: msg.state
          }));
        }
      }
    }
  });

  ws.on('close', () => {
    if (deviceId) {
      peers.delete(deviceId);
      console.log(`Device ${deviceId} disconnected`);
    }
  });
});

server.listen(8080, () => {
  console.log('Converge Messages server on ws://localhost:8080');
});
```

### Client Code (Modify converge-messages-app.html)

Replace the setInterval sync with WebSocket:

```javascript
// Remove this:
// setInterval(() => { manualSync(); }, 2000);

// Add this:
let ws = null;

function connectWebSocket(deviceId) {
  ws = new WebSocket(`ws://localhost:8080`);

  ws.onopen = () => {
    // Register this device
    ws.send(JSON.stringify({
      type: 'register',
      deviceId: deviceId
    }));
    console.log(`Device ${deviceId} registered`);
  };

  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);

    if (msg.type === 'state') {
      const otherState = msg.state;
      const otherDeviceId = msg.from;

      // Merge peer's state into local state
      const changed = states[deviceId].merge(otherState);
      if (changed) {
        persistState(deviceId);
        renderChat(deviceId);
        updateStatus(deviceId);
      }
    }
  };

  ws.onerror = (err) => {
    console.error('WebSocket error:', err);
    // Retry after 3 seconds
    setTimeout(() => connectWebSocket(deviceId), 3000);
  };

  ws.onclose = () => {
    console.log('Disconnected. Reconnecting...');
    setTimeout(() => connectWebSocket(deviceId), 3000);
  };
}

// Periodically send local state to peers
setInterval(() => {
  if (ws && ws.readyState === WebSocket.OPEN) {
    ws.send(JSON.stringify({
      type: 'state',
      state: states[1].getState()  // or states[2] for Bob
    }));
  }
}, 5000);

// Connect on page load
window.addEventListener('load', () => {
  const deviceId = localStorage.getItem('deviceId') || 1;
  connectWebSocket(deviceId);
});
```

### Run It

```bash
# Terminal 1: Start server
node server.js

# Terminal 2: Open Alice's client
open converge-messages-app.html

# Terminal 3: Open Bob's client (in different browser or incognito)
open converge-messages-app.html
```

Now Alice and Bob can message across different browsers/machines!

---

## Option 2: Bluetooth LE (iOS/Android)

### Swift Code (iOS)

```swift
import CoreBluetooth

class ConvergeMessagesPeripheral: NSObject, CBPeripheralManagerDelegate {
  var peripheralManager: CBPeripheralManager!
  let serviceUUID = CBUUID(string: "12345678-1234-5678-1234-567812345678")
  let stateCharUUID = CBUUID(string: "87654321-4321-8765-4321-876543218765")

  override init() {
    super.init()
    peripheralManager = CBPeripheralManager(
      delegate: self,
      queue: DispatchQueue.main
    )
  }

  func peripheralManagerDidUpdateState(_ peripheral: CBPeripheralManager) {
    if peripheral.state == .poweredOn {
      let service = CBMutableService(type: serviceUUID, primary: true)
      let stateChar = CBMutableCharacteristic(
        type: stateCharUUID,
        properties: [.read, .write, .notify],
        value: nil
      )
      service.characteristics = [stateChar]
      peripheralManager.add(service)
      
      let advData: [String: Any] = [
        CBAdvertisementDataServiceUUIDsKey: [serviceUUID]
      ]
      peripheralManager.startAdvertising(advData)
    }
  }

  func peripheralManager(
    _ peripheral: CBPeripheralManager,
    didReceiveWrite requests: [CBATTRequest]
  ) {
    for request in requests {
      if let data = request.value,
         let stateJSON = String(data: data, encoding: .utf8) {
        let otherState = try? JSONDecoder().decode(
          ConversationState.self,
          from: stateJSON.data(using: .utf8)!
        )
        if let otherState = otherState {
          localState.merge(otherState)
          updateUI()
        }
      }
      peripheral.respond(to: request, withResult: .success)
    }
  }
}

class ConvergeMessagesCentral: NSObject, CBCentralManagerDelegate, CBPeripheralDelegate {
  var centralManager: CBCentralManager!
  let serviceUUID = CBUUID(string: "12345678-1234-5678-1234-567812345678")

  override init() {
    super.init()
    centralManager = CBCentralManager(delegate: self, queue: DispatchQueue.main)
  }

  func centralManagerDidUpdateState(_ central: CBCentralManager) {
    if central.state == .poweredOn {
      centralManager.scanForPeripherals(
        withServices: [serviceUUID],
        options: nil
      )
    }
  }

  func centralManager(
    _ central: CBCentralManager,
    didDiscover peripheral: CBPeripheral,
    advertisementData: [String: Any],
    rssi RSSI: NSNumber
  ) {
    centralManager.connect(peripheral, options: nil)
  }

  func centralManager(
    _ central: CBCentralManager,
    didConnect peripheral: CBPeripheral
  ) {
    peripheral.delegate = self
    peripheral.discoverServices([serviceUUID])
  }

  func peripheral(
    _ peripheral: CBPeripheral,
    didDiscoverCharacteristicsFor service: CBService,
    error: Error?
  ) {
    for char in service.characteristics ?? [] {
      // Write local state to peer
      let stateData = localState.getState().toJSON()
      peripheral.writeValue(stateData, for: char, type: .withResponse)
      peripheral.setNotifyValue(true, for: char)
    }
  }

  func peripheral(
    _ peripheral: CBPeripheral,
    didUpdateValueFor characteristic: CBCharacteristic,
    error: Error?
  ) {
    if let data = characteristic.value {
      let otherState = try? JSONDecoder().decode(
        ConversationState.self,
        from: data
      )
      if let otherState = otherState {
        localState.merge(otherState)
        updateUI()
      }
    }
  }
}
```

---

## Option 3: IPFS (Decentralized)

### JavaScript (Helia + IPFS)

```javascript
import { create } from 'helia';
import { json } from '@helia/json';
import { createLibp2p } from 'libp2p';

let h = null;
let j = null;

async function initIPFS() {
  h = await create();
  j = json(h);

  // Subscribe to new peers
  h.libp2p.addEventListener('peer:discovery', (evt) => {
    console.log('Peer discovered:', evt.detail.id.toString());
  });

  // Start periodic state replication
  setInterval(async () => {
    const state = states[1].getState();
    try {
      const cid = await j.put(state);
      console.log('State stored:', cid.toString());

      // Announce to DHT
      await h.libp2p.contentRouting.provide(cid);

      // Fetch peers' state
      const allCIDs = await h.libp2p.contentRouting.findProviders(cid);
      for await (const provider of allCIDs) {
        try {
          const peerState = await j.get(cid);
          if (peerState) {
            states[1].merge(peerState);
            renderChat(1);
          }
        } catch (e) {
          // Peer not reachable
        }
      }
    } catch (e) {
      console.error('IPFS error:', e);
    }
  }, 10000);
}

// Start IPFS on page load
window.addEventListener('load', () => {
  initIPFS();
});
```

---

## Option 4: WiFi Direct (Android)

### Kotlin Code

```kotlin
import android.net.wifi.p2p.WifiP2pManager
import android.net.wifi.p2p.WifiP2pConfig

class ConvergeMessagesWifiP2p(context: Context) {
  private val wifiP2pMgr = context.getSystemService(Context.WIFI_P2P_SERVICE) as WifiP2pManager
  private val channel = wifiP2pMgr.initialize(context, Looper.getMainLooper(), null)

  fun discoverPeers() {
    wifiP2pMgr.discoverPeers(channel, object : WifiP2pManager.ActionListener {
      override fun onSuccess() {
        Log.d("WiFi Direct", "Peer discovery started")
      }

      override fun onFailure(reason: Int) {
        Log.e("WiFi Direct", "Discovery failed: $reason")
      }
    })
  }

  fun connectToPeer(deviceAddress: String) {
    val config = WifiP2pConfig().apply {
      deviceAddress = deviceAddress
    }

    wifiP2pMgr.connect(channel, config, object : WifiP2pManager.ActionListener {
      override fun onSuccess() {
        Log.d("WiFi Direct", "Connection initiated")
      }

      override fun onFailure(reason: Int) {
        Log.e("WiFi Direct", "Connection failed: $reason")
      }
    })
  }

  fun syncState(peerAddress: String) {
    val socket = Socket(peerAddress, 8080)
    val out = PrintWriter(socket.getOutputStream(), true)
    val state = states[1].getState()
    out.println(state.toJSON())
    out.close()
    socket.close()
  }
}
```

---

## Option 5: Matrix Homeserver (Open Standard)

Use the Matrix protocol (decentralized, open-source messaging):

```javascript
import * as sdk from 'matrix-js-sdk';

const client = sdk.createClient({
  baseUrl: 'https://matrix.example.com',
  userId: '@alice:matrix.example.com',
  accessToken: 'token'
});

client.on('sync', (state) => {
  if (state === 'PREPARED') {
    // Ready to send/receive
  }
});

client.on('Room.timeline', (event, room) => {
  const content = event.getContent();
  const remoteState = JSON.parse(content.body);
  
  // Merge peer state
  states[1].merge(remoteState);
  renderChat(1);
});

// Send state as room message
setInterval(() => {
  const state = states[1].getState();
  client.sendTextMessage(roomId, JSON.stringify(state));
}, 5000);

client.startClient();
```

---

## Comparison

| Transport | Pros | Cons | Best For |
|-----------|------|------|----------|
| **WebSocket** | Simple, works anywhere | Requires server | Internet messaging |
| **Bluetooth LE** | P2P, no server, low power | Short range, iOS/Android only | Mobile devices near each other |
| **IPFS** | Decentralized, content-addressed | Slower, NAT traversal hard | Long-term data storage |
| **WiFi Direct** | P2P, no infrastructure | Android only, manual pairing | Local mesh networks |
| **Matrix** | Open standard, decentralized, mature | More overhead | Communities, bridges |

---

## Testing Multiple Devices

### Real Scenario: Test with Two Machines

1. **Machine A (Alice):**
   ```bash
   open http://localhost:3000 # Start server
   open converge-messages-app.html # Alice's client
   ```

2. **Machine B (Bob):**
   ```bash
   open http://192.168.1.100:3000 # Same server, different machine
   ```

3. **Send a message from Alice:**
   - Message appears in Alice's chat
   - Server broadcasts to Bob
   - Message appears in Bob's chat within 500ms

4. **Check localStorage on both:**
   ```javascript
   localStorage.getItem('state1') // Both should have identical JSON
   ```

---

## Debugging Network Sync

### Server Logs

```javascript
// Add logging to server.js
ws.on('message', (data) => {
  const msg = JSON.parse(data);
  console.log(`[${new Date().toISOString()}] Device ${msg.deviceId} sent state with ${msg.state.messagesByID.length} messages`);
});
```

### Client Console

```javascript
// Add to client:
const origMerge = ConversationState.prototype.merge;
ConversationState.prototype.merge = function(otherState) {
  console.log(`Before merge: ${this.messagesByID.size} messages`);
  const changed = origMerge.call(this, otherState);
  console.log(`After merge: ${this.messagesByID.size} messages (changed: ${changed})`);
  return changed;
};
```

---

## Security Considerations

### What's Already Secure
- Ed25519 signing (can't forge messages)
- Invariant validation (can't create cycles)

### What Needs Work
- **Transport encryption:** TLS/SSL for WebSocket, encrypted Bluetooth pairing
- **Key exchange:** Establish shared encryption keys for AES-256-GCM
- **Peer authentication:** Verify peers are who they claim to be
- **Replay prevention:** Add sequence numbers or timestamps
- **Access control:** Membership management (who can join the conversation)

### Quick Hardening

```javascript
// Sign all network messages with your keypair
const networkMsg = {
  type: 'state',
  state: localState.getState(),
  timestamp: Date.now()
};
networkMsg.signature = signMessage(networkMsg, secretKey);

// Verify on receive
if (!verifySignature(incomingMsg, incomingMsg.signature, incomingMsg.author)) {
  console.error('Invalid message signature!');
  return;
}
```

---

## Deployment Checklist

- [ ] Choose transport layer (WebSocket, IPFS, Bluetooth, etc)
- [ ] Set up server/relay infrastructure (if needed)
- [ ] Test with 2+ real devices
- [ ] Add TLS/SSL encryption for network traffic
- [ ] Implement peer authentication
- [ ] Add membership management
- [ ] Test offline + reconnect scenarios
- [ ] Monitor sync latency and message loss
- [ ] Load test with 100+ messages
- [ ] Document for production support

---

## Next Steps

1. **Choose a transport** from the options above
2. **Adapt the client code** to use that transport
3. **Test with real devices** on your local network
4. **Add TLS** for production
5. **Deploy** a relay server (if using WebSocket)
6. **Iterate** based on user feedback

The Duality state machine remains identical — only the networking layer changes.
