---
title: Wasm + WebRTC
slug: wasm-webrtc
difficulty: advanced
tags: [api]
order: 36
description: RustとWebAssemblyを使って、WebRTCによるピアツーピア通信を構築します — ピア接続、データチャネル、シグナリング、ICE候補。
starter_code: |
  use std::collections::HashMap;

  // 純粋なRustでWebRTCのピアツーピアメッセージングをシミュレーション。
  // 実際のWasmでは、ブラウザのRTCPeerConnection、
  // RTCDataChannelなどへのweb-sysバインディングを使用します。

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
          println!("[{}] オファーを作成", self.id);
          sdp
      }

      fn create_answer(&mut self, _offer: &str) -> String {
          let sdp = format!("v=0\r\no={} 0 0 IN IP4 0.0.0.0\r\ns=answer\r\nt=0 0\r\n", self.id);
          self.local_description = Some(sdp.clone());
          self.state = ConnectionState::Connecting;
          println!("[{}] アンサーを作成", self.id);
          sdp
      }

      fn set_remote_description(&mut self, sdp: &str) {
          self.remote_description = Some(sdp.to_string());
          println!("[{}] リモートディスクリプションを設定 ({}バイト)", self.id, sdp.len());
      }

      fn add_ice_candidate(&mut self, candidate: IceCandidate) {
          println!("[{}] ICE候補を追加: {}", self.id, candidate.candidate);
          self.ice_candidates.push(candidate);
      }

      fn connect(&mut self) {
          if self.local_description.is_some() && self.remote_description.is_some() {
              self.state = ConnectionState::Connected;
              println!("[{}] 状態 -> 接続済み", self.id);
          }
      }

      fn create_data_channel(&mut self, label: &str, ordered: bool) {
          let channel = DataChannel {
              label: label.to_string(),
              messages: Vec::new(),
              ordered,
          };
          self.channels.insert(label.to_string(), channel);
          println!("[{}] データチャネルを作成: '{}'", self.id, label);
      }

      fn send(&mut self, channel: &str, msg: &str) {
          if self.state != ConnectionState::Connected {
              println!("[{}] 送信不可 — 未接続", self.id);
              return;
          }
          if let Some(ch) = self.channels.get_mut(channel) {
              ch.messages.push(msg.to_string());
              println!("[{}] '{}'で送信: {}", self.id, channel, msg);
          }
      }

      fn receive(&mut self, channel: &str, msg: &str) {
          if let Some(ch) = self.channels.get_mut(channel) {
              ch.messages.push(msg.to_string());
              println!("[{}] '{}'で受信: {}", self.id, channel, msg);
          }
      }
  }

  fn main() {
      println!("=== WebRTCピア接続シミュレーション ===\n");

      // 2つのピアを作成
      let mut alice = PeerConnection::new("Alice");
      let mut bob = PeerConnection::new("Bob");

      // ステップ1: Aliceがオファーを作成
      let offer = alice.create_offer();

      // ステップ2: シグナリングサーバーがオファーをBobに転送
      bob.set_remote_description(&offer);

      // ステップ3: Bobがアンサーを作成
      let answer = bob.create_answer(&offer);

      // ステップ4: シグナリングサーバーがアンサーをAliceに転送
      alice.set_remote_description(&answer);

      // ステップ5: ICE候補を交換
      let candidate = IceCandidate {
          sdp_mid: "0".to_string(),
          sdp_mline_index: 0,
          candidate: "candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host".to_string(),
      };
      alice.add_ice_candidate(candidate.clone());
      bob.add_ice_candidate(candidate);

      // ステップ6: 接続確立
      alice.connect();
      bob.connect();

      // ステップ7: データチャネルを作成しメッセージを交換
      alice.create_data_channel("chat", true);
      bob.create_data_channel("chat", true);

      alice.send("chat", "Hello Bob!");
      bob.receive("chat", "Hello Bob!");

      bob.send("chat", "Hi Alice!");
      alice.receive("chat", "Hi Alice!");

      println!("\n接続状態:");
      println!("  Alice: {:?}", alice.state);
      println!("  Bob:   {:?}", bob.state);
  }
expected_output: |
  === WebRTCピア接続シミュレーション ===

  [Alice] オファーを作成
  [Bob] リモートディスクリプションを設定 (54バイト)
  [Bob] アンサーを作成
  [Alice] リモートディスクリプションを設定 (56バイト)
  [Alice] ICE候補を追加: candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host
  [Bob] ICE候補を追加: candidate:1 1 UDP 2130706431 192.168.1.10 5000 typ host
  [Alice] 状態 -> 接続済み
  [Bob] 状態 -> 接続済み
  [Alice] データチャネルを作成: 'chat'
  [Bob] データチャネルを作成: 'chat'
  [Alice] 'chat'で送信: Hello Bob!
  [Bob] 'chat'で受信: Hello Bob!
  [Bob] 'chat'で送信: Hi Alice!
  [Alice] 'chat'で受信: Hi Alice!

  接続状態:
    Alice: Connected
    Bob:   Connected
---

## WebRTCとは？

**WebRTC**（Web Real-Time Communication）は、ブラウザ間で直接ピアツーピアの音声、動画、データ転送を可能にします — サーバーリレーは不要です。Rust/Wasmは、エンコーディング、プロトコル処理、アプリケーション状態など、クライアント側のロジック全体を駆動できます。

```
  従来のクライアント-サーバー       WebRTCピアツーピア

  ┌──────┐     ┌──────┐     ┌──────┐   ┌──────┐          ┌──────┐
  │Peer A│────>│Server│<────│Peer B│   │Peer A│<========>│Peer B│
  └──────┘     └──────┘     └──────┘   └──────┘  直接    └──────┘
                  ▲                                 │
             すべてのデータが                  低レイテンシ
             サーバーを経由                    サーバーコスト不要
                                              E2E暗号化
```

## WebRTCアーキテクチャの概要

WebRTC接続にはいくつかのコンポーネントが関与します：

```
  ┌─────────────────────────────────────────────────────────┐
  │                    ブラウザ (Peer A)                      │
  │                                                         │
  │  ┌─────────────┐   ┌──────────────┐   ┌─────────────┐  │
  │  │  Rust        │──>│RTCPeerConn.  │──>│  ICEエージェント│  │
  │  │  Wasmコード  │   │  (ブラウザ)   │   │ (STUN/TURN) │  │
  │  └─────────────┘   └──────────────┘   └──────┬──────┘  │
  │         │                  │                   │         │
  │         ▼                  ▼                   │         │
  │  ┌─────────────┐   ┌──────────────┐           │         │
  │  │データチャネル│   │ メディア     │           │         │
  │  │(テキスト/バイナリ)│ │ ストリーム  │           │         │
  │  └─────────────┘   │(音声/動画)   │           │         │
  │                    └──────────────┘           │         │
  └───────────────────────────────────────────────┼─────────┘
                                                   │
                          ┌────────────────────────┘
                          │  STUN / TURN / 直接接続
                          ▼
  ┌─────────────────────────────────────────────────────────┐
  │                    ブラウザ (Peer B)                      │
  │              （上記と同じ構成）                           │
  └─────────────────────────────────────────────────────────┘
```

## シグナリングプロセス

ピア同士が直接接続する前に、シグナリングサーバー（通常はWebSocketサーバー）を通じて**オファー**、**アンサー**、**ICE候補**を交換する必要があります：

```
  Alice                   シグナリングサーバー             Bob
    │                            │                        │
    │── 1. オファー作成 ────────> │                        │
    │                            │── 2. オファー転送 ────> │
    │                            │                        │
    │                            │ <── 3. アンサー作成 ───│
    │ <── 4. アンサー転送 ────── │                        │
    │                            │                        │
    │── 5. ICE候補 ────────────> │                        │
    │                            │── 6. ICE転送 ────────> │
    │                            │                        │
    │                            │ <── 7. ICE候補 ────────│
    │ <── 8. ICE転送 ─────────── │                        │
    │                            │                        │
    │ <══════════ 直接P2P接続 ══════════════════════════> │
    │            （シグナリングサーバーは不要に）           │
```

### SDP（Session Description Protocol）

オファーとアンサーは、機能を記述する**SDP**文字列です：

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

## Rust/WasmでのWebRTC（web-sys使用）

`web-sys`クレートは、すべてのWebRTCブラウザAPIへのバインディングを提供します：

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

### ピア接続の作成

```rust
use web_sys::{RtcPeerConnection, RtcConfiguration, RtcIceServer};
use wasm_bindgen::prelude::*;
use js_sys::{Array, Object, Reflect};

fn create_peer_connection() -> Result<RtcPeerConnection, JsValue> {
    // STUN/TURNサーバーを設定
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

### オファーの作成

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

### データチャネルのセットアップ

```rust
fn setup_data_channel(pc: &RtcPeerConnection) -> RtcDataChannel {
    let mut init = RtcDataChannelInit::new();
    init.ordered(true);

    let channel = pc.create_data_channel_with_data_channel_dict("chat", &init);

    // 受信メッセージの処理
    let onmessage = Closure::wrap(Box::new(move |evt: MessageEvent| {
        if let Some(text) = evt.data().as_string() {
            web_sys::console::log_1(&format!("受信: {}", text).into());
        }
    }) as Box<dyn FnMut(_)>);

    channel.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
    onmessage.forget();

    // チャネルオープンの処理
    let channel_clone = channel.clone();
    let onopen = Closure::wrap(Box::new(move |_: JsValue| {
        channel_clone.send_with_str("Hello from Rust/Wasm!").unwrap();
    }) as Box<dyn FnMut(_)>);

    channel.set_onopen(Some(onopen.as_ref().unchecked_ref()));
    onopen.forget();

    channel
}
```

## ICE: ピア間の経路の発見

**ICE**（Interactive Connectivity Establishment）は、ピア間の最適なネットワーク経路を発見します：

```
  ICE候補の種類（優先順）:

  1. ホスト候補            ← 直接LAN接続
     ┌──────┐               ┌──────┐
     │Peer A│═══════════════│Peer B│
     └──────┘  同一ネットワーク └──────┘

  2. サーバーリフレクシブ (srflx) ← STUN経由（パブリックIPを学習）
     ┌──────┐     ┌──────┐     ┌──────┐
     │Peer A│────>│ STUN │<────│Peer B│
     └──────┘     │サーバー│     └──────┘
       NAT ─ ─ ─ └──────┘ ─ ─ ─ NAT
     Peer A ═════════════════════ Peer B
              （NATを通して）

  3. リレー (relay)          ← TURN経由（中継トラフィック）
     ┌──────┐     ┌──────┐     ┌──────┐
     │Peer A│────>│ TURN │<────│Peer B│
     └──────┘     │サーバー│     └──────┘
                  └──────┘
               （全データを中継）
```

### RustでのICE候補の処理

```rust
fn setup_ice_handling(pc: &RtcPeerConnection, ws: &WebSocket) {
    let ws_clone = ws.clone();

    let onicecandidate = Closure::wrap(Box::new(move |evt: RtcPeerConnectionIceEvent| {
        if let Some(candidate) = evt.candidate() {
            // シリアライズしてシグナリングサーバー経由で送信
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

## データチャネルのメッセージ型

データチャネルはテキストとバイナリの両方のデータをサポートします：

```rust
// テキストメッセージ
channel.send_with_str("Hello!")?;

// バイナリメッセージ（例：ゲーム状態、ファイルチャンク）
let data: Vec<u8> = vec![0x01, 0x02, 0x03, 0x04];
let array = js_sys::Uint8Array::from(&data[..]);
channel.send_with_array_buffer(&array.buffer())?;
```

### 順序付きチャネルと非順序チャネル

| 特性                   | 順序付き (`ordered: true`) | 非順序 (`ordered: false`)    |
|------------------------|---------------------------|------------------------------|
| メッセージ順序          | 保証あり                  | 順序が入れ替わる可能性あり   |
| レイテンシ             | 高い（先頭ブロッキング）  | 低い                         |
| ユースケース           | チャット、コマンド        | ゲーム状態、動画フレーム     |
| 再送                   | あり                      | オプション (`maxRetransmits`) |

```rust
// 非信頼・非順序 — リアルタイムゲーム状態に最適
let mut init = RtcDataChannelInit::new();
init.ordered(false);
init.max_retransmits(0);  // 送りっぱなし

let game_channel = pc.create_data_channel_with_data_channel_dict("game-state", &init);
```

## WebSocketによるシグナリング

最小限のシグナリングサーバーがピア間でSDPとICEメッセージを中継します：

```rust
// クライアント側: シグナリングサーバーに接続
use web_sys::WebSocket;

let ws = WebSocket::new("wss://signal.example.com/room/abc")?;

let onmessage = Closure::wrap(Box::new(move |evt: MessageEvent| {
    let text = evt.data().as_string().unwrap();
    // パースして処理: オファー、アンサー、またはICE候補
    // その後 pc.set_remote_description() または pc.add_ice_candidate() を呼ぶ
}) as Box<dyn FnMut(_)>);

ws.set_onmessage(Some(onmessage.as_ref().unchecked_ref()));
onmessage.forget();
```

### シグナリングプロトコルの例

```json
// Alice -> サーバー -> Bob
{"type": "offer",  "sdp": "v=0\r\no=..."}

// Bob -> サーバー -> Alice
{"type": "answer", "sdp": "v=0\r\no=..."}

// 双方向
{"type": "ice", "candidate": "candidate:1 1 UDP ...",
 "sdpMid": "0", "sdpMLineIndex": 0}
```

## 接続のライフサイクル

```
  状態マシン:

  ┌─────┐  オファー/アンサー作成  ┌────────────┐  ICE完了  ┌───────────┐
  │ New │───────────────────────>│ Connecting │───────────>│ Connected │
  └─────┘                       └────────────┘            └─────┬─────┘
                                      │                         │
                                      │ 失敗                    │ ネットワーク
                                      ▼                         │ 変更
                                ┌──────────┐                    ▼
                                │  Failed  │              ┌──────────────┐
                                └──────────┘              │Disconnected  │
                                                          └──────┬───────┘
                                                                 │ ICE
                                                                 │ リスタート
                                                                 ▼
                                                          ┌───────────┐
                                                          │ Connected │
                                                          └───────────┘
```

## Rust/Wasm WebRTCのパフォーマンスヒント

1. **バイナリデータをRustで処理する** — 動画フレームのデコード、ゲーム状態の圧縮、ペイロードの暗号化をWasm側でデータチャネルを通じて送信する前に行います。

2. **`SharedArrayBuffer`を使用して**WasmとJavaScript間でゼロコピーデータ転送を行います（クロスオリジン分離ヘッダーが必要）。

3. **小さなメッセージをバッチ処理する** — 各`send()`呼び出しにはオーバーヘッドがあります。複数のゲームイベントを単一のバイナリメッセージにまとめましょう。

4. **リアルタイムデータには非順序チャネルを使用する** — すべての更新よりも最新の状態が重要な場合。

| パターン                        | レイテンシ | 信頼性 | ユースケース         |
|--------------------------------|---------|-------------|----------------------|
| 順序付き + 信頼性あり           | 高い    | 完全        | チャット、ファイル転送 |
| 順序付き + maxRetransmits(3)    | 中程度  | 部分的      | 重要なゲームイベント   |
| 非順序 + maxRetransmits(0)      | 最低    | なし        | 位置情報の更新        |

## まとめ

WebRTCにより、Rust/Wasmアプリケーションはリレーサーバーなしでピアツーピア通信が可能になります。シグナリングフェーズ（オファー、アンサー、ICE候補）はWebSocketサーバーを通じて行われますが、接続が確立されるとすべてのデータはブラウザ間で直接流れます。データチャネルは柔軟で低レイテンシのメッセージングを提供します — 正確性が重要なデータには順序付き/信頼性ありを、リアルタイムストリームには非順序/非信頼を選択しましょう。
