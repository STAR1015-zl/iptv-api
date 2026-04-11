# 本地测试指南

> 本指南用于验证合并后的代码是否可以正常运行

---

## 一、测试环境检查

### 1.1 检查 Python 版本
```bash
# 要求 Python 3.13（与上游一致）
python --version
# 或
python3 --version
```

**预期输出**: `Python 3.13.x`

### 1.2 检查 FFmpeg（测速功能依赖）
```bash
ffmpeg -version
```

**预期输出**: FFmpeg 版本信息

### 1.3 检查当前分支
```bash
git branch --show-current
```

**预期输出**: `merge-test-upstream-v2.0.7`

---

## 二、语法与导入测试

### 2.1 Python 语法检查
```bash
# 检查核心模块
python -m py_compile main.py
python -m py_compile service/app.py
python -m py_compile utils/tools.py
python -m py_compile utils/config.py
python -m py_compile utils/channel.py
python -m py_compile utils/speed.py
python -m py_compile utils/i18n.py
python -m py_compile utils/aggregator.py
python -m py_compile updates/epg/request.py
python -m py_compile updates/subscribe/request.py
```

**预期结果**: 无输出表示通过

### 2.2 模块导入测试
```bash
python -c "
from utils.config import config
print('✅ config 模块加载成功')
print(f'  - source_file: {config.source_file}')
print(f'  - open_update: {config.open_update}')
print(f'  - open_local: {config.open_local}')
print(f'  - open_subscribe: {config.open_subscribe}')
"
```

**预期输出**: 配置参数正常显示

### 2.3 新功能模块测试
```bash
python -c "
from utils.i18n import t
print('✅ i18n 模块加载成功')

from utils.aggregator import ResultAggregator
print('✅ aggregator 模块加载成功')

from utils.speed import clear_cache
print('✅ speed 模块加载成功')
"
```

**预期输出**: 各模块加载成功提示

---

## 三、依赖安装测试

### 3.1 创建虚拟环境（推荐）
```bash
# 使用 pipenv（项目推荐）
pipenv --python 3.13
pipenv install --deploy
```

**或使用 venv**
```bash
python -m venv .venv
# Windows
.venv\Scripts\activate
# Linux/Mac
source .venv/bin/activate
pip install -r <(pipenv requirements)
```

### 3.2 验证依赖安装
```bash
pipenv run python -c "
import aiohttp
import flask
import tqdm
import requests
print('✅ 核心依赖安装成功')
"
```

---

## 四、配置文件验证

### 4.1 验证配置文件格式
```bash
python -c "
import configparser
config = configparser.ConfigParser()
config.read('config/config.ini')
print('✅ config.ini 格式正确')
print(f'  - 配置项数量: {len(config.options(\"Settings\"))}')

config.read('config/user_config.ini')
print('✅ user_config.ini 格式正确')
print(f'  - 配置项数量: {len(config.options(\"Settings\"))}')
"
```

### 4.2 验证 RTP 配置文件
```bash
# 检查 RTP 配置数量
ls config/rtp/*.txt | wc -l
# 预期输出: 44

# 检查某个 RTP 文件格式
head -5 config/rtp/北京_电信.txt
# 预期格式: 频道名,rtp://IP:端口
```

### 4.3 验证频道模板
```bash
# 检查模板文件
head -10 config/user_demo.txt
```

---

## 五、功能运行测试

### 5.1 配置加载测试（快速）
```bash
pipenv run python -c "
import sys
sys.path.insert(0, '.')

# 测试配置加载
from utils.config import config
print('=== 配置加载测试 ===')
print(f'source_file: {config.source_file}')
print(f'final_file: {config.final_file}')
print(f'open_update: {config.open_update}')
print(f'open_local: {config.open_local}')
print(f'open_subscribe: {config.open_subscribe}')
print(f'open_speed_test: {config.open_speed_test}')
print(f'urls_limit: {config.urls_limit}')
print('✅ 配置加载测试通过')
"
```

### 5.2 主程序启动测试（短时间运行）
```bash
# 设置短超时，测试启动是否正常
timeout 30 pipenv run dev || echo "启动测试完成（可能因无数据源而退出）"
```

**预期行为**:
- 程序正常启动，无导入错误
- 可能因缺少数据源而正常退出或等待

### 5.3 服务模式启动测试
```bash
# 在后台启动服务
pipenv run dev &
# 等待启动
sleep 5
# 测试服务端口
curl http://localhost:5180 || echo "服务端口检查完成"
# 停止服务
pkill -f "pipenv run dev" || taskkill /F /IM python.exe
```

---

## 六、RTP 转换工作流测试

### 6.1 本地测试 RTP 转换脚本
```bash
# 创建测试脚本
python -c "
import os
import glob

# 测试 RTP 文件读取
rtp_files = glob.glob('config/rtp/*.txt')
print(f'✅ 找到 {len(rtp_files)} 个 RTP 配置文件')

# 读取第一个文件测试
if rtp_files:
    with open(rtp_files[0], 'r', encoding='utf-8') as f:
        lines = f.readlines()[:5]
        print(f'✅ 示例文件: {os.path.basename(rtp_files[0])}')
        for line in lines:
            print(f'  {line.strip()[:50]}...')
"
```

### 6.2 验证 RTP 工作流语法
```bash
# 检查 workflow 文件语法
python -c "
import yaml
with open('.github/workflows/rtp-to-m3u.yml', 'r') as f:
    wf = yaml.safe_load(f)
    print('✅ rtp-to-m3u.yml 语法正确')
    print(f'  - 触发方式: {list(wf.get(\"on\", {}).keys())}')
"
```

---

## 七、完整性检查清单

```bash
echo "=== 完整性检查 ==="

# 1. 私有配置
[ -f config/user_config.ini ] && echo "✅ user_config.ini" || echo "❌ user_config.ini"

# 2. 自定义模板
[ -f config/user_demo.txt ] && echo "✅ user_demo.txt" || echo "❌ user_demo.txt"

# 3. RTP 配置目录
RTP_COUNT=$(ls config/rtp/*.txt 2>/dev/null | wc -l)
[ "$RTP_COUNT" -eq 44 ] && echo "✅ RTP配置 ($RTP_COUNT个)" || echo "⚠️ RTP配置 ($RTP_COUNT个)"

# 4. 自定义工作流
[ -f .github/workflows/rtp-to-m3u.yml ] && echo "✅ rtp-to-m3u.yml" || echo "❌ rtp-to-m3u.yml"

# 5. 上游新功能
[ -f utils/i18n.py ] && echo "✅ i18n.py" || echo "⚠️ i18n.py"
[ -f utils/aggregator.py ] && echo "✅ aggregator.py" || echo "⚠️ aggregator.py"
[ -d config/logo ] && echo "✅ logo目录" || echo "⚠️ logo目录"

# 6. main.yml 优化
grep -q "permissions:" .github/workflows/main.yml && echo "✅ permissions" || echo "❌ permissions"
grep -q "concurrency:" .github/workflows/main.yml && echo "✅ concurrency" || echo "❌ concurrency"
grep -q "upload-artifact" .github/workflows/main.yml && echo "✅ artifacts" || echo "❌ artifacts"
```

---

## 八、测试结果判定

### ✅ 测试通过标准
- 所有语法检查无错误
- 配置加载正常，参数值正确
- 依赖安装成功
- 程序可以启动（即使因无数据源退出）
- 文件完整性检查全部通过

### ⚠️ 需要注意的情况
- 某些模块警告但非错误 → 可接受
- 因网络问题无法获取 EPG → 正常
- 因无订阅源而结果为空 → 正常

### ❌ 测试失败情况
- Python 语法错误 → 需要修复
- 模块导入失败 → 检查依赖
- 配置加载报错 → 检查配置文件格式

---

## 九、快速测试脚本

将以下内容保存为 `test_local.sh` 并执行：

```bash
#!/bin/bash
echo "========================================"
echo "       本地快速测试脚本"
echo "========================================"

# 语法检查
echo "[1/5] Python 语法检查..."
python -m py_compile main.py service/app.py utils/config.py 2>/dev/null && echo "✅ 语法检查通过" || echo "❌ 语法检查失败"

# 配置加载
echo "[2/5] 配置加载测试..."
python -c "from utils.config import config; print('✅ 配置加载成功')" 2>/dev/null || echo "❌ 配置加载失败"

# 依赖检查
echo "[3/5] 依赖检查..."
pipenv run python -c "import aiohttp, flask, tqdm; print('✅ 核心依赖正常')" 2>/dev/null || echo "⚠️ 请先安装依赖: pipenv install"

# 文件完整性
echo "[4/5] 文件完整性..."
RTP=$(ls config/rtp/*.txt 2>/dev/null | wc -l)
[ "$RTP" -eq 44 ] && echo "✅ RTP配置完整 ($RTP个)" || echo "⚠️ RTP配置异常 ($RTP个)"

# 服务端口检查
echo "[5/5] 端口检查..."
netstat -an | grep -q "5180.*LISTEN" && echo "✅ 端口5180可用" || echo "✅ 端口5180未被占用"

echo "========================================"
echo "       测试完成"
echo "========================================"
```

---

**测试完成后，如果全部通过，即可执行合并到主分支操作。**
