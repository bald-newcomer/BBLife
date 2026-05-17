# 创建和使用python环境

## 1. 为你的项目创建虚拟环境
python3 -m venv aegis

## 2. 激活虚拟环境
source aegis/bin/activate

# 3. 安装依赖
pip install -r requirements.txt

## 4. 当你完成工作后，可以退出虚拟环境
deactivate

--------
# 执行测试用例
python -m pytest tests/test_config.py -v

python -m pytest tests/test_config.py::TestAppConfig::test_default_llm_providers_called -v

## 执行测试用例 单元测试
python -m pytest tests/test_config.py::TestAppConfig::test_default_llm_providers_called -v

python -m pytest tests/test_department.py::test_analyze_extends_round_for_tool_only_responses -v

python -m pytest tests/test_department.py::test_analyze -v

python -m pytest tests/test_yfinance_client.py::TestYfinanceClient::test_get_stock_basic -v

python -m pytest tests/test_yfinance_client.py::TestYfinanceClient::test_get_stock_basic -v

--------
# debug
import pdb; pdb.set_trace()

n(ext)	n	执行下一行
s(tep)	s	进入函数调用
c(ontinue)	c	继续执行直到下一个断点
r(eturn)	r	执行直到当前函数返回
q(uit)	q	退出调试器 

l(ist)	显示当前代码位置
ll	显示当前函数的全部代码
w(here)	显示调用栈


命令	说明
p <expression>	打印表达式的值
pp <expression>	漂亮打印（适合复杂对象）
whatis <variable>	显示变量类型

# python的注入？


