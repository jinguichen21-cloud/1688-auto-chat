---
name: 1688-auto-chat
description: |
  1688 商家沟通自动化-自动获取现货/贴标/包装尺寸/重量信息并回写钉钉表格
  使用场景："1688 商家沟通" "询问现货情况" "获取包装信息" "自动回写表格" "批量沟通供应商"
version: 2.0.0
capabilities:
tools:
  - dingtalk-ai-table
  - browser-use
tags:
  - AI table
  - 1688
  - browser automation
---

# 1688 商家沟通自动化 Skill

## 严格禁止 (NEVER DO)

- **不要编造包装尺寸/重量等数据** — 必须从页面或商家真实回复中提取，提取失败标记为 null
- **不要在信息不完整时就结束会话** — 必须针对缺失字段追问，直到获取全部4项信息或超时
- **不要并行超过3个会话** — 避免触发1688反爬机制和系统资源过载
- **不要在链接失效时强行执行** — 先验证链接有效性，无效链接标记为"链接失效"并跳过
- **不要在未登录状态下尝试发送消息** — 旺旺 web IM 需要登录态，未登录时提示用户先登录
- **不要使用 `.odDetail-sendBtn` 选择器** — 该选择器在当前1688页面不存在，已废弃
- **不要用 JavaScript 操作 NPS 评价弹窗** — IM 中的 "期待您的真实反馈" 弹窗，强行 display:none 会导致页面空白，直接忽略即可

## 参数提取规则

**baseId 和 tableId 以用户发送的钉钉 AI 表格链接为准**，不要使用硬编码默认值。

提取规则：
- **链接格式**: `https://alidocs.dingtalk.com/i/nodes/{baseId}?corpId=...&iframeQuery=entrance%3Ddata%26sheetId%3D{tableId}%26viewId%3D...`
- **baseId**: 从 URL path 中提取，即 `/i/nodes/` 后面的部分（`?` 之前）
- **tableId**: 从 `iframeQuery` 查询参数中解码后提取 `sheetId` 的值

示例：
```
链接: https://alidocs.dingtalk.com/i/nodes/2Amq4vjg89lXamOySQwEan5yJ3kdP0wQ?corpId=ding71d82db5b076cf6fa1320dcb25e91351&iframeQuery=entrance%3Ddata%26sheetId%3DPVVCLN1%26viewId%3DUNRyn4Z
→ baseId = 2Amq4vjg89lXamOySQwEan5yJ3kdP0wQ
→ tableId = PVVCLN1
```

如果用户未提供链接，使用以下默认值：

| 参数 | 值 | 说明 |
|------|------|------|
| base_id | `2Amq4vjg89lXamOySQwEan5yJ3kdP0wQ` | 钉钉 AI 表格 Base ID（亚马逊选品数据——海外） |
| table_id | `PVVCLN1` | 1688同款表 ID |
| link_field | `uZbKrdV` | 商品链接字段 ID |
| result_field | `hh1eXb2` | 商家确认信息字段 ID |
| structured_field | `O41WlpD` | 1688商家沟通记录字段 ID |

## 核心工作流（经实测验证）

整体流程分为三个阶段：**读表筛选 → 页面提取+旺旺沟通 → 回写表格**

### 阶段1: 读取表格，筛选待沟通记录

使用 `dingtalk-ai-table` MCP 工具：

```
# 1.1 查询表格记录（只取关键字段减少 token）
mcp__dingtalk-ai-table__query_records(
  baseId="{baseId}",  # 从用户提供的链接中提取
  tableId="{tableId}",  # 从用户提供的链接中提取
  fieldIds=["6a52Q3h", "uZbKrdV", "hh1eXb2", "O41WlpD", "65It1AN"],
  limit=100
)

# 1.2 在返回结果中筛选：有 uZbKrdV(商品链接) 且 hh1eXb2(商家确认信息) 为空的记录
incomplete = [r for r in records if r.cells.get("uZbKrdV") and not r.cells.get("hh1eXb2")]
```

### 阶段2: 对每条记录执行信息采集

#### 步骤2.1: 打开商品详情页，提取页面已有数据

```
# 用 browser-use 打开商品页（去掉 URL 追踪参数，只保留核心链接）
mcp__browser-use__navigate_page(type="url", url="https://detail.1688.com/offer/{offerId}.html")
mcp__browser-use__take_snapshot()
```

**页面可直接提取的数据（优先使用，无需旺旺沟通）：**

| 数据项 | 页面位置 | 提取方式 |
|--------|---------|---------|
| 包装尺寸/重量 | "包装信息" 区域（heading "商品件重尺"） | 从 snapshot 中读取表格：规格/颜色/长/宽/高/体积/重量 |
| 库存数量 | SKU 选择区域 | 每个尺寸旁的 "库存 XXXXX 个" |
| 是否支持贴标 | 页面横幅 "免费贴标" / 商品属性区域 | 搜索页面文本 |
| 供应商名称 | 页面顶部店铺信息 | heading level=1 中的公司名 |

#### 步骤2.2: 提取卖家旺旺 ID（用于打开 IM）

```javascript
// 在商品页通过 evaluate_script 提取
mcp__browser-use__evaluate_script(function=`() => {
  const scripts = document.querySelectorAll('script');
  let sellerLoginId = null;
  scripts.forEach(s => {
    const match = s.textContent.match(/sellerLoginId['":\\s]*['"]([^'"]+)['"]/);
    if (match) sellerLoginId = match[1];
  });
  return { sellerLoginId };
}`)
// 示例返回: { sellerLoginId: "飞驰体育用品实力工厂" }
```

#### 步骤2.3: 打开旺旺 web IM（直接导航，不通过商品页按钮）

```
# 直接打开 web IM 页面（经实测验证的可靠方式）
mcp__browser-use__new_page(
  url="https://air.1688.com/app/ocms-fusion-components-1688/def_cbu_web_im/index.html?touid=cnalichn{sellerLoginId}&siteid=cnalichn"
)

# 等待聊天窗口加载
mcp__browser-use__wait_for(text="请输入消息")
```

**注意**: 不要尝试在商品页点击"客服"按钮唤起旺旺——该按钮使用 React 事件绑定，`click()` 无法触发。直接导航到 web IM URL 是唯一可靠方式。

#### 步骤2.4: 发送询问消息（含商品链接，IM 自动生成卡片）

旺旺 web IM 的输入框在 iframe 内，需要通过 iframe document 操作：

```javascript
// 1. 先发送商品链接（IM 自动解析为卡片）
mcp__browser-use__evaluate_script(function=`() => {
  const iframe = document.querySelector('iframe');
  const iframeDoc = iframe.contentDocument;
  const editor = iframeDoc.querySelector('pre.edit[contenteditable="true"]');
  editor.focus();
  editor.innerHTML = '';
  iframeDoc.execCommand('insertText', false, 'https://detail.1688.com/offer/{offerId}.html');
  const sendBtn = iframeDoc.querySelector('.send-btn');
  sendBtn.click();
  return { success: true };
}`)

// 2. 再发送询问文本（所有问题放在一行，用空格分隔）
mcp__browser-use__evaluate_script(function=`() => {
  const iframe = document.querySelector('iframe');
  const iframeDoc = iframe.contentDocument;
  const editor = iframeDoc.querySelector('pre.edit[contenteditable="true"]');
  editor.focus();
  editor.innerHTML = '';
  const message = "你好，想了解一下这款商品：1.目前有现货吗？大概能发多少？ 2.单个包装尺寸是多少（长x宽x高）？ 3.单个重量大概多少？ 4.可以贴我们自己的标吗？起订量多少？";
  iframeDoc.execCommand('insertText', false, message);
  const sendBtn = iframeDoc.querySelector('.send-btn');
  sendBtn.click();
  return { success: true };
}`)
```

**实测关键发现**:
- 消息中包含 1688 商品 URL，IM 会自动将其解析为带图片和价格的商品卡片
- `execCommand('insertText')` **不支持 `\n` 换行**——换行后的内容会丢失，只发送第一行。所有问题必须写在同一行
- 发送按钮的选择器是 `.send-btn`（在 iframe 内），不要用 `button` 遍历
- 可以在同一个 evaluate_script 中先 insertText 再 click 发送按钮，减少调用次数

#### 步骤2.5: 监控商家回复

```
# 最可靠的方式：直接用 take_snapshot 读取最新消息内容
# 轮询：每 30 秒取一次 snapshot，提取商家回复
mcp__browser-use__take_snapshot()

# snapshot 中商家消息的识别特征：
# - 消息前有 "商家名:客服名" 的 StaticText（如 "飞驰体育用品实力工厂:阿良"）
# - 消息后有时间戳 StaticText
# - 自己的消息前有 "七炎楼"（或当前登录用户名）
```

**注意**: IM 可能弹出 "期待您的真实反馈" NPS 评价弹窗，**直接忽略即可，不要用 JavaScript 操作该弹窗**（强行 display:none 会导致页面空白，需刷新才能恢复）。该弹窗不影响 snapshot 读取消息内容。

**商家回复解读**:
- 商家可能分多条消息回复，且不一定按你的问题顺序
- "你讲"/"您说" = 商家在等你发问，需要发送具体问题
- 如果商家只回答了部分问题，需要单独追问未回答的具体问题

#### 步骤2.6: 信息提取（优先页面数据，旺旺回复补充）

信息提取优先级：
1. **页面 "包装信息" 区域** → 包装尺寸、重量（最可靠）
2. **页面 SKU 区域** → 库存/现货状态
3. **页面横幅/IM 卖家档案** → 贴标能力（IM 右侧面板 "加工定制" 字段）
4. **旺旺商家回复** → 补充确认以上信息，或获取页面未展示的信息

IM 右侧卖家档案面板包含的有用信息（无需询问即可获取）：
- `加工定制`: 如 "100个起订 | 支持贴牌 | 可接外贸订单"
- `经营模式`: 生产厂家 / 贸易商
- `生产档期`: 可判断是否有产能

### 阶段3: 回写钉钉表格

```
# 3.1 回写 商家确认信息(hh1eXb2)
mcp__dingtalk-ai-table__update_records(
  baseId="{baseId}",
  tableId="{tableId}",
  records=[{
    recordId: "{recordId}",
    cells: {
      "hh1eXb2": "现货：有(库存60847+);贴标：支持贴牌;包装尺寸：40x40x1cm(40cm款);重量：200g(40cm款)",
      "O41WlpD": "【沟通时间】{timestamp}\n【数据来源】页面提取+旺旺确认\n【页面包装信息】浅红色 直径40cm: 40x40x1cm, 200g; 直径50cm: 50x50x2cm, 330g ...\n【旺旺回复】{merchant_reply}\n【提取信息】现货：有 | 贴标：支持贴牌 | 包装尺寸：见上 | 重量：见上"
    }
  }]
)
```

## 上下文传递规则

| 步骤 | 产出数据 | 传递给 |
|------|---------|--------|
| query_records | recordId, product_url(uZbKrdV), supplier(65It1AN) | 后续所有步骤 |
| 商品页 snapshot | 包装信息表格、库存数据、贴标信息 | 信息提取、回写 |
| evaluate_script(sellerLoginId) | 卖家旺旺 ID | web IM URL 构造 |
| IM 卖家档案面板 | 加工定制信息、经营模式 | 贴标能力判断 |
| IM 商家回复 | 文本回复内容 | 信息提取、追问判断 |

## 错误处理

| 错误场景 | 处理方式 |
|---------|---------|
| **链接失效** (404/验证码拦截) | 标记 `hh1eXb2="链接失效"`，跳过该记录 |
| **未登录** (页面出现登录弹窗) | 暂停执行，提示用户扫码登录后继续 |
| **商家超时未回复** | 如页面已有包装信息，直接用页面数据回写；否则标记 `hh1eXb2="商家未回复(超时)"` |
| **sellerLoginId 提取失败** | 尝试从店铺 URL 或页面其他位置提取；仍失败则标记并跳过 |
| **iframe 输入失败** | 重试 execCommand；仍失败则尝试 press_key 逐字输入 |
| **1688 反爬触发验证码** | 暂停执行，通知用户手动处理验证码 |

## 性能指标

- **准确率目标**: >=99%（信息提取准确，无误判）
- **完成率目标**: >=95%（成功获取至少3/4项信息）
- **平均耗时**: <=180秒/记录（含追问）
- **追问成功率**: >=90%（追问后成功获取缺失信息）
