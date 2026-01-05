# 报告查询网站部署指南

## 📋 项目结构

```
zy.jktac.top/
├── backend/                # 后端 API 服务
│   ├── app.py              # Flask 主程序
│   ├── requirements.txt    # Python 依赖
│   ├── init_sample_data.py # 初始化示例数据
│   └── reports.db          # SQLite 数据库（自动生成）
├── deploy/                 # 部署配置文件
│   ├── nginx.conf          # Nginx/OpenResty 配置
│   ├── systemd-flask.service  # Systemd 服务文件
│   ├── Dockerfile          # Docker 镜像
│   └── docker-compose.yml  # Docker Compose 配置
├── WeChat/                 # 前端页面
│   └── rQuery.html         # 报告查询页面
├── Common/                 # 公共 JS 文件
├── Content/                # CSS 样式文件
├── Scripts/                # jQuery 等库文件
└── images/                 # 图片资源
```

## 🚀 部署步骤

### 方案一：传统部署（推荐）

#### 步骤 1: 上传文件到服务器

通过 SFTP 或 1Panel 文件管理器，将整个项目上传到 `/opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/`

```bash
# 目录结构应该是：
/opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/
├── backend/
├── WeChat/
├── Common/
├── Content/
├── Scripts/
├── images/
└── favicon.ico
```

#### 步骤 2: 配置 Python 环境

SSH 登录服务器执行：

```bash
# 进入后端目录
cd /opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/backend

# 创建 Python 虚拟环境
python3 -m venv venv

# 激活虚拟环境
source venv/bin/activate

# 安装依赖
pip install -r requirements.txt

# 初始化数据库
python app.py &
# 按 Ctrl+C 停止

# 添加示例数据（可选）
python init_sample_data.py
```

#### 步骤 3: 配置 Systemd 服务

```bash
# 复制服务文件
sudo cp /opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/deploy/systemd-flask.service /etc/systemd/system/report-query.service

# 重载 systemd
sudo systemctl daemon-reload

# 启动服务
sudo systemctl start report-query

# 设置开机自启
sudo systemctl enable report-query

# 查看服务状态
sudo systemctl status report-query
```

#### 步骤 4: 配置 1Panel + OpenResty

1. 登录 1Panel 面板
2. 进入 **网站** → **创建网站**
3. 选择 **静态网站**
4. 域名填写：`zy.jktac.top`
5. 根目录选择：`/opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top`
6. 创建完成后，点击 **配置** → **配置文件**
7. 在配置文件中添加以下内容（在 `server {}` 块内）：

```nginx
    # 处理 /WeChat/rQuery 路由（不带 .html 后缀）
    location = /WeChat/rQuery {
        try_files /WeChat/rQuery.html =404;
    }

    # API 代理到 Flask 后端
    location /WeChat/GetReportInfo {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    location /WeChat/GetReportUrl {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    # 后端管理 API
    location /api/ {
        proxy_pass http://127.0.0.1:5000;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
```

8. 保存配置并重载 OpenResty

#### 步骤 5: 配置 SSL 证书

在 1Panel 中：
1. 进入 **网站** → 选择你的网站 → **HTTPS**
2. 申请/上传 SSL 证书
3. 开启 **强制 HTTPS**

---

### 方案二：Docker 部署

```bash
# 进入部署目录
cd /opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/deploy

# 构建并启动
docker-compose up -d

# 查看日志
docker-compose logs -f
```

---

## 📊 数据管理

### 添加新报告

通过 API 添加报告数据：

```bash
curl -X POST https://zy.jktac.top/api/add_report \
  -H "Content-Type: application/json" \
  -d '{
    "testingReportNo": "HY188-240033",
    "testingInstituteName": "某某检测有限公司",
    "testingOrderNo": "WT2024003",
    "reportDate": "2024-02-01",
    "unitName": "某某建筑公司",
    "testingDate": "2024-01-28",
    "testingTypeName": "委托",
    "projectName": "某某工程",
    "projectSection": "主体结构",
    "sampleNo": "YP2024003",
    "sampleName": "混凝土试块",
    "productiveUnit": "某某搅拌站",
    "testingBasisItems": "GB/T 50081-2019",
    "specification": "C35",
    "sampleLevel": "",
    "isDelete": 0,
    "reportUrl": "https://example.com/report.pdf"
  }'
```

返回示例：
```json
{
  "success": true,
  "rId": "abc1234",
  "message": "报告添加成功"
}
```

### 查看所有报告

```bash
curl https://zy.jktac.top/api/reports
```

### 数据库字段说明

| 字段名 | 说明 | 示例 |
|--------|------|------|
| rId | 报告唯一标识（自动生成） | e22el6a |
| testingReportNo | 报告编号 | HY188-240031 |
| testingInstituteName | 检测机构名称 | 某某检测有限公司 |
| testingOrderNo | 委托编号 | WT2024001 |
| reportDate | 签发日期 | 2024-01-15 |
| unitName | 委托单位 | 某某建筑公司 |
| testingDate | 委托日期 | 2024-01-10 |
| samplingDate | 抽样日期 | 2024-01-08 |
| testingTypeName | 检测类型 | 委托/抽样/认证抽样 |
| projectName | 工程名称 | 某某住宅楼 |
| projectSection | 工程部位 | 主体结构 |
| sampleNo | 样品编号 | YP2024001 |
| sampleName | 样品名称 | 混凝土试块 |
| productiveUnit | 生产单位 | 某某搅拌站 |
| testingBasisItems | 检验依据 | GB/T 50081-2019 |
| specification | 规格型号 | C30 |
| sampleLevel | 样品等级 | |
| isDelete | 是否作废(0有效/1无效) | 0 |
| reportUrl | 报告下载链接 | https://... |

---

## 🔗 访问地址

部署完成后，可以通过以下地址访问：

- 查询页面：`https://zy.jktac.top/WeChat/rQuery?rId=e22el6a&rNo=HY188-240031`
- 管理 API：`https://zy.jktac.top/api/reports`

---

## 🔧 常见问题

### 1. API 返回 502 错误
- 检查 Flask 服务是否正常运行：`systemctl status report-query`
- 查看日志：`journalctl -u report-query -f`

### 2. 页面样式不显示
- 检查静态文件是否上传完整
- 检查文件权限：`chmod -R 755 /opt/1panel/apps/openresty/openresty/www/sites/zy.jktac.top/`

### 3. 数据库错误
- 确保 backend 目录有写权限
- 检查 reports.db 文件是否存在

---

## 📝 你的数据抓取脚本

根据你提到的数据源字段，你可以编写 Python 脚本批量导入数据：

```python
import requests

def add_report(data):
    url = "https://zy.jktac.top/api/add_report"
    response = requests.post(url, json=data)
    return response.json()

# 示例：添加报告
report = {
    "testingReportNo": "你的报告编号",
    "testingInstituteName": "检测机构名称",
    "testingOrderNo": "委托编号",
    "reportDate": "2024-01-15",
    "unitName": "委托单位",
    "testingDate": "2024-01-10",
    "testingTypeName": "委托",
    "projectName": "工程名称",
    "projectSection": "工程部位",
    "sampleNo": "样品编号",
    "sampleName": "样品名称",
    "productiveUnit": "生产单位",
    "testingBasisItems": "检验依据",
    "specification": "规格型号",
    "sampleLevel": "",
    "isDelete": 0,
    "reportUrl": ""
}

result = add_report(report)
print(f"添加结果: {result}")
print(f"查询地址: https://zy.jktac.top/WeChat/rQuery?rId={result['rId']}&rNo={report['testingReportNo']}")
```
