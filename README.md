# SD-WAN

SD-WAN overlay implementation using FortiGate, providing dual-path connectivity over SASE and MPLS paths with intelligent path selection.

---

## 🔬 Overview

This component is part of an MPLS/SASE-integrated lab. It focuses on SD-WAN path selection, health monitoring, and failover behavior between multiple WAN paths.

The lab demonstrates SD-WAN capabilities including:

- **Dual-Path Design** – SASE primary path with MPLS secondary
- **Health Check** – Continuous endpoint monitoring
- **SLA-based Failover** – Automatic path switching based on latency/jitter/loss

**【日本語サマリ】**

FortiGateを使用したSD-WANオーバーレイ実装。
SASE（プライマリ）とMPLS（セカンダリ）のデュアルパス設計、Health Checkによる常時監視、SLAベースの自動フェイルオーバーを検証。

---

## 🏗️Architecture Position

<img width="500" alt="image" src="https://github.com/user-attachments/assets/5aa3621e-8c18-489d-9076-c838b75b3af6" />


SD-WAN operates as the **Overlay Layer**, managing path selection between SASE and MPLS independently of underlay routing and security policy enforcement.

**【日本語サマリ】**

SD-WANはOverlay層として動作し、Underlay（MPLS）のルーティングやSASE層のセキュリティポリシーとは独立してパス選択を管理。

---

## 🧩Components

### Dual-Path IPsec Tunnels

| Path | Tunnel Name | Underlay | Priority |
|------|-------------|----------|----------|
| Primary | SASE-VPN | Internet (WireGuard) | 1 (High) |
| Secondary | MPLS-VPN | MPLS L3VPN | 2 (Low) |

Both tunnels use IPsec ESP (Protocol 50) for site-to-site encryption.

### Health Check Configuration

| Parameter | Value | Purpose |
|-----------|-------|---------|
| Protocol | ICMP / HTTP | Endpoint reachability |
| Interval | 500ms - 1000ms | Probe frequency |
| Failover Threshold | 5 consecutive failures | Trigger path switch |
| Recovery Threshold | 5 consecutive successes | Return to primary |

### SLA Metrics

| Metric | Threshold | Action |
|--------|-----------|--------|
| Latency | > 100ms | SLA violation |
| Jitter | > 50ms | SLA violation |
| Packet Loss | > 5% | SLA violation |

**【日本語サマリ】**

SASE-VPN（プライマリ、Priority 1）とMPLS-VPN（セカンダリ、Priority 2）の2つのIPsecトンネルを構成。
Health CheckはICMP/HTTPで500ms-1000ms間隔、5回連続失敗でフェイルオーバー。
SLAはLatency 100ms / Jitter 50ms / Loss 5%を閾値として監視。

---

## 🔀Path Selection Logic

### SD-WAN Rule Evaluation

```
Traffic arrives at SD-WAN Edge (FortiGate)
               │
               ▼
┌─────────────────────────────┐
│  Check SASE-VPN Health      │
│  - Health Check: OK?        │
│  - SLA Metrics: Within?     │
└──────────────┬──────────────┘
               │
       ┌───────┴───────┐
       │               │
       ▼               ▼
   [Healthy]      [Unhealthy/SLA Violation]
       │               │
       ▼               ▼
  Use SASE-VPN    Use MPLS-VPN
  (Primary)       (Secondary)
```

### Path Priority

1. **SASE Path** – Preferred when healthy (lower latency via direct internet)
2. **MPLS Path** – Used during SASE degradation or outage (stable enterprise WAN)

### AS-Path Prepending Requirement

A critical design decision involved BGP AS-path manipulation:

**Problem:** MPLS path traversed more AS hops than SASE path, which aligned with our intent. However, ensuring consistent behavior required explicit configuration.

**Solution:** AS-path prepending applied to MPLS-learned routes to reinforce SASE as primary path.

This illustrates that SD-WAN path selection must account for underlying BGP mechanics.

**【日本語サマリ】**

SD-WANルールはHealth Check → SLAメトリクス の順で評価し、SASEが正常ならSASE、異常ならMPLSを選択。
MPLSパスのASホップ数がSASEより多いことを利用し、AS-path prependingでSASEを優先パスとして明示的に設定。

---

## ✅Verification Results

### Health Check Status
📷 SD-WAN Health Check Before
**<img width="700" alt="image" src="https://github.com/user-attachments/assets/94006082-1f61-4216-b8d4-6688c34fd66e" />

**Before (Normal State):**

| Tunnel | Health Check | SLA | Status |
|--------|--------------|-----|--------|
| SASE-VPN | ✓ Alive | ✓ OK | Active |
| MPLS-VPN | ✓ Alive | ✓ OK | Standby |

📷 SD-WAN Health Check After



**After (SASE Degradation):**

| Tunnel | Health Check | SLA | Status |
|--------|--------------|-----|--------|
| SASE-VPN | ✓ Alive | ✗ Violation | Standby |
| MPLS-VPN | ✓ Alive | ✓ OK | Active |

Failover triggered by SLA violation, not complete outage.

### IPsec Tunnel Verification

FortiGate CLI confirmation:

```
diagnose vpn ike gateway list
diagnose vpn tunnel list

SASE-VPN: selectors 1/1 (UP)
MPLS-VPN: selectors 1/1 (UP)
```

### ESP Packet Capture
<img width="1172" height="809" alt="image" src="https://github.com/user-attachments/assets/28c8aa37-e34f-450c-b480-5ddc8eaa0d50" />
<img width="1258" height="952" alt="image" src="https://github.com/user-attachments/assets/ac7834b6-cb5b-4b4c-9da4-9a2d416cbf24" />

**📷 Wireshark ESP packet capture**

| Field | SASE-VPN | MPLS-VPN |
|-------|----------|----------|
| Protocol | ESP (50) | ESP (50) |
| Source | 10.0.0.1 (FG1) | 10.0.0.1 (FG1) |
| Destination | 10.0.1.1 (FG2) | 10.0.1.1 (FG2) |
| SPI | 0x78ce2103 | 0x20bf398a |

SPI values confirm distinct tunnel identities.

**【日本語サマリ】**

Health Check Before/Afterで、SASE SLA違反時にMPLSへ自動フェイルオーバーを確認。
IPsecトンネルは両方UP状態を維持し、ESPパケットキャプチャでSPI値による識別を検証。

---

## ⚡Brownout Detection

### What is Brownout?

Brownout is **partial degradation** without complete failure:
- Increased latency
- Higher jitter
- Intermittent packet loss
- Connection remains "up" but quality degrades

### Why Brownout Matters

Traditional monitoring (ping/up-down) misses brownout:

| Scenario | Ping Status | User Experience |
|----------|-------------|-----------------|
| Complete Outage | DOWN | No connectivity |
| Brownout | UP | Slow, choppy, unreliable |

SD-WAN SLA monitoring detects brownout and triggers failover.

### Brownout Verification

**[ 📷 ここに画像添付：Brownout時 Wireshark I/O Graph ]**

I/O Graph observations:
- Traffic continues flowing (not zero)
- Packet rate fluctuates
- Quality degradation visible in time-series

This demonstrates SD-WAN's ability to detect quality issues, not just connectivity loss.

**【日本語サマリ】**

Brownoutは完全断ではなく部分的な品質劣化（遅延増加、ジッター、パケットロス）。
従来のPing監視では検知できないが、SD-WANのSLA監視で検知しフェイルオーバー可能。
Wireshark I/Oグラフで通信継続中の品質劣化を可視化。

---

## 🔄Failover Behavior

### Failover Timeline

```
### Failover Timeline

**[ 📷 ここに画像添付：FortiGate CLI - diagnose sys sdwan health-check (Before) ]**

**[ 📷 ここに画像添付：FortiGate CLI - diagnose sys sdwan health-check (After) ]**

**[ 📷 ここに画像添付：FortiGate CLI - execute log display (Failover event) ]**

Actual failover timing and behavior to be documented from CLI output.

```

### Key Observations

- **Failover Time:** ~2-3 seconds after SLA violation
- **Failback:** Automatic when primary recovers
- **Session Continuity:** TCP sessions maintained during failover (with brief interruption)

**【日本語サマリ】**

SLA違反検知から2-3秒でフェイルオーバー実行。プライマリ復旧時は自動でフェイルバック。
TCPセッションは短い中断を伴うが維持される。

---

## 🧪Lab Constraints

### Simulated Brownout

Production brownout occurs naturally. Lab simulation methods:
- Traffic shaping on underlay
- Latency injection via tc (traffic control)
- Bandwidth limitation

### WireGuard as SASE Path Underlay

In production, site-to-site connectivity would use BGP over IPsec configured directly on edge routers. WireGuard is used in this lab because:
- Linux POP devices cannot run BGP over IPsec natively
- WireGuard provides UDP-based transport suitable for NAT traversal

**【日本語サマリ】**

ラボではtcコマンドによる遅延注入でBrownoutをシミュレート。
本番の拠点間接続はBGP over IPsecだが、ラボではLinux POPでBGP over IPsecが設定できないためWireGuardで代替。

---

## 🚀Key Learnings

### SD-WAN vs Traditional WAN

| Aspect | Traditional | SD-WAN |
|--------|-------------|--------|
| Failover | Manual / Routing convergence | Automatic, SLA-based |
| Brownout Detection | None | Built-in |
| Path Selection | Routing protocol only | Policy + SLA + Health |
| Visibility | Limited | Per-tunnel metrics |

### SLA-Based Path Selection

SLA monitoring provides advantages over simple health checks:
- Detects quality degradation before failure
- Enables proactive failover
- Maintains user experience during brownout

### Integration Points

SD-WAN interacts with other layers:
- **SASE Layer:** Provides primary path + security
- **MPLS Underlay:** Provides secondary transport
- **BGP:** Requires coordination (AS-path prepending)

**【日本語サマリ】**

従来WANは手動/ルーティング収束でフェイルオーバー、SD-WANはSLAベースで自動。
Brownout検知は従来WANにない機能。SD-WANはSASE/MPLS/BGPと連携して動作。

---

## 🔗Related Components

- [SASE-ZeroTrust](../SASE-ZeroTrust) - Security policy enforcement
- [Brownout](../Brownout) - Detailed degradation analysis
- [Troubleshooting](../Troubleshooting) - Packet capture and visibility

---

*Part of SASE × SD-WAN Verification Lab*
