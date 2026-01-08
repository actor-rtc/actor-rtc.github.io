### proto file store

#### 解决方案概述
1. **新增表**：在 `ServiceRegistryStorage` 中添加 `service_specs` 表，专门存储 Proto 内容。
2. **数据流**：
   - **保存**：Actor 注册服务时，如果 `ServiceSpec` 存在，则提取 Proto 并保存到 `service_specs` 表。
   - **查询**：通过 `actr_type` 和 `service_fingerprint` 查询 Proto。
   - **清理**：使用保守的 TTL（默认 7 天），基于最后更新时间，避免影响兼容性协商。
3. **集成**：修改 `ServiceRegistryStorage` 的方法，确保与现有逻辑兼容。

#### 设计原则
- **访问驱动 TTL**：Proto 内容只有在被查询时才延长生命周期，未访问的条目会自然过期。
- **统一清理**：所有过期清理在一个方法中完成，减少复杂性。
- **性能考虑**：查询时更新时间戳，但使用异步任务避免阻塞。

#### 详细设计

##### 1. 新表结构 (`service_specs`)
```sql
CREATE TABLE IF NOT EXISTS service_specs (
    actr_type_manufacturer TEXT NOT NULL,
    actr_type_name TEXT NOT NULL,
    service_fingerprint TEXT NOT NULL,
    proto_content BLOB NOT NULL,
    last_accessed INTEGER NOT NULL,  -- 最后访问时间（Unix timestamp）
    expires_at INTEGER NOT NULL,     -- 过期时间（last_accessed + TTL）
    PRIMARY KEY (actr_type_manufacturer, actr_type_name, service_fingerprint)
);

CREATE INDEX IF NOT EXISTS idx_service_specs_expires_at ON service_specs(expires_at);
CREATE INDEX IF NOT EXISTS idx_service_specs_last_accessed ON service_specs(last_accessed);
```

- **字段说明**：
  - `last_accessed`：记录最后一次查询的时间戳。
  - `expires_at`：基于 `last_accessed` 计算的过期时间，每次访问时更新为 `last_accessed + TTL`。

##### 2. 修改 `ServiceRegistryStorage` 结构体
- 添加新字段：`proto_ttl_secs: u64`（默认 604800 秒，即 7 天）。
- 在 `new()` 方法中初始化。

##### 3. 新增/修改方法
- **`save_proto_spec(actr_type: &ActrType, service_spec: &ServiceSpec) -> Result<()>`**：
  - 插入或更新 `service_specs` 表。
  - 设置 `last_accessed` 和 `expires_at` 为当前时间 + TTL。

- **`get_proto_by_fingerprint(actr_type: &ActrType, fingerprint: &str) -> Result<Option<Vec<u8>>>`**：
  - 查询匹配的 Proto。
  - 如果找到，更新 `last_accessed` 和 `expires_at`（异步执行，避免阻塞查询）。
  - 如果过期，返回 `None`（但不立即删除）。

- **`cleanup_expired_proto_specs() -> Result<u64>`**：（可选，保留但不单独调用，与 `cleanup_expired()` 合并）。

##### 4. 集成到现有流程
- **保存时机**：在 `save_service()` 中，如果 `service.service_spec` 存在，则调用 `save_proto_spec()`。
- **查询时机**：在服务发现或兼容性检查时，通过 `get_proto_by_fingerprint()` 获取 Proto。
- **清理时机**：在 `cleanup_expired()` 中添加对 `service_specs` 表的清理。

##### 5. 清理策略
- **统一清理**：在 `cleanup_expired()` 方法中，同时删除两个表的过期条目。
- **TTL 默认**：7 天（可配置），基于访问延长。
- **性能**：清理是批量操作，不影响正常查询。

##### 6. 代码实现示例（修改部分）
```rust
impl ServiceRegistryStorage {
    // 新增方法
    pub async fn save_proto_spec(&self, actr_type: &ActrType, service_spec: &ServiceSpec) -> Result<()> {
        let fingerprint = extract_fingerprint(service_spec)?;
        let proto_content = compress_proto(service_spec.proto_content.clone())?;
        let now = current_timestamp();
        let expires_at = now + self.proto_ttl_secs;

        sqlx::query(
            r#"
            INSERT INTO service_specs (actr_type_manufacturer, actr_type_name, service_fingerprint, proto_content, last_accessed, expires_at)
            VALUES (?1, ?2, ?3, ?4, ?5, ?6)
            ON CONFLICT(actr_type_manufacturer, actr_type_name, service_fingerprint)
            DO UPDATE SET proto_content = excluded.proto_content, last_accessed = excluded.last_accessed, expires_at = excluded.expires_at
            "#,
        )
        .bind(&actr_type.manufacturer)
        .bind(&actr_type.name)
        .bind(&fingerprint)
        .bind(&proto_content)
        .bind(now as i64)
        .bind(expires_at as i64)
        .execute(&self.pool)
        .await?;
        Ok(())
    }

    pub async fn get_proto_by_fingerprint(&self, actr_type: &ActrType, fingerprint: &str) -> Result<Option<Vec<u8>>> {
        let now = current_timestamp();
        
        // 先查询
        let row = sqlx::query(
            r#"SELECT proto_content FROM service_specs WHERE actr_type_manufacturer = ?1 AND actr_type_name = ?2 AND service_fingerprint = ?3"#,
        )
        .bind(&actr_type.manufacturer)
        .bind(&actr_type.name)
        .bind(fingerprint)
        .fetch_optional(&self.pool)
        .await?;

        if let Some(row) = row {
            let proto_content = decompress_proto(row.get("proto_content"))?;
            
            // 异步更新访问时间和过期时间
            let pool_clone = self.pool.clone();
            let manufacturer = actr_type.manufacturer.clone();
            let name = actr_type.name.clone();
            let fingerprint = fingerprint.to_string();
            let ttl = self.proto_ttl_secs;
            tokio::spawn(async move {
                let new_expires_at = now + ttl;
                let _ = sqlx::query(
                    r#"UPDATE service_specs SET last_accessed = ?1, expires_at = ?2 WHERE actr_type_manufacturer = ?3 AND actr_type_name = ?4 AND service_fingerprint = ?5"#,
                )
                .bind(now as i64)
                .bind(new_expires_at as i64)
                .bind(&manufacturer)
                .bind(&name)
                .bind(&fingerprint)
                .execute(&pool_clone)
                .await;
            });
            
            Ok(Some(proto_content))
        } else {
            Ok(None)
        }
    }

    // 修改 cleanup_expired() 以同时清理 service_specs
    pub async fn cleanup_expired(&self) -> Result<u64> {
        let now = current_timestamp();

        // 清理 service_registry
        let registry_deleted = sqlx::query("DELETE FROM service_registry WHERE expires_at <= ?1")
            .bind(now as i64)
            .execute(&self.pool)
            .await?
            .rows_affected();

        // 清理 service_specs
        let specs_deleted = sqlx::query("DELETE FROM service_specs WHERE expires_at <= ?1")
            .bind(now as i64)
            .execute(&self.pool)
            .await?
            .rows_affected();

        let total_deleted = registry_deleted + specs_deleted;
        if total_deleted > 0 {
            info!("Cleaned up {} expired entries from cache (registry: {}, specs: {})", total_deleted, registry_deleted, specs_deleted);
        }

        Ok(total_deleted)
    }
}
```
