# 签字报表确认自动化工具 — 版本1

## 功能概述

将考勤签字报表数据通过飞书机器人发送给员工确认，收集回复并生成进度报告。

## 工作流程

```
签字报表.xlsx
    ↓
步骤1: 查询open_id          →  确认状态.csv（初始化）
    ↓
步骤2: 发送确认消息          →  飞书消息卡片
    ↓
步骤3: 轮询接收回复          →  更新确认状态.csv
    ↓
步骤4: 生成进度报告          →  进度报告.html
    ↓
步骤5: 上传多维表格          →  飞书bitable
    ↓
步骤6: 多维表格看板          →  多维表格看板.html
```

## 文件说明

| 文件 | 作用 |
|------|------|
| `初始化状态表.py` | 从签字报表读取员工信息，生成确认状态.csv |
| `步骤1_查询open_id.py` | 批量查询员工的飞书open_id |
| `步骤2_发送确认消息.py` | 通过飞书机器人发送确认卡片 |
| `feishu_poll.py` | 飞书长轮询，监听员工回复（独立常驻进程） |
| `步骤3_轮询接收回复.py` | 解析飞书回复，更新确认状态.csv |
| `步骤4_进度报告.py` | 生成HTML进度报告 |
| `步骤5_上传多维表格.py` | 将数据上传至飞书多维表格 |
| `步骤6_多维表格看板.py` | 生成多维表格看板HTML |
| `校验工具.py` | 签字报表与打卡记录/补签/休假交叉校验 |
| `确认状态.csv` | 所有员工的状态追踪表 |
| `bp白名单.csv` | BP白名单（无需确认的人员） |

## 飞书应用配置

- **App ID**: `cli_aa8ef6308ae41cb6`
- **App Secret**: `qVdoVV7M4QMPsBAhm5ETucG6VVwIcAja`
- **多维表格 Base ID**: `YwaAbg2lfaBDhhsCkVAcBvwhnMf`

## 使用步骤

### 前提条件

- Python 3.8+
- 安装依赖：`pip install requests openpyxl pandas`
- 飞书应用已开通「批量发送消息」权限

### 完整执行流程

**第1步：准备数据**
```
把签字报表.xlsx 放到项目目录，命名为「美区签字报表_OT已清除.xlsx」
```

**第2步：初始化状态表**
```
python 初始化状态表.py
```
生成 `确认状态.csv`，包含所有员工工号、部门信息。

**第3步：查询飞书open_id**
```
python 步骤1_查询open_id.py
```
读取签字报表中的工号，批量查询对应飞书用户的open_id，写入确认状态.csv。

**第4步：发送确认消息**
```
python 步骤2_发送确认消息.py
```
调用飞书批量消息API，逐一发送确认卡片给每位员工。

**第5步：启动轮询监听回复**
```bash
# 方式A：后台常驻（推荐用于正式运行）
nohup python feishu_poll.py >> 轮询日志.txt 2>&1 &

# 方式B：一次性轮询（调试用）
python 步骤3_轮询接收回复.py
```
> `feishu_poll.py` 是常驻进程，每5分钟自动检查新消息。
> `步骤3_轮询接收回复.py` 是单次执行版本。

**第6步：生成进度报告**
```
python 步骤4_进度报告.py
```
输出 `进度报告.html`，可在浏览器中打开查看实时进度。

**第7步：上传飞书多维表格（可选）**
```
python 步骤5_上传多维表格.py
python 步骤6_多维表格看板.py
```
将数据同步至飞书bitable，并生成本地看板HTML。

### 数据校验（可选）

```
python 校验工具.py
```
交叉验证签字报表、打卡记录、补签申请、休假流程的逻辑一致性，输出 `校验报告.csv`。

## 定时任务配置（macOS launchd）

如果需要长期后台运行，可配置 launchd plist：

```xml
<key>ProgramArguments</key>
<array>
  <string>/usr/bin/python3</string>
  <string>/Users/masc/bin/feishu_poll.py</string>
</array>
<key>StandardOutPath</key>
<string>/Users/masc/Library/Logs/feishu_poll.log</string>
<key>StandardErrorPath</key>
<string>/Users/masc/Library/Logs/feishu_poll.log</string>
```

注意：脚本路径不能含空格（launchd sandbox 限制），建议将脚本放在 `~/bin/` 下。

## 当前运行状态

- 总人数：304 人
- 已确认：1 人（GL502561 梁景悦）
- 有异议：0 人
- 待确认：303 人
- 轮询服务：每5分钟自动运行

## 注意事项

1. **路径含空格问题**：macOS launchd 无法执行路径含空格的脚本，务必将脚本放在无空格目录（如 `~/bin/`）
2. **飞书消息内容限制**：卡片内容过长会报 `code=9499`，需精简字段
3. **多维表格权限**：飞书应用只能访问自己创建的 base，无法访问用户分享的 base
4. **open_id查询限制**：批量查询有频率限制，304人约需数分钟
5. **BP白名单**：`bp白名单.csv` 中的人员不会被发送消息

## 版本记录

- **v1**：完成全流程闭环 —— 校验→发送→轮询→报告→bitable同步
