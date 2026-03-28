# 1688-auto-chat 技能部署说明

##技能包已生成完成

技能包位置：`/Users/kitano/.real/workspace/1688-auto-chat-skill/`

包含文件：
```
1688-auto-chat-skill/
├── SKILL.md                      # 主入口文件（132行）
├── package.json                  # 元数据配置
├── README.md                     #使用说明
├── references/
│   ├── api-reference.md          # Tool Schema详情
│   └── error-codes.md            #错误码与调试流程
└── tests/
    ├── testcases.json            # 5个评测用例
    └── plan.json                 # 评测计划
```

##手动安装步骤

由于沙盒环境限制，需要手动将技能包复制到.skills目录：

###方法1：使用终端命令

```bash
# 1.复制技能包到.skills目录
cp -r /Users/kitano/.real/workspace/1688-auto-chat-skill \
      /Users/kitano/.real/.skills/1688-auto-chat

# 2.扫描并启用技能
real-cli skills scan local --json '{"paths": ["/Users/kitano/.real/.skills/1688-auto-chat"]}'

# 3.验证技能已安装
real-cli skills list --json '{}' | grep "1688-auto-chat"

# 4.运行技能（可选测试）
real-cli skill run 1688-auto-chat
```

###方法2：使用技能创建助手

```bash
#如果real-cli不可用，可以重新扫描workspace
real-cli skills scan local --json '{"paths": ["/Users/kitano/.real/workspace"]}'
```

##运行评测

安装完成后，运行评测验证技能质量：

```bash
#运行完整评测
real-cli eval run 1688-auto-chat --testcases /Users/kitano/.real/.skills/1688-auto-chat/tests/testcases.json

#或运行单个测试用例
real-cli eval run 1688-auto-chat --filter tc_001_complete_reply
```

**目标准确率：≥99%**

## 使用方式

###手动触发

在钉钉对话中直接说：
- "执行1688 商家沟通"
- "帮我询问供应商现货情况"
- "批量联系1688 商家获取包装信息"

###带参数执行

```python
#通过代码调用
result = skill.run(
    name="1688-auto-chat",
    params={
        "base_id": "2Amq4vjg89lXamOySQwEan5yJ3kdP0wQ",
        "table_id": "PVVCLN1",
        "max_concurrent": 3,
        "timeout_minutes": 3
    }
)
```

###定时执行（可选）

配置cron任务每30 分钟自动轮询：

```bash
real-cli cron add --json '{
  "name": "1688 商家沟通定时任务",
  "schedule": {"expr": "*/30 * * * *"},
  "payload": {
    "kind": "skill",
    "skill_name": "1688-auto-chat"
  }
}'
```

##核心功能验证

执行后检查钉钉表格中的以下字段：

1. **商家确认信息** (`hh1eXb2`):应包含✅标记的结构化文本
2. **结构化记录** (`O41WlpD`):应包含JSON 格式的4项核心数据

示例输出：
```
✅ 现货：浅灰色有现货，深灰色需要等3天
✅ 贴标：可以贴牌贴标，起订量100个
✅ 尺寸：42*25*5cm
✅ 重量：0.6kg
沟通时间：2026-03-14T19:00:00+08:00
```

## 故障排查

###问题1：技能未找到

**解决：**
```bash
#检查技能是否在列表中
real-cli skills list --json '{}'

#如果不在，重新扫描
real-cli skills scan local --json '{"paths": ["/Users/kitano/.real/workspace"]}'
```

###问题2：消息发送失败

**原因：**旺旺前端框架限制

**解决：**技能已通过evaluate注入事件方式解决，确保SKILL.md中的NEVER DO条款被遵守。

###问题3：信息提取不准确

**解决：**
1.检查`references/api-reference.md`中的正则表达式
2.根据实际商家回复调整extraction_rules
3.运行评测定位失败用例

###问题4：沙盒权限错误

**错误信息：** `Operation not permitted`或`Bad file descriptor`

**解决：**这是沙盒环境的正常限制，不影响技能执行。技能实际运行时使用的是浏览器自动化和钉钉 MCP，不依赖沙盒文件系统。

##性能监控

执行后可查看日志：

```bash
#查看详细执行日志
export DEBUG=1688_auto_chat:*
real-cli skill run 1688-auto-chat --verbose
```

关键指标：
-准确率：≥99%
-完成率：≥95%
-平均耗时：≤180秒/记录
-追问成功率：≥90%

##版本信息

- **当前版本**: v1.0.0
- **创建日期**: 2026-03-14
- **作者**:北野川
- **组织**: bug砖家

##下一步

1.按上述步骤安装技能
2.运行评测验证质量
3.在真实数据上小批量测试（建议先测3-5条记录）
4.确认无误后批量执行

如有问题，查看`references/error-codes.md`中的错误处理流程。
