# TikTok Live Webcast Engine & WebSocket Gateway

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)
[![Python 3.10+](https://img.shields.io/badge/python-3.10+-blue.svg)](https://www.python.org/downloads/)
[![WebSocket](https://img.shields.io/badge/WebSocket-Gateway-green.svg)](https://websockets.readthedocs.io/)
[![Protobuf](https://img.shields.io/badge/Protobuf-betterproto-red.svg)](https://github.com/danielgtaylor/python-betterproto)
[![Playwright](https://img.shields.io/badge/Playwright-Headless_Chromium-purple.svg)](https://playwright.dev/python/)

An enterprise-grade, high-concurrency **TikTok Live Webcast reverse-engineering engine and WebSocket Gateway** written in Python. It captures real-time live events directly via TikTok's binary GoIM WebCast WebSocket protocol, decodes Protocol Buffers (Protobuf) streams on the fly, bypasses Edge CDN anti-bot fingerprinting, and broadcasts standardized JSON events to any downstream tech stack (**Node.js, C# / .NET, Go, PHP, Unity, Web Frontends, and OBS Browser Sources**).

> **⚠️ Disclaimer:**  
> This project is developed strictly for **educational, research, and interoperability purposes**. It is not affiliated with, authorized by, sponsored by, or in any way officially connected with TikTok, ByteDance Ltd., or any of their subsidiaries or affiliates. All live stream data processed by this software constitutes publicly accessible broadcasts transmitted over open web protocols.

---

## 📌 Attribution & Heritage

This project was originally forked and re-engineered from [**PirateTok/live-py**](https://github.com/PirateTok/live-py).

While the original `live-py` provided a solid foundation for basic in-process Python callbacks, real-world high-traffic deployments encountered severe limitations, blocking issues, and missing enterprise features. This repository represents a ground-up architectural overhaul designed to overcome those hurdles.

---

## ⚡ Why Was This Re-engineering Necessary? (Root Cause & Anti-Bot Challenges)

### 1. The TikTok Edge Anti-Bot Wall (`DEVICE_BLOCKED` & HTTP 415 / 429 / 403)
In recent TikTok Live Webcast updates, TikTok's edge CDN aggressively flags non-browser traffic:
* **`DEVICE_BLOCKED` (HTTP 415):** Occurs when the `ttwid` tracking cookie is absent, expired, or generated via basic curl requests lacking complete browser execution context.
* **TLS / JA3 / JA4 Fingerprinting:** Edge servers detect standard Python TLS handshakes and drop WebSocket upgrade requests with HTTP 403 Forbidden.
* **Rate Limits (HTTP 429):** Connecting multiple workers from the same IP quickly triggers CDN edge rate limiters if connections are not multiplexed.

**💡 Our Solution:**
* **Headless Playwright TTWID Engine:** Integrates automated headless Chromium to simulate authentic browser context, extracting valid `ttwid` tokens with persistent 72-hour caching.
* **Self-Healing Dynamic Token Rotation:** If a worker encounters `DEVICE_BLOCKED`, the engine automatically spins up Playwright in the background, mints a fresh token, and reconnects without dropping downstream client sockets or requiring server restarts.

### 2. Cross-Language & Ecosystem Barrier
The original library only functioned as an in-process Python script. External microservices (such as Node.js backend services, C# game engines, PHP e-commerce systems, or OBS stream widgets) could not access the stream data.

**💡 Our Solution:**
* Built a dedicated **WebSocket Gateway Server (`ws_server.py`)** that ingests raw Protobuf packets and broadcasts clean, standardized JSON events to any external system over a single WebSocket connection.

### 3. Missing TikTok Shop (E-Commerce OEC) Integration
Streamers frequently pin e-commerce products during live shopping. The original library lacked comprehensive support for `WebcastOECLiveShoppingMessage`, dropping product titles, HD thumbnails, and store links during periodic card refresh keepalives.

**💡 Our Solution:**
* Added complete **TikTok Shop PDP Parsing** (`SetPinProduct` and `CardRefresh` actions) with persistent stateful product caching, delivering clean canonical URLs (free of captcha/redirect loops), seller details, and sold counters.

### 4. Flawed Gift Combo Streaks & Rollback on Likes
* **Gift Streaks:** TikTok streams cumulative `repeat_count` values instead of delta bursts, leading to incorrect gift tallies when packets drop.
* **Like Counts:** Out-of-order sharded packets caused like counters to jump backwards.

**💡 Our Solution:**
* Implemented `GiftStreakTracker` (precise delta diamond and gift count calculations) and `LikeAccumulator` (strictly monotonic, non-decreasing total likes).

---

## 🚀 Key Upgrades & Architecture Highlights

| Feature | Original `live-py` | Our Enhanced Engine |
| :--- | :---: | :---: |
| **Architectural Model** | In-process Python script only | **Multi-Client WebSocket Gateway Server** |
| **External Interoperability** | Python only | **Node.js, C#, PHP, Go, Web, OBS, Unity** |
| **Anti-Bot Defense** | Static curl extraction (frequent 415) | **Playwright Headless + Auto-Rotating TTWID** |
| **TikTok Shop (OEC Live)** | Unsupported / Partial | **Full PDP (Title, HD Image, Clean URL, Sold Count)** |
| **Connection Multiplexing** | 1 connection per listener | **`RoomHubManager` (1 TikTok connection : N clients)** |
| **High Concurrency** | Bottlenecked at ~10 workers | **Tested & verified on 30–65 streams/IP (< 1.3MB/room)** |
| **Event Broadcast Speed** | Serial async dispatch | **Parallel non-blocking (`asyncio.gather`) in < 1ms** |
| **Gift Math** | Raw cumulative counts | **`GiftStreakTracker` delta diamond & count math** |
| **Like Stability** | Unstable (rollback on shard delays) | **`LikeAccumulator` strictly monotonic** |
| **Event Coverage** | Basic Chat / Gift / Like | **19+ Events: VIP, Pin, Banners, Goals, Captions, PK...** |

---

## 🏗️ Architecture & Data Flow

```mermaid
sequenceDiagram
    autonumber
    actor Client as External Client (Node.js/C#/Web)
    participant GW as ws_server.py (Gateway)
    participant Hub as RoomHubManager
    participant Engine as TikTokLiveClient
    participant TT as TikTok Edge WebCast Server

    Client->>GW: Connect to ws://localhost:8765/live?username=swatchesbybaobao
    GW->>Hub: get_or_create_room("swatchesbybaobao")
    alt RoomHub does not exist
        Hub->>Engine: Spawn dedicated worker & attach Playwright TTWID
        Engine->>TT: Handshake & EnterRoom (WSS Binary GoIM)
        TT-->>Engine: Stream Gzip Protobuf Frames
    else RoomHub already active
        Note over Hub: Re-use existing upstream TikTok socket!
    end
    Engine->>Hub: Decode Protobuf -> Enrich (Shop/Streak/Likes)
    Hub->>GW: Broadcast JSON Payload
    GW-->>Client: Real-time Event (Chat, Gift, Shop PDP, Like...)
```

---

## 📦 Event Coverage Matrix (19+ Event Categories)

Our engine parses and translates 100% of the binary Protobuf frames into standardized JSON payloads:

1. **`chat`**: Comment content, User ID, Nickname, Avatar HD, Badges (`is_host`, `is_mod`, `is_sub`, `is_fan`, Fan Club level).
2. **`gift`**: Gift ID, Gift Name, Diamonds, HD Image, Combo Streak calculations (Delta diamonds, Total diamonds, Active/Final state).
3. **`like`**: Real-time incremental likes and strictly monotonic total room likes.
4. **`oec_live_shopping`**: Pinned product title, Product ID, Thumbnail #1 HD URL, Clean SEO purchase link, Seller store name, Sold count.
5. **`room_user_seq`**: Live active viewer count, Total unique visitors, Top 10 leaderboard rankings.
6. **`join` / `follow` / `share`**: Viewer joined room (with viewer count), user followed host, user shared stream.
7. **`privilege_advance`**: VIP upgrade announcement, diamond cost, and privilege badges.
8. **`room_pin`**: Streamer pinned message or promotional deal announcement.
9. **`in_room_banner`**: Discount vouchers, promotion banners, and campaign activity.
10. **`goal_update`**: Live stream goal progress, target metrics, and contributor counts.
11. **`caption`**: Real-time AI Speech-to-Text captions of streamer's voice.
12. **`envelope`**: Red packet / Treasure box lucky drops with diamond counts and winner capacity.
13. **`question_new`**: Audience Q&A inquiries.
14. **`link_mic_battle`**: Streamer PK battles and duel state changes.
15. **`sub_notify`**: Subscriber renewal and new membership alerts.
16. **`emote_chat`**: Custom sticker and emote messages.
17. **`connected` / `disconnected` / `reconnecting`**: Granular socket lifecycle states.
18. **`live_ended`**: Stream termination notification.
19. **`unknown`**: Fallback passthrough for experimental Protobuf messages (**Guarantees Zero Event Loss**).

---

## 🛠️ Quick Start Guide

### 1. Requirements
* **Python 3.10+** (Tested on Python 3.10, 3.11, 3.12, 3.13 on Windows / Linux / macOS).

### 2. Installation
```bash
# Clone the repository
git clone https://github.com/dangtuandat123/tiktoklive_api.git
cd tiktoklive_api

# Install required Python dependencies
pip install -r requirements.txt

# Install Playwright Chromium (Required for automated anti-bot token minting)
python -m playwright install chromium
```

> **💡 Windows 1-Click Setup:** If you are on Windows, simply double-click **`install.bat`** inside the `tiktok_live_service` folder!

### 3. Launching the WebSocket Gateway Server
```bash
# Start server on default port 8765
python ws_server.py

# Or customize host, port, or proxy
python ws_server.py --host 0.0.0.0 --port 9000 --proxy "http://user:pass@proxy-ip:port"
```

---

## 🔌 Connecting from Any Programming Language

### 1. Instant Connection via Query Parameter (Recommended)
Simply point any WebSocket client to:
```text
ws://localhost:8765/live?username=<streamer_username>
```
*Example:* `ws://localhost:8765/live?username=swatchesbybaobao`

---

### 2. Code Snippets by Language

#### 🟢 Node.js / TypeScript
```javascript
const WebSocket = require('ws');
const ws = new WebSocket('ws://localhost:8765/live?username=swatchesbybaobao');

ws.on('message', (rawData) => {
  const msg = JSON.parse(rawData);
  if (msg.event === 'chat') {
    console.log(`💬 [CHAT] ${msg.data.user.nickname}: ${msg.data.comment}`);
  } else if (msg.event === 'oec_live_shopping') {
    console.log(`🛍️ [SHOP] Pinned: ${msg.data.product_title}`);
    console.log(`   🔗 Direct Link: ${msg.data.product_url}`);
  } else if (msg.event === 'gift') {
    console.log(`🎁 [GIFT] ${msg.data.user.nickname} sent ${msg.data.gift.name} (x${msg.data.combo.total_gift_count})`);
  }
});
```

#### 🔵 C# / .NET
```csharp
using System;
using System.Net.WebSockets;
using System.Text;
using System.Text.Json;
using System.Threading;
using System.Threading.Tasks;

class Program {
    static async Task Main() {
        using ClientWebSocket ws = new ClientWebSocket();
        await ws.ConnectAsync(new Uri("ws://localhost:8765/live?username=swatchesbybaobao"), CancellationToken.None);

        byte[] buffer = new byte[8192];
        while (ws.State == WebSocketState.Open) {
            var result = await ws.ReceiveAsync(new ArraySegment<byte>(buffer), CancellationToken.None);
            string json = Encoding.UTF8.GetString(buffer, 0, result.Count);
            using JsonDocument doc = JsonDocument.Parse(json);
            string evt = doc.RootElement.GetProperty("event").GetString();
            Console.WriteLine($"Received Event: {evt}");
        }
    }
}
```

#### 🐘 PHP
```php
<?php
require 'vendor/autoload.php';
$client = new WebSocket\Client("ws://localhost:8765/live?username=swatchesbybaobao");

while (true) {
    $msg = json_decode($client->receive(), true);
    if ($msg['event'] === 'chat') {
        echo "💬 " . $msg['data']['user']['nickname'] . ": " . $msg['data']['comment'] . "\n";
    }
}
```

#### 🌐 HTML5 / Web / OBS Overlay
```html
<script>
  const ws = new WebSocket("ws://localhost:8765/live?username=swatchesbybaobao");
  ws.onmessage = (event) => {
    const msg = JSON.parse(event.data);
    if (msg.event === "chat") {
      document.body.innerHTML += `<p><b>${msg.data.user.nickname}:</b> ${msg.data.comment}</p>`;
    }
  };
</script>
```

---

## 📊 Standardized JSON Output Examples

### Chat Event (`event: "chat"`)
```json
{
  "event": "chat",
  "username": "swatchesbybaobao",
  "room_id": "7679237785261837074",
  "timestamp": "2026-09-10T08:30:00.123Z",
  "data": {
    "user": {
      "id": "111222333",
      "nickname": "Alex",
      "unique_id": "alex99",
      "avatar_url": "https://p16-sign-va.tiktokcdn.com/...",
      "is_host": false,
      "is_mod": true,
      "is_sub": false,
      "is_fan": true,
      "fan_club": {
        "name": "VIP Club",
        "level": 12
      }
    },
    "comment": "Does this product ship internationally?"
  }
}
```

### TikTok Shop Event (`event: "oec_live_shopping"`)
```json
{
  "event": "oec_live_shopping",
  "username": "swatchesbybaobao",
  "room_id": "7679237785261837074",
  "timestamp": "2026-09-10T08:30:05.456Z",
  "data": {
    "action_type": 1,
    "action_name": "SetPinProduct (Ghim sản phẩm mới)",
    "product_id": "1734309253794202883",
    "product_title": "Olay Body Cellscience B5 Whitening Lotion 260g",
    "product_image": "https://p16-oec-sg.ibyteimg.com/...webp",
    "product_url": "https://shop.tiktok.com/vn/pdp/olay-body-cellscience/1734309253794202883",
    "seller": "P&G Beauty Official Store",
    "sold_count": "126.2K sold"
  }
}
```

---

## 📂 Standalone Distribution Package

For plug-and-play deployment onto Windows or Linux VPS environments, the repository includes a self-contained directory:
👉 **[`tiktok_live_service/`](./tiktok_live_service/)**

Inside you will find:
* `ws_server.py`: The standalone gateway server.
* `install.bat`: 1-click batch installer that verifies Python, upgrades pip, installs dependencies, and pulls Chromium.
* `start_server.bat`: 1-click server runner for Windows CMD.
* `test_client.bat`: 1-click interactive Python test console.
* `ws_client_example.html`: Built-in web dashboard for visual testing.
* `WEBSOCKET_GUIDE.md`: Deep-dive integration manual for all client platforms.

---

## 📄 License

This project is licensed under the [MIT License](LICENSE) — see the LICENSE file for details.  
Originally inspired by and forked from [PirateTok/live-py](https://github.com/PirateTok/live-py).
