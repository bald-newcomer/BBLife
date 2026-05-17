# Beilef
每个智能体都有自己的信念，都基于不同的分析方法和数据

```python
# 就像三个分析师对同一支股票有不同的看法：
beliefs = {
    "技术分析师": "应该买入，因为MACD金叉",      # 信念1
    "基本面分析师": "应该持有，估值合理",        # 信念2  
    "情绪分析师": "应该卖出，市场情绪悲观"       # 信念3
}
```

## Consensus Action（共识动作）
共识动作，大家投票决定的最终方案
```shell
# 三个分析师投票：
技术分析师: BUY     (权重0.9)
基本面分析师: HOLD  (权重0.7)  
情绪分析师: SELL    (权重0.6)

# 共识计算：
BUY: 0.9分
HOLD: 0.7分  
SELL: 0.6分

# 共识动作 = BUY（得分最高）
```
## Consensus Confidence（共识置信度）
达成共识的可信度
```python
# 计算共识置信度：
总置信度 = 0.9 + 0.7 + 0.6 = 2.2
支持BUY的置信度 = 0.9
共识置信度 = 0.9 / 2.2 ≈ 0.41

# 这意味着：共识的可靠性为41%
```

##  Conflicts（冲突）
团队内部的分歧程度
```python
# 分析冲突：
共识动作: BUY
反对者: 基本面分析师(HOLD), 情绪分析师(SELL)
反对比例: 2/3 ≈ 0.67

# 冲突分析结果：
存在冲突: 是
冲突程度: 67% (高度冲突)
反对动作: [HOLD, SELL]

# 冲突级别：
# 无冲突：所有人都同意
# 轻度冲突：少数人反对
# 重度冲突：意见严重分歧
```

# Department
在智能体系统中，部门是一个重要的组织概念，用于对智能体进行功能划分和专业化管理。

部门就是把大问题拆分成小问题，让专业的人（智能体）做专业的事！就像公司里的不同专业团队，每个部门负责特定的分析维度

```python
class Department(Enum):
    TECHNICAL_ANALYSIS = "technical"      # 技术分析部
    FUNDAMENTAL_ANALYSIS = "fundamental"  # 基本面分析部
    RISK_MANAGEMENT = "risk"              # 风险管理部
    MARKET_SENTIMENT = "sentiment"        # 市场情绪部
    PORTFOLIO_STRATEGY = "portfolio"      # 组合策略部

# 举例
class TechnicalDepartment(Agent):
    def analyze(self, context):
        # 专注于技术指标
        return Belief(
            action=基于技术指标的计算结果,
            confidence=技术信号强度,
            reasoning="MACD金叉, RSI超卖反弹"
        )
    def make_decision(self, stock_data):
        return consensus
```

# Function Calling
在大模型调用时，若判断当前需要更多信息，且需要的信息同时在提供的函数中能获取到，会触发Function Calling

## LLM 判断是否需要 Function Calling 的依据：
* 当前 prompt/上下文是否包含决策所需的全部信息；
* tools 参数中是否有可用的函数描述；
* LLM 的推理能力（模型本身会根据任务和工具描述自动决定是否调用）；
* tool_choice 参数的设置。

场景：部门智能体做股票决策，LLM需要补充数据

```shell
初始输入
股票代码：600519.SH
交易日：2024-10-18
features、market_snapshot等基础信息
```
## 多轮调用过程
### 第1轮：
* LLM收到初始prompt，分析后回复：“请补充近30日的daily行情数据和daily_basic基本面数据。”
* 响应中包含工具调用请求：fetch_data，参数为需要的表和窗口。

系统处理：识别到工具调用，调用 _handle_tool_call，自动查询数据库，返回近30日daily和daily_basic数据。
把数据补充到对话历史，并作为tool响应发回LLM。

### 第2轮：

* LLM收到补充的数据，再次分析，回复：“根据动量信号，建议BUY_M，置信度0.72，主要依据：近30日涨幅明显，基本面稳定。”
* 响应为结构化文本或JSON，包含action、confidence、signals等字段。

系统处理：解析LLM回复，提取决策结果，终止多轮调用。


### 特殊情况：
如果LLM再次提出新的数据需求（如“请补充最新新闻情绪数据”），系统会继续补充数据，进入第3轮。

如果达到最大轮次还未得到决策，则用最后一次回复作为结果。

---------

# codemate学习

addLocalRetrievalProvider: 本地检索系统的核心