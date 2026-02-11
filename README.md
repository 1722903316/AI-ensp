# eNSP AI 平台

本项目是一个面向 **eNSP（华为网络模拟器）实验/教学场景** 的前后端分离平台：支持导入 `.topo` 拓扑、自动连接本地 eNSP 设备 Console 抓取 `display current-configuration`，并结合拓扑与配置进行故障分析、输出修复建议与命令。同时集成 Ping 工具、子网计算器、网络扫描等常用辅助能力。

> 适用场景：课程实验、网络排障训练、批量配置生成、拓扑可视化与工具化分析。

---

## 1. 核心能力概览

### 1.1 AI 智能修复（重点）

- 上传 eNSP 导出的 `.topo` 文件
- 平台解析拓扑并识别设备（AR/S 系列交换机/USG 防火墙/PC 等）
- 对每台设备自动连接 `127.0.0.1:<com_port>`（`.topo` 中自带的 Console 端口）
- 自动执行 `display current-configuration` 抓取设备当前配置（支持自动翻页，避免 `---- More ----` 截断）
- 规则化“预分析”（确定性证据）+ LLM 深度分析（可选）
- 输出：问题分析、排查步骤、按设备下发的修复命令，并附带“采集到的配置内容”

### 1.2 网络工具集

- Ping 工具：支持持续 ping、SSE 流式返回、历史记录
- 子网计算器：辅助 IPv4 子网划分/地址规划
- 网络扫描：接口发现、扫描历史记录（以本机网络信息与探测结果为主）

### 1.3 用户与历史

- 轻量用户注册/登录（SQLite）
- Ping/扫描历史记录（SQLite，默认保留最近 50 条）

---

## 2. 技术栈与目录结构

### 2.1 技术栈

- 前端：Vue 3 + Vite + Element Plus + Pinia + Vue Router
- 后端：Python Flask + Flask-CORS
- 拓扑解析：lxml（解析 eNSP `.topo` XML）
- AI 能力：LangChain + langchain-openai（可配置任意兼容 OpenAI API 的网关）
- 数据库：SQLite（`users.db`）

### 2.2 目录结构（核心）

```
ensp/
  ensp-frontend/          # 前端（Vue3/Vite）
  ensp-backend/           # 后端（Flask）
  test.topo               # 示例 topo 文件
```

后端关键文件：

- `ensp-backend/app.py`：Flask API 入口、拓扑解析、AI 修复、Ping/扫描等
- `ensp-backend/telnet_client.py`：eNSP Console Telnet 采集实现（含翻页与控制码清理）
- `ensp-backend/llm_service.py`：LLM 调用与输出结构（RepairSolution）
- `ensp-backend/database.py`：SQLite 用户与历史记录
- `ensp-backend/topo_generator.py`：拓扑/配置生成相关能力（与 AI 修复不同链路）

前端关键文件：

- `ensp-frontend/src/views/AIRepairView.vue`：AI 修复页面（采集统计、配置输出、预分析证据、修复结果）
- `ensp-frontend/src/router/index.js`：路由与登录态守卫

---

## 3. AI 修复的工作原理（端到端）

AI 修复链路的目标是：**先确保拿到“真实配置”，再做“基于证据的分析”**。

### 3.1 拓扑解析

`.topo` 本质是 XML。平台解析得到：

- `nodes[]`：设备 id/name(model)/com_port/坐标
- `edges[]`：设备间连接关系（当前未解析端口号，仅保留设备级连接）

其中 `com_port` 是 eNSP 分配的 Console 端口（例如 2000/2001…），用于后续自动采集配置。

### 3.2 配置采集（Telnet Console）

平台对每台设备：

1. 连接 `127.0.0.1:<com_port>`
2. 发送命令：`screen-length 0 temporary`（尽量关闭分页）
3. 执行：`display current-configuration | no-more`（优先拿全量）
4. 若设备不支持 `| no-more`，自动回退到 `display current-configuration`
5. 采集过程中若出现分页提示 `---- More ----`，会自动发送回车继续直到结束
6. 清理终端输出中的 ANSI 光标控制码（例如 `\x1b[42D`），避免“乱码”

采集结果会以每台设备维度返回：

- `ok / skipped / error`：采集是否成功、是否跳过、失败原因
- `config_len`：配置长度
- `config_key_lines`：提取的关键配置块/关键行
- `config_preview`：配置预览（为避免页面过大，默认截断到前 12000 字符）

### 3.3 规则化预分析（确定性证据）

仅靠大模型容易出现“看到 OSPF 就归因 OSPF”的问题。为避免这种“幻觉式推理”，平台在 LLM 之前增加了 **确定性预分析**：

- 从配置解析接口块，提取每个 `interface` 下的 `ip address`
- 如果某设备 **没有任何接口 IP**，直接给出高置信发现（例如“源设备缺少接口 IP”）
- 根据拓扑链路检查：链路两端 L3 设备是否存在共同网段（否则标记为可疑）
- 从问题描述中尝试抽取“源设备 + 目标 IP”，并识别目标 IP 的归属设备/接口（如果能从配置里匹配到）

预分析结果会以 `precheck` 字段返回，并注入大模型上下文，让模型优先解释“证据明确的低级错误”（如接口未配 IP）。

### 3.4 LLM 深度分析（可选）

如果配置了 `OPENAI_API_KEY` 等环境变量，平台会调用模型输出结构化 JSON：

- `analysis`：基于证据的原因分析
- `solution_steps`：排查/修复步骤
- `commands`：按设备给出可执行命令列表

若 LLM 不可用，平台会返回可用的降级提示，并保留采集的配置与预分析证据，仍可用于人工排障。

---

## 4. 常见误判的根因与本项目的改进点

在 eNSP 实验中，“删接口 IP 导致不通”这类问题非常常见。早期版本的 AI 结果容易出现：

- 只给“OSPF/路由传递问题”这种泛化结论
- 未明确指出“接口 IP 缺失/网段不一致”这类硬证据

主要根因：

1. LLM 在没有强证据时容易做经验推断
2. 拓扑解析未包含端口级映射，模型难以做精确链路定位
3. 配置上下文过长/过散，模型抓不到“缺失配置”的关键证据

本项目的改进：

- Telnet 全量采集 + 自动翻页
- 清理控制码，保证输出可读
- 引入规则化预分析（高置信证据）
- 将“证据”在前端直接展示，避免黑盒

---

## 5. API 设计（摘要）

后端默认地址：`http://localhost:5000`

### 5.1 AI 修复

- `POST /api/ai-repair`
  - FormData：
    - `file`: `.topo`
    - `issue`: 问题描述文本
  - 返回：
    - `analysis/solution_steps/commands`
    - `device_collection`：设备采集明细（含配置预览）
    - `precheck`：预分析证据

### 5.2 LLM 配置

- `GET/POST /api/system/llm-config`
  - 用于读取/写入 `.env` 中的模型配置（注意不要提交 `.env` 到 GitHub）

---

## 6. 本地运行

### 6.1 前置条件

- Windows 环境（eNSP 主要运行于 Windows）
- eNSP 已启动且拓扑中设备已运行完成
- Python（建议使用虚拟环境）
- Node.js / npm

### 6.2 启动后端

进入 `ensp-backend/`：

```bash
pip install -r requirements.txt
python app.py
```

默认监听：`http://127.0.0.1:5000`

### 6.3 启动前端

进入 `ensp-frontend/`：

```bash
npm install
npm run dev -- --host 0.0.0.0 --port 5173
```

如果 5173 被占用，Vite 会自动换到 5174/5175…，以控制台输出为准。


