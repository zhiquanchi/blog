# PRD Skill 对比分析表（31款）

> 共 31 款 PRD 相关 AI Skill / Agent 工具的主要作用与优缺点对比 · 2026年8月

## 完整对比表

| # | Skill 名称 | 类型 | 角色数 | 主要作用 | 优点 | 缺点 |
|---|-----------|------|--------|---------|------|------|
| 1 | [check-prd-skill](https://github.com/Liquid-erigeronphiladelphicus552/check-prd-skill) | 评审型 | — | 对B端PRD或系统设计文档进行14维度严格质量审查，覆盖业务分析、场景设计、数据建模、交互设计、商业分析等，输出P0-P3分级问题清单 | ✓ 14个审查维度极其全面，针对B端/SaaS/企业系统有差异化规则<br>✓ 每条发现锚定PRD具体位置，并给出可直接执行的改进示例<br>✓ 有明确输出规范和最低质量标准，保证审查深度一致性 | ✗ 重度依赖大量reference文件，脱离完整仓库效果大打折扣<br>✗ 14个维度逐一输出，流程冗长，用户易疲劳<br>✗ 主要面向B端/企业级，C端产品适用性有限 |
| 2 | [ralph / PRD Generator](https://github.com/snarktank/ralph/blob/main/skills/prd/SKILL.md) | 生成型 | — | 通过3-5个选项式澄清问题快速收集信息，生成含用户故事、功能需求、非目标等的结构化PRD，保存为Markdown | ✓ 流程简洁高效，选项式澄清交互成本极低<br>✓ PRD结构完整，含用户故事、验收标准、成功指标等核心章节<br>✓ 面向初级开发者/AI Agent优化，表述明确、需求编号便于引用 | ✗ 仅做生成，无内置审查/迭代机制，质量依赖单次输出<br>✗ 澄清问题固定3-5个，复杂需求覆盖不足<br>✗ 缺少与后续开发流程的衔接（任务拆分、技术设计等） |
| 3 | [TestAny prd-reviewer](https://github.com/TestAny-io/testany-agent-skills/blob/main/plugins/testany-eng/skills/prd-reviewer/SKILL.md) | 评审型 | — | PRD进入HLD阶段的"守门人"，从结构完整性、业务逻辑、可测试性等10大维度360度审查，按P0/P1/P2分级，通过则颁发准出证书 | ✓ 审查标准严格，有明确准出/不准出判定规则<br>✓ 强制校验追溯元数据，可通过脚本自动化检查<br>✓ 支持1:N拆分场景审查（BRD拆多PRD的覆盖率验证） | ✗ 重度依赖TestAny生态的reference和脚本，独立使用困难<br>✗ 审查维度过多，执行时间长，小型PRD可能过度审查<br>✗ 只做审查不做修改，发现问题后闭环能力弱 |
| 4 | [TestAny prd-studio](https://github.com/TestAny-io/testany-agent-skills/blob/main/plugins/testany-eng/skills/prd-studio/SKILL.md) | 全流程型 | 2 | PRD全流程编排器，协调Writer和Reviewer两个隔离subagent，自动完成"写→审→改→审"循环，最多3轮迭代直到准出通过 | ✓ 全自动工作流，Writer与Reviewer上下文隔离，避免互相污染<br>✓ 支持BRD到多PRD的1:N拆分，含拆分评估和覆盖率验证<br>✓ 迭代过程透明，每轮有状态记录和审查历程 | ✗ 架构复杂，依赖subagent机制和大量模板，部署调试成本高<br>✗ 最多3轮迭代硬限制，复杂需求可能遗留P0/P1问题<br>✗ 完全自动流转缺人工干预，Writer理解偏差可能越走越偏 |
| 5 | [TestAny prd-writer](https://github.com/TestAny-io/testany-agent-skills/blob/main/plugins/testany-eng/skills/prd-writer/SKILL.md) | 生成型 | — | 专业PRD写作助手，写作前强制扫描项目上下文（已有PRD/HLD/API文档等），基于证据撰写符合约定的PRD，严格区分What/Why与How边界 | ✓ 上下文收集严谨，先扫描再确认后读取，确保与现状一致<br>✓ 严格内容边界控制，明确PRD不应包含API路径、表结构等HLD内容<br>✓ 支持5种PRD类型，内置BRD拆分评估和User Journey映射 | ✗ 前置流程过重（多阶段上下文收集），简单需求效率低<br>✗ 强制嵌入追溯元数据，增加写作复杂度且依赖外部schema<br>✗ "先读后写"需大量文件扫描，大型项目中可能耗时较长 |
| 6 | [prd-debate](https://github.com/betseyliu/prd-debate) | 辩论对抗型 | 3 | 跨模型对抗式辩论skill，Host（Claude主持）+ Proposer（GPT提案）+ Reviewer（Claude子代理审查）三角色结构化辩论，产出高质量PRD | ✓ 跨模型对抗设计独特，不同模型能发现彼此盲区<br>✓ 内置回溯校验算法，自动检查新结论是否破坏前期共识<br>✓ 支持可选代码库调研，使产品设计 grounded in 实际代码现状 | ✗ 依赖两个不同模型（Claude + GPT），部署门槛高、成本高<br>✗ 6个阶段每阶段2-5轮辩论，生成周期长<br>✗ 仅聚焦产品视角（What/Why），不涉及技术实现 |
| 7 | [requirement-review](https://github.com/LunaLovegood76/requirement-review) | 评审型 | 4 | 模拟真实互联网公司需求评审会，扮演技术负责人、交互设计师、测试工程师、业务方四个角色轮番质疑挑战，帮助PM会前发现盲区 | ✓ 四角色模拟真实感强，各角色有人设、语气和独立关注点<br>✓ 交互式设计好，每次只问一个问题，节奏可控<br>✓ 输出健康度仪表盘和结构化评审报告，行动项明确 | ✗ 依赖大量人工交互，完整评审可能需要很多轮对话<br>✗ 审查深度受限于模型自身知识储备，特定行业专业度不足<br>✗ 更偏向"训练/模拟"定位，批量自动化审查能力弱 |
| 8 | [goal-workflow / PRD](https://github.com/smallnest/goal-workflow/blob/master/skills/prd/SKILL.md) | 全流程型 | — | 功能增强版PRD生成器，根据需求复杂度动态调整澄清问题数量（简单2-3/典型3-5/复杂6-8），支持后续衔接技术设计和任务拆分 | ✓ 澄清问题数随复杂度动态调整，更灵活合理<br>✓ 内置8种fallback场景处理，健壮性好<br>✓ 与下游工作流衔接顺畅，可直接转技术设计和任务拆分 | ✗ 单轮生成+用户审阅模式，缺内置自动审查和迭代修正<br>✗ 无项目上下文扫描能力，PRD可能与项目现状脱节<br>✗ 对BRD拆分、多PRD协作等复杂场景缺乏支持 |
| 9 | [copilot-chat-modes / PRD](https://github.com/LorcanChinnock/copilot-chat-modes/blob/master/prd.chatmode.md) | 生成型 | — | GitHub Copilot的PRD聊天模式，以资深PM+架构师角色，按"接收→澄清→调研→起草→迭代→定稿"流程，输出严格JSON schema的结构化PRD | ✓ 输出严格JSON格式，九大模块，机器可读性强<br>✓ 深度集成GitHub Copilot工具链（20+工具），可基于代码库生成<br>✓ 丰富控制短语，用户可灵活控制交互节奏和粒度 | ✗ 强依赖GitHub Copilot Chat环境，其他平台无法直接使用<br>✗ JSON schema庞大复杂，简单功能也生成大量字段<br>✗ 技术部分写得过深，可能越界到HLD/技术设计范畴 |
| 10 | [Taya technical-pm 系列](https://github.com/TayaIntelligence/skills) | 全流程型 | — | 含3个skill：technical-pm（技术型PM全流程）、prd-review（PRD完备性审查）、technical-pm-guided（苏格拉底式教练教学），覆盖从发现到交付完整PM工作流 | ✓ 一个skill覆盖8大PM工作流，功能全面<br>✓ 内置AI/ML产品专用视角（评估集、阈值、精确率/召回率等）<br>✓ 支持多种优先级框架（RICE/MoSCoW/Kano/WSJF等）和估算方法 | ✗ 单个skill承载8种工作流，上下文可能不够聚焦<br>✗ PRD深度审查能力较弱，更偏向"写"而非"审"<br>✗ reference文件较多，按需加载增加执行复杂度 |
| 11 | [multi-agent-prd-reviewer](https://github.com/dimospapadopoulos/multi-agent-prd-reviewer) | 评审型 | 4 | 4个专业化Agent（验证器、技术怀疑论者、UX审查员、法务合规）串行协作，从完整性、技术可行性、UX、合规性四维度输出结构化评审结果 | ✓ 多Agent专业化分工，覆盖技术、UX、法务等多维度<br>✓ 支持Slack Bot集成和多文件格式，接入协作流程方便<br>✓ 规则验证+AI批判混合模式，评审效率从30分钟降至60秒 | ✗ 串行流水线架构，token消耗较高（约3500 tokens/次）<br>✗ 依赖Anthropic Claude API，成本和可用性受第三方约束<br>✗ Python项目部署需技术门槛，非开箱即用SaaS |
| 12 | [prd-creator-skill](https://github.com/huchi996/prd-creator-skill) | 生成型 | 多角色 | 6阶段SOP工作流（需求孵化→方案预研→项目构建→模块构建→功能细化→多角色评审），三层文档结构（项目P→模块M→功能F）创建标准化PRD | ✓ 三层结构（P-M-F）文档组织清晰，命名规范统一<br>✓ 6阶段工作流完整覆盖从想法到评审全过程<br>✓ 支持多角色评审（产品/研发/测试/设计/运维），内置检查清单 | ✗ 流程较重，6个阶段对小型需求可能过于繁琐<br>✗ 主要面向Kimi CLI设计，其他平台适配性未明确<br>✗ 依赖mermaid图表渲染，环境不支持则图表功能受限 |
| 13 | [prd-meeting-review](https://github.com/bert995/prd-meeting-review) | 评审型 | — | 将飞书PRD云文档与飞书妙记（会议逐字稿）结合审阅，基于会议已确认结论，自动识别PRD中的缺失项、冲突点和待决策项，可写回文档 | ✓ 以会议逐字稿为证据来源，评审结论可追溯性强<br>✓ 写回机制安全严谨，多次人工确认避免误改<br>✓ 支持评论、block替换、黄色建议块等多种写回方式 | ✗ 强依赖飞书生态（文档+妙记），非飞书用户无法使用<br>✗ 仅聚焦会议结论与PRD一致性，不覆盖技术/UX等其他维度<br>✗ 写回前需多次人工确认，自动化程度有限 |
| 14 | [claude-plugin-prd-workflow](https://github.com/Yassinello/claude-plugin-prd-workflow) | 全流程型 | 多角色 | Claude Code的完整PRD工作流管理插件，覆盖想法→PRD→评审→编码→质量→发布全生命周期，集成AI评审、Git worktree编排、自动化质量门禁 | ✓ 功能极为全面，延伸到代码实现、安全扫描、进度跟踪<br>✓ 支持双模式（完整工作流/快速发布），适配不同规模需求<br>✓ Git worktree强制隔离，支持多特性并行开发 | ✗ 体积庞大（18条命令、17个Agent、13个Skill），学习曲线陡峭<br>✗ 强绑定Claude Code平台，其他IDE/Agent无法直接使用<br>✗ 推荐"超宽松权限"配置，存在安全和数据风险 |
| 15 | [ai-product-manager-skills](https://github.com/PANGKAIFENG/ai-product-manager-skills) | 全流程型 | 13个Skill | 中文优先的AI PM Skill库，含13个独立Skill（协作校准、竞品分析、PRD起草、PRD评审、拆Issue、UI线框、方案拷问等），覆盖从脑暴到交付全链路 | ✓ 13个Skill覆盖PM全工作链路，从问题定义到研发交付闭环<br>✓ 支持Loop Extension（决策循环、研究雷达等），适合复杂迭代<br>✓ 有明确路由规则，避免多Skill混淆，与Superpowers体系衔接好 | ✗ Skill数量多，安装配置复杂，初学者难快速选对<br>✗ 跨Skill状态同步和数据流转依赖handoff，可能有信息损耗<br>✗ 中文优先，英文文档和示例薄弱，国际化团队有障碍 |
| 16 | [prd-generator](https://github.com/millerzhang0723-cmd/prd-generator) | 生成型 | — | 将原型图和需求说明转化为可评审的Markdown PRD，优先产出功能级需求详述，强调可实现、可测试、可验收，支持按页面类型和AI/策略类需求生成 | ✓ 工程化导向，输出偏技术细节（字段、约束、异常、验收要点）<br>✓ 不臆造无依据信息，未知项明确标记"待确认"，诚实性好<br>✓ 支持按页面类型（列表/表单/看板等）和AI类需求定制化生成 | ✗ 功能相对单一，仅聚焦PRD生成，不含评审/迭代/项目管理<br>✗ 依赖原型图作为输入，没有原型时生成质量可能受限<br>✗ 仓库结构简单，缺少完整示例和文档说明 |
| 17 | [prd-to-reviewable-prototype](https://github.com/jonathankong12138-ai/skill-prd-to-reviewable-prototype) | 生成型 | — | 将早期产品想法、简略PRD转化为可运行、可浏览器分享的交互式原型评审工作台，支持可点击屏幕、用户流视角、评审评论和数据导出 | ✓ 从文字PRD跨越到可交互原型，评审者能直观体验产品<br>✓ 产物为单文件HTML，无需Node.js即可分享，分发便利<br>✓ 内置评审评论系统和review-notes.json导出，协作友好 | ✗ 原型基于Vite+React+TS+Tailwind生成，对运行环境有技术栈要求<br>✗ 主要聚焦UI交互原型，对业务逻辑、数据模型覆盖不足<br>✗ 生成的原型为静态/模拟交互，与真实后端无连接 |
| 18 | [awesome-prd-craft-skill](https://github.com/jiekingwu/awesome-prd-craft-skill) | 辩论对抗型 | 2 | 纯Prompt工程驱动的PRD生成与审查，内置"产品经理"和"用户视角审查官"双角色对弈，六维度量化评分，最多5轮循环打磨质量 | ✓ 双角色对抗性审查模拟真实团队博弈，避免AI自说自话<br>✓ 六维度量化评分使迭代有明确度量标准<br>✓ 纯Prompt工程实现，可在任何支持Markdown的AI对话工具中使用 | ✗ 单模型模拟双角色时，上下文隔离效果可能不彻底<br>✗ 最多5轮循环可能消耗大量token，成本高且耗时<br>✗ 审查官"每轮至少3条优化点"规则可能导致凑数低价值问题 |
| 19 | [ql-spec (quantum-loop)](https://github.com/andyzengmath/quantum-loop/blob/master/skills/ql-spec/SKILL.md) | 生成型 | — | quantum-loop自主开发流水线中的规格生成阶段（brainstorm→spec→plan→execute→review→verify），生成结构化PRD供下游转机器可读任务，强调验收标准可验证 | ✓ 与自主开发流水线深度集成，PRD直接对接下游自动执行<br>✓ 验收标准严格要求"机器可验证"，杜绝模糊表述<br>✓ 有phase-skip机制（文件指纹），相同输入重复调用可跳过 | ✗ 是大系统的一个环节，独立使用价值有限，强依赖流水线生态<br>✗ 高度工程化导向，对非技术背景产品经理不够友好<br>✗ 用例拆分规则偏细碎，可能导致PRD过于碎片化 |
| 20 | [prd-review-skill](https://github.com/yihannangua/prd-review-skill) | 评审型 | 4角色模拟 | 深度PRD评审Agent，模拟资深PM、业务分析师、测试负责人、解决方案架构师联合评审，从业务模式、数据关系、流程权限等维度识别问题 | ✓ 评审维度深入全面，覆盖业务对象、数据关系、异常路径等深层问题<br>✓ 不是模板检查器，而是深度理解后给出修改建议和补充文案<br>✓ 输出结构化报告（领域模型、模块地图、问题清单、测试场景） | ✗ 仅做评审不做生成，需要已有PRD作为输入<br>✗ 深度评审需充分理解业务，对输入PRD质量有一定要求<br>✗ 仓库规模小（10次提交），生态和社区支持有限 |
| 21 | [prd-review-panel](https://github.com/rance811-maker/prd-review-panel) | 辩论对抗型 | 4 | 4个并行AI Agent模拟真实PRD评审会，各角色站在不同利益相关者立场发言，产生冲突和交锋，针对AI Agent类PRD优化了评审维度 | ✓ 多角色对抗式评审模拟真实会议场景，角色间有交锋<br>✓ 针对AI Agent类PRD专门优化评审维度（智能体边界、fallback等）<br>✓ 提供PRD合规大纲，支持"先大纲自审，再Panel评审"两步法 | ✗ 仅支持Claude Code平台，适用性受限<br>✗ 角色数量固定为4个，灵活性不足<br>✗ 纯prompt-based skill，无后端逻辑和状态持久化 |
| 22 | [plan-bender](https://github.com/jasonraimondi/plan-bender) | 全流程型 | 多阶段 | AI编码代理的结构化规划管道，覆盖需求访谈→PRD撰写→问题分解→评审→自主实现→归档，将模糊想法转化为agent-ready issues并驱动实现 | ✓ 端到端完整流程，每个阶段有专门skill<br>✓ 支持多代理平台（claude-code、opencode、openclaw、pi）<br>✓ 自主实现调度器支持并行git-worktree，依赖排序不污染主分支 | ✗ 预1.0版本，CLI表面虽稳定但仍有粗糙边缘<br>✗ 自主实现循环仅Claude Code可用，其他代理需手动执行<br>✗ 需要Go环境安装，非技术用户有门槛 |
| 23 | [pecker-prd-review-agent](https://github.com/albedolyu/pecker-prd-review-agent) | 评审型 | 4+编排器 | 人在环中的PRD评审系统，4个有界专家代理独立运行，编排器整合输出，顾问做覆盖度交叉检查，最终由PM确认后生成报告 | ✓ 四个有界专家独立运行，每个最多输出3条发现，失败隔离<br>✓ 证据可追溯，每条发现必须引用确切的源行号<br>✓ PM确认环节保证质量，最终报告只含被接受或编辑过的建议 | ✗ 公开版本用确定性规则而非真实LLM，仅展示工作流<br>✗ SQLite仅适合本地单用户演示，不支持多用户/分布式<br>✗ 无认证授权、租户隔离和生产级审计追踪 |
| 24 | [prd-agents-framework](https://github.com/vshidlovsky/prd-agents-framework) | 全流程型 | 多Agent | 基于Claude Code的多代理PRD框架，研究员、撰写员、评审员等组成人工门控管道，应用BABOK、PBR和需求气味检测等学术方法进行结构化需求工作 | ✓ 评审方法有学术依据（BABOK、PBR、需求气味检测、NASA缺陷分类）<br>✓ 模块化section packs设计，可按项目类型启用不同章节<br>✓ 支持模型配置优化（可靠型/成本优化型），运行日志可对比 | ✗ 仅支持Claude Code，平台绑定度高<br>✗ 初始设置较复杂（约15分钟），需配置project-context.md<br>✗ 运行成本取决于API调用量，全Opus方案较昂贵 |
| 25 | [ai-product-field-review](https://github.com/MisonL/ai-product-field-review) | AI产品专项 | — | Codex插件，专门审查AI产品PRD、Agent方案、企业工作流自动化计划。从产品定位、真实用户证据、指标漂移、组织节奏等维度发现隐藏风险 | ✓ 专为AI产品设计，覆盖AI特有问题（信任、失败恢复、权限边界等）<br>✓ 基于105页/约7.5万字长文复盘提炼，有实战深度<br>✓ 适用场景广：PRD审查、发布计划、用户访谈、竞品策略等 | ✗ 仅支持Codex平台，其他AI工具无法使用<br>✗ 是判断框架而非自动化工具，输出质量依赖输入材料丰富度<br>✗ 文档较简略，缺少详细使用示例和最佳实践 |
| 26 | [AIX prd (知识库)](https://github.com/kelvin381539960-cyber/prd) | 知识库型 | — | AIX公司的AI可读PRD和产品知识库仓库，维护结构化的产品事实、迭代PRD、参考数据、历史PRD和外部依赖文档，服务多条金融产品线 | ✓ 完整的AI可读结构化知识体系，目录分类清晰<br>✓ 严格运营规则：知识库更新走ingestion流程，未确认项走knowledge-gaps<br>✓ 提供AI查询路由入口，指导AI从哪里获取信息 | ✗ 是特定公司内部知识库，普适性差，其他公司无法直接复用<br>✗ 更偏向知识管理和文档组织，而非主动评审工具<br>✗ 与特定金融业务（钱包、卡、KYC等）深度绑定 |
| 27 | [PM-PRD-Review](https://github.com/cutieoon/PM-PRD-Review) | 评审型 | 多视角 | 面向PM的Cursor Agent Skill，借鉴cross-review多角色互审模式，从用户价值、系统一致性、QA可测性、范围成本、AI边界等视角交叉评审PRD和原型 | ✓ 多视角交叉评审，覆盖可执行性、多视图一致性、PRD与原型匹配度<br>✓ 输出分层清晰：结论/必修/建议/可延后/决策点，便于优先级排序<br>✓ 专门检查PRD与原型的逻辑、文案、状态、数据字段不一致问题 | ✗ 仅支持Cursor平台<br>✗ 文档较简略，角色定义和评审标准不够详细<br>✗ 项目只有一次提交，成熟度较低 |
| 28 | [PRD_reviewer](https://github.com/jkresse23/PRD_reviewer) | 评审型 | 3 | Cursor的多代理PRD评审工具，三个代理分别评审问题、方案、需求三部分，使用共享绿/黄/红评分标准给出VP风格反馈 | ✓ 按PRD结构分角色评审：Pete（问题）、Sally（方案）、Ralph（需求）<br>✓ 共享评分标准，保证评价一致性<br>✓ VP风格反馈简洁有力，每准则有绿/黄/红评分和总结表 | ✗ 仅支持Cursor<br>✗ 三个角色覆盖维度有限，缺少安全、运维、数据等视角<br>✗ 各代理独立评审，无整体汇总和交叉检查机制 |
| 29 | [PRD-Review (串行对抗)](https://github.com/piguren/PRD-Review) | 辩论对抗型 | 4 + PM | 运行在Claude Code里的PRD串行对抗评审harness。四个AI角色（产品老板、前端、后端、测试）串行对抗，PM作为真实参与者被质疑并当场拍板 | ✓ 串行对抗设计独到：角色能看到其他角色质疑后继续追击<br>✓ PM深度参与：不是旁观者，是被质疑方也是决策者<br>✓ 辩论日志落磁盘，防止长对话上下文失真 | ✗ 仅支持Claude Code<br>✗ 串行模式Token消耗大，评审耗时较长<br>✗ 需要PM全程参与回应，自动化程度相对低 |
| 30 | [prd-review-skill (9角色)](https://github.com/122245951-lab/prd-review-skill) | 评审型 | 9 | AI多角色需求评审专家，模拟产品研发全链路9个关键干系人视角，全方位结构化评审。三层递进框架（方向层→结构层→细节层），三种评审模式 | ✓ 9个角色覆盖最全面：产品、开发、算法、UI、测试、运维、安全、项目、数据分析<br>✓ 三层递进框架：先抓大再放小，评审有条理<br>✓ 三种评审模式适配不同成熟度：概念版/完整版/快速版 | ✗ 仅有Initial commit，项目成熟度低，未经过实战验证<br>✗ 文档较简略，缺少具体评审标准和示例输出<br>✗ 9角色同时运行可能导致Token消耗大 |
| 31 | [KARIMO](https://github.com/opensesh/KARIMO) | 全流程型 | 子代理团队 | Claude Code的harness工程插件，PRD驱动的自主代理编排系统。从PRD执行到自动化评审、代码评审和CI友好的代理团队，支持分阶段采用 | ✓ 功能最全面：覆盖PRD访谈、任务分解、并行执行、自动评审、CI集成<br>✓ 支持子代理和代理团队，任务依赖管理、卡住检测和崩溃恢复<br>✓ 分阶段采用（4个阶段），可从简单PRD执行逐步升级 | ✗ 学习曲线陡峭，概念和配置项多，新手上手成本高<br>✗ 主要面向开发者和设计师，纯产品经理使用门槛较高<br>✗ 深度绑定Claude Code生态，依赖GitHub CLI和Git工作流 |

---

## 分类速览

| 类型 | 数量 | 代表 Skill |
|------|------|-----------|
| 生成型 | 9 款 | #2 ralph、#5 prd-writer、#9 copilot PRD、#12 prd-creator、#16 prd-generator、#17 prd-to-prototype、#19 ql-spec |
| 评审型 | 13 款 | #1 check-prd-skill、#3 prd-reviewer、#7 requirement-review、#11 multi-agent-prd-reviewer、#20 prd-review-skill、#30 9角色版 |
| 全流程型 | 8 款 | #4 prd-studio、#8 goal-workflow、#10 technical-pm、#14 claude-plugin、#15 ai-product-manager、#22 plan-bender、#24 prd-agents-framework、#31 KARIMO |
| 辩论对抗型 | 4 款 | #6 prd-debate、#18 prd-craft、#21 prd-review-panel、#29 PRD-Review串行对抗 |
| AI产品专项 | 3 款 | #21（Agent PRD）、#25 ai-product-field-review、#10（AI/ML视角） |
| 知识库型 | 1 款 | #26 AIX prd |

---

## 选型建议

| 场景 | 推荐 Skill | 理由 |
|------|-----------|------|
| 快速生成轻量PRD | #2 ralph / #8 goal-workflow | 简洁高效，动态问题数，下游衔接好 |
| 工程化团队严格评审 | #3 TestAny prd-reviewer | 守门人机制+追溯元数据+1:N拆分支持 |
| 写审全自动闭环 | #4 TestAny prd-studio | Writer+Reviewer双Agent自动循环 |
| 多角色模拟评审会 | #7 requirement-review / #30 9角色版 | 交互感强 / 覆盖最广 |
| 对抗式辩论打磨质量 | #6 prd-debate / #29 PRD-Review | 跨模型 / 串行对抗+PM参与 |
| AI产品专项审查 | #25 ai-product-field-review | AI特有风险维度最全 |
| 从PRD到代码全流程 | #31 KARIMO / #22 plan-bender | 功能最全 / 多平台支持 |
| 中文PM全能力套件 | #15 ai-product-manager-skills | 13个Skill全覆盖，中文优先 |
