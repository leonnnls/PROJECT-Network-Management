# Product Requirements Document (PRD)

## IRAMON - Network Monitoring & Traffic Analysis System

**Version:** 1.1  
**Last Updated:** April 6, 2026  
**Author:** Development Team  
**Status:** Production  
**Changelog:** Cleanup & bug fixes (file cleanup, log reorganization, service status fix)

---

## Table of Contents

1. [Executive Summary](#1-executive-summary)
2. [Product Overview](#2-product-overview)
3. [System Architecture](#3-system-architecture)
4. [Core Features](#4-core-features)
5. [Technical Specifications](#5-technical-specifications)
6. [Database Schema](#6-database-schema)
7. [Data Flow](#7-data-flow)
8. [API Endpoints](#8-api-endpoints)
9. [Scheduled Tasks](#9-scheduled-tasks)
10. [Security Considerations](#10-security-considerations)
11. [Dependencies](#11-dependencies)
12. [Deployment Requirements](#12-deployment-requirements)
13. [Known Limitations](#13-known-limitations)
14. [Future Roadmap](#14-future-roadmap)
15. [Appendix A: File Structure](#appendix-a-file-structure)
16. [Appendix B: Glossary](#appendix-b-glossary)
17. [Appendix C: Blocked Domain Detection](#appendix-c-blocked-domain-detection)

---

## 1. Executive Summary

IRAMON (Industrial Robotic Automation - Network Monitoring) is a comprehensive network monitoring and traffic analysis system designed for enterprise environments. It provides real-time packet capture, traffic analysis, device monitoring, and automated reporting through a modern web dashboard and Telegram bot interface.

### Key Capabilities
- **Real-time Network Sniffing** using TShark/Wireshark engine
- **Traffic Analysis** with Local/Public/CCTV/Blocked categorization
- **Server Health Monitoring** (CPU, RAM, Disk, Temperature)
- **Automated Daily Reports** via PDF + Telegram
- **Device Management** (IP, MAC, CCTV registration)
- **Multi-Platform Access** (Web Dashboard + Telegram Bot)

---

## 2. Product Overview

### 2.1 Problem Statement

Enterprise networks require continuous monitoring to:
- Track bandwidth usage per device/user
- Identify unauthorized devices on the network
- Monitor blocked website access attempts (YouTube, Facebook, etc.)
- Track CCTV/NVR network traffic separately
- Detect anomalous traffic patterns
- Generate daily/weekly reports for management

### 2.2 Solution

IRAMON provides an automated, multi-service solution that:
1. Captures all network traffic during working hours (07:30 - 15:50)
2. Classifies traffic into categories (Local, Public, CCTV, Blocked)
3. Provides real-time visibility via web dashboard
4. Sends automated PDF reports via Telegram bot
5. Monitors server infrastructure health

### 2.3 Target Users

| User Role | Usage |
|-----------|-------|
| **IT Administrator** | Configure devices, monitor traffic, manage sniffer |
| **IT Manager** | View dashboard, receive daily reports, check server status |
| **Management** | Receive automated PDF reports via Telegram |

---

## 3. System Architecture

### 3.1 High-Level Architecture

```
┌─────────────────────────────────────────────────────────────────┐
│                     IRAMON ECOSYSTEM                            │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  🌐 LARAVEL WEB APP (Port 8000)                                │
│  ├─ DashboardController → Server metrics & ping monitoring     │
│  ├─ MonitorController → Traffic logs, live view, analytics     │
│  └─ ManagementController → IP/MAC/CCTV/Server CRUD             │
│                                                                 │
│  🐍 PYTHON SERVICES                                            │
│  ├─ python/main.py → TShark packet sniffer (multi-threaded)    │
│  └─ python/system_monitor.py → CPU/RAM/Temp collector          │
│                                                                 │
│  🤖 TELEGRAM BOT                                               │
│  └─ app/Console/Commands/TelegramMonitor.php                   │
│     ├─ Interactive commands (/status, /history, /id)           │
│     └─ PDF report generation & delivery                        │
│                                                                 │
│  🗄️ DATABASE (MySQL)                                          │
│  ├─ Core tables: server_metrics, monitored_servers             │
│  ├─ Mapping tables: registered_ips, registered_macs           │
│  ├─ Config tables: cctv_configs, excluded_ips                 │
│  └─ Dynamic tables: log_*_{timestamp} (per session)           │
│                                                                 │
│  ⚙️ EXTERNAL TOOLS                                            │
│  ├─ TShark (Wireshark) → Packet capture engine                 │
│  └─ Network Share → Auto-copy reports to NAS                   │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

### 3.2 Component Interaction

```
┌──────────┐     ┌──────────┐     ┌──────────┐     ┌──────────┐
│ TShark   │────▶│ Python   │────▶│ MySQL    │◀────│ Laravel  │
│ Capture  │     │ Sniffer  │     │ Database │     │ Web App  │
└──────────┘     └──────────┘     └──────────┘     └──────────┘
                                       │                   │
                                       │                   │
                                       ▼                   ▼
                                 ┌──────────┐     ┌──────────┐
                                 │ Telegram │◀────│ PDF      │
                                 │ Bot      │     │ Report   │
                                 └──────────┘     └──────────┘
```

---

## 4. Core Features

### 4.1 Dashboard Module

**Purpose:** Real-time server health & infrastructure monitoring

**Features:**
- **Performance Widget**
  - CPU utilization (%)
  - Memory usage (GB used / total)
  - Disk activity (%)
  - GPU name & status
  - CPU temperature (°C) with line chart (60-min history)

- **Service Status Widget**
  - TShark Sniffer status (Running/Stopped)
    - Status berdasarkan heartbeat database (60 detik terakhir)
    - Python sniffer update heartbeat setiap 3 detik saat aktif
    - **Logic:** `WHERE last_heartbeat >= DATE_SUB(NOW(), INTERVAL 60 SECOND)`
    - ✅ **Fixed (v1.1):** Removed false positive `$isActivelyWriting` logic
  - Telegram Bot status (Running/Stopped)
    - Status berdasarkan heartbeat database (60 detik terakhir)
    - Bot update heartbeat setiap loop iteration
    - **Logic:** `WHERE last_heartbeat >= DATE_SUB(NOW(), INTERVAL 60 SECOND)`
  - Live heartbeat indicator
  - **Note:** Status di-update via AJAX setiap 3 detik setelah page load

- **Server List Widget**
  - Ping monitoring for configured servers
  - Auto-detect server type (Server/Cloud/Database)
  - Status: ONLINE/DOWN with response time
  - Signal strength indicator
  - Click to open web URL

**Data Sources:**
- `server_metrics` table (updated every 1 second)
- `monitored_servers` table (ping check every 3 seconds)
- `service_status` table (heartbeat tracking)

---

### 4.2 Monitoring Module

#### 4.2.1 Session Logs

**Purpose:** View all captured traffic sessions

**Features:**
- List all capture sessions with metadata
- Status indicators:
  - 🟢 **LIVE** - Currently capturing
  - ✅ **PROCESSED** - Analytics ready
  - ⏳ **PENDING** - Requires processing
- Batch process multiple sessions
- Delete multiple sessions
- Export to Excel

**Workflow:**
1. Sniffer creates `log_{interface}_{timestamp}` table
2. Session appears in list with packet count
3. User clicks "Process" to generate analytics
4. Backend aggregates data into `session_analytics`
5. Session marked as "PROCESSED" with full analytics

---

#### 4.2.2 Live View

**Purpose:** Real-time packet capture visualization

**Features:**
- Live packet stream (1-second refresh)
- Protocol distribution chart
- Source IP ranking by volume (bytes)
- Packet table with:
  - Timestamp
  - Source IP + Port
  - Destination IP + Port
  - Protocol (color-coded)
  - Packet length
  - Info/Description

**Protocol Colors:**
- TCP: Green
- UDP: Yellow
- ICMP: Red
- QUIC: Teal
- TLS/SSL: Purple

---

#### 4.2.3 Session Detail

**Purpose:** Deep dive into a specific capture session

**Features:**
- **Summary Card**
  - Total packets
  - Total volume (MB)
  - Unique source IPs
  - Session metadata

- **Analytics Tabs** (after processing)
  - **Local Traffic** - Internal network communication
  - **Public Traffic** - External/Internet communication
  - Each tab shows:
    - Top 10 Source IPs (with progress bars)
    - Protocol distribution
    - Top destination ports

- **Security & Insights**
  - **CCTV Traffic** - Registered CCTV devices & bandwidth
  - **Unregistered IPs** - Unknown devices on local network
  - **Blocked Access** - Attempts to access restricted sites

- **Raw Packet Logs**
  - Paginated table (50 per page)
  - Filter by traffic type (Local/Public/All)
  - Search by IP, protocol, port, info
  - Click any IP to analyze

---

#### 4.2.4 IP Analysis

**Purpose:** Analyze traffic from/to a specific IP address

**Features:**
- **IP Info Header**
  - IP address
  - User/device label (if registered)
  - MAC address + vendor
  - Register MAC modal (quick action)
  - Total packets, volume, unique destinations

- **Top Destinations**
  - IPs this device communicates with
  - Packet count per destination

- **Top Ports**
  - Most used destination ports
  - Service name mapping (e.g., 80=HTTP, 443=HTTPS)

- **Activity Log Stream**
  - Filterable packet history
  - Filter by: Dst IP, Src IP, Protocol, Port, Info
  - MAC address labels on each packet
  - Pagination

---

### 4.3 Management Module

#### 4.3.1 Monitored Servers

**Purpose:** Configure servers shown on dashboard ping list

**Fields:**
- Server Name (e.g., "OwnCloud")
- IP Address (e.g., "192.168.3.250")
- Web URL (optional)
- Auto-detect type from name:
  - "Cloud"/"Drive" → Cloud icon
  - "DB"/"SQL"/"Data" → Database icon
  - Others → Server icon

---

#### 4.3.2 Registered IPs

**Purpose:** Map IP addresses to user/device names

**Fields:**
- IP Address (unique)
- Label (e.g., "Laptop HRD", "PC Manager")

**Usage:** Labels appear in monitoring tables instead of raw IPs.

---

#### 4.3.3 Registered MACs

**Purpose:** Map MAC addresses to device names

**Fields:**
- MAC Address (unique, auto-formatted to uppercase)
- Device Name (e.g., "HP Printer", "Work Laptop")

**Usage:** MAC labels appear in IP analysis and live view.

---

#### 4.3.4 CCTV Configuration

**Purpose:** Register CCTV/NVR devices for separate traffic tracking

**Fields:**
- IP Address (unique)
- Device Name/Location (e.g., "NVR Lobby Utama")

**Usage:** CCTV traffic appears in dedicated card on session detail page.

---

#### 4.3.5 Excluded IPs (Sniffer Blacklist)

**Purpose:** Exclude specific IPs from packet capture to save storage

**Fields:**
- IP Address (unique)
- Label/Reason (e.g., "CCTV Camera", "Backup Server")

**Behavior:** Python sniffer applies BPF filter to ignore these IPs. Changes take effect on next sniffer restart.

---

### 4.4 Export & Reports

#### 4.4.1 Session Export (Excel)

**Purpose:** Export session data to Excel format

**Data Included:**
- Session metadata
- Top IPs, protocols, ports
- Local & public traffic tables
- CCTV traffic
- Blocked access attempts

---

#### 4.4.2 IP Analysis Export (Excel)

**Purpose:** Export single IP analysis to Excel

**Data Included:**
- IP info & summary
- Top destinations
- Top ports
- Recent activity logs (sample 2000 records)

---

#### 4.4.3 PDF Report

**Purpose:** Generate comprehensive PDF report for a session

**Sections:**
1. **Header** - Session ID, date, interface, status
2. **Summary Box** - Total packets, volume, capture time
3. **Top 10 Source IPs** - Packets & volume
4. **Top 10 Protocols** - Packets & volume
5. **Top 10 Ports** - Hits & volume
6. **Local Traffic** - Top 46 connections
7. **Public Traffic** - Top 46 connections
8. **CCTV Traffic** - Device names & activity
9. **Unregistered Devices** - Unknown IPs
10. **Blocked Access** - Restricted site attempts
11. **Network Timeline** - Hourly traffic graph (bar chart)

**Delivery:**
- Download via browser
- Sent via Telegram bot
- Auto-copied to network share: `\\192.168.3.250\IT Warehouse HR\Capture network\`

---

### 4.5 Telegram Bot

**Purpose:** Provide monitoring access via Telegram mobile app

**Commands:**

| Command | Description | Response |
|---------|-------------|----------|
| `/start` or `/menu` | Show main menu | Keyboard with buttons |
| `/status` or 📊 Status | Server & sniffer status | Text with metrics |
| `/history` or 📜 History Capture | List recent sessions | Inline buttons |
| `/id` or 🆔 Cek ID | Get chat ID | Text with ID |

**Interactive Features:**
- Click inline button → Generate PDF report → Send as document
- Auto-copy PDF to network share
- Caption includes session info & capture time

**Heartbeat:** Updates `service_status` table every loop iteration.

---

## 5. Technical Specifications

### 5.1 Tech Stack

| Component | Technology | Version |
|-----------|-----------|---------|
| **Backend Framework** | Laravel | 12.0 |
| **Language** | PHP | 8.2+ (Production: 8.3.26) |
| **Database** | MySQL | 8.0+ (Development: SQLite for testing) |
| **Frontend** | Bootstrap 5 + TailwindCSS 4 | Latest |
| **Charts** | Chart.js | CDN |
| **Build Tool** | Vite | 7.0 |
| **PDF Engine** | barryvdh/laravel-dompdf | 3.1 |
| **Excel Export** | maatwebsite/excel | 3.1 |
| **Telegram SDK** | irazasyed/telegram-bot-sdk | 3.15 |
| **Packet Capture** | TShark (Wireshark CLI) | Latest |
| **Sniffer Script** | Python | 3.x |
| **System Monitor** | Python (psutil, mysql-connector) | 3.x |

### 5.2 System Requirements

| Resource | Minimum | Recommended |
|----------|---------|-------------|
| **CPU** | 4 cores | 8+ cores |
| **RAM** | 8 GB | 16+ GB |
| **Storage** | 100 GB SSD | 500+ GB SSD |
| **OS** | Windows 10/11 | Windows Server 2019+ |
| **Network** | 1 Gbps | 1 Gbps+ |

### 5.3 External Dependencies

| Tool | Path | Purpose |
|------|------|---------|
| **TShark** | `C:\Program Files\Wireshark\tshark.exe` | Packet capture engine |
| **PHP** | `C:\laragon\bin\php\php-8.3.26-Win32-vs16-x64\php.exe` | Laravel runtime |
| **Python** | System PATH | Sniffer & monitor scripts |
| **Network Share** | `\\192.168.3.250\IT Warehouse HR\Capture network` | Report storage |

### 5.4 Performance Characteristics

| Metric | Value | Notes |
|--------|-------|-------|
| **Capture Rate** | ~10K packets/batch | Python sniffer |
| **Processing Chunk** | 5M rows/batch | Daily analytics |
| **Live View Refresh** | 1 second | AJAX polling |
| **Dashboard Update** | 3 seconds | AJAX polling |
| **Server Ping** | 3 seconds | Per server |
| **Heartbeat Interval** | 3 seconds (sniffer) | 1 second (monitor) |
| **CPU Temp Cache** | 60 seconds | Avoid frequent sensor reads |

---

## 6. Database Schema

### 6.1 Core Tables

#### `server_metrics`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `cpu_load` | FLOAT | CPU utilization % |
| `ram_used` | FLOAT | RAM used (GB) |
| `ram_total` | FLOAT | Total RAM (GB) |
| `ram_percent` | FLOAT | RAM utilization % |
| `disk_percent` | FLOAT | Disk utilization % |
| `gpu_name` | VARCHAR(255) | GPU name or "Offline" |
| `cpu_temp` | FLOAT | CPU temperature (°C) |
| `created_at` | TIMESTAMP | Record time |

#### `monitored_servers`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `name` | VARCHAR(255) | Server display name |
| `ip_address` | VARCHAR(45) | Server IP |
| `url` | VARCHAR(255) | Web URL (nullable) |
| `type` | VARCHAR(50) | server/cloud/database |
| `created_at` | TIMESTAMP | Record time |

#### `registered_ips`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `ip_address` | VARCHAR(45) | IP address (unique) |
| `label` | VARCHAR(100) | User/device name |
| `created_at` | TIMESTAMP | Record time |

#### `registered_macs`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `mac_address` | VARCHAR(17) | MAC address (unique, uppercase) |
| `label` | VARCHAR(100) | Device name |
| `created_at` | TIMESTAMP | Record time |

#### `cctv_configs`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `ip_address` | VARCHAR(45) | CCTV IP (unique) |
| `device_name` | VARCHAR(50) | Device/location name |
| `created_at` | TIMESTAMP | Record time |

#### `excluded_ips`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `ip_address` | VARCHAR(45) | Excluded IP (unique) |
| `label` | VARCHAR(100) | Reason/note |
| `created_at` | TIMESTAMP | Record time |

#### `service_status`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `service_name` | VARCHAR(50) | Service identifier (unique) |
| `status` | VARCHAR(20) | running/stopped/active |
| `last_heartbeat` | TIMESTAMP | Last update time |

### 6.2 Dynamic Session Tables

#### `log_{interface}_{YYYYMMDD}_{HHMMSS}`
| Column | Type | Description |
|--------|------|-------------|
| `id` | BIGINT UNSIGNED PK | Auto-increment |
| `src_ip` | VARCHAR(45) | Source IP |
| `dst_ip` | VARCHAR(45) | Destination IP |
| `src_port` | INT | Source port (nullable) |
| `dst_port` | INT | Destination port (nullable) |
| `protocol` | VARCHAR(50) | Protocol name |
| `packet_len` | INT | Packet size (bytes) |
| `info` | TEXT | Packet description |
| `mac_src` | VARCHAR(20) | Source MAC |
| `mac_dst` | VARCHAR(20) | Destination MAC |
| `captured_at` | TIMESTAMP | Capture time |

**Indexes:**
- `id` (Primary Key)
- `captured_at` (Query optimization)
- `src_ip` (Query optimization)
- `dst_ip` (Query optimization)

### 6.3 Analytics Tables

#### `session_analytics`
| Column | Type | Description |
|--------|------|-------------|
| `session_id` | VARCHAR(100) | Source table name |
| `type` | VARCHAR(10) | local/public |
| `src_ip` | VARCHAR(45) | Source IP |
| `dst_ip` | VARCHAR(45) | Destination IP |
| `dst_port` | INT | Destination port |
| `protocol` | VARCHAR(50) | Protocol name |
| `packet_count` | BIGINT | Aggregated packets |
| `total_bytes` | BIGINT | Aggregated bytes |
| `created_at` | TIMESTAMP | Record time |

#### `daily_summaries`
| Column | Type | Description |
|--------|------|-------------|
| `log_date` | DATE | Date (indexed) |
| `src_ip` | VARCHAR(45) | Source IP |
| `dst_ip` | VARCHAR(45) | Destination IP |
| `dst_port` | INT | Destination port |
| `protocol` | VARCHAR(50) | Protocol name |
| `packet_count` | BIGINT | Aggregated packets |
| `total_bytes` | BIGINT | Aggregated bytes |

---

## 7. Data Flow

### 7.1 Capture Phase (07:30 - 15:50)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Python Sniffer starts at 07:30                           │
│    ├─ Detect network interface (TShark -D)                  │
│    ├─ Create/load session table: log_{iface}_{date}_{time}  │
│    └─ Apply excluded IPs filter (BPF)                       │
│                                                             │
│ 2. TShark captures packets                                 │
│    ├─ Output format: pipe-separated fields                  │
│    ├─ Queue packets in memory                               │
│    └─ Update heartbeat every 3 seconds                      │
│                                                             │
│ 3. DB Writer Thread                                         │
│    ├─ Batch insert: 10,000 packets/commit                   │
│    ├─ Commit interval: 2 seconds                            │
│    └─ Retry on connection loss                              │
│                                                             │
│ 4. Stop at 15:50                                            │
│    ├─ Flush remaining packets                               │
│    └─ Close session table                                   │
└─────────────────────────────────────────────────────────────┘
```

### 7.2 Processing Phase (16:00 / 23:35)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. Trigger: Scheduled (23:35) or Manual                     │
│                                                             │
│ 2. For each session table today:                            │
│    ├─ Reset existing analytics                              │
│    ├─ Chunk processing: 5M rows/batch                       │
│    ├─ INSERT INTO session_analytics (local)                 │
│    ├─ INSERT INTO session_analytics (public)                │
│    └─ INSERT INTO daily_summaries (aggregated)              │
│                                                             │
│ 3. Generate PDF Report                                      │
│    ├─ Query analytics data                                  │
│    ├─ Render PDF template                                   │
│    ├─ Save to storage/app/                                  │
│    ├─ Copy to network share                                 │
│    └─ Send via Telegram bot                                 │
│                                                             │
│ 4. Cleanup                                                  │
│    └─ Delete local PDF file                                 │
└─────────────────────────────────────────────────────────────┘
```

### 7.3 Monitoring Phase (Real-time)

```
┌─────────────────────────────────────────────────────────────┐
│ 1. System Monitor (python/system_monitor.py)                │
│    ├─ Read CPU, RAM, Disk every 1 second                    │
│    ├─ Cache CPU temp (60s interval)                         │
│    └─ Insert into server_metrics                            │
│                                                             │
│ 2. Laravel Dashboard (AJAX 3s interval)                     │
│    ├─ GET /monitor/api/stats → Latest metrics               │
│    ├─ GET /monitor/api/ping-check → Server status           │
│    └─ Update UI: charts, widgets, badges                    │
│                                                             │
│ 3. Live View (AJAX 1s interval)                             │
│    ├─ GET /monitoring/api/live-stream/{table}               │
│    ├─ Query last 30 packets                                 │
│    ├─ Aggregate protocol stats                              │
│    └─ Update packet table & source list                     │
└─────────────────────────────────────────────────────────────┘
```

---

## 8. API Endpoints

### 8.1 Web Routes

| Method | Path | Controller | Description |
|--------|------|-----------|-------------|
| GET | `/` | DashboardController@index | Main dashboard |
| GET | `/monitor/api/stats` | MonitorController@getStatsApi | Get server metrics |
| GET | `/monitor/api/ping-check?ip=` | MonitorController@pingServer | Ping server |
| GET | `/monitoring` | MonitorController@monitoringList | Session list |
| GET | `/monitoring/live-current` | MonitorController@live | Redirect to active session |
| GET | `/monitoring/live-view/{table}` | MonitorController@live_view | Live view page |
| GET | `/monitoring/api/live` | MonitorController@getLiveData | Live status API |
| GET | `/monitoring/api/live-stream/{table}` | MonitorController@live_stream_api | Live packet stream |
| GET | `/monitoring/{tableName}` | MonitorController@monitoringDetail | Session detail |
| GET | `/monitoring/{table}/ip/{ip}` | MonitorController@analyzeIp | IP analysis |
| POST | `/monitoring/process/init` | MonitorController@initProcess | Init processing |
| POST | `/monitoring/process/chunk` | MonitorController@processChunk | Process chunk |
| POST | `/monitoring/delete` | MonitorController@deleteSessions | Delete sessions |
| GET | `/monitoring/export/{table}` | MonitorController@exportSession | Export session |
| GET | `/monitoring/export/{table}/ip/{ip}` | MonitorController@exportIpAnalysis | Export IP |
| GET | `/management` | ManagementController@index | Management page |
| POST | `/management/ip/add` | ManagementController@storeIp | Add IP |
| DELETE | `/management/ip/{id}` | ManagementController@deleteIp | Delete IP |
| POST | `/management/mac/add` | ManagementController@storeMac | Add MAC |
| DELETE | `/management/mac/{id}` | ManagementController@deleteMac | Delete MAC |
| POST | `/management/server/add` | ManagementController@storeServer | Add server |
| DELETE | `/management/server/{id}` | ManagementController@deleteServer | Delete server |
| POST | `/management/cctv` | ManagementController@storeCctv | Add CCTV |
| DELETE | `/management/cctv/{id}` | ManagementController@deleteCctv | Delete CCTV |
| POST | `/management/sniffer/store` | ManagementController@storeExcludedIp | Add excluded IP |
| DELETE | `/management/sniffer/{id}` | ManagementController@destroyExcludedIp | Delete excluded IP |

### 8.2 Console Commands

| Command | Description |
|---------|-------------|
| `php artisan bot:run` | Start Telegram bot |
| `php artisan sniffer:control {start|stop}` | Control Python sniffer |
| `php artisan monitor:process-daily` | Process daily sessions + PDF |
| `php artisan monitor:process {table}` | Process single session |
| `php artisan monitor:stream` | System monitor daemon (legacy) |
| `php artisan monitor:record-metrics` | Record server metrics (legacy) |

---

## 9. Scheduled Tasks

### 9.1 Laravel Scheduler (routes/console.php)

| Time | Command | Description |
|------|---------|-------------|
| `23:30` | `sniffer:control stop` | Stop packet capture |
| `23:35` | `monitor:process-daily` | Process today's sessions |
| `00:00` | `sniffer:control start` | Start new day's capture |

### 9.2 Python Internal Schedule

| Time | Action | Description |
|------|--------|-------------|
| `07:30` | Start TShark | Begin packet capture |
| `15:50` | Stop TShark | End packet capture |
| `16:00` | Trigger Processing | Call `php artisan monitor:process-daily` |

### 9.3 Setup Instructions

```bash
# Add to Windows Task Scheduler:
schtasks /create /tn "IRAMON Scheduler" /tr "php artisan schedule:run" /sc minute
```

---

## 10. Security Considerations

### 10.1 Current Security Measures

| Measure | Implementation |
|---------|---------------|
| **CSRF Protection** | Laravel default middleware |
| **SQL Injection Prevention** | Query builder (no raw queries) |
| **Input Validation** | Controller validation rules |
| **XSS Prevention** | Blade template escaping |
| **File Upload Validation** | IP/MAC format validation |

### 10.2 Known Gaps

| Gap | Risk | Recommendation |
|-----|------|----------------|
| **No Authentication** | Anyone can access dashboard | Add login system with `auth` middleware |
| **No Rate Limiting** | API endpoints can be abused | Add `throttle` middleware |
| **No Role-Based Access** | All users have full access | Implement roles (admin/viewer) |
| **Sensitive Data in Logs** | IP addresses exposed in logs | Implement log rotation & encryption |
| **Network Share Credentials** | NAS path hardcoded | Use mounted drive with service account |

### 10.3 Data Privacy

- IP addresses and MAC addresses are stored in database
- Traffic patterns can reveal user behavior
- Reports contain detailed network activity
- **Recommendation:** Implement user authentication and access logging

---

## 11. Dependencies

### 11.1 PHP Dependencies (composer.json)

```json
{
    "require": {
        "php": "^8.2",
        "laravel/framework": "^12.0",
        "barryvdh/laravel-dompdf": "^3.1",
        "irazasyed/telegram-bot-sdk": "^3.15",
        "maatwebsite/excel": "^3.1"
    }
}
```

### 11.2 JavaScript Dependencies (package.json)

```json
{
    "devDependencies": {
        "@tailwindcss/vite": "^4.0.0",
        "axios": "^1.11.0",
        "laravel-vite-plugin": "^2.0.0",
        "tailwindcss": "^4.0.0",
        "vite": "^7.0.7"
    }
}
```

### 11.3 Python Dependencies

```
mysql-connector-python
psutil
```

### 11.4 External Software

| Software | Purpose | Required Version |
|----------|---------|-----------------|
| **Wireshark** | TShark packet capture | Latest |
| **MySQL Server** | Database | 8.0+ |
| **PHP** | Laravel runtime | 8.3.26 |
| **Python** | Sniffer & monitor | 3.x |

---

## 12. Deployment Requirements

### 12.1 Server Setup

**Operating System:** Windows 10/11 or Windows Server 2019+  
**Web Server:** PHP built-in server (dev) or Nginx/Apache (prod)  
**Database:** MySQL 8.0+  
**PHP Version:** 8.3.26 (Laragon environment)

### 12.2 Installation Steps

```bash
# 1. Clone repository
git clone <repo> iramon
cd iramon

# 2. Install dependencies
composer install
npm install

# 3. Setup environment
cp .env.example .env
php artisan key:generate

# 4. Configure database (edit .env)
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=iramon_db
DB_USERNAME=root
DB_PASSWORD=

# 5. Run migrations
php artisan migrate

# 6. Build assets
npm run build

# 7. Start services
# - Open program/RUN_SNIFFER.bat (Python sniffer)
# - Open program/RUN_BOT.bat (Telegram bot)
# - php artisan serve (Web app)
```

### 12.3 Environment Variables

```env
# Application
APP_NAME=IRAMON
APP_ENV=production
APP_DEBUG=false
APP_URL=http://localhost:8000

# Database
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_DATABASE=iramon_db
DB_USERNAME=root
DB_PASSWORD=your_password

# Telegram
TELEGRAM_BOT_TOKEN=your_bot_token
TELEGRAM_ADMIN_ID=your_chat_id

# Network
TSHARK_PATH=C:\Program Files\Wireshark\tshark.exe
PHP_BINARY=C:\laragon\bin\php\php-8.3.26-Win32-vs16-x64\php.exe
```

### 12.4 Network Requirements

| Requirement | Description |
|-------------|-------------|
| **Network Interface** | Must have access to target network segment |
| **Wireshark/TShark** | Installed with capture permissions |
| **MySQL** | Running and accessible |
| **Telegram Bot** | Created via @BotFather, token configured |
| **Network Share** | `\\192.168.3.250` accessible (for report storage) |

---

## 13. Known Limitations

### 13.1 Technical Limitations

| Limitation | Impact | Workaround |
|------------|--------|------------|
| **Windows-only** | Cannot deploy on Linux | Python scripts use Windows-specific commands |
| **Single Interface** | Can only capture one NIC at a time | Configure `INTERFACE_NAME` in main.py |
| **Working Hours Only** | Capture limited to 07:30-15:50 | Modify schedule in main.py |
| **No Real-Time Alerts** | No instant notifications for anomalies | Check dashboard manually or via Telegram |
| **Large Data Processing** | PDF generation can be slow for huge sessions | Chunking algorithm handles up to 5M rows/batch |

### 13.2 Performance Limitations

| Scenario | Limitation | Mitigation |
|----------|-----------|------------|
| **100M+ packets/session** | Processing takes 10-30 minutes | Chunking (5M rows), run during off-hours |
| **Concurrent users** | No authentication = no session isolation | Plan for auth system in future |
| **Long-term storage** | Raw packet tables grow quickly | Archive old sessions, keep only analytics |

### 13.3 Functional Gaps

- No user management or authentication
- No multi-interface support
- No real-time alerting (e.g., bandwidth threshold)
- No historical trend analysis (weekly/monthly reports)
- No device discovery (auto-detect new devices)

### 13.4 Fixed Issues

| Issue | Status | Fix Date | Description |
|-------|--------|----------|-------------|
| **Service Status False Positive** | ✅ Fixed | April 2026 | TShark status showed "Running" even when stopped. Fixed by removing `$isActivelyWriting` logic and using only heartbeat validation (60 seconds). |
| **Unused Root Files** | ✅ Fixed | April 2026 | Removed `Melayani`, `Mencatat`, `Mengupdate`, `Update` files from root directory. |
| **Log Files in Wrong Location** | ✅ Fixed | April 2026 | Moved `Log-DataPreprocessing.txt` from `public/` and `sniffer.log` from root to `storage/logs/`. |
| **Duplicate Dashboard Views** | ✅ Fixed | April 2026 | Removed `monitor/index.blade.php` (unused) and `management/excluded.blade.php` (duplicate of section in main management page). |
| **Dead Controller Method** | ✅ Fixed | April 2026 | Removed `MonitorController::index()` method (no route, view deleted). |

---

## 14. Future Roadmap

### 14.1 Short Term (v1.x)

| Feature | Priority | Status | Description |
|---------|----------|--------|-------------|
| **Authentication System** | 🔴 High | ⏳ Pending | Login, roles, access control |
| **Config File for Paths** | 🟡 Medium | ⏭️ Skipped | Move hardcoded paths to `.env` (not needed for single-machine deployment) |
| **Log Management** | 🟡 Medium | ✅ Done | Centralized logging in `storage/logs/`, rotation needed |
| **API Rate Limiting** | 🟡 Medium | ⏳ Pending | Throttle AJAX endpoints |
| **Unit Tests** | 🟢 Low | ⏳ Pending | PHPUnit coverage for critical paths |
| **Code Cleanup** | 🟡 Medium | ✅ Done | Removed unused files, views, and dead code |

### 14.2 Medium Term (v2.0)

| Feature | Priority | Description |
|---------|----------|-------------|
| **Real-Time Alerts** | 🔴 High | Threshold-based notifications |
| **Multi-Interface Support** | 🟡 Medium | Capture from multiple NICs |
| **Historical Reports** | 🟡 Medium | Weekly/monthly PDF reports |
| **Device Discovery** | 🟡 Medium | Auto-detect new devices on network |
| **Dashboard Customization** | 🟢 Low | Widget rearrangement, custom metrics |

### 14.3 Long Term (v3.0)

| Feature | Priority | Description |
|---------|----------|-------------|
| **WebSocket Integration** | 🟡 Medium | Replace AJAX polling with real-time streaming |
| **REST API** | 🟡 Medium | External API for third-party integrations |
| **Mobile App** | 🟢 Low | React Native / Flutter companion app |
| **Cloud Deployment** | 🟢 Low | Support for AWS/GCP/Azure |
| **Machine Learning** | 🟢 Low | Anomaly detection, predictive analytics |

---

## Appendix A: File Structure

```
iramon/
├── app/
│   ├── Console/Commands/
│   │   ├── TelegramMonitor.php          # Telegram bot handler
│   │   ├── ProcessDailyTraffic.php      # Daily processing + PDF
│   │   ├── SnifferControl.php           # Start/stop sniffer
│   │   ├── ProcessSession.php           # Single session processor
│   │   ├── MonitorDaemon.php            # System monitor (legacy)
│   │   └── RecordServerMetrics.php      # Metrics recorder (legacy)
│   ├── Http/Controllers/
│   │   ├── DashboardController.php      # Dashboard page
│   │   ├── MonitorController.php        # Monitoring logic (1073 lines)
│   │   └── ManagementController.php     # CRUD operations
│   ├── Models/
│   │   ├── ServerMetric.php
│   │   ├── MonitoredServer.php
│   │   ├── RegisteredIp.php
│   │   ├── RegisteredMac.php
│   │   ├── CctvConfig.php
│   │   ├── ExcludedIp.php
│   │   └── User.php
│   └── Exports/
│       ├── SessionExport.php            # Excel export for sessions
│       └── IpExport.php                 # Excel export for IP analysis
├── config/
│   └── *.php                            # Laravel configuration
├── database/
│   ├── migrations/
│   └── seeders/
├── program/
│   ├── RUN_SNIFFER.bat                  # Python sniffer launcher
│   └── RUN_BOT.bat                      # Telegram bot launcher
├── python/
│   ├── main.py                          # TShark sniffer engine
│   └── system_monitor.py                # CPU/RAM/Temp collector
├── resources/
│   ├── views/
│   │   ├── dashboard.blade.php          # Main dashboard
│   │   ├── monitoring/
│   │   │   ├── index.blade.php          # Session list
│   │   │   ├── show.blade.php           # Session detail
│   │   │   ├── live.blade.php           # Live packet stream
│   │   │   └── ip_detail.blade.php      # IP analysis
│   │   ├── management/
│   │   │   └── index.blade.php          # Device management (all-in-one)
│   │   └── exports/
│   │       ├── session_pdf.blade.php    # PDF report template
│   │       └── ip_report.blade.php      # Excel IP report
│   └── js/, css/
├── routes/
│   ├── web.php                          # Web routes
│   └── console.php                      # Scheduled tasks
├── storage/
│   ├── logs/
│   │   ├── sniffer.log                  # Sniffer output (moved from root)
│   │   └── Log-DataPreprocessing.txt    # Preprocessing log (moved from public/)
│   └── app/                             # Generated PDFs
└── public/
    ├── index.php                        # Entry point
    └── Images/                          # Static assets
```

**Notes:**
- ❌ Removed files: `Melayani`, `Mencatat`, `Mengupdate`, `Update` (root)
- ❌ Removed views: `monitor/index.blade.php`, `management/excluded.blade.php`
- ❌ Removed method: `MonitorController::index()` (dead code)
- ✅ Moved log files to `storage/logs/` for proper organization

---

## Appendix B: Glossary

| Term | Definition |
|------|-----------|
| **BPF** | Berkeley Packet Filter - used by TShark to filter captured packets |
| **Chunking** | Processing large datasets in smaller batches to avoid memory overflow |
| **Heartbeat** | Periodic signal to indicate a service is alive and running |
| **Session** | A single capture period, stored as a `log_*` table |
| **TShark** | Command-line version of Wireshark for packet capture |
| **Local Traffic** | Communication between devices on the same network (192.168.x.x, 10.x.x.x, 172.x.x.x) |
| **Public Traffic** | Communication between local devices and external networks |
| **CCTV Traffic** | Network traffic from registered CCTV/NVR devices |
| **Blocked Traffic** | Attempts to access restricted domains (YouTube, Facebook, etc.) |

---

## Appendix C: Blocked Domain Detection

The system identifies blocked traffic by matching destination IP prefixes:

| Prefix | Service | Category |
|--------|---------|----------|
| `142.250.*` | YouTube/Google | Blocked |
| `172.217.*` | YouTube/Google | Blocked |
| `216.58.*` | YouTube/Google | Blocked |
| `173.194.*` | YouTube/Google | Blocked |
| `209.85.*` | YouTube/Google | Blocked |
| `74.125.*` | YouTube/Google | Blocked |
| `8.8.8.*` | Google DNS | Blocked |
| `8.8.4.4` | Google DNS | Blocked |
| `157.240.*` | Facebook/Instagram | Blocked |
| `31.13.*` | Facebook/Instagram | Blocked |
| `69.171.*` | Facebook/Instagram | Blocked |

---

**Document End**

*This PRD is a living document and should be updated as the product evolves.*
