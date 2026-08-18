# 交易策略深度分析（Strategies Deep Dive）

> - **分析日期**：2026-08-13
> - **分析对象**：`strategies/` 目录下 15 个内置策略（自然语言 YAML）
> - **数据契约**：工具字段与行为以 `src/agent/tools/` 实际实现为准（`data_tools.py`、`analysis_tools.py`、`search_tools.py`、`market_tools.py`）
> - **基线状态**：`required_tools` 已补齐（3 个策略 4 处缺口已修复）、乖离率严进阈值已由 5% 调整为 7%（龙头放宽 10%/谨慎线 13%）
> - **说明**：本文档记录分类、逐策略解析与优化建议；优化建议仅为分析结论，不代表已实施

---

## 1. 策略总览

### 1.1 策略清单

| # | name | display_name | category | priority | market_regimes | 默认状态 |
|---|------|-------------|----------|----------|----------------|----------|
| 1 | bull_trend | 默认多头趋势 | trend | 10 | trending_up | `default_active` + `default_router` |
| 2 | ma_golden_cross | 均线金叉 | trend | 20 | trending_up | — |
| 3 | volume_breakout | 放量突破 | trend | 30 | trending_up | — |
| 4 | hot_theme | 热点题材 | framework | 35 | sector_hot | — |
| 5 | shrink_pullback | 缩量回踩 | trend | 40 | trending_down, sideways | `default_router` |
| 6 | event_driven | 事件驱动 | framework | 45 | sector_hot, volatile | — |
| 7 | box_oscillation | 箱体震荡 | framework | 50 | sideways | — |
| 8 | growth_quality | 成长质量 | framework | 55 | trending_up | — |
| 9 | bottom_volume | 底部放量 | reversal | 60 | trending_down | — |
| 10 | expectation_repricing | 预期重估 | framework | 65 | volatile, sector_hot | — |
| 11 | chan_theory | 缠论 | framework | 70 | volatile | — |
| 12 | wave_theory | 波浪理论 | framework | 80 | volatile | — |
| 13 | dragon_head | 龙头策略 | trend | 90 | sector_hot | — |
| 14 | emotion_cycle | 情绪周期 | framework | 100 | sector_hot | — |
| 15 | one_yang_three_yin | 一阳夹三阴 | pattern | 110 | **无配置** ⚠️ | — |

### 1.2 分类维度

**按官方 category（YAML 元数据）**

| 类别 | 数量 | 策略 |
|---|---|---|
| trend（趋势） | 5 | bull_trend、ma_golden_cross、volume_breakout、shrink_pullback、dragon_head |
| pattern（形态） | 1 | one_yang_three_yin |
| reversal（反转） | 1 | bottom_volume |
| framework（框架） | 8 | box_oscillation、chan_theory、wave_theory、emotion_cycle、event_driven、expectation_repricing、growth_quality、hot_theme |

**按市场环境（market_regimes，auto 路由匹配键）**

| 环境 | 策略 |
|---|---|
| trending_up | bull_trend、ma_golden_cross、volume_breakout、growth_quality |
| trending_down | shrink_pullback、bottom_volume |
| sideways | box_oscillation、shrink_pullback |
| sector_hot | hot_theme、event_driven、expectation_repricing、dragon_head、emotion_cycle |
| volatile | chan_theory、wave_theory、event_driven、expectation_repricing |
| 无配置 | one_yang_three_yin（auto 模式不可达） |

**按分析本质**

| 维度 | 策略 |
|---|---|
| 纯技术/价格（9） | bull_trend、ma_golden_cross、volume_breakout、shrink_pullback、one_yang_three_yin、bottom_volume、box_oscillation、chan_theory、wave_theory |
| 事件/信息（3） | event_driven、expectation_repricing、hot_theme |
| 基本面（1） | growth_quality |
| 情绪/行为（1） | emotion_cycle |
| 板块/相对强度（1） | dragon_head |

**按默认路由状态**

| 字段 | 配置 | 含义 |
|---|---|---|
| `default_active` | 仅 bull_trend | 未指定技能时的唯一默认策略 |
| `default_router` | bull_trend、shrink_pullback | fallback 技能池（auto 之外模式的兜底集合） |

### 1.3 运行机制背景

理解策略如何生效，需要了解以下链路：

1. **选择**（`src/agent/skills/router.py`）：auto 模式通过 `_detect_regime()` 从技术面推演市场状态（trending_up / trending_down / sideways / volatile / sector_hot），再按 `market_regimes` 匹配技能；手动模式按 `agent_skills` 配置；未指定时回退 `default_router` 池。
2. **执行**（`src/agent/skills/skill_agent.py`）：每个被选中的策略由一个独立 SkillAgent 评估，工具集由 `required_tools` 过滤——**未声明的工具对 SkillAgent 不可见**。
3. **输出契约**：SkillAgent 输出 JSON——`signal`（strong_buy/buy/hold/sell/strong_sell）、`confidence`（0-1）、`conditions_met/missed`、`score_adjustment`（-20~+20）、`reasoning`。
4. **融合**（`src/agent/skills/synthesis.py`）：`score_adjustment >= 8` 视为正向观点、`<= -8` 视为负向，进入多策略合成后由 DecisionAgent 汇总最终建议。

> 注意：策略指令中的 `` `sentiment_score` `` 与 SkillAgent 实际输出字段 `score_adjustment` 是同一语义，由 LLM 语义映射完成，系统内不冲突。

---

## 2. 逐策略深入分析

### 2.1 bull_trend（默认多头趋势）— priority 10

**定位**：全系统默认策略，承担"常规个股分析"的基线判断，`default_active` 与 `default_router` 双角色。

**判定标准**：
1. 趋势确认（最高优先级）：`analyze_trend` 判断 MA5≥MA10≥MA20 且 MA20 斜率向上；显著跌破 MA20 则降低看多权重。
2. 位置与节奏：优先"回踩不破"而非高位追涨；距 MA5/MA10 过远时提示等待回踩；放量突破有效阻力可提高胜率评级。
3. 量价验证：`get_daily_history` 检查突破日/反弹日是否放量；缩量上涨谨慎、放量滞涨警惕分歧。
4. 交易建议：明确"买入/观望/减仓"倾向与触发条件，必须给出止损参考（MA20 下方或结构低点）；无清晰优势时明确"暂不出手"。

**评分调整**：多头排列+趋势强度良好 `+12`；回踩关键均线企稳 `+8`；放量突破关键阻力 `+10`；跌破 MA20 或趋势转弱 `-12`。

**工具依赖**：`get_daily_history`、`analyze_trend`（已声明完整，且指令未引用其他工具，一致）。

**优势**：
- 逻辑稳健，趋势、位置、量价三要素齐全，止损要求明确；
- 与理念 1（严进）、2（趋势）、3（量能）对齐，是全系统评分基准的锚点。

**局限**：
- 不引用 `get_realtime_quote`，只用历史数据判断"当前"位置，对当日盘中实时变化不敏感；
- 无独立"输出要求"块（有步骤 4 交易建议输出，语义覆盖但结构不如箱体/波浪完整）。

**优化建议**：
- 作为默认策略，可补 `get_realtime_quote` 做当日价格/量比确认（需同步 required_tools）；
- 可选：补齐输出要求块，与其他框架类策略结构对齐。

---

### 2.2 ma_golden_cross（均线金叉）— priority 20

**定位**：经典趋势反转/延续信号，优先级第二高，仅次于默认策略。

**判定标准**：
1. 金叉检测：主信号 MA5 最近 3 个交易日内上穿 MA10；强信号 MA10 上穿 MA20；检查 MACD 是否金叉或零轴上方金叉。
2. 量能确认：金叉日成交量高于 5 日均量（`get_daily_history` 验证），量比 > 1.2 为积极信号。
3. 趋势背景分级：盘整后金叉=最强信号；上升趋势中金叉=延续信号；深度下跌中金叉=弱信号需更多确认。
4. 价格位置：价格应在交叉均线附近或上方，乖离率 < 7% 避免追高。

**评分调整**：MA5×MA10 金叉+量能 `+10`；MA10×MA20 金叉 `+8`；MACD 零轴上方金叉 `+5`。

**工具依赖**：`get_daily_history`、`analyze_trend`（一致）。

**优势**：信号定义精确（3 日内、双级别金叉、MACD 佐证），场景分级（盘整/上升/下跌）体现对信号质量的甄别。

**局限（🔴 环境配置矛盾）**：指令明确"**盘整后金叉：最强信号**"，但 `market_regimes` 只配置 `trending_up`——最强场景（sideways 盘整）在 auto 模式下永远匹配不到，策略最有效的用法被自己的环境配置屏蔽。

**优化建议**：`market_regimes: [trending_up, sideways]`。

---

### 2.3 volume_breakout（放量突破）— priority 30

**定位**：放量突破阻力位的顺势介入信号，与 ma_golden_cross 互补（金叉看均线交叉，突破看价格行为）。

**判定标准**：
1. 阻力位识别：`analyze_trend` → `resistance_levels`，通常为 20 日高点或前期震荡平台顶部。
2. 量能确认：当日成交量 > 5 日均量 2 倍；`get_realtime_quote` → `volume_ratio > 2.0`；`get_daily_history` 交叉验证。
3. 价格确认：收盘站上阻力位；收盘位于当日振幅上方 30%（强势收盘）；乖离率 < 7% 避免追高。
4. 后续验证：次日开盘在突破位之上，区分真/假突破。
5. 风险过滤：`search_stock_news` 无重大利空；PE 不应过高（避免泡沫型突破）。

**评分调整**：放量突破确认 `+12`；突破伴随板块共振 `+5`。买点设突破位附近，止损突破位下方 3%。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`、`search_stock_news`（已声明完整）。

**优势**：15 个策略中完成度最高之一——识别→量能→价格→次日验证→风险过滤五步闭环，假突破有量化确认机制（次日开盘验证），止损位明确（3%）。

**局限**：无明显缺陷。"次日验证"依赖后续交易日数据，当日分析时只能作为待确认条件。

**优化建议**：无实质问题。

---

### 2.4 hot_theme（热点题材）— priority 35

**定位**：政策/产业/资金热点场景的核心策略，sector_hot 环境下优先级最高（35）。

**判定标准**：
1. 热点强度：`get_sector_rankings` 判断板块涨幅/成交额/人气是否前列；观察热点从核心股向板块扩散；单股异动、板块未共振则降低信号权重。
2. 个股相关性：`search_stock_news` 核实业务/订单/产能/客户/公告与热点的直接关联；区分"实质受益/间接受益/概念关联较弱"。
3. 相对强弱：个股涨幅、量比、换手率是否强于板块平均；强势热点股回调不破关键均线。
4. 节奏与风险：不在连续加速、高乖离率位置追涨；新闻集中在"已大涨/资金追捧/游资博弈"时警惕短线情绪顶；重大利空、监管问询、澄清公告一票降级。

**评分调整**：热点启动/扩散期+实质受益 `+12`；强于板块+量能确认 `+6`；分化或退潮 `-8`；蹭概念+乖离过高 `-12`。

**工具依赖**：`get_sector_rankings`、`search_stock_news`、`get_realtime_quote`、`analyze_trend`（一致）。

**优势**：热点阶段划分（启动/扩散/分化/退潮）是全场最完整的题材生命周期模型；"实质受益 vs 蹭概念"的甄别直击题材炒作的核心风险；与 dragon_head 形成"选板块→选个股"的递进关系。

**局限**：热点生命周期判断高度依赖新闻时效性，`search_stock_news` 结果质量直接决定策略效果。

**优化建议**：无实质问题。

---

### 2.5 shrink_pullback（缩量回踩）— priority 40，`default_router`

**定位**：上升趋势中的低吸策略，与 default_active 的 bull_trend 共同构成 fallback 池，是"非追高"买点偏好的代表。

**判定标准**：
1. 前提：股票处于上升趋势（MA5 > MA10 > MA20，`analyze_trend` 确认多头排列）。
2. 回踩检测：价格回踩 MA5（误差 1% 内）或 MA10（误差 2% 内）；回调期成交量 < 5 日均量 70%；`volume_status` 应显示缩量。
3. 反弹信号：当前价格守住均线支撑；MA5 乖离率 < 2% 为最佳买入区间。
4. 确认条件：`search_stock_news` 无利空；`get_chip_distribution` → `profit_ratio` 在 50-80% 区间。

**评分调整**：缩量回踩 MA5 `+10`；回踩 MA10 且量能 < 0.6 倍均量 `+8`。理想买点 MA5、次优 MA10，止损 MA20。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`、`get_chip_distribution`、`search_stock_news`（已声明完整，2026-08 补齐筹码工具）。

**优势**：入场条件量化最细的策略之一（1%/2% 误差带、70% 缩量线、2% 乖离买点），是"理念 4 买点偏好"的执行范本。

**局限（🟡 阈值不一致）**：入场条件写"成交量 < 5 日均量 **70%**"，评分调整写"量能 < **0.6 倍** 均量 `+8`"——同一指标两个值，LLM 执行时语义边界模糊。

**优化建议**：统一阈值表述（70% 为入场线，0.6 倍为更强加分档需明示），消除二义性。

---

### 2.6 event_driven（事件驱动）— priority 45

**定位**：围绕业绩、政策、并购、订单、产品发布等事件的催化强度评估，sector_hot 与 volatile 双环境。

**判定标准**：
1. 事件分类：`search_stock_news` 梳理近期事件，分业绩/政策/订单产品/资本运作/监管风险五类；过期或时间未知的信息不作主要依据。
2. 影响路径：判断影响收入、利润率、估值、融资能力、市场份额还是仅情绪；重大订单/政策利好要说明兑现周期与不确定性；监管、减持、处罚、诉讼风险优先。
3. 市场反应：`get_realtime_quote` + `analyze_trend` 判断事件是否已被价格充分反映；放量上涨未过阻力可等确认；高位放量滞涨或冲高回落警惕兑现压力。
4. 交易计划：事件未兑现前强调仓位控制与时间窗口；兑现后重新评估"预期交易→业绩验证"切换；负面事件先看风险释放充分度。

**评分调整**：高可信正向事件+未充分反映 `+14`；正向事件已大幅兑现 `-6`；负面事件仍在发酵 `-15`；信息冲突时维持中性并降置信度。

**工具依赖**：`search_stock_news`、`get_realtime_quote`、`analyze_trend`（一致）。

**优势**：唯一强制要求"操作建议必须包含**失效条件**"的策略（公告不及预期、跌破关键支撑、热度消退），风险控制意识全场最强；事件分类与影响路径分析结构化程度高。

**局限**：依赖新闻检索的时效与覆盖；"已反映程度"判断依赖 LLM 对量价的主观解读，无量化锚点。

**优化建议**：无实质问题。

---

### 2.7 box_oscillation（箱体震荡）— priority 50

**定位**：横盘行情的波段策略，sideways 环境下与 shrink_pullback 共同覆盖。

**判定标准**：
1. 箱体识别：`get_daily_history` 60~120 日数据；顶/底各至少触碰 2-3 次确认；`analyze_trend` 的 support/resistance_levels 辅助定位。
2. 当前位置：`get_realtime_quote` 现价对比边界——箱底区域（距支撑 ≤5%）买入，箱中（中间 1/3）观望，箱顶区域（距阻力 ≤5%）减仓。
3. 量能辅助：箱底放量企稳=强信号可较重仓；箱顶缩量滞涨=卖出信号；放量超均量 2 倍突破→转趋势策略（新目标=箱体高度延伸）。
4. 箱体宽度：< 5% 不参与；5%~15% 标准波段；> 15% 大箱体。
5. 假突破识别：单日触及后快速回撤收盘回箱内=假突破；连续两日收盘突破+放量=真突破改策略。

**评分调整**：箱底企稳+缩量 `+10`；箱底放量攻顶 `+12`；向上有效突破 `+15` 转趋势；箱顶区域 `-5`（不追高）；箱底有效跌破 `-15` 离场。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`（一致）。

**优势**：全场结构最完整——输出要求块、宽度分级、假突破量化标准（连续两日收盘+放量）齐备，是框架类策略的范本。

**局限**：箱体识别（触碰次数、边界取值）依赖 LLM 主观解读 K 线，无程序化箱体计算工具支撑；"转趋势策略"的衔接未引用 volume_breakout 的突破标准，两策略标准可能不一致。

**优化建议**：可选——明确突破后衔接 volume_breakout 的量化标准（2 倍量、收盘站上、次日验证），避免同一"突破"在两个策略中标准漂移。

---

### 2.8 growth_quality（成长质量）— priority 55

**定位**：唯一的基本面主导策略，弥补全系统"技术分析为主、基本面薄弱"的短板。

**判定标准**：
1. 成长性：财报字段中的营业收入、归母净利润、经营现金流、ROE；判断收入/利润是否同向、"增收不增利"风险；只有概念热度无财报验证则降低确定性。
2. 质量：ROE 越高且稳定越好；经营现金流与净利润方向一致；现金流显著弱于利润时提示回款/存货/应收风险。
3. 估值承受力：PE/PB/市值判断是否透支成长；高成长可承受高估值但需说明增长能否覆盖。
4. 趋势确认：`analyze_trend` 判断长期成长逻辑是否被资金确认；基本面好但技术面未确认时给观察条件而非直接追买。

**评分调整**：收入/利润/现金流/ROE 同向改善 `+15`；行业景气与新闻互证 `+6`；高估值成长未验证 `-8`；增收不增利或现金流恶化 `-12`。

**工具依赖**：`get_stock_info`、`get_realtime_quote`、`search_stock_news`、`analyze_trend`（一致）。

**优势**：四维基本面评估（成长/质量/估值/趋势确认）逻辑严密，"现金流方向"与"增收不增利"是识别财务粉饰的有效信号。

**局限（🟡 两个）**：
1. **数据契约风险**：指令要求的财报字段依赖 `get_stock_info` → `fundamental_context` 的 `growth`/`earnings` blocks，这两个 block 的覆盖取决于数据源且可能 `status: failed`；指令没有降级指引，LLM 拿不到数据时存在编造风险。
2. **环境配置矛盾**：指令买点包括"回踩长期均线、估值回落"——对应 sideways/trending_down 场景，但 `market_regimes: [trending_up]`，回踩买点场景在 auto 模式匹配不到。

**优化建议**：`market_regimes: [trending_up, sideways]`；指令补充"基本面字段缺失或 status=failed 时降级为技术面判断并降低置信度"。

---

### 2.9 bottom_volume（底部放量）— priority 60

**定位**：全仓库唯一的反转类策略，趋势下跌环境中捕捉底部反转信号。

**判定标准**：
1. 持续下跌确认：`get_daily_history` 30 日，20 日高点至近期低点跌幅 > 15%；`trend_status` 应为 BEAR 或 STRONG_BEAR。
2. 量能异动：当日成交量 > 5 日均量 3 倍；`get_realtime_quote` → `volume_ratio > 3.0`；异动应出现在前期极度缩量之后。
3. 价格企稳：当日收阳（收盘 > 开盘）；守住近期低点；最好长下影线显示买方支撑。
4. 确认因素：`search_stock_news` 基本面催化；`get_chip_distribution` → `avg_cost` 接近现价（成本收敛）。
5. 风险提示：反转信号风险高于趋势跟踪；仓位最多 2-3 成；止损严格（近期低点下方）。

**评分调整**：底部放量确认 `+8`；配合阳线+新闻催化 `+5`。止损近期低点。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`、`get_chip_distribution`、`search_stock_news`（2026-08 已补齐 3 个缺口并接入筹码工具）。

**优势**：逻辑链条完整（下跌确认→缩量→放量→阳线企稳→新闻/筹码确认→仓位与止损约束），风险提示段明确"反转信号风险高于趋势跟踪"，认知诚实。

**局限（🟡 三个）**：
1. **主信号加分全仓库最低（+8）**，相比 volume_breakout +12、emotion_cycle +14，反转信号即使完全成立对最终决策的影响也偏弱；
2. **量比 > 3.0 阈值偏严**——比放量突破的 2.0 还严格 50%，真实底部反转多为温和放量，3 倍会漏掉大量有效信号；
3. 无假信号排除机制（高开低走放量、单日脉冲后缩量、量比 > 8 对倒）与输出要求块。

**优化建议**：量比阈值降为 2.5 并分层（> 5 倍为强信号额外加分）；主信号加分提升至 +10~12；补假信号排除段与输出要求块。

---

### 2.10 expectation_repricing（预期重估）— priority 65

**定位**：预期差交易框架，与 event_driven 互补（事件驱动看催化本身，本策略看"预期 vs 现实"的差值）。

**判定标准**：
1. 预期来源：`search_stock_news` 识别改变市场预期的信息；区分硬信息（公告/财报/订单）与软信息（传闻/观点/情绪）。
2. 预期差方向：正向预期差=悲观背景下好于预期；负向预期差=乐观背景下低于预期；已被连续大涨充分反映时提示兑现风险。
3. 估值重估：`get_stock_info` 的 PE/PB、市值、ROE、现金流判断重估是否有基本面支撑；估值提升需匹配盈利质量、增长持续性、行业空间；回落时区分一次性扰动 vs 长期逻辑变化。
4. 价格确认：`analyze_trend` 判断预期是否转化为趋势；放量突破=资金确认，缩量反弹=修复观察；高位放量滞涨、利好不涨、跌破关键支撑=预期转弱。

**评分调整**：正向预期差+未充分反映 `+15`；已被连续大涨兑现 `-5`；负向预期差或核心假设证伪 `-15`；信息不充分但存在潜在修复时中性+降置信度。

**工具依赖**：`search_stock_news`、`get_stock_info`、`get_realtime_quote`、`analyze_trend`（一致）。

**优势**：硬/软信息区分是全场独有的信息质量甄别维度；"预期差方向+兑现程度"双轴判断框架清晰；观察点输出（下一份财报、订单兑现、政策落地）可执行性强。

**局限**：同 growth_quality——ROE/现金流等字段依赖 `get_stock_info` 数据覆盖，无降级指引。

**优化建议**：指令补充数据缺失时的降级说明（与 growth_quality 同款问题）。

---

### 2.11 chan_theory（缠论）— priority 70

**定位**：技术分析中结构最严谨的框架之一，volatile 环境下的结构定位工具。

**判定标准**：
1. 中枢识别：`get_daily_history` 60 日数据，识别连续 3 段走势重叠的中枢；判断震荡中枢 vs 脱离中枢的趋势段。
2. 背驰判断（最高优先级）：顶背驰=价格新高但 MACD 红柱面积缩小→卖出；底背驰=价格新低但绿柱面积缩小→买入。
3. 买卖点：一买（下跌趋势末中枢底背驰，最强）、二买（离开中枢后回调不破中枢高点）、三买（中枢后向上突破不回中枢）；一/二/三卖对称。
4. 级别与仓位：日线级别 30-50%、周线级别 50-80%；多级别共振信号最强。
5. 输出：当前趋势状态、背驰信号与级别、买卖点类型（无则明写"暂无明确买卖点"）、止损（前低/前高）。

**评分调整**：底背驰+一买 `+15`；二/三买共振 `+10`；中枢震荡维持基准；顶背驰/趋势向下 `-15`。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`（一致）。

**优势**：全场唯一按"级别"给出仓位管理的策略；输出要求明确要求"无买卖点就明说"，防过度交易意识强；结构分析（中枢/背驰/买卖点）层次清晰。

**局限（🟡 工具契约限制）**：背驰判断要求比较 MACD 柱**面积的历史序列**，但 `analyze_trend` 只返回**当前** `macd_bar` 单个值，`get_daily_history` 不含 MACD。LLM 无法精确比较柱面积，只能近似推断，存在幻觉风险。

**优化建议**：指令明确"用当前 MACD 柱值与价格高低点做近似背驰判断，并降低置信度"，或将背驰判断标注为辅助信号。

---

### 2.12 wave_theory（波浪理论）— priority 80

**定位**：volatile 环境下的大局观定位工具，判断所处浪型与目标价。

**判定标准**：
1. 浪型识别（120 日数据）：推动浪特征（1 浪温和放量、3 浪最强绝不为最短、5 浪量能弱于 3 浪且顶背离）；调整浪特征（A 浪下跌量大、B 浪弱反弹陷阱、C 浪力度超 A）。
2. 黄金位置：2 浪回调 38.2%~61.8%；3 浪目标 1.618~2.618 倍 1 浪；4 浪不得进入 1 浪价格区域（违反规则需重新归数）；C 浪目标 ≥ A 浪长度。
3. 最优买点：2 浪回调企稳（最安全，止损 1 浪起点）> 4 浪回调企稳（止损 1 浪顶部）> 3 浪初期突破；避免 5 浪末端追高。
4. 风险提示：B 浪不宜重仓；波浪计数主观，需其他指标验证。
5. 输出：当前浪型位置、斐波那契支撑/阻力位、买卖时机判断、计数置信度（高/中/低）。

**评分调整**：2 浪底企稳 `+15`；3 浪突破 `+12`；5 浪末端/顶背离 `-10`；C 浪下跌中 `-12`。

**工具依赖**：`get_daily_history`、`analyze_trend`、`get_realtime_quote`（一致）。

**优势**：少数强制输出**置信度**的策略；"4 浪侵入 1 浪需重新归数"的规则自纠机制体现了对主观性缺陷的自觉；买点按浪型分级+止损位绑定浪结构，可执行性强。

**局限**：波浪计数主观性是理论固有属性，已通过置信度输出缓解；120 日窗口对刚上市或数据不足的标的会受限。

**优化建议**：无实质问题。

---

### 2.13 dragon_head（龙头策略）— priority 90

**定位**：板块轮动中的个股选择器，sector_hot 环境下承接 hot_theme 之后的"选股"环节。

**判定标准**：
1. 板块领涨：`get_sector_rankings` 检查所在板块是否涨幅前列；确认是否板块启动周期中率先上涨或涨停。
2. 换手与动能：换手率 > 5%、量比 > 1.5 说明活跃交易兴趣。
3. 相对强度：个股涨跌幅对比板块均值，上涨日跑赢板块 2% 以上。
4. 新闻催化：`search_stock_news` 板块级催化剂（政策/事件/业绩）。
5. 乖离率检查（已随理念 1 调整）：龙头可放宽至 10%，超过 13% 仍需谨慎。

**评分调整**：确认为龙头 `+10`；板块主动轮动期 `+5`。

**工具依赖**：`get_realtime_quote`、`get_sector_rankings`、`search_stock_news`（一致）。

**优势**：唯一做"相对强度"量化（跑赢板块 2%）的策略；换手率/量比阈值与 emotion_cycle 的换手率体系（> 5% 高热度）互相印证，全系统阈值自洽；龙头放宽梯度（基础 7% → 龙头 10%）与理念 7 对齐。

**局限**：无。

**优化建议**：无实质问题。

---

### 2.14 emotion_cycle（情绪周期）— priority 100

**定位**：全系统唯一的情绪/行为维度策略，逆情绪布局（恐慌底买入、狂热顶离场）。

**判定标准**：
1. 换手率分层：< 0.5%/日=潜在底部；0.5%~2%=正常；2%~5%=活跃不追高；> 5%=高热度警惕情绪顶；> 10%=极度过热通常短期顶部。
2. 连续换手率走势（20 日）：由高向低+缩量=退潮等待；由低向高+放量=情绪启动可介入；单日暴量超前期 5 倍=疑似出货警惕。
3. 新闻情绪面：利好兑现/机构推荐集中=可能过热；利空/破位集中=悲观可能造就底部；散户情绪极端负面=反向指标。
4. 均线与波动率：三线粘合=蓄势待定；ATR 低位=蓄势爆发前兆。
5. 情绪底部特征（满足 3 项以上买入区）：换手率近一年低位、缩量至 60 日均量 50% 以下、新闻低调/负面、股价 MA20 附近无恐慌暴跌、机构持仓稳定。
6. 情绪顶部特征（满足 3 项以上减仓区）：5 日换手率 > 20 日均值 2 倍、成交量脉冲放大、利好兑现/目标价上调/散户追捧、偏离 MA5 超 8%、MACD 顶背离。

**评分调整**：底部特征 3 项+ `+14`、5 项全中 `+20`；顶部特征 3 项+ `-12`、5 项全中 `-20`；平稳区间不调整。全系统最完整的评分梯度，也是唯一有 `+20` 满分项的策略。

**工具依赖**：`get_daily_history`、`get_realtime_quote`、`analyze_trend`、`search_stock_news`（一致）。

**优势**：换手率分层+底部/顶部特征清单是全系统最详尽的量化体系；"5 项全中 ±20"的极端分数设置体现了对极端情绪反转的重视；反身性思想（大众恐慌我贪婪）表述清晰。

**局限（🔴 环境配置矛盾 + 🟡 数据基准）**：
1. **核心价值被屏蔽一半**：`market_regimes: [sector_hot]`——但"恐慌底部买入"场景对应 trending_down/volatile，而非 sector_hot。当前配置下只在热点火爆时被选中，用的恰是"顶部警惕"那一半；"底部买入"这一核心价值在 auto 模式永远用不上。
2. 换手率分层需要"近一年换手率均值"（约 250 日数据），指令未写明 `get_daily_history` 的 `days=250` 参数，LLM 默认 60 天会算错基准。

**优化建议**：`market_regimes: [sector_hot, trending_down, volatile]`；指令写明 `days=250` 数据要求。

---

### 2.15 one_yang_three_yin（一阳夹三阴）— priority 110

**定位**：全仓库唯一的 K 线组合形态策略，识别上升趋势中的整理结束信号。

**判定标准**（最近 5 个交易日）：
1. 第 1 日：大阳线（收 > 开，实体 > 股价 2%）。
2. 第 2-4 日：连续三根阴线或小 K——最低价不破第 1 日开盘价；成交量逐步萎缩（量比 < 0.8）；收在第 1 日实体范围内。
3. 第 5 日：又一根阳线，收盘突破第 1 日收盘价。
4. `analyze_trend` 确认多头排列（MA5 > MA10 > MA20）。

**评分调整**：形态成立+趋势看多 `+15`；形态成立但趋势不明 `+5`。买点第 5 日收盘附近，止损第 1 日开盘价下方。

**工具依赖**：`get_daily_history`、`analyze_trend`（一致）。

**优势**：形态定义精确到逐日（实体幅度、量比阈值、不破位条件），止损位锚定形态结构（第 1 日开盘价），是 pattern 类的规范写法。

**局限（🔴 环境配置缺失）**：全仓库唯一**没有 `market_regimes`** 的策略——auto 模式 `_detect_regime()` 匹配不到任何技能，永远不被选中；priority 110 也是全场最低。目前只能通过手动指定技能或 fallback 池外的方式使用。

**优化建议**：补 `market_regimes: [sideways, trending_up]`（整理形态常见于盘整或上升中继）。可选：形态检查可借助 `analyze_pattern` 工具（系统中存在），但该形态过于特定，LLM 按逐日条件手动比对更灵活，不强制。

---

## 3. 横切共性问题

| # | 问题 | 涉及策略 | 严重度 |
|---|---|---|---|
| 1 | **market_regimes 与核心买点场景矛盾**——策略最有效的场景在 auto 模式匹配不到 | ma_golden_cross（盘整后金叉）、emotion_cycle（恐慌底）、growth_quality（回踩买点） | 🔴 高 |
| 2 | **缺 market_regimes**，auto 模式不可达 | one_yang_three_yin | 🔴 高 |
| 3 | 同一指标阈值表述不一致 | shrink_pullback（70% vs 0.6 倍） | 🟡 中 |
| 4 | 工具只给当前值，策略需要历史序列 | chan_theory（MACD 柱面积背驰） | 🟡 中 |
| 5 | 数据源可能无数据但指令无降级指引 | growth_quality（财报字段）、expectation_repricing（ROE/现金流）、emotion_cycle（250 日基准未写明） | 🟡 中 |
| 6 | 主信号加分幅度无全局基准（+8~+20 差距 2.5 倍） | bottom_volume（+8 最低）vs emotion_cycle（+20 最高） | 🟡 中 |
| 7 | 输出要求块不统一（10 个有，5 个无） | bull_trend、ma_golden_cross、bottom_volume、dragon_head、volume_breakout | 🔵 低 |
| 8 | `` `sentiment_score` `` 命名 vs SkillAgent 实际输出字段 `score_adjustment`，靠 LLM 语义映射 | 全部 15 个（能用，但存在歧义空间） | 🔵 低 |

## 4. 优化建议优先级

**P0（影响策略可达性，建议优先）**：
1. 修 3 个策略的 `market_regimes` 矛盾：ma_golden_cross 加 `sideways`、emotion_cycle 加 `trending_down, volatile`、growth_quality 加 `sideways`
2. one_yang_three_yin 补 `market_regimes: [sideways, trending_up]`

**P1（影响信号质量）**：
3. bottom_volume 量比阈值放宽/分层（2.5/5）+ 假信号排除 + 主信号加分提升
4. shrink_pullback 缩量阈值表述统一
5. chan_theory、growth_quality、expectation_repricing 补数据降级指引；emotion_cycle 写明 `days=250`

**P2（一致性打磨）**：
6. 统一输出要求块结构（5 个缺失的策略补齐）
7. 主信号评分幅度全局基准梳理
8. `sentiment_score`/`score_adjustment` 命名对齐评估
