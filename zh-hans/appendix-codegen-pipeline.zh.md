# 附录：代码生成流水线（Code Generation Pipeline）

本文档以中文说明 Actor-RTC（actr）项目中的代码生成流水线，帮助你从 `.proto` 与 Actr.toml 出发，稳定地得到可直接使用、类型安全的生成代码与绑定。

## 架构总览

```
actr/crates/framework-protoc-codegen/   (生成器源码)
  ├── src/
  │   ├── bin/protoc-gen-actrframework.rs   (protoc 插件入口)
  │   ├── modern_generator.rs                (现行生成器实现)
  │   └── ...
  └── Cargo.toml

        ↓ (cargo build + install)

~/.cargo/bin/protoc-gen-actrframework     (已安装的可执行文件)

        ↓ (由 protoc 调用)

examples/*/src/generated/                 (生成代码输出目录)
```

要点：
- 代码生成由 `protoc` + 自定义插件完成，插件名称为 `protoc-gen-actrframework`。
- 生成器的唯一可信源码位于 `actr/crates/framework-protoc-codegen/`。
- CLI 命令 `actr gen` 会统一负责构建/安装插件并调用 `protoc` 进行生成。

## 关键组件

### 1) framework-protoc-codegen（唯一可信实现）
- 位置：`actr/crates/framework-protoc-codegen/`
- 组成：
  - `modern_generator.rs`：当前使用的生成器实现
  - `bin/protoc-gen-actrframework.rs`：protoc 插件入口
- 约定：所有生成逻辑的修改都应在这里进行。

### 2) actr CLI（构建与生成的编排者）
- 作用：通过 `actr gen` 命令统一完成构建、安装插件与调用 `protoc`。
- 典型过程：
  1. 构建 `protoc-gen-actrframework` 可执行文件
  2. 安装到 `~/.cargo/bin/`
  3. 调用 `protoc`，并将插件注入到生成流程
  4. 将生成产物输出到指定目录（通常为 `src/generated/`）

## 正确工作流

### 修改代码生成器时

1. 编辑生成器源码（例如修复模板或新增特性）
2. 重新构建并安装插件
3. 使用 `actr gen` 重新生成代码
4. 构建并验证示例或业务工程

示例命令（参考）：
```bash
# 1) 编辑生成器源码
vim actr/crates/framework-protoc-codegen/src/modern_generator.rs

# 2) 构建并安装插件
cargo install --path actr/crates/framework-protoc-codegen --bin protoc-gen-actrframework

# 3) 在示例项目中重新生成
cd examples/shell-actr-echo/server
actr gen proto src/generated

# 4) 构建与运行验证
cargo build
./start.sh
```

### 新增示例工程时
```bash
cd examples/my-new-example

# 1) 创建 Actr.toml
cat > Actr.toml <<'EOF'
[package]
name = "my-new-example"

[package.actr_type]
manufacturer = "acme"
name = "my-new-service"

[system.deployment]
realm = 0
EOF

# 2) 新建 proto 目录并编写 .proto
mkdir proto
vim proto/my_service.proto

# 3) 生成代码
actr gen proto src/generated

# 4) 编写业务逻辑
vim src/my_service.rs

# 5) 构建
cargo build
```

## 常见陷阱（请避免）

### 误区 1：修改错误的位置
- 不要在历史遗留/被删除的目录中修改（例如早期的重复目录）。
- 正确位置：`actr/crates/framework-protoc-codegen/`。

### 误区 2：忘记重新安装插件
```bash
# 错误示例：只改了源码就直接生成，仍会使用旧的二进制
vim crates/framework-protoc-codegen/src/modern_generator.rs
actr gen proto src/generated  # 仍在用旧版！
```
```bash
# 正确示例：先重新安装，再生成
vim crates/framework-protoc-codegen/src/modern_generator.rs
cargo install --path crates/framework-protoc-codegen --bin protoc-gen-actrframework
actr gen proto src/generated
```

### 误区 3：手动调用 protoc
```bash
# 不建议：手动拼接 --plugin 与 *_out 参数，容易出错
protoc --plugin=protoc-gen-actrframework=... --actrframework_out=...
```
```bash
# 推荐：使用 actr gen 命令
actr gen proto src/generated
```

## 验证方式

### 确认使用的是最新插件二进制
```bash
# 1) 查看插件路径
which protoc-gen-actrframework

# 2) 查看修改时间并对比源码更新时间
stat ~/.cargo/bin/protoc-gen-actrframework
stat actr/crates/framework-protoc-codegen/src/modern_generator.rs
```

### 检查生成代码是否包含预期变更
```bash
# 以 echo 示例为例
cd examples/shell-actr-echo/server
# 根据你的改动，检查生成文件中的目标片段
rg "fn actor_type" src/generated -n
```

## 历史与清理
- 过去曾存在重复的生成器目录，已清理合并为唯一实现：`actr/crates/framework-protoc-codegen/`。
- 常见问题包括：编辑源码后未重新安装插件、模板字段解析错误等。
- 相关清理提交包含：修复模板、移除重复目录等。

## 参考与延伸阅读
- `actr/crates/framework-protoc-codegen/README.md`：生成器实现细节
- `examples/*/README.md`：示例的具体使用方式
- 相关文档：
  - 《2.4-project-manifest-and-cli》：CLI 与工程清单
  - 《3.1-attach-and-codegen》：Attach 与生成物如何接入系统
