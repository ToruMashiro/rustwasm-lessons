---
title: Wasm + WebRTC
slug: wasm-webrtc
difficulty: advanced
tags: [api]
order: 36
description: Use Rust and WebAssembly to build peer-to-peer communication with WebRTC — peer connections, data channels, signaling, and ICE candidates.
starter_code: |
  use std::collections::HashMap;

  // Simulating WebRTC peer-to-peer messaging in plain Rust.
  // In real Wasm, you'd use web-sys bindings to the browser's
  // RTCPeerConnection, RTCDataChannel, etc.

  #[derive(Debug, Clone)]
  struct IceCandidate {
      sdp_mid: String,
      sdp_mline_index: u32,
      candidate: String,
  }

  #[derive(Debug, Clone)]
  enum SignalMessage {
      Offer(String),
      Answer(String),
      IceCandidate(IceCandidate),
  }

  #[derive(Debug, Clone, PartialEq)]
  enum ConnectionState {
      New,
      Connecting,
      Connected,
      Disconnected,
  }

  struct DataChannel {
      label: String,
      messages: Vec<String>,
      ordered: bool,
  }

  struct PeerConnection {
      id: String,
      state: ConnectionState,
      local_description: Option<String>,
      remote_description: Option<String>,
      ice_candidates: Vec<IceCandidate>,
      channels: HashMap<String, DataChannel>,
  }

  impl PeerConnection {
      fn new(id: &str) -> Self {
          PeerConnection {
              id: id.to_string(),
              state: ConnectionState::New,
              local_description: None,
              remote_description: None,
              ice_candidates: Vec::new(),
              channels: HashMap::new(),
          }
      }

      fn create_offer(&mut self) -> String {
          let sdp = format!("v=0\r\no={} 0 0 IN IP4 0.0.0.0\r\ns=-\r\nt=0 0\r\n", self.id);
          self.local_description = Some(sdp.clone());
          self.state = ConnectionState::Connecting;
          println!("[{}] Created offer", self.id);
          sdp
      }

      fn create_answer(&mut self, _offer: &str) -> String {
          let sdp = format!("v=0\r\no={} 0 0 IN IP4 0.0.0.0\r\ns=answer\r\nt=0 0\r\n", self.id);
          self.local_description = Some(sdp.clone());
          self.state = ConnectionState::Connecting;
          println!("[{}] Created answer", self.id);
          sdp
      }

      fn set_remote_description(&mut self, sdp: &str) {
          self.remote_description = Some(sdp.to_string());
          println!("[{}] Set remote description ({} bytes)", self.id, sdp.len());
      }

      fn add_ice_candidate(&mut self, candidate: IceCandidate) {
          println!("[{}] Added ICE candidate: {}", self.id, candidate.candidate);
          self.ice_candidates.push(candidate);
      }

      fn connect(&mut self) {
          if self.local_description.is_some() && self.remote_description.is_some() {
              self.state = ConnectionState::Connected;
              println!("[{}] State -> Connected", self.id);
          }
      }

      fn create_data_channel(&mut self, label: &str, ordered: bool) {
          let channel = DataChannel {
              label: label.to_string(),
              messages: Vec::new(),
              ordered,
          };
          self.channels.insert(label.to_string(), channel);
          println!("[{}] Created data channel: '{}'", self.id, label);
      }

      fn send(&mut self, channel: &str, msg: &str) {
          if self.state != ConnectionState::Connected {
              println!("[{}] Cannot send — not connected", self.id);
              return;
          }
          if let Some(ch) = self.channels.get_mut(channel) {
              ch.messages.push(msg.to_string());
              println!("[{}] Sent on '{}': {}", self.id, channel, msg);
          }
      }

      fn receive(&mut self, channel: &str, msg: &str) {
          if let Some(ch) = self.channels.get_mut(channel) {
              ch.messages.push(msg.to_string());
              println!("[{}] Received on '{}': {}", self.id, channel, msg);
          }
      }
  }

  fn main() {
      println!("=== WebRTC Peer Connection Simulation ===\n");

      // Create two peers
      let mut alice = PeerConnection::new("Alice");
      let mut bob = PeerConnection::new("Bob");

      // Step 1: Alice creates an offer
      let offer = alice.create_offer();

      // Step 2: Signaling server delivers offer to Bob
      bob.set_remote_description(&offer);

      // Step 3: Bob creates an answer
      let answer = bob.create_answer(&offer);

      // Step 4: Signaling server delivers answer to Alice
      alice.set_remote_description(&answer);

      // Step 5: Exchange ICE candidates
      let candidate = IceCandidate {
          sdp_mid: "0".to_string(),
          sdp_mline_index: 0,
          candidate: "candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host".to_string(),
      };
      alice.add_ice_candidate(candidate.clone());
      bob.add_ice_candidate(candidate);

      // Step 6: Connection established
      alice.connect();
      bob.connect();

      // Step 7: Create data channels and exchange messages
      alice.create_data_channel("chat", true);
      bob.create_data_channel("chat", true);

      alice.send("chat", "Hello Bob!");
      bob.receive("chat", "Hello Bob!");

      bob.send("chat", "Hi Alice!");
      alice.receive("chat", "Hi Alice!");

      println!("\nConnection states:");
      println!("  Alice: {:?}", alice.state);
      println!("  Bob:   {:?}", bob.state);
  }
expected_output: |
  === WebRTC Peer Connection Simulation ===

  [Alice] Created offer
  [Bob] Set remote description (54 bytes)
  [Bob] Created answer
  [Alice] Set remote description (56 bytes)
  [Alice] Added ICE candidate: candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host
  [Bob] Added ICE candidate: candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host
  [Alice] State -> Connected
  [Bob] State -> Connected
  [Alice] Created data channel: 'chat'
  [Bob] Created data channel: 'chat'
  [Alice] Sent on 'chat': Hello Bob!
  [Bob] Received on 'chat': Hello Bob!
  [Bob] Sent on 'chat': Hi Alice!
  [Alice] Received on 'chat': Hi Alice!

  Connection states:
    Alice: Connected
    Bob:   Connected
---

## What is WebRTC?

**WebRTC** (Web Real-Time Communication) enables peer-to-peer audio, video, and data transfer directly between browsers — with no server relay. Rust/Wasm can drive the entire client-side logic: encoding, protocol handling, and application state.

```
  Traditional Client-Server             WebRTC Peer-to-Peer

  ┌──────┐     ┌──────┐     ┌──────┐   ┌──────┐          ┌──────┐
  │Peer A│────>│Server│<────│Peer B│   │Peer A│<========>│Peer B│
  └──────┘     └──────┘     └──────┘   └──────┘  direct  └──────┘
                  ▲                                 │
             All data                         Lower latency
             goes through                     No server cost
             server                           E2E encrypted
```

## WebRTC Architecture Overview

A WebRTC connection involves several components:

```
  ┌─────────────────────────────────────────────────────────┐
  │                    Browser (Peer A)                      │
  │                                                         │
  │  ┌─────────────┐   ┌──────────────┐   ┌─────────────┐  │
  │  │  Your Rust   │──>│RTCPeerConn.  │──>│  ICE Agent  │  │
  │  │  Wasm Code   │   │  (browser)   │   │ (STUN/TURN) │  │
  │  └─────────────┘   └──────────────┘   └──────┬──────┘  │
  │         │                  │                   │         │
  │         ▼                  ▼                   │         │
  │  ┌─────────────┐   ┌──────────────┐           │         │
  │  │Data Channel │   │ Media Stream │           │         │
  │  │ (text/bin)  │   │ (audio/video)│           │         │
  │  └─────────────┘   └──────────────┘           │         │
  └───────────────────────────────────────────────┼─────────┘
                                                   │
                          ┌────────────────────────┘
                          │  STUN / TURN / Direct
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    Browser (Peer B)                      │
  │              (mirror of above)                          │
  └─────────────────────────────────────────────────────────┘
```

## The Signaling Process

Before peers can connect directly, they must exchange **offers**, **answers**, and **ICE candidates** through a signaling server (typically a WebSocket server):

```
  Alice                    Signaling Server              Bob
    │                            │                        │
    │── 1. Create Offer ───────> │                        │
    │                            │── 2. Forward Offer ──> │
    │                            │                        │
    │                            │ <── 3. Create Answer ──│
    │ <── 4. Forward Answer ──── │                        │
    │                            │                        │
    │── 5. ICE Candidate ──────> │                        │
    │                            │── 6. Forward ICE ────> │
    │                            │                        │
    │                            │ <── 7. ICE Candidate ──│
    │ <── 8. Forward ICE ─────── │                        │
    │                            │                        │
    │ <══════════ Direct P2P Connection ═════════════════> │
    │            (signaling server no longer needed)       │
```

### SDP (Session Description Protocol)

The offer and answer are **SDP** strings that describe capabilities:

```
v=0
o=- 4567890 2 IN IP4 127.0.0.1
s=-
t=0 0
a=group:BUNDLE 0
m=application 9 UDP/DTLS/SCTP webrtc-datachannel
c=IN IP4 0.0.0.0
a=ice-ufrag:abcd
a=ice-pwd:efghijklmnop
a=fingerprint:sha-256 AA:BB:CC:...
a=setup:actpass
a=mid:0
a=sctp-port:5000
```

## WebRTC in Rust/Wasm with web-sys

The `web-sys` crate provides bindings to all WebRTC browser APIs:

```toml
[dependencies.web-sys]
version = "0.3"
features = [
    "RtcPeerConnection",
    "RtcPeerConnectionIceEvent",
    "RtcSessionDescriptionInit",
    "RtcSdpType",
    "RtcDataChannel",
    "RtcDataChannelInit",
    "RtcDataChannelEvent",
    "RtcIceCandidateInit",
    "RtcConfiguration",
    "RtcIceServer",
    "MessageEvent",
]
```

### Creating a Peer Connection

```rust
use web_sys::{RtcPeerConnection, RtcConfiguration, RtcIceServer};
use wasm_bindgen::prelude::*;
use js_sys::{Array, Object, Reflect};

fn create_peer_connection() -> Result<RtcPeerConnection, JsValue> {
    // Configure STUN/TURN servers
    let ice_server = Object::new();
    Reflect::set(&ice_server, &"urls".into(),
        &"stun:stun.l.google.com:19302".into())?;

    let ice_servers = Array::new();
    ice_servers.push(&ice_server);

    let config = RtcConfiguration::new();
    config.set_ice_servers(&ice_servers);

    RtcPeerConnection::new_with_configuration(&config)
}
```

### Creating an Offer

```rust
use wasm_bindgen_futures::JsFuture;

async fn create_offer(pc: &RtcPeerConnection) -> Result<String, JsValue> {
    let offer = JsFuture::from(pc.create_offer()).await?;
    let offer_sdp = Reflect::get(&offer, &"sdp".into())?
        .as_string()
        .unwrap();

    let mut desc = RtcSessionDescriptionInit::new(RtcSdpType::Offer);
    desc.sdp(&offer_sdp);
    JsFuture::from(pc.set_local_description(&desc)).await?;

    Ok(offer_sdp)
}
```

### Setting Up a Data Channel

```rust
fn setup_data_channel(pc: &RtcPeerConnection) -> RtcDataChannel {
    let mut init = RtcDataChannelInit::new();
    init.ordered(true);

    let channel = pc.create_data_channel_with_data_channel_dict("chat", &init);

    // Handle incoming messages
    let onmessage = Closure::wrap(Box::new(move |evt: MessageEvent| {
        if let Some(text) = evt.data().as_string() {
            web_sys::console::log_1(&format!("Received: {}", text).into());
        }
    }) as Box<dyn FnMut(_)>);

    channel.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
    onmessage.forget();

    // Handle channel open
    let channel_clone = channel.clone();
    let onopen = Closure::wrap(Box::new(move |_: JsValue| {
        channel_clone.send_with_str("Hello from Rust/Wasm!").unwrap();
    }) as Box<dyn FnMut(_)>);

    channel.set_onopen(Some(onopen.as_ref().unchecked_ref()));
    onopen.forget();

    channel
}
```

## ICE: Finding a Path Between Peers

**ICE** (Interactive Connectivity Establishment) discovers the best network path between peers:

```
  ICE Candidate Types (in preference order):

  1. Host Candidate         ← Direct LAN connection
     ┌──────┐               ┌──────┐
     │Peer A│═══════════════│Peer B│
     └──────┘  same network └──────┘

  2. Server-Reflexive (srflx) ← Via STUN (learns public IP)
     ┌──────┐     ┌──────┐     ┌──────┐
     │Peer A│────>│ STUN │<────│Peer B│
     └──────┘     │Server│     └──────┘
       NAT ─ ─ ─ └──────┘ ─ ─ ─ NAT
     Peer A ═════════════════════ Peer B
              (through NAT)

  3. Relay (relay)          ← Via TURN (relayed traffic)
     ┌──────┐     ┌──────┐     ┌──────┐
     │Peer A│────>│ TURN │<────│Peer B│
     └──────┘     │Server│     └──────┘
                  └──────┘
               (all data relayed)
```

### Handling ICE Candidates in Rust

```rust
fn setup_ice_handling(pc: &RtcPeerConnection, ws: &WebSocket) {
    let ws_clone = ws.clone();

    let onicecandidate = Closure::wrap(Box::new(move |evt: RtcPeerConnectionIceEvent| {
        if let Some(candidate) = evt.candidate() {
            // Serialize and send via signaling server
            let msg = format!(
                r#"{{"type":"ice","candidate":"{}","sdpMid":"{}","sdpMLineIndex":{}}}"#,
                candidate.candidate(),
                candidate.sdp_mid().unwrap_or_default(),
                candidate.sdp_m_line_index().unwrap_or(0)
            );
            ws_clone.send_with_str(&msg).unwrap();
        }
    }) as Box<dyn FnMut(_)>);

    pc.set_onicecandidate(Some(onicecandidate.as_ref().unchecked_ref()));
    onicecandidate.forget();
}
```

## Data Channel Message Types

Data channels support both text and binary data:

```rust
// Text message
channel.send_with_str("Hello!")?;

// Binary message (e.g., game state, file chunks)
let data: Vec<u8> = vec![0x01, 0x02, 0x03, 0x04];
let array = js_sys::Uint8Array::from(&data[..]);
channel.send_with_array_buffer(&array.buffer())?;
```

### Ordered vs Unordered Channels

| Property               | Ordered (`ordered: true`) | Unordered (`ordered: false`) |
|------------------------|---------------------------|------------------------------|
| Message order          | Guaranteed                | May arrive out of order      |
| Latency                | Higher (head-of-line)     | Lower                        |
| Use case               | Chat, commands            | Game state, video frames     |
| Retransmission         | Yes                       | Optional (`maxRetransmits`)  |

```rust
// Unreliable, unordered — ideal for real-time game state
let mut init = RtcDataChannelInit::new();
init.ordered(false);
init.max_retransmits(0);  // fire-and-forget

let game_channel = pc.create_data_channel_with_data_channel_dict("game-state", &init);
```

## Signaling with WebSocket

A minimal signaling server relays SDP and ICE messages between peers:

```rust
// Client-side: connect to signaling server
use web_sys::WebSocket;

let ws = WebSocket::new("wss://signal.example.com/room/abc")?;

let onmessage = Closure::wrap(Box::new(move |evt: MessageEvent| {
    let text = evt.data().as_string().unwrap();
    // Parse and handle: offer, answer, or ICE candidate
    // Then call pc.set_remote_description() or pc.add_ice_candidate()
}) as Box<dyn FnMut(_)>);

ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
onmessage.forget();
```

### Signaling Protocol Example

```json
// Alice -> Server -> Bob
{"type": "offer",  "sdp": "v=0\r\no=..."}

// Bob -> Server -> Alice
{"type": "answer", "sdp": "v=0\r\no=..."}

// Both directions
{"type": "ice", "candidate": "candidate:1 1 UDP ...",
 "sdpMid": "0", "sdpMLineIndex": 0}
```

## Connection Lifecycle

```
  State Machine:

  ┌─────┐  create offer/answer  ┌────────────┐  ICE done  ┌───────────┐
  │ New │───────────────────────>│ Connecting │───────────>│ Connected │
  └─────┘                       └────────────┘            └─────┬─────┘
                                      │                         │
                                      │ failure                 │ network
                                      ▼                         │ change
                                ┌──────────┐                    ▼
                                │  Failed  │              ┌──────────────┐
                                └──────────┘              │Disconnected  │
                                                          └──────┬───────┘
                                                                 │ ICE
                                                                 │ restart
                                                                 ▼
                                                          ┌───────────┐
                                                          │ Connected │
                                                          └───────────┘
```

## Performance Tips for Rust/Wasm WebRTC

1. **Process binary data in Rust** — decode video frames, compress game state, encrypt payloads on the Wasm side before sending through data channels.

2. **Use `SharedArrayBuffer`** for zero-copy data transfer between Wasm and JavaScript (requires cross-origin isolation headers).

3. **Batch small messages** — each `send()` call has overhead. Pack multiple game events into a single binary message.

4. **Use unordered channels** for real-time data where the latest state matters more than every update.

| Pattern                        | Latency | Reliability | Use Case             |
|--------------------------------|---------|-------------|----------------------|
| Ordered + reliable             | High    | Full        | Chat, file transfer  |
| Ordered + maxRetransmits(3)    | Medium  | Partial     | Important game events|
| Unordered + maxRetransmits(0)  | Lowest  | None        | Position updates     |

## Summary

WebRTC lets your Rust/Wasm application communicate peer-to-peer without a relay server. The signaling phase (offers, answers, ICE candidates) happens through a WebSocket server, but once connected, all data flows directly between browsers. Data channels give you flexible, low-latency messaging — choose ordered/reliable for correctness-critical data and unordered/unreliable for real-time streams.
