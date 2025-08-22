# Claude Code Proxy - 二进制打包指南

本文档详细介绍如何将 Claude Code Proxy Python 项目打包成二进制可执行程序。

## 打包概述

我们使用 PyInstaller 将 Python FastAPI 应用打包成独立运行的二进制程序，无需目标系统安装 Python 环境。

### 生成的二进制文件

- **目录版本**: `dist/claude-code-proxy/` (6.4MB 可执行文件 + 49MB 依赖库)
- **单文件版本**: `dist/claude-code-proxy-single` (24MB 独立可执行文件)

## 环境要求

### 开发环境

- Python 3.9+
- uv 包管理器
- 操作系统: Linux/macOS/Windows

### 安装打包工具

```bash
# 使用 uv 安装 PyInstaller
uv add --dev pyinstaller
```

## 打包步骤

### 1. 目录版本打包（推荐用于开发/测试）

```bash
# 使用自定义 spec 文件
uv run pyinstaller claude-proxy.spec
```

### 2. 单文件版本打包（推荐用于部署）

```bash
# 快速单文件打包
uv run pyinstaller --onefile --name claude-code-proxy-single src/main.py
```

### 3. 自定义配置打包

```bash
# 包含特定模块
uv run pyinstaller --onefile --hidden-import=src.api.endpoints src/main.py

# 包含数据文件
uv run pyinstaller --onefile --add-data="src:src" src/main.py
```

## PyInstaller 配置详解

### claude-proxy.spec 配置

```python
# 主要配置项说明
a = Analysis(
    ['src/main.py'],           # 入口文件
    pathex=['.'],              # 搜索路径
    binaries=[],               # 二进制文件
    datas=[('src', 'src')],    # 数据文件
    hiddenimports=[...],       # 隐藏导入模块
)

# 包含的关键隐藏导入
hiddenimports=[
    'src.api.endpoints',
    'src.core.config',
    'src.core.client',
    'uvicorn.logging',
    'uvicorn.loops.auto',
    'fastapi.openapi.utils',
    'pydantic.v1',
]
```

## 跨平台打包

### Linux (当前环境)

```bash
# 在 Linux 系统上构建
uv run pyinstaller --onefile src/main.py
# 输出: claude-code-proxy-single (Linux ELF)
```

### Windows

```bash
# 在 Windows 系统上构建
pyinstaller --onefile src/main.py
# 输出: claude-code-proxy-single.exe
```

### macOS

```bash
# 在 macOS 系统上构建
pyinstaller --onefile src/main.py
# 输出: claude-code-proxy-single (macOS 可执行文件)
```

## 使用方法

### 基本运行

```bash
# 目录版本
./dist/claude-code-proxy/claude-code-proxy

# 单文件版本
./dist/claude-code-proxy-single
```

### 配置环境变量

```bash
# 设置必需的环境变量
export OPENAI_API_KEY="your-api-key-here"

# 可选环境变量
export ANTHROPIC_API_KEY="anthropic-key"
export HOST="0.0.0.0"
export PORT="8082"
export LOG_LEVEL="INFO"
```

### 查看帮助

```bash
./claude-code-proxy --help
```

### 启动服务器

```bash
# 前台运行
OPENAI_API_KEY="your-key" ./claude-code-proxy

# 后台运行
OPENAI_API_KEY="your-key" nohup ./claude-code-proxy &
```

## 部署说明

### 系统要求

- **内存**: 至少 512MB RAM
- **磁盘**: 至少 100MB 可用空间
- **网络**: 能够访问 OpenAI API
- **权限**: 无需 root 权限

### 生产环境部署

```bash
# 1. 上传二进制文件到服务器
scp claude-code-proxy-single user@server:/opt/claude-proxy/

# 2. 设置环境变量
cat > /opt/claude-proxy/.env << EOF
OPENAI_API_KEY=sk-your-actual-key
ANTHROPIC_API_KEY=your-anthropic-key
HOST=0.0.0.0
PORT=8082
LOG_LEVEL=WARNING
EOF

# 3. 创建系统服务 (systemd)
cat > /etc/systemd/system/claude-proxy.service << EOF
[Unit]
Description=Claude Code Proxy
After=network.target

[Service]
Type=simple
User=www-data
WorkingDirectory=/opt/claude-proxy
EnvironmentFile=/opt/claude-proxy/.env
ExecStart=/opt/claude-proxy/claude-code-proxy-single
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

# 4. 启动服务
systemctl daemon-reload
systemctl start claude-proxy
systemctl enable claude-proxy
```

### Docker 容器化部署

```dockerfile
FROM alpine:latest
RUN apk add --no-cache libstdc++
COPY claude-code-proxy-single /usr/local/bin/claude-code-proxy
EXPOSE 8082
CMD ["claude-code-proxy"]
```

## 故障排除

### 常见问题

1. **权限错误**

   ```bash
   # 添加执行权限
   chmod +x claude-code-proxy
   ```

2. **缺少依赖错误**

   - 重新生成 spec 文件包含更多隐藏导入
   - 检查 PyInstaller 版本是否兼容

3. **环境变量未设置**

   ```bash
   # 检查环境变量
   env | grep -E "(OPENAI|ANTHROPIC)"
   ```

4. **端口占用**

   ```bash
   # 查找占用进程
   lsof -i :8082
   # 或使用不同端口
   PORT=8083 ./claude-code-proxy
   ```

5. **内存不足**
   - 单文件版本需要更多内存解压
   - 考虑使用目录版本

### 日志调试

```bash
# 启用详细日志
LOG_LEVEL=DEBUG ./claude-code-proxy

# 查看系统日志
journalctl -u claude-proxy -f
```

## 性能优化

### 减小二进制大小

1. **使用 UPX 压缩** (仅 Windows)

   ```bash
   pyinstaller --onefile --upx-dir=/path/to/upx src/main.py
   ```

2. **移除不必要的依赖**

   - 清理 requirements.txt
   - 使用 --exclude 排除大模块

3. **选择性导入**
   - 只导入需要的模块
   - 避免通配符导入

### 启动优化

```bash
# 预加载优化
export PYTHONOPTIMIZE=2

# 内存优化
export PYTHONDONTWRITEBYTECODE=1
```

## 文件结构说明

```
dist/
├── claude-code-proxy/           # 目录版本
│   ├── claude-code-proxy        # 主可执行文件 (6.4MB)
│   └── _internal/               # 依赖库 (49MB)
│       ├── base_library.zip
│       ├── libpython3.10.so.1.0
│       └── ...
└── claude-code-proxy-single     # 单文件版本 (24MB)
```

## 版本管理

### 更新二进制

1. 重新构建项目
2. 备份旧版本
3. 部署新版本
4. 重启服务

### 回滚策略

```bash
# 保持上一个版本
cp claude-code-proxy claude-code-proxy.bak

# 回滚命令
mv claude-code-proxy.bak claude-code-proxy
systemctl restart claude-proxy
```

## 安全注意事项

1. **API 密钥安全**

   - 使用环境变量而非硬编码
   - 定期轮换密钥
   - 限制文件权限

2. **网络安全**

   - 使用 HTTPS
   - 配置防火墙规则
   - 监控访问日志

3. **系统安全**
   - 运行在非 root 用户
   - 定期更新系统
   - 监控资源使用

## 后续步骤

1. **测试**: 在目标环境验证功能
2. **监控**: 设置日志轮转和监控
3. **备份**: 定期备份配置和二进制文件
4. **文档**: 更新部署和维护文档

## 联系支持

如遇到问题，请检查：

1. 系统日志: `journalctl -u claude-proxy`
2. 应用日志: 检查 stdout/stderr 输出
3. 网络连接: 测试 API 端点连通性
4. 依赖完整性: 验证二进制文件完整性

---

_最后更新: 2024年8月22日_
_PyInstaller 版本: 6.15.0_
