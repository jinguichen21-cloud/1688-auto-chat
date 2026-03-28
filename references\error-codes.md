# 1688-auto-chat错误码与调试流程

##错误分类与处理

### 1.链接相关错误

#### LINK_INVALID (链接失效)

**触发条件:**
-访问商品页返回404
-页面被重定向到首页
-触发验证码拦截无法继续

**检测方式:**
```python
response = requests.get(product_url, headers={'User-Agent': ua.random})
if response.status_code == 404:
    error = "LINK_INVALID"
elif "验证码" in response.text or "滑块" in response.text:
    error = "CAPTCHA_DETECTED"
```

**处理流程:**
```
检测到链接失效
  ↓
更新表格：result_field = "链接失效"
  ↓
记录错误日志：{record_id, url, error_type, timestamp}
  ↓
跳过该记录，继续下一条
```

---

### 2.旺旺相关错误

#### WANGWANG_NOT_ACTIVATED (旺旺未激活)

**触发条件:**
-点击客服按钮后无反应
-等待30秒仍未显示聊天窗口
-页面提示"请先下载旺旺"

**检测方式:**
```javascript
//等待聊天窗口出现
wait_for(text='您发送的消息将发送至', timeout=30000)
  .catch(() => error = "WANGWANG_NOT_ACTIVATED")
```

**处理流程:**
```
检测到旺旺未激活
  ↓
重试1:刷新页面，重新点击客服按钮
  ↓
重试2:清除浏览器缓存，重新打开商品页
  ↓
重试3:检查本地是否安装旺旺客户端
  ↓
仍失败 → 更新表格：result_field = "旺旺唤起失败"
```

#### MESSAGE_SEND_FAILED (消息发送失败)

**触发条件:**
-输入框写入内容但不上屏
-点击发送按钮无反应
-发送后消息不出现在聊天历史中

**原因分析:**
- React/Vue框架的状态管理限制
-输入框事件未正确触发

**解决方案（已实现）:**
```javascript
//正确的发送方式
const input = document.querySelector('pre.edit[contenteditable="true"]');
input.textContent = message;
input.dispatchEvent(new Event('input', {bubbles: true}));
input.dispatchEvent(new Event('change', {bubbles: true}));

//使用系统级键盘模拟（备选方案）
pyautogui.write(message)
pyautogui.press('enter')
```

---

### 3.回复监控错误

#### TIMEOUT_NO_REPLY (超时未回复)

**触发条件:**
-等待超过timeout_minutes仍未收到回复
-商家不在线或未及时响应

**检测方式:**
```python
start_time = time.time()
while time.time() - start_time < timeout_minutes * 60:
    if new_message_detected():
        break
else:
    error = "TIMEOUT_NO_REPLY"
```

**处理流程:**
```
检测到超时
  ↓
检查已获取的信息数量
  ├─ 已获取≥3项 → 标记为"信息基本完整（缺X项）"
  └─ 已获取<3项 → 标记为"商家未回复(超时)"
  ↓
记录超时时间戳
  ↓
可稍后重试（建议间隔1小时以上）
```

#### EXTRACTION_FAILED (提取失败)

**触发条件:**
-商家回复了但正则表达式匹配失败
-回复格式不符合预期

**常见场景:**
```
商家回复："有的亲"  → 缺少具体颜色信息
商家回复："可以定制" → 未明确说贴牌
商家回复："不小呢"   → 没有具体尺寸数字
商家回复："挺轻的"   → 没有具体重量数字
```

**处理流程:**
```
检测到提取失败
  ↓
保留原始对话内容
  ↓
标记structured_field.extraction_failed = true
  ↓
生成追问话术，针对性询问缺失字段
  ↓
最多追问2次，仍失败则人工介入
```

---

### 4.网络相关错误

#### NETWORK_ERROR (网络异常)

**触发条件:**
- DNS解析失败
-连接超时
- SSL证书错误

**处理流程:**
```
检测到网络错误
  ↓
重试1:等待5秒后重试
  ↓
重试2:更换DNS服务器（8.8.8.8 / 1.1.1.1）
  ↓
重试3:检查本地网络连接
  ↓
仍失败 → 标记为"网络异常"，暂停任务
```

#### RATE_LIMITED (频率限制)

**触发条件:**
-短时间内访问过多商品页
-并发会话数超过限制
- 1688 反爬机制触发

**检测方式:**
```python
if "访问过于频繁" in response.text or "请稍后再试" in response.text:
    error = "RATE_LIMITED"
```

**处理流程:**
```
检测到频率限制
  ↓
立即停止所有并发请求
  ↓
等待cooldown_period（默认15分钟）
  ↓
降低max_concurrent参数（如从3降到1）
  ↓
恢复执行
```

---

##调试命令

### verbose模式

开启详细日志：
```bash
export DEBUG=1688_auto_chat:*
real-cli skill run 1688-auto-chat --verbose
```

日志输出：
```
[2026-03-14 19:00:00] INFO:读取表格记录50条
[2026-03-14 19:00:01] INFO:筛选出待沟通记录12条
[2026-03-14 19:00:02] DEBUG:打开商品页https://detail.1688.com/offer/xxx.html
[2026-03-14 19:00:05] DEBUG:定位客服按钮.odDetail-sendBtn
[2026-03-14 19:00:06] INFO:唤起旺旺成功
[2026-03-14 19:00:07] DEBUG:发送商品卡片
[2026-03-14 19:00:08] DEBUG:发送询问文本
[2026-03-14 19:00:18] INFO:等待商家回复...
[2026-03-14 19:01:30] DEBUG:检测到新消息
[2026-03-14 19:01:31] INFO:提取现货状态：浅灰有现货 ✅
[2026-03-14 19:01:32] INFO:提取贴标能力：可以贴标 ✅
[2026-03-14 19:01:33] INFO:提取包装尺寸：42*25*5cm ✅
[2026-03-14 19:01:34] INFO:提取包装重量：0.6kg ✅
[2026-03-14 19:01:35] INFO:回写结果到表格record_id=1WCIl72Ied
```

###单步调试

逐条记录执行：
```bash
real-cli skill run 1688-auto-chat \
  --single_record "1WCIl72Ied" \
  --step_by_step
```

每步确认后继续：
```
步骤1:读取表格 → [按Enter继续]
步骤2:打开商品页 → [按Enter继续]
步骤3:唤起旺旺 → [按Enter继续]
...
```

---

##性能优化建议

### 1.并发控制

推荐配置：
-普通场景：`max_concurrent=3`
-高价值商品：`max_concurrent=1`（专注跟进）
-批量初筛：`max_concurrent=5`（快速覆盖）

### 2.超时设置

推荐配置：
-初次询问：`timeout_minutes=3`
-第一次追问：`timeout_minutes=2`
-第二次追问：`timeout_minutes=1`

### 3.重试策略

```python
retry_config = {
    "max_retries": 3,
    "backoff_factor": 5,  # 5s, 10s, 20s...
    "retry_on": ["NETWORK_ERROR", "WANGWANG_NOT_ACTIVATED"]
}
```

### 4.缓存机制

避免重复沟通：
```python
#检查历史记录
if record['fields']['hh1eXb2'] and '✅' in record['fields']['hh1eXb2']:
    skip(record_id)  #已完成的跳过
```

---

##监控告警

###成功率监控

```python
#每小时统计
success_rate = success_count / total_count
if success_rate < 0.95:
    alert("成功率低于95%，请检查")
```

###异常告警

触发以下情况时告警：
-连续5条记录都失败
-同一错误类型出现≥10次
-平均耗时超过300秒/记录

告警方式：
```python
send_work_notification(
    content=f"""
【1688-auto-chat异常告警】
时间：{datetime.now()}
错误类型：{error_type}
失败次数：{fail_count}
建议操作：{suggestion}
    """
)
```
