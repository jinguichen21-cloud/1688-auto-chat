# 1688-auto-chat API Reference

## Tool Schema详情

### read_table_records

读取钉钉 AI 表格记录。

```json
{
  "name": "read_table_records",
  "description": "读取钉钉 AI 表格中的记录",
  "inputSchema": {
    "type": "object",
    "properties": {
      "base_id": {
        "type": "string",
        "description": "表格 Base ID"
      },
      "table_id": {
        "type": "string",
        "description": "表格ID"
      }
    },
    "required": ["base_id", "table_id"]
  }
}
```

**返回示例:**
```json
{
  "records": [
    {
      "record_id": "1WCIl72Ied",
      "fields": {
        "uZbKrdV": "https://detail.1688.com/offer/xxx.html",
        "hh1eXb2": "",
        "O41WlpD": null
      }
    }
  ]
}
```

---

### filter_incomplete_records

筛选需要沟通的记录（有链接但信息不完整）。

```json
{
  "name": "filter_incomplete_records",
  "description": "筛选出有商品链接但未完成沟通的记录",
  "inputSchema": {
    "type": "object",
    "properties": {
      "link_field": {
        "type": "string",
        "description": "商品链接字段 ID"
      },
      "result_field": {
        "type": "string",
        "description": "沟通结果字段 ID"
      }
    },
    "required": ["link_field", "result_field"]
  }
}
```

**过滤逻辑:**
```python
def filter(records):
    return [r for r in records 
            if r['fields'].get(link_field) 
            and (not r['fields'].get(result_field) 
                 or '待沟通' in r['fields'].get(result_field, ''))]
```

---

### open_product_page

打开1688 商品详情页。

```json
{
  "name": "open_product_page",
  "description": "使用浏览器打开1688 商品详情页面",
  "inputSchema": {
    "type": "object",
    "properties": {
      "product_url": {
        "type": "string",
        "description": "1688 商品详情页URL"
      }
    },
    "required": ["product_url"]
  }
}
```

**实现方式:**
```javascript
use_browser(action='navigate', url=product_url)
```

---

### activate_wangwang_session

唤起旺旺聊天并激活会话。

```json
{
  "name": "activate_wangwang_session",
  "description": "点击客服按钮，唤起旺旺聊天窗口",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

**实现方式:**
```javascript
//定位客服按钮
backbone = use_browser(action='backbone')
contact_btn = search(query='.odDetail-sendBtn')
click(ref=contact_btn.ref)

//等待旺旺窗口激活
wait_for(text='您发送的消息将发送至')
```

---

### send_product_card

发送商品链接卡片给商家。

```json
{
  "name": "send_product_card",
  "description": "点击发送链接按钮，将商品卡片发送给商家",
  "inputSchema": {
    "type": "object",
    "properties": {}
  }
}
```

**实现方式:**
```javascript
//点击商品详情卡片的发送按钮
evaluate('''
  const btn = document.querySelector('.odDetail-sendBtn');
  if (btn) {
    btn.click();
    btn.dispatchEvent(new Event('click', {bubbles: true}));
  }
''')
```

**关键点:**
-必须先发送商品卡片，否则商家不知道询问的是哪个商品
-使用`dispatchEvent`确保触发React/Vue的事件监听

---

### send_inquiry_message

发送标准化询问文本。

```json
{
  "name": "send_inquiry_message",
  "description": "向商家发送询问消息",
  "inputSchema": {
    "type": "object",
    "properties": {
      "inquiry_text": {
        "type": "string",
        "description": "询问内容"
      }
    },
    "required": ["inquiry_text"]
  }
}
```

**标准询问文本:**
```
您好，请问这款商品：
1.有现货吗？
2.可以贴条码/贴牌吗？
3.包装尺寸是多少？
4.包装重量是多少？
谢谢！
```

**实现方式:**
```javascript
//定位输入框（在第一个 iframe 内）
iframe = document.querySelector('iframe')
input_box = iframe.contentDocument.querySelector('pre.edit[contenteditable="true"]')

//写入文本并触发事件
input_box.textContent = inquiry_text
input_box.dispatchEvent(new Event('input', {bubbles: true}))
input_box.dispatchEvent(new Event('change', {bubbles: true}))

//点击发送按钮或按回车
evaluate('document.querySelector(".send-btn").click()')
//或
press(key='Enter')
```

---

### monitor_merchant_reply

监控商家回复并提取关键信息。

```json
{
  "name": "monitor_merchant_reply",
  "description": "监控商家回复，使用正则提取关键信息",
  "inputSchema": {
    "type": "object",
    "properties": {
      "timeout_minutes": {
        "type": "integer",
        "description": "超时时间（分钟）",
        "default": 3
      },
      "extraction_rules": {
        "type": "object",
        "description": "正则提取规则",
        "properties": {
          "stock_status": {"type": "string"},
          "labeling_capability": {"type": "string"},
          "package_dimensions": {"type": "string"},
          "package_weight": {"type": "string"}
        }
      }
    }
  }
}
```

**默认提取规则:**
```python
extraction_rules = {
    "stock_status": r"(有|无|没)现货|库存|现成的",
    "labeling_capability": r"(可以|能|支持|可)贴(牌|标|logo|商标)",
    "package_dimensions": r"\d+[*x×]\d+[*x×]\d+\s*(cm|厘米)",
    "package_weight": r"\d+\.?\d*\s*(g|kg|克|千克)"
}
```

**监控循环:**
```python
start_time = time.time()
while time.time() - start_time < timeout_minutes * 60:
    #获取聊天记录
    chat_history = use_browser(action='readability')
    
    #提取最新回复
    latest_reply = extract_latest_message(chat_history)
    
    #应用正则提取
    extracted = {}
    for field, pattern in extraction_rules.items():
        match = re.search(pattern, latest_reply, re.IGNORECASE)
        if match:
            extracted[field] = match.group(0)
    
    #检查是否全部提取成功
    if len(extracted) == 4:
        break
    
    #等待10秒后再次检查
    time.sleep(10)
```

---

### follow_up_missing_fields

针对缺失字段追问。

```json
{
  "name": "follow_up_missing_fields",
  "description": "针对未获取到的信息进行追问",
  "inputSchema": {
    "type": "object",
    "properties": {
      "missing_fields": {
        "type": "array",
        "items": {"type": "string"},
        "description": "缺失的字段列表"
      }
    },
    "required": ["missing_fields"]
  }
}
```

**追问话术模板:**
```python
follow_up_templates = {
    "stock_status": "请问还有现货吗？",
    "labeling_capability": "可以贴牌或贴条码吗？",
    "package_dimensions": "包装尺寸是多少呢？长×宽×高",
    "package_weight": "包装重量是多少？"
}

#生成追问文本
follow_up_text = "您好，还想确认一下：" + ", ".join(
    follow_up_templates[field] for field in missing_fields
)
```

---

### write_back_result

将结果回写钉钉表格。

```json
{
  "name": "write_back_result",
  "description": "将沟通结果回写到钉钉 AI 表格",
  "inputSchema": {
    "type": "object",
    "properties": {
      "record_id": {
        "type": "string",
        "description": "记录ID"
      },
      "fields": {
        "type": "object",
        "description": "要更新的字段",
        "properties": {
          "result_field": {"type": "string"},
          "structured_field": {"type": "object"}
        }
      }
    },
    "required": ["record_id", "fields"]
  }
}
```

**回写格式:**
```python
#文本字段（商家确认信息）
result_text = f"""
✅ 现货：{stock_status or '未提供'}
✅ 贴标：{labeling_capability or '未提供'}
✅ 尺寸：{package_dimensions or '未提供'}
✅ 重量：{package_weight or '未提供'}
沟通时间：{datetime.now()}
"""

#结构化字段
structured_data = {
    "stock_status": stock_status,
    "labeling_capability": labeling_capability,
    "package_dimensions": package_dimensions,
    "package_weight": package_weight,
    "extraction_confidence": 0.95,
    "follow_up_count": 1
}

#调用钉钉 MCP更新
dingtalk_mcp.update_record(
    base_id=base_id,
    table_id=table_id,
    record_id=record_id,
    fields={
        result_field: result_text,
        structured_field: json.dumps(structured_data)
    }
)
```

---

##错误码

|错误码|含义|处理方式|
|-------|------|---------|
| `LINK_INVALID` |商品链接失效(404/验证码) |标记为"链接失效"，跳过该记录|
| `TIMEOUT_NO_REPLY` |商家超时未回复|标记为"商家未回复(超时)",可重试|
| `EXTRACTION_FAILED` |正则匹配失败|保留原始对话，标记extraction_failed=true |
| `WANGWANG_NOT_ACTIVATED` |旺旺窗口未唤起|重试3次，仍失败则标记错误 |
| `NETWORK_ERROR` |网络异常|重试3次（间隔5秒），仍失败则标记错误|
| `RATE_LIMITED` |触发1688 反爬限制|暂停执行，通知用户手动处理|
