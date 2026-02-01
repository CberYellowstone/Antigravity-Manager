# API 反代流量消耗机制分析

## 项目概述

Antigravity Tools 是一个基于 Tauri 的桌面应用,提供 Google AI (Gemini/Claude) 的 API 反向代理服务。该项目通过管理多个 Google 账号,实现智能的流量分配和配额管理。

---

## 一、核心架构

### 1.1 技术栈
- **前端**: React + TypeScript + Vite
- **后端**: Rust (Tauri) + Axum (Web 框架)
- **数据存储**: 
  - SQLite (流量统计、代理日志、IP 监控)
  - JSON 文件 (账号信息、配置)

### 1.2 核心模块

```
src-tauri/src/
├── proxy/
│   ├── token_manager.rs      # Token 池管理与账号调度
│   ├── server.rs              # Axum HTTP 服务器
│   ├── rate_limit.rs          # 限流跟踪器
│   └── sticky_config.rs       # 会话粘性配置
├── modules/
│   ├── account.rs             # 账号 CRUD 操作
│   ├── quota.rs               # 配额查询与保护
│   └── token_stats.rs         # Token 使用统计
└── models/
    ├── account.rs             # 账号数据模型
    ├── quota.rs               # 配额数据模型
    └── token.rs               # Token 数据模型
```

---

## 二、流量消耗机制详解

### 2.1 账号数据结构

每个账号包含以下核心信息:

```rust
pub struct Account {
    pub id: String,                          // 账号唯一 ID
    pub email: String,                       // Google 邮箱
    pub token: TokenData,                    // OAuth Token 信息
    pub quota: Option<QuotaData>,            // 配额信息
    pub disabled: bool,                      // 是否全局禁用
    pub proxy_disabled: bool,                // 是否禁用代理功能
    pub protected_models: HashSet<String>,   // 受配额保护的模型列表
    pub created_at: i64,
    pub last_used: i64,
}
```

**Token 数据**:
```rust
pub struct TokenData {
    pub access_token: String,      // 访问令牌
    pub refresh_token: String,     // 刷新令牌
    pub expires_in: i64,           // 过期时间(秒)
    pub expiry_timestamp: i64,     // 过期时间戳
    pub project_id: Option<String>, // Google Cloud 项目 ID
}
```

**配额数据**:
```rust
pub struct QuotaData {
    pub models: Vec<ModelQuota>,           // 各模型配额
    pub subscription_tier: Option<String>, // 订阅等级 (FREE/PRO/ULTRA)
    pub is_forbidden: bool,                // 是否被禁止访问
}

pub struct ModelQuota {
    pub name: String,        // 模型名称
    pub percentage: i32,     // 剩余配额百分比 (0-100)
    pub reset_time: String,  // 配额重置时间
}
```

### 2.2 Token 池管理 (TokenManager)

**核心职责**:
1. 从磁盘加载所有可用账号
2. 智能选择账号分配给请求
3. 自动刷新过期的 Token
4. 跟踪限流状态和配额保护

**关键字段**:
```rust
pub struct TokenManager {
    tokens: Arc<DashMap<String, ProxyToken>>,  // 账号 ID -> Token 映射
    current_index: Arc<AtomicUsize>,           // 轮询索引
    last_used_account: Arc<Mutex<Option<(String, Instant)>>>, // 60s 锁定机制
    rate_limit_tracker: Arc<RateLimitTracker>, // 限流跟踪器
    session_accounts: Arc<DashMap<String, String>>, // 会话 -> 账号绑定
    preferred_account_id: Arc<RwLock<Option<String>>>, // 固定账号模式
    health_scores: Arc<DashMap<String, f32>>,  // 账号健康分数
}
```

### 2.3 账号调度策略

#### 2.3.1 优先级排序

在每次请求时,账号按以下优先级排序:

```rust
// 1. 订阅等级优先级: ULTRA > PRO > FREE
// 2. 同等级内按剩余配额降序排序
// 3. 配额相同时按健康分数降序排序

tokens_snapshot.sort_by(|a, b| {
    // 第一优先级: 订阅等级
    let tier_priority = |tier: &Option<String>| match tier.as_deref() {
        Some("ULTRA") => 0,
        Some("PRO") => 1,
        Some("FREE") => 2,
        _ => 3,
    };
    
    let tier_cmp = tier_priority(&a.subscription_tier)
        .cmp(&tier_priority(&b.subscription_tier));
    
    if tier_cmp != std::cmp::Ordering::Equal {
        return tier_cmp;
    }
    
    // 第二优先级: 剩余配额百分比 (高优先)
    let quota_a = a.remaining_quota.unwrap_or(0);
    let quota_b = b.remaining_quota.unwrap_or(0);
    let quota_cmp = quota_b.cmp(&quota_a);
    
    if quota_cmp != std::cmp::Ordering::Equal {
        return quota_cmp;
    }
    
    // 第三优先级: 健康分数 (高优先)
    b.health_score.partial_cmp(&a.health_score)
        .unwrap_or(std::cmp::Ordering::Equal)
});
```

**设计理由**:
- **ULTRA/PRO 优先**: 这些账号配额重置快,优先消耗可最大化总体可用性
- **高配额优先**: 避免低配额账号被用光,保持账号池的健康度
- **健康分数**: 根据历史成功率动态调整优先级

#### 2.3.2 调度模式

系统支持三种调度模式:

1. **CacheFirst (缓存优先)**
   - 启用 60s 全局锁定
   - 启用会话粘性 (Session Sticky)
   - 适合需要上下文连续性的场景

2. **Balance (平衡模式)**
   - 启用会话粘性
   - 不启用 60s 锁定
   - 兼顾性能和连续性

3. **PerformanceFirst (性能优先)**
   - 纯轮询模式
   - 不启用任何锁定机制
   - 最大化并发性能

#### 2.3.3 固定账号模式 (FIX #820)

支持指定优先使用某个账号:

```rust
// 如果设置了 preferred_account_id
if let Some(ref pref_id) = preferred_id {
    if let Some(preferred_token) = tokens_snapshot.iter().find(|t| &t.account_id == pref_id) {
        // 检查账号是否可用 (未限流、未被配额保护)
        if !is_rate_limited && !is_quota_protected {
            // 直接使用优先账号,跳过轮询逻辑
            return Ok((token.access_token, project_id, token.email, 0));
        }
    }
}
```

### 2.4 配额保护机制

#### 2.4.1 模型级配额保护 (Issue #621)

当某个模型的配额低于阈值时,自动将该模型加入账号的 `protected_models` 列表:

```rust
// 配置示例
{
  "quota_protection": {
    "enabled": true,
    "threshold_percentage": 10,  // 低于 10% 触发保护
    "monitored_models": [
      "gemini-3-flash",
      "claude-sonnet-4-5",
      "gemini-3-pro-high"
    ]
  }
}
```

**保护流程**:
1. 加载账号时检查每个模型的配额
2. 如果 `percentage <= threshold_percentage`,将模型名加入 `protected_models`
3. 在 `get_token()` 时,跳过包含目标模型的账号
4. 当配额恢复时,自动从 `protected_models` 移除

**优势**:
- 精细化保护,不会因为单个模型配额低而禁用整个账号
- 其他模型仍可正常使用该账号

#### 2.4.2 配额恢复机制

```rust
// 当配额恢复到阈值以上时
if percentage > threshold {
    // 从 protected_models 中移除该模型
    account_json["protected_models"]
        .as_array_mut()
        .unwrap()
        .retain(|m| m.as_str() != Some(model_name));
    
    // 保存到磁盘
    std::fs::write(account_path, serde_json::to_string_pretty(account_json).unwrap())?;
}
```

### 2.5 限流跟踪 (RateLimitTracker)

**功能**:
- 记录每个账号的限流状态 (429 错误)
- 记录每个账号的 5xx 错误 (熔断机制)
- 自动清理过期记录 (每 15 秒)

**数据结构**:
```rust
pub struct RateLimitTracker {
    // account_id -> (reset_timestamp, model_name)
    records: Arc<DashMap<String, (i64, Option<String>)>>,
    
    // account_id -> 5xx 错误计数
    error_counts: Arc<DashMap<String, u32>>,
}
```

**使用示例**:
```rust
// 记录限流
tracker.record_rate_limit(&account_id, Some(&model_name), 60); // 60 秒后重置

// 检查是否限流
if tracker.is_rate_limited(&account_id, Some(&model_name)) {
    // 跳过该账号
}

// 清除限流记录
tracker.clear(&account_id);
```

### 2.6 会话粘性 (Session Sticky)

**目的**: 保持同一会话的请求使用同一账号,避免上下文丢失

**实现**:
```rust
// 1. 首次请求时绑定会话与账号
if let Some(sid) = session_id {
    self.session_accounts.insert(sid.to_string(), candidate.account_id.clone());
}

// 2. 后续请求复用绑定的账号
if let Some(bound_id) = self.session_accounts.get(sid).map(|v| v.clone()) {
    if let Some(bound_token) = tokens_snapshot.iter().find(|t| t.account_id == bound_id) {
        // 检查账号是否仍然可用
        if !is_rate_limited && !is_quota_protected {
            target_token = Some(bound_token.clone());
        } else {
            // 账号不可用,解绑并切换
            self.session_accounts.remove(sid);
        }
    }
}
```

### 2.7 Token 自动刷新

**触发条件**: Token 距离过期时间小于 5 分钟

```rust
let now = chrono::Utc::now().timestamp();
if now >= token.timestamp - 300 {  // 提前 5 分钟刷新
    match crate::modules::oauth::refresh_access_token(&token.refresh_token).await {
        Ok(token_response) => {
            // 更新内存中的 Token
            token.access_token = token_response.access_token.clone();
            token.expires_in = token_response.expires_in;
            token.timestamp = now + token_response.expires_in;
            
            // 同步到 DashMap
            if let Some(mut entry) = self.tokens.get_mut(&token.account_id) {
                entry.access_token = token.access_token.clone();
                entry.expires_in = token.expires_in;
                entry.timestamp = token.timestamp;
            }
            
            // 持久化到磁盘
            self.save_refreshed_token(&token.account_id, &token_response).await?;
        }
        Err(e) if e.contains("invalid_grant") => {
            // Refresh Token 已失效,禁用账号
            self.disable_account(&token.account_id, &format!("invalid_grant: {}", e)).await?;
            self.tokens.remove(&token.account_id);
        }
    }
}
```

---

## 三、多账号消耗实现

### 3.1 账号加载流程

```rust
pub async fn load_accounts(&self) -> Result<usize, String> {
    let accounts_dir = self.data_dir.join("accounts");
    
    // 清空现有 Token 池
    self.tokens.clear();
    self.current_index.store(0, Ordering::SeqCst);
    
    // 遍历账号目录
    for entry in std::fs::read_dir(&accounts_dir)? {
        let path = entry?.path();
        
        // 只处理 .json 文件
        if path.extension().and_then(|s| s.to_str()) != Some("json") {
            continue;
        }
        
        // 加载单个账号
        match self.load_single_account(&path).await {
            Ok(Some(token)) => {
                self.tokens.insert(token.account_id.clone(), token);
                count += 1;
            }
            Ok(None) => {
                // 跳过禁用或配额保护的账号
            }
            Err(e) => {
                tracing::debug!("加载账号失败 {:?}: {}", path, e);
            }
        }
    }
    
    Ok(count)
}
```

### 3.2 账号过滤逻辑

在 `load_single_account()` 中,以下账号会被跳过:

1. **全局禁用** (`disabled: true`)
2. **代理禁用** (`proxy_disabled: true`)
3. **配额保护触发** (所有监控模型都低于阈值)

```rust
async fn load_single_account(&self, path: &PathBuf) -> Result<Option<ProxyToken>, String> {
    let content = std::fs::read_to_string(path)?;
    let mut account: serde_json::Value = serde_json::from_str(&content)?;
    
    // 1. 检查全局禁用
    if account.get("disabled").and_then(|v| v.as_bool()).unwrap_or(false) {
        return Ok(None);
    }
    
    // 2. 检查配额保护
    if self.check_and_protect_quota(&mut account, path).await {
        return Ok(None);
    }
    
    // 3. 检查代理禁用
    if account.get("proxy_disabled").and_then(|v| v.as_bool()).unwrap_or(false) {
        return Ok(None);
    }
    
    // 4. 提取账号信息并加载
    Ok(Some(ProxyToken {
        account_id: account["id"].as_str()?.to_string(),
        email: account["email"].as_str()?.to_string(),
        access_token: account["token"]["access_token"].as_str()?.to_string(),
        refresh_token: account["token"]["refresh_token"].as_str()?.to_string(),
        // ... 其他字段
    }))
}
```

### 3.3 请求分配流程

**完整流程图**:

```
客户端请求
    ↓
[1] 解析请求参数 (model, session_id)
    ↓
[2] 固定账号模式检查
    ├─ 有 preferred_account_id → 优先使用
    └─ 无 → 继续
    ↓
[3] 会话粘性检查 (如果有 session_id)
    ├─ 已绑定账号 → 检查可用性
    │   ├─ 可用 → 复用
    │   └─ 不可用 → 解绑并继续
    └─ 未绑定 → 继续
    ↓
[4] 60s 全局锁定检查 (CacheFirst 模式)
    ├─ 距上次使用 < 60s → 复用上次账号
    └─ 否则 → 继续
    ↓
[5] 轮询选择账号
    ├─ 按优先级排序
    ├─ 跳过已尝试的账号
    ├─ 跳过限流的账号
    ├─ 跳过配额保护的账号
    └─ 选择第一个可用账号
    ↓
[6] Token 刷新检查
    ├─ 距过期 < 5 分钟 → 刷新 Token
    └─ 否则 → 继续
    ↓
[7] 返回 (access_token, project_id, email)
    ↓
[8] 代理请求到 Google API
    ↓
[9] 记录使用统计
    ├─ token_stats.db (Token 消耗)
    ├─ proxy_logs.db (请求日志)
    └─ 更新 last_used 时间戳
```

### 3.4 流量统计

#### 3.4.1 数据库结构

**token_usage 表** (原始记录):
```sql
CREATE TABLE token_usage (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    timestamp INTEGER NOT NULL,
    account_email TEXT NOT NULL,
    model TEXT NOT NULL,
    input_tokens INTEGER NOT NULL DEFAULT 0,
    output_tokens INTEGER NOT NULL DEFAULT 0,
    total_tokens INTEGER NOT NULL DEFAULT 0
);
```

**token_stats_hourly 表** (小时聚合):
```sql
CREATE TABLE token_stats_hourly (
    hour_bucket TEXT NOT NULL,           -- "2024-01-15 14:00"
    account_email TEXT NOT NULL,
    total_input_tokens INTEGER NOT NULL DEFAULT 0,
    total_output_tokens INTEGER NOT NULL DEFAULT 0,
    total_tokens INTEGER NOT NULL DEFAULT 0,
    request_count INTEGER NOT NULL DEFAULT 0,
    PRIMARY KEY (hour_bucket, account_email)
);
```

#### 3.4.2 记录流程

```rust
// 在请求完成后记录
pub fn record_usage(
    account_email: &str,
    model: &str,
    input_tokens: u32,
    output_tokens: u32,
) -> Result<(), String> {
    let conn = connect_db()?;
    let timestamp = chrono::Utc::now().timestamp();
    let total_tokens = input_tokens + output_tokens;
    
    // 1. 插入原始记录
    conn.execute(
        "INSERT INTO token_usage (timestamp, account_email, model, input_tokens, output_tokens, total_tokens)
         VALUES (?1, ?2, ?3, ?4, ?5, ?6)",
        params![timestamp, account_email, model, input_tokens, output_tokens, total_tokens],
    )?;
    
    // 2. 更新小时聚合 (使用 UPSERT)
    let hour_bucket = chrono::Utc::now().format("%Y-%m-%d %H:00").to_string();
    conn.execute(
        "INSERT INTO token_stats_hourly (hour_bucket, account_email, total_input_tokens, total_output_tokens, total_tokens, request_count)
         VALUES (?1, ?2, ?3, ?4, ?5, 1)
         ON CONFLICT(hour_bucket, account_email) DO UPDATE SET
            total_input_tokens = total_input_tokens + ?3,
            total_output_tokens = total_output_tokens + ?4,
            total_tokens = total_tokens + ?5,
            request_count = request_count + 1",
        params![hour_bucket, account_email, input_tokens, output_tokens, total_tokens],
    )?;
    
    Ok(())
}
```

#### 3.4.3 统计查询

**按账号统计**:
```rust
pub fn get_account_stats(hours: i64) -> Result<Vec<AccountTokenStats>, String> {
    let cutoff = chrono::Utc::now() - chrono::Duration::hours(hours);
    let cutoff_bucket = cutoff.format("%Y-%m-%d %H:00").to_string();
    
    let mut stmt = conn.prepare(
        "SELECT account_email,
            SUM(total_input_tokens) as input, 
            SUM(total_output_tokens) as output,
            SUM(total_tokens) as total,
            SUM(request_count) as count
         FROM token_stats_hourly 
         WHERE hour_bucket >= ?1
         GROUP BY account_email
         ORDER BY total DESC"
    )?;
    
    // 返回每个账号的统计数据
}
```

**按模型统计**:
```rust
pub fn get_model_stats(hours: i64) -> Result<Vec<ModelTokenStats>, String> {
    let cutoff = chrono::Utc::now().timestamp() - (hours * 3600);
    
    let mut stmt = conn.prepare(
        "SELECT model,
            SUM(input_tokens) as input,
            SUM(output_tokens) as output,
            SUM(total_tokens) as total,
            COUNT(*) as count
         FROM token_usage
         WHERE timestamp >= ?1
         GROUP BY model
         ORDER BY total DESC"
    )?;
    
    // 返回每个模型的统计数据
}
```

### 3.5 配额查询与更新

#### 3.5.1 配额 API

Google 提供的配额查询接口:

```rust
const QUOTA_API_URL: &str = "https://daily-cloudcode-pa.sandbox.googleapis.com/v1internal:fetchAvailableModels";

pub async fn fetch_quota(access_token: &str, email: &str) -> Result<(QuotaData, Option<String>), AppError> {
    // 1. 获取 project_id 和订阅等级
    let (project_id, subscription_tier) = fetch_project_id(access_token, email).await;
    
    // 2. 查询配额
    let client = create_client();
    let payload = json!({
        "project": project_id.as_deref().unwrap_or("bamboo-precept-lgxtn")
    });
    
    let response = client
        .post(QUOTA_API_URL)
        .bearer_auth(access_token)
        .json(&payload)
        .send()
        .await?;
    
    // 3. 解析响应
    let quota_response: QuotaResponse = response.json().await?;
    let mut quota_data = QuotaData::new();
    
    for (name, info) in quota_response.models {
        if let Some(quota_info) = info.quota_info {
            let percentage = quota_info.remaining_fraction
                .map(|f| (f * 100.0) as i32)
                .unwrap_or(0);
            
            let reset_time = quota_info.reset_time.unwrap_or_default();
            
            // 只保留关心的模型
            if name.contains("gemini") || name.contains("claude") {
                quota_data.add_model(name, percentage, reset_time);
            }
        }
    }
    
    quota_data.subscription_tier = subscription_tier;
    Ok((quota_data, project_id))
}
```

#### 3.5.2 批量刷新配额

```rust
pub async fn refresh_all_quotas_logic() -> Result<String, String> {
    let accounts = list_accounts()?;
    let mut success = 0;
    let mut failed = 0;
    
    // 并发查询 (批次大小 5)
    for batch in accounts.chunks(5) {
        let mut handles = Vec::new();
        
        for account in batch {
            let account = account.clone();
            let handle = tokio::spawn(async move {
                // 刷新 Token (如果需要)
                let token = ensure_fresh_token(&account.token).await?;
                
                // 查询配额
                let (quota, project_id) = fetch_quota(&token.access_token, &account.email).await?;
                
                // 更新账号文件
                let mut updated_account = account.clone();
                updated_account.quota = Some(quota);
                if let Some(pid) = project_id {
                    updated_account.token.project_id = Some(pid);
                }
                save_account(&updated_account)?;
                
                Ok::<_, String>(())
            });
            handles.push(handle);
        }
        
        for handle in handles {
            match handle.await {
                Ok(Ok(_)) => success += 1,
                _ => failed += 1,
            }
        }
    }
    
    Ok(format!("Refreshed {}/{} accounts", success, success + failed))
}
```

---

## 四、关键优化点

### 4.1 性能优化

1. **DashMap 无锁并发**: 使用 `DashMap` 替代 `Mutex<HashMap>`,提升并发性能
2. **预排序**: 在请求前对账号排序,避免每次请求时重复排序
3. **批量操作**: 配额刷新等操作使用批量并发处理
4. **连接池**: HTTP 客户端使用连接池复用连接

### 4.2 可靠性优化

1. **Token 自动刷新**: 提前 5 分钟刷新,避免请求时 Token 过期
2. **限流自动跳过**: 主动检测限流状态,避免重复请求
3. **配额保护**: 低配额账号自动保护,避免耗尽
4. **熔断机制**: 5xx 错误累积到阈值后暂时禁用账号
5. **乐观重置**: 所有账号限流时,等待最短时间后自动重试

### 4.3 用户体验优化

1. **会话粘性**: 保持对话连续性
2. **智能调度**: 根据订阅等级和配额智能分配
3. **实时统计**: 提供详细的使用统计和趋势分析
4. **配额可视化**: 实时显示每个账号的配额状态

---

## 五、典型使用场景

### 5.1 场景一: 高并发 API 调用

**需求**: 100 个并发请求,需要快速响应

**配置**:
- 调度模式: `PerformanceFirst`
- 账号数量: 10+ (建议 ULTRA/PRO 账号)
- 配额保护: 启用,阈值 10%

**流程**:
1. 请求到达,纯轮询选择账号
2. 跳过限流和配额保护的账号
3. 按订阅等级和配额优先级分配
4. 记录使用统计

**结果**: 
- 平均响应时间: < 500ms
- 账号利用率: 均衡分布
- 配额消耗: 优先消耗 ULTRA/PRO

### 5.2 场景二: 长对话场景

**需求**: 保持对话上下文,同一会话使用同一账号

**配置**:
- 调度模式: `CacheFirst` 或 `Balance`
- 会话粘性: 启用
- 60s 锁定: 启用 (CacheFirst)

**流程**:
1. 首次请求分配账号 A,绑定 session_id
2. 后续请求携带相同 session_id,复用账号 A
3. 如果账号 A 限流,自动解绑并切换到账号 B
4. 新的 session_id 绑定账号 B

**结果**:
- 对话连续性: 100%
- 切换次数: 最小化
- 用户体验: 无感知切换

### 5.3 场景三: 配额保护场景

**需求**: 避免 FREE 账号配额耗尽

**配置**:
- 配额保护: 启用,阈值 10%
- 监控模型: `["gemini-3-flash", "claude-sonnet-4-5"]`
- 自动刷新: 每小时

**流程**:
1. 定时刷新所有账号配额
2. 检测到账号 A 的 `gemini-3-flash` 配额 < 10%
3. 将 `gemini-3-flash` 加入账号 A 的 `protected_models`
4. 后续请求 `gemini-3-flash` 时跳过账号 A
5. 其他模型仍可使用账号 A
6. 配额恢复后自动移除保护

**结果**:
- 配额利用率: 最大化
- 账号可用性: 精细化保护
- 恢复速度: 自动化

---

## 六、总结

### 6.1 核心优势

1. **智能调度**: 多维度优先级排序,最大化资源利用
2. **精细化保护**: 模型级配额保护,避免过度禁用
3. **高可用性**: 限流跟踪、熔断机制、乐观重置
4. **灵活配置**: 支持多种调度模式和固定账号
5. **完整统计**: 详细的使用统计和趋势分析

### 6.2 适用场景

- ✅ 多账号管理和负载均衡
- ✅ 高并发 API 代理
- ✅ 配额精细化管理
- ✅ 长对话场景
- ✅ 企业级 API 网关

### 6.3 未来优化方向

1. **机器学习调度**: 基于历史数据预测账号可用性
2. **动态阈值**: 根据使用模式自动调整配额保护阈值
3. **跨实例同步**: 支持多实例部署时的账号池同步
4. **成本优化**: 根据订阅成本优化账号使用策略

---

## 附录: 配置示例

### A.1 应用配置 (app_config.json)

```json
{
  "proxy": {
    "enabled": true,
    "host": "127.0.0.1",
    "port": 8045,
    "scheduling_mode": "CacheFirst",
    "request_timeout": 60
  },
  "quota_protection": {
    "enabled": true,
    "threshold_percentage": 10,
    "monitored_models": [
      "gemini-3-flash",
      "claude-sonnet-4-5",
      "gemini-3-pro-high",
      "gemini-3-pro-image"
    ]
  },
  "circuit_breaker": {
    "enabled": true,
    "error_threshold": 5,
    "timeout_seconds": 300
  }
}
```

### A.2 账号文件示例 (accounts/xxx.json)

```json
{
  "id": "abc123",
  "email": "user@gmail.com",
  "name": "User Name",
  "token": {
    "access_token": "ya29.xxx",
    "refresh_token": "1//xxx",
    "expires_in": 3599,
    "expiry_timestamp": 1706000000,
    "token_type": "Bearer",
    "project_id": "bamboo-precept-lgxtn"
  },
  "quota": {
    "models": [
      {
        "name": "gemini-3-flash",
        "percentage": 85,
        "reset_time": "2024-01-15T00:00:00Z"
      },
      {
        "name": "claude-sonnet-4-5",
        "percentage": 5,
        "reset_time": "2024-01-15T00:00:00Z"
      }
    ],
    "subscription_tier": "PRO",
    "last_updated": 1706000000,
    "is_forbidden": false
  },
  "disabled": false,
  "proxy_disabled": false,
  "protected_models": ["claude-sonnet-4-5"],
  "created_at": 1705000000,
  "last_used": 1706000000
}
```

---

**文档版本**: 1.0  
**最后更新**: 2026-01-30  
**作者**: Antigravity Analysis
