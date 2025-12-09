# OCI-Manager Go 系统详细设计方案

**版本**: v1.0  
**创建日期**: 2025-12-09  
**项目代号**: oci-manager-go  
**技术栈**: Go 1.21+

---

## 📋 目录

1. [项目概述](#1-项目概述)
2. [系统架构](#2-系统架构)
3. [功能模块设计](#3-功能模块设计)
4. [数据库设计](#4-数据库设计)
5. [API 设计](#5-api-设计)
6. [技术选型](#6-技术选型)
7. [项目结构](#7-项目结构)
8. [开发计划](#8-开发计划)
9. [部署方案](#9-部署方案)

---

## 1. 项目概述

### 1.1 项目背景

基于对现有 Java 版 OCI-Start 项目的分析，决定使用 Go 语言重新开发一套高性能的 Oracle Cloud Infrastructure (OCI) 管理系统，专注于实例抢购（抢机）场景。

### 1.2 核心功能

| 模块 | 功能描述 | 优先级 |
|------|----------|--------|
| **系统资源监控** | 监控本地系统 CPU、内存、磁盘等资源 | P1 |
| **OCI 租户管理** | 多租户配置、API 密钥管理、区域管理 | P0 |
| **OCI 开机管理** | 实例抢购、批量开机、状态监控 | P0 |
| **系统日志** | 操作日志、抢机日志、错误日志 | P1 |
| **通知管理** | Telegram、钉钉、邮件等多渠道通知 | P1 |

### 1.3 设计目标

| 目标 | 指标 |
|------|------|
| 启动时间 | < 100ms |
| 内存占用 | < 50MB |
| 并发能力 | 10,000+ goroutine |
| 轮询间隔 | 最小 1 秒 |
| 可用性 | 99.9% |

---

## 2. 系统架构

### 2.1 整体架构图

```
┌─────────────────────────────────────────────────────────────────┐
│                        OCI-Manager-Go                           │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐             │
│  │   Web UI    │  │  REST API   │  │   CLI       │             │
│  │  (Vue/React)│  │  (Gin)      │  │  (Cobra)    │             │
│  └──────┬──────┘  └──────┬──────┘  └──────┬──────┘             │
│         │                │                │                     │
│         └────────────────┼────────────────┘                     │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Service Layer                          │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │  │
│  │  │ Tenant   │ │ Instance │ │ Monitor  │ │ Notify   │     │  │
│  │  │ Service  │ │ Service  │ │ Service  │ │ Service  │     │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Core Layer                             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌──────────┐     │  │
│  │  │ OCI SDK  │ │ Scheduler│ │ Logger   │ │ Config   │     │  │
│  │  │ Client   │ │ (Cron)   │ │ (Zap)    │ │ (Viper)  │     │  │
│  │  └──────────┘ └──────────┘ └──────────┘ └──────────┘     │  │
│  └──────────────────────────────────────────────────────────┘  │
│                          │                                      │
│                          ▼                                      │
│  ┌──────────────────────────────────────────────────────────┐  │
│  │                    Data Layer                             │  │
│  │  ┌──────────┐ ┌──────────┐ ┌──────────┐                  │  │
│  │  │ SQLite   │ │ Cache    │ │ File     │                  │  │
│  │  │ (GORM)   │ │ (Memory) │ │ Storage  │                  │  │
│  │  └──────────┘ └──────────┘ └──────────┘                  │  │
│  └──────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
                          │
                          ▼
        ┌─────────────────────────────────────┐
        │         External Services           │
        │  ┌─────┐ ┌─────┐ ┌─────┐ ┌─────┐   │
        │  │ OCI │ │ TG  │ │DingTalk│Email│   │
        │  │ API │ │ API │ │ API │ │SMTP │   │
        │  └─────┘ └─────┘ └─────┘ └─────┘   │
        └─────────────────────────────────────┘
```

### 2.2 模块交互流程

```
抢机流程:

┌──────┐    ┌──────────┐    ┌──────────┐    ┌─────────┐
│ 定时 │───▶│ Instance │───▶│ OCI SDK  │───▶│ OCI API │
│ 任务 │    │ Service  │    │ Client   │    │         │
└──────┘    └──────────┘    └──────────┘    └─────────┘
                │                                │
                │ 成功                           │
                ▼                                ▼
         ┌──────────┐                     ┌─────────┐
         │ Notify   │◀────────────────────│ 响应    │
         │ Service  │                     └─────────┘
         └──────────┘
                │
                ▼
         ┌──────────┐
         │ Telegram │
         │ DingTalk │
         │ Email    │
         └──────────┘
```

---

## 3. 功能模块设计

### 3.1 系统资源监控模块

| 功能 | 描述 |
|------|------|
| CPU 监控 | 实时监控 CPU 使用率 |
| 内存监控 | 监控内存使用情况 |
| 磁盘监控 | 监控磁盘空间和 I/O |
| 网络监控 | 监控网络流量 |
| 进程监控 | 监控应用进程状态 |

**技术实现**:
```go
// 使用 gopsutil 库
import "github.com/shirou/gopsutil/v3"

type SystemMetrics struct {
    CPUUsage    float64   `json:"cpu_usage"`
    MemoryUsage float64   `json:"memory_usage"`
    DiskUsage   float64   `json:"disk_usage"`
    NetworkIO   NetworkIO `json:"network_io"`
    Timestamp   time.Time `json:"timestamp"`
}
```

---

### 3.2 OCI 租户管理模块

| 功能 | 描述 |
|------|------|
| 租户 CRUD | 添加、编辑、删除、查询租户 |
| API 密钥管理 | 安全存储和使用 OCI API 密钥 |
| 区域订阅 | 管理租户可用区域 |
| 配额查询 | 查询租户资源配额 |
| 连接测试 | 测试租户 API 连接状态 |

**数据模型**:
```go
type Tenant struct {
    ID          uint      `gorm:"primaryKey"`
    Name        string    `gorm:"uniqueIndex;not null"`
    UserID      string    `gorm:"not null"`
    Fingerprint string    `gorm:"not null"`
    TenancyID   string    `gorm:"not null"`
    Region      string    `gorm:"not null"`
    KeyFile     string    `gorm:"not null"`
    Status      string    `gorm:"default:'active'"`
    CreatedAt   time.Time
    UpdatedAt   time.Time
}
```

---

### 3.3 OCI 开机管理模块（核心）

| 功能 | 描述 |
|------|------|
| 实例抢购 | 高频轮询抢购 OCI 实例 |
| 批量开机 | 多租户多区域并发开机 |
| 状态监控 | 实时监控抢机任务状态 |
| 任务管理 | 启动、停止、暂停抢机任务 |
| 配置管理 | Shape、OS、网络配置 |
| 资源预热 | 提前创建 VCN/Subnet |

**抢机任务模型**:
```go
type InstanceTask struct {
    ID            uint      `gorm:"primaryKey"`
    TenantID      uint      `gorm:"index"`
    Name          string    
    Shape         string    // VM.Standard.A1.Flex
    OCPU          float32   
    Memory        float32   
    DiskSize      int64     
    OS            string    // Ubuntu 22.04
    Region        string    
    Interval      int       // 轮询间隔(秒)
    Status        string    // pending/running/success/failed
    RetryCount    int       
    PublicIP      string    
    InstanceID    string    
    StartedAt     *time.Time
    CompletedAt   *time.Time
    CreatedAt     time.Time
    UpdatedAt     time.Time
}
```

**抢机引擎设计**:
```go
type GrabEngine struct {
    tasks      sync.Map              // 任务池
    ociClients map[uint]*oci.Client  // OCI客户端池
    scheduler  *gocron.Scheduler     // 调度器
    notifier   NotifyService         // 通知服务
    logger     *zap.Logger           // 日志
}

// 并发控制
func (e *GrabEngine) StartTask(task *InstanceTask) error {
    go func() {
        ticker := time.NewTicker(time.Duration(task.Interval) * time.Second)
        for {
            select {
            case <-ticker.C:
                if result := e.tryCreateInstance(task); result.Success {
                    e.notifier.Send(result)
                    return
                }
            case <-task.stopChan:
                return
            }
        }
    }()
    return nil
}
```

---

### 3.4 系统日志模块

| 日志类型 | 描述 |
|----------|------|
| 操作日志 | 用户操作记录 |
| 抢机日志 | 每次抢机尝试记录 |
| 错误日志 | 系统错误和异常 |
| 审计日志 | 安全相关操作 |

**日志模型**:
```go
type Log struct {
    ID        uint      `gorm:"primaryKey"`
    Level     string    `gorm:"index"` // INFO/WARN/ERROR
    Type      string    `gorm:"index"` // grab/system/operation
    Message   string    
    Details   string    // JSON 详情
    TenantID  *uint     
    TaskID    *uint     
    CreatedAt time.Time `gorm:"index"`
}
```

---

### 3.5 通知管理模块

| 渠道 | 支持状态 |
|------|----------|
| Telegram | ✅ 支持 |
| 钉钉机器人 | ✅ 支持 |
| 邮件 (SMTP) | ✅ 支持 |
| Webhook | ✅ 支持 |
| Server酱 | 🔄 计划中 |

**通知配置模型**:
```go
type NotifyChannel struct {
    ID        uint   `gorm:"primaryKey"`
    Type      string `gorm:"not null"` // telegram/dingtalk/email/webhook
    Name      string 
    Config    string // JSON 配置
    Enabled   bool   `gorm:"default:true"`
    CreatedAt time.Time
    UpdatedAt time.Time
}

// Telegram 配置
type TelegramConfig struct {
    BotToken string `json:"bot_token"`
    ChatID   string `json:"chat_id"`
}

// 钉钉配置
type DingTalkConfig struct {
    Webhook string `json:"webhook"`
    Secret  string `json:"secret"` // 加签密钥
}
```

---

## 4. 数据库设计

### 4.1 ER 图

```
┌──────────────┐       ┌──────────────┐
│   tenants    │       │ notify_channels│
├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │
│ name         │       │ type         │
│ user_id      │       │ name         │
│ fingerprint  │       │ config       │
│ tenancy_id   │       │ enabled      │
│ region       │       │ created_at   │
│ key_file     │       │ updated_at   │
│ status       │       └──────────────┘
│ created_at   │
│ updated_at   │
└──────┬───────┘
       │ 1:N
       ▼
┌──────────────┐       ┌──────────────┐
│instance_tasks│       │    logs      │
├──────────────┤       ├──────────────┤
│ id (PK)      │       │ id (PK)      │
│ tenant_id(FK)│       │ level        │
│ name         │       │ type         │
│ shape        │       │ message      │
│ ocpu         │       │ details      │
│ memory       │       │ tenant_id    │
│ disk_size    │       │ task_id      │
│ os           │       │ created_at   │
│ region       │       └──────────────┘
│ interval     │
│ status       │       ┌──────────────┐
│ retry_count  │       │system_metrics│
│ public_ip    │       ├──────────────┤
│ instance_id  │       │ id (PK)      │
│ started_at   │       │ cpu_usage    │
│ completed_at │       │ memory_usage │
│ created_at   │       │ disk_usage   │
│ updated_at   │       │ created_at   │
└──────────────┘       └──────────────┘
```

### 4.2 数据库选型

| 方案 | 适用场景 |
|------|----------|
| **SQLite** (默认) | 单机部署，简单便捷 |
| PostgreSQL | 生产环境，需要高可用 |
| MySQL | 已有 MySQL 环境 |

---

## 5. API 设计

### 5.1 API 概览

| 模块 | 端点 | 方法 |
|------|------|------|
| 租户 | `/api/v1/tenants` | GET/POST/PUT/DELETE |
| 任务 | `/api/v1/tasks` | GET/POST/PUT/DELETE |
| 任务控制 | `/api/v1/tasks/:id/start` | POST |
| 任务控制 | `/api/v1/tasks/:id/stop` | POST |
| 日志 | `/api/v1/logs` | GET |
| 通知 | `/api/v1/notify/channels` | GET/POST/PUT/DELETE |
| 监控 | `/api/v1/monitor/system` | GET |
| 监控 | `/api/v1/monitor/tasks` | GET |

### 5.2 API 示例

**创建租户**:
```http
POST /api/v1/tenants
Content-Type: application/json

{
  "name": "my-oci-tenant",
  "user_id": "ocid1.user.oc1..xxxx",
  "fingerprint": "aa:bb:cc:dd:ee...",
  "tenancy_id": "ocid1.tenancy.oc1..xxxx",
  "region": "ap-tokyo-1",
  "key_file": "/path/to/key.pem"
}
```

**创建抢机任务**:
```http
POST /api/v1/tasks
Content-Type: application/json

{
  "tenant_id": 1,
  "name": "抢ARM机器",
  "shape": "VM.Standard.A1.Flex",
  "ocpu": 4,
  "memory": 24,
  "disk_size": 50,
  "os": "Ubuntu 22.04",
  "region": "ap-tokyo-1",
  "interval": 5,
  "root_password": "your-password"
}
```

**启动任务**:
```http
POST /api/v1/tasks/1/start
```

---

## 6. 技术选型

### 6.1 核心依赖

| 类别 | 库 | 版本 | 用途 |
|------|-----|------|------|
| **Web框架** | Gin | v1.9+ | HTTP 路由和中间件 |
| **ORM** | GORM | v2.0+ | 数据库操作 |
| **OCI SDK** | oci-go-sdk | v65+ | OCI API 调用 |
| **配置** | Viper | v1.17+ | 配置管理 |
| **日志** | Zap | v1.26+ | 高性能日志 |
| **定时任务** | gocron | v2.0+ | 任务调度 |
| **系统监控** | gopsutil | v3.0+ | 系统指标采集 |
| **CLI** | Cobra | v1.8+ | 命令行工具 |

### 6.2 go.mod 示例

```go
module oci-manager-go

go 1.21

require (
    github.com/gin-gonic/gin v1.9.1
    gorm.io/gorm v1.25.5
    gorm.io/driver/sqlite v1.5.4
    github.com/oracle/oci-go-sdk/v65 v65.55.0
    github.com/spf13/viper v1.17.0
    github.com/spf13/cobra v1.8.0
    go.uber.org/zap v1.26.0
    github.com/go-co-op/gocron/v2 v2.2.0
    github.com/shirou/gopsutil/v3 v3.23.11
)
```

---

## 7. 项目结构

```
oci-manager-go/
├── cmd/
│   └── main.go                 # 程序入口
├── internal/
│   ├── api/
│   │   ├── router.go           # 路由定义
│   │   ├── middleware/         # 中间件
│   │   └── handlers/           # 处理器
│   │       ├── tenant.go
│   │       ├── task.go
│   │       ├── log.go
│   │       ├── notify.go
│   │       └── monitor.go
│   ├── config/
│   │   └── config.go           # 配置管理
│   ├── models/
│   │   ├── tenant.go           # 租户模型
│   │   ├── task.go             # 任务模型
│   │   ├── log.go              # 日志模型
│   │   └── notify.go           # 通知模型
│   ├── service/
│   │   ├── tenant_service.go   # 租户服务
│   │   ├── task_service.go     # 任务服务
│   │   ├── grab_engine.go      # 抢机引擎
│   │   ├── monitor_service.go  # 监控服务
│   │   ├── log_service.go      # 日志服务
│   │   └── notify/
│   │       ├── notifier.go     # 通知接口
│   │       ├── telegram.go     # Telegram
│   │       ├── dingtalk.go     # 钉钉
│   │       └── email.go        # 邮件
│   ├── oci/
│   │   ├── client.go           # OCI客户端
│   │   ├── compute.go          # 计算服务
│   │   ├── network.go          # 网络服务
│   │   └── identity.go         # 身份服务
│   └── pkg/
│       ├── logger/             # 日志工具
│       └── utils/              # 通用工具
├── web/                        # 前端资源 (可选)
│   └── dist/
├── configs/
│   ├── config.yaml             # 主配置文件
│   └── config.example.yaml     # 配置示例
├── data/
│   └── oci-manager.db          # SQLite数据库
├── logs/
│   └── app.log                 # 应用日志
├── scripts/
│   ├── build.sh                # 构建脚本
│   └── install.sh              # 安装脚本
├── Dockerfile
├── docker-compose.yml
├── Makefile
├── go.mod
├── go.sum
└── README.md
```

---

## 8. 开发计划

### 8.1 里程碑

| 阶段 | 内容 | 工期 | 状态 |
|------|------|------|------|
| **M1: 基础框架** | 项目结构、配置、日志 | 2天 | 🔄 待开始 |
| **M2: 租户管理** | 租户 CRUD、OCI 客户端 | 3天 | 📋 计划中 |
| **M3: 抢机核心** | 抢机引擎、任务管理 | 5天 | 📋 计划中 |
| **M4: 通知系统** | Telegram、钉钉、邮件 | 2天 | 📋 计划中 |
| **M5: 监控日志** | 系统监控、日志管理 | 2天 | 📋 计划中 |
| **M6: API 完善** | REST API、文档 | 2天 | 📋 计划中 |
| **M7: 测试优化** | 单元测试、性能优化 | 3天 | 📋 计划中 |

### 8.2 开发优先级

```
优先级 P0 (必须):
├── OCI 租户管理
├── 抢机引擎核心
└── Telegram 通知

优先级 P1 (重要):
├── 系统资源监控
├── 日志管理
├── 钉钉通知
└── REST API

优先级 P2 (可选):
├── Web UI
├── 邮件通知
└── Docker 部署
```

---

## 9. 部署方案

### 9.1 单机部署

```bash
# 下载并解压
wget https://github.com/your-repo/oci-manager-go/releases/latest/download/oci-manager-linux-amd64.tar.gz
tar -xzf oci-manager-linux-amd64.tar.gz

# 配置
cp config.example.yaml config.yaml
vim config.yaml

# 运行
./oci-manager serve
```

### 9.2 Docker 部署

```yaml
# docker-compose.yml
version: '3.8'
services:
  oci-manager:
    image: oci-manager-go:latest
    container_name: oci-manager
    ports:
      - "8080:8080"
    volumes:
      - ./config.yaml:/app/config.yaml
      - ./data:/app/data
      - ./keys:/app/keys:ro
    restart: unless-stopped
```

### 9.3 Systemd 服务

```ini
# /etc/systemd/system/oci-manager.service
[Unit]
Description=OCI Manager Go
After=network.target

[Service]
Type=simple
User=oci-manager
WorkingDirectory=/opt/oci-manager
ExecStart=/opt/oci-manager/oci-manager serve
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```

---

## 10. 配置文件示例

```yaml
# config.yaml
server:
  host: "0.0.0.0"
  port: 8080
  mode: "release"  # debug/release

database:
  driver: "sqlite"
  dsn: "./data/oci-manager.db"

log:
  level: "info"
  file: "./logs/app.log"
  max_size: 100      # MB
  max_backups: 3
  max_age: 7         # days

grab:
  default_interval: 5          # 默认轮询间隔(秒)
  min_interval: 1              # 最小间隔
  max_retry: 1000              # 最大重试次数
  concurrent_limit: 10         # 并发任务上限

notify:
  telegram:
    enabled: true
    bot_token: ""
    chat_id: ""
  dingtalk:
    enabled: false
    webhook: ""
    secret: ""
```

---

## 11. 确认事项

> [!IMPORTANT]
> 请确认以下事项是否满足您的需求：

### 功能确认

- [ ] 系统资源监控是否需要更多指标？
- [ ] OCI 租户管理功能是否完整？
- [ ] 抢机任务配置项是否足够？
- [ ] 需要支持哪些通知渠道？
- [ ] 是否需要 Web 管理界面？

### 技术确认

- [ ] 使用 SQLite 还是其他数据库？
- [ ] 是否需要 Docker 部署支持？
- [ ] 是否需要 CLI 命令行工具？
- [ ] 日志保留策略是否合适？

---

*文档创建时间: 2025-12-09 21:29 UTC*
