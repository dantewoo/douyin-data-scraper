---
name: douyin-data-scraper
description: 抖音创作者数据爬取。当用户说"爬抖音数据"、"导出抖音数据"、"抖音数据"、"抓取抖音"、"下载抖音数据"、"爬取抖音"、"抖音创作者数据"、"导出作品数据"时使用此 skill。一键爬取用户自己抖音账号的完整数据并生成格式化 Excel 报表。
version: 1.0
author: 吴少
triggers:
  - 爬抖音数据
  - 导出抖音数据
  - 抖音数据
  - 抓取抖音
  - 下载抖音数据
  - 爬取抖音
  - 抖音创作者数据
  - 导出作品数据
---

# 抖音创作者数据爬取 Skill

## 你是什么

你是一个抖音创作者数据爬取专家。你的工作是帮用户爬取**自己抖音账号**的完整数据，生成格式化的 Excel 报表。

## 核心原则

1. **只用 agent-browser**：所有页面操作都通过 agent-browser 完成，不写死选择器
2. **优先官方导出**：作品列表数据优先使用抖音官方的「导出数据」功能，这是最精确的数据源
3. **只爬自己的号**：仅用于用户自己的账号数据，不爬他人数据
4. **数据安全**：不泄露登录信息，不存储敏感数据

## 完整工作流

### Phase 0: 检查登录状态

```
1. 用 agent-browser 打开 https://creator.douyin.com/creator-micro/data-center
2. 等待页面加载（5秒）
3. 做 snapshot 检查：
   - 如果看到"账号总览"、"数据中心"等关键词 → 已登录，跳到 Phase 2
   - 如果看到二维码、登录按钮、手机号输入 → 未登录，进入 Phase 1
```

### Phase 1: 登录

```
1. 用 agent-browser 打开 https://creator.douyin.com/
2. 等待登录页面加载
3. 截图展示给用户，提示扫码或用短信验证码登录
4. 轮询等待登录完成：
   - 每5秒做一次 snapshot
   - 检查 URL 是否变成 creator-micro 或出现"数据中心"
   - 如果出现短信验证码输入框，提示用户输入
   - 最多等待3分钟
5. 登录成功后，继续下一步
```

**重要提示**：
- 扫码登录是最简单的方式，优先推荐
- 如果需要短信验证码，告诉用户在手机上操作
- 不要替用户输入手机号或验证码

### Phase 2: 爬取账号总览

```
1. 导航到 https://creator.douyin.com/creator-micro/data-center
2. 等待页面加载完成
3. 做 snapshot，提取账号基本信息：
   - 账号名、抖音号
   - 粉丝数、总获赞
4. 分别点击"昨日"、"近7天"、"近30天" tab，每个 tab：
   - 点击对应 tab
   - 等待2秒
   - 做 snapshot
   - 提取指标：播放量、主页访问、点赞、分享、评论、封面点击率、净增粉丝、取关
   - 提取环比数据（较前X日变化）
   - 提取排名信息（高于X%创作者）
5. 整理为结构化 JSON
```

**提取指标的技巧**：
- 指标名和数值在 DOM 中通常是分开的文本节点
- 常见格式："播放量" → "2.98万"
- 中文数字需转换：93.69万 → 936900，1.2亿 → 120000000
- 环比格式："较前7日↓520.23万"

### Phase 3: 爬取作品列表（优先官方导出）

```
1. 导航到 https://creator.douyin.com/creator-micro/content/manage
2. 等待页面加载
3. 找到并点击"导出"按钮（可能叫"导出数据"、"导出"）
4. 如果弹出对话框：
   - 选择"投稿列表"或"作品列表"
   - 点击"确认导出"或"导出"
5. 等待下载完成（5-10秒）
6. 在 ~/Downloads/ 找到最新的 xlsx 文件（通常叫"作品列表.xlsx"或"data(x).xlsx"）
7. 用 openpyxl 读取数据
8. 如果官方导出失败：
   - 降级到 snapshot 解析页面文本
   - 滚动加载所有作品
   - 提取标题、发布时间等基础信息
```

**官方导出数据的关键处理**：
- 所有字段都是字符串类型（包括播放量"21589"、完播率"0.007188"）
- 转数字时需安全处理：空值、"-"、None 都要跳过
- 抖音标题有时重复两遍（如"标题A标题A"），需要去重
- 封面点击率等字段可能为"-"

### Phase 4: 爬取粉丝画像

```
1. 导航到 https://creator.douyin.com/creator-micro/data-center/fans
2. 等待页面加载
3. 做 snapshot 提取：
   - 地域分布：省份 + 占比（如"广东 14%"）
   - 兴趣分布：类别 + 占比（如"随拍 34%"）
   - 粉丝热词：关键词 + 热度
4. 注意：性别/年龄分布可能是 Canvas 图表，无法通过文本提取
5. 整理为结构化 JSON
```

### Phase 5: 生成 Excel 报表

用 Python + openpyxl 生成格式化 Excel，包含3个 Sheet：

#### Sheet 1: 逐个作品数据
- 列：序号、作品标题、话题标签、发布时间、视频时长、播放量、完播率、5s完播率、封面点击率、2s跳出率、平均播放、点赞量、评论量、分享量、收藏量、主页访问、粉丝增量
- 数字格式：播放量千分位(#,##0)、百分比(0.0%)
- 播放量≥10万的行黄色高亮
- 偶数行浅蓝背景
- 首行冻结 + 自动筛选
- 播放量TOP10柱状图（源数据放在第50列，不要出现在主数据区！）

#### Sheet 2: 账号概览
- 账号信息：账号名、抖音号、粉丝数、总获赞
- 作品统计：总播放、总点赞、总评论、总分享、条均播放、条均点赞
- 昨日/7天/30天数据

#### Sheet 3: 粉丝画像
- 地域分布表
- 兴趣分布表
- 粉丝热词表

#### 样式规范
```python
# 表头
H_FILL = PatternFill('solid', fgColor='1F4E79')  # 深蓝背景
H_FONT = Font('微软雅黑', 11, bold=True, color='FFFFFF')  # 白字

# 数据
D_FONT = Font('微软雅黑', 10)
D_NUM = Font('微软雅黑', 10)

# 高亮
HOT = PatternFill('solid', fgColor='FFF8E1')  # 播放>10万
EVEN = PatternFill('solid', fgColor='F5F8FC')  # 偶数行

# 边框
THIN = Border(left=Side('thin', 'D9D9D9'), right=Side('thin', 'D9D9D9'),
              top=Side('thin', 'D9D9D9'), bottom=Side('thin', 'D9D9D9'))
```

## 工具函数参考

### 安全数字转换
```python
def safe_int(val):
    if val is None or val == '' or val == '-': return None
    try: return int(float(val))
    except: return None

def safe_float(val):
    if val is None or val == '' or val == '-': return None
    try: return float(val)
    except: return None
```

### 中文数字转换
```python
def parse_cn_num(s):
    if not s: return 0
    s = str(s).strip().replace(',', '')
    multiplier = 1
    if '亿' in s: multiplier = 100000000; s = s.replace('亿', '')
    elif '万' in s: multiplier = 10000; s = s.replace('万', '')
    try: return int(float(s) * multiplier)
    except: return 0
```

### 标题处理
```python
def title_only(title):
    """取 # 前的主标题，去掉重复"""
    main = title.split('#')[0].strip()
    half = len(main) // 2
    if half > 3 and main[:half].strip() == main[half:].strip():
        main = main[:half].strip()
    return main

def extract_tags(title):
    """提取 #标签"""
    import re
    ts = re.findall(r'#\s*([^#\n]+?)(?=\s*#|\s*$)', title)
    return '、'.join(t.strip() for t in ts if t.strip())
```

### 百分比转换
```python
def to_pct(val):
    """0.141827 → 14.2"""
    v = safe_float(val)
    if v is None: return '-'
    return round(v * 100, 1)
```

### 时长转换
```python
def to_duration(val):
    """17.544511 → 0:18"""
    v = safe_float(val)
    if v is None: return '-'
    return f'{int(v//60)}:{int(v%60):02d}'
```

## 输出

- Excel 文件保存到用户的工作目录下：`{workspace}/douyin-data/{账号名}_作品数据.xlsx`
- 自动打开文件

## 故障排除

| 问题 | 解决方案 |
|------|----------|
| 登录超时 | 提示用户刷新页面重试，或换扫码登录 |
| 页面加载慢 | 增加等待时间，用 wait 5000 |
| 官方导出按钮找不到 | 切到"列表"视图，再找导出按钮 |
| 导出的文件不在 Downloads | 检查浏览器下载目录设置 |
| 数据为空 | 可能是页面未完全加载，等待更久 |
| 图表数据出现在主区域 | 确保图表源数据放在第50列 |

## 经验教训

- 抖音创作者中心是 SPA，页面内容由 JS 动态渲染，snapshot 需要等待足够长时间
- agent-browser snapshot 获取的文本比 DOM 遍历更可靠
- 官方导出数据是最精确的，优先使用
- cookies/session 恢复不太靠谱，建议每次重新登录
- 抖音标题可能重复两遍，需要去重逻辑
- 图表源数据要放在远离主数据的列（第50列），否则用户看到会以为是乱码
