# 修仙世界 · 第一版技术规格说明书（SPEC）

- 文档版本：v1.1（技术栈由 Go 切换为 Python）
- 变更记录：v1.0 初始 Go 版；v1.1 按用户决定改用 Python 实现，玩法规则、数据库与结算算法不变
- 对应需求源：`Order.md`
- 适用范围：当前仓库 `/home/wyh/project/MyGame`
- 开发原则：**逐个版本迭代开发，每个版本可运行、可测试、可存档；未提及的功能后续以“新增属性/方法/配置”的方式扩展，不推翻现有结构。**

---

## 1. 背景与总体定位

### 1.1 需求来源

本规格根据 `Order.md` 及多轮口头确认整理。`Order.md` 中定义的基础类清单如下：

| 类别 | 类名 |
|---|---|
| 基础类 | 人物类、动作类、Tile 类、Region 类、时间类、模拟器类、前端类、灵气类、修仙等级类、灵根类、寿命类、事件类、NPC AI 类、角色性格类、ID 类、动物类、Item 类、灵石类、植物类 |
| 额外文件 | 名称文件、图片文件、LLM 文件、配置文件、IO 文件 |

### 1.2 游戏形态

- 纯文字、单机、PVE、无登录系统。
- 以方格地图为基础，玩家在地图上移动。
- **移动一格 = 世界推进一个月 = 模拟器进行一次完整结算。**
- 不移动时世界暂停；服务器关闭时世界暂停，重新启动后从存档继续。
- 第一版优先通过 VSCode 运行和测试，部署问题后续再议。

### 1.3 架构选型：经典类 + 抽象接口（方案 A，Python 实现）

本版**不采用 ECS**，采用用户确认的方案 A，并使用 Python 实现：

- `Person`、`Action`、`Tile`、`Region`、`Simulator` 等均为普通 Python 类，数据类优先使用 `@dataclass`；
- 继承只用于“接口契约”：动作类统一实现 `Action` 抽象基类，Region 子类、AI 策略、前端实现通过接口扩展；
- 每个动作类实现同一个 `Action` 接口；
- Region 后续子类（城市、修炼区域、普通区域）通过 `kind` 字段 + 接口扩展，不修改原有 Map/Tile 逻辑；
- 后续若某个类属性爆炸，再对局部做组件化重构，对外接口保持不变。

---

## 2. 本版已确认的核心决策（不可再随意更改）

| 编号 | 决策 | 说明 |
|---|---|---|
| D1 | 组织方式 | Python 经典类 + 抽象接口，不用 ECS |
| D2 | 世界时间推进 | 玩家移动 1 格 = 世界推进 1 个月 = 模拟器结算 1 次；不移动则世界暂停 |
| D3 | 月内精度 | 每次月结算内部按 **30 天逐日回放**；年=12月，月=30天 |
| D4 | 日结算顺序 | 当天动作 → 延寿事件按生效日应用 → 年龄+1天 → 死亡检查（死亡最后判定） |
| D5 | 动作进度 | 动作按 `已完成天数/总天数` 累计百分比，**未到 100% 不结算效果** |
| D6 | 动作衔接 | 动作在月中完成时，队列中下一个动作从下一日起继续使用当月剩余天数 |
| D7 | 死亡规则 | 死亡日之前已完成的动作保留收益；进行中的动作不再推进；未开始的动作清空并记录“因寿终中断” |
| D8 | 延寿事件 | 采用“延寿事件字典”：每次延寿记录生效时间与增加年数，模拟时按时间顺序应用，死亡检查前汇总已生效事件 |
| D9 | 地图 | 64×64 方格，由 `world.yaml` 定义 Region 与 Shape，启动时自动生成；预留随机生成开关 |
| D10 | Tile 职责 | Tile 只记录 x、y、所属 Region；灵气、可进入性、事件等行为全部由 Region 与 Map 统一管理 |
| D11 | 数据库 | SQLite 单文件存档，WAL + synchronous=FULL + 外键 + 自动备份，关键数据不丢 |
| D12 | 时间比例 | 第一版为纯回合制：世界时间只随移动推进，不按现实时间推进；`time_scale` 字段预留但默认不启用 |
| D13 | 存档 | 单存档；关服暂停、重启续档；第一版无离线收益（离线收益以后通过现实时间补算开放） |
| D14 | 并发 | 所有世界状态只能由模拟器这一个异步任务（asyncio 单线程事件循环）修改；LLM 调用可异步，但只能向模拟器提交“动作提案” |
| D15 | NPC | 初始 30 个 NPC，人口上限 100；每年出生、每月规则 AI 决策 |
| D16 | 世界事件 | 第一版固定每 6 个游戏月触发一次；以后按 Region 属性与随机时间扩展 |
| D17 | 玩家寿终 | 进入“大限将至”状态：不能修炼、不能突破，存档保留，可 GM 回滚；第一版无转世 |
| D18 | 灵根 | 采用五行占比模型：金/木/水/火/土各 0–100，总和 100；GM 只能在新建人物时设定，运行中不允许修改 |
| D19 | 灵石 | 独立货币，不放入 Item 类 |
| D20 | 动物/植物 | 各先提供 20 个默认物种配置，后续按需增删；动物位置持久化，植物分幼苗/成熟/枯萎三阶段 |
| D21 | 名称生成 | `surnames.txt`（姓氏）+ `male_names.txt`（男名）+ `female_names.txt`（女名）随机组合 |
| D22 | 图片 | 第一版不加载图片，但所有实体配置预留 `sprite` 字段，供后续图形化使用 |
| D23 | LLM | 第一版只实现 DeepSeek 配置与接口骨架（`enabled: false`），异步调用、失败回退规则 AI |
| D24 | 配置热更 | 第一版不做运行中热更；改配置后重启加载，启动时做边界与引用完整性校验 |
| D25 | 前端 | 第一版为终端文字客户端 + ASCII 地图（周围 10×10） |

---

## 3. 本版明确不做（后续版本再扩展）

以下内容**不进入第一版实现**，但结构上预留扩展点：

- 登录、账号、多玩家、PVP、聊天、宗门、交易；
- 战斗系统、任务系统；
- 转世、道侣、宠物、天气、炼丹炼器；
- 离线收益、真实时间挂机；
- WebSocket、Redis、PostgreSQL/MySQL、MongoDB；
- 图片渲染、动画、微信小程序、2D 画面；
- 配置运行中热更新；
- 任务系统（第一版只做“地块/区域事件”）；
- 动物繁衍。

---

## 4. 技术栈

| 项目 | 选择 | 理由 |
|---|---|---|
| 语言 | Python 3.12+ | 用户熟悉，开发迭代最快；当前单机回合制规模完全无性能压力 |
| 数据库 | SQLite | 标准库 `sqlite3` 内置，零额外依赖；单文件存档 |
| 配置格式 | YAML | `PyYAML`，与 Python dict 天然对应 |
| 文本文件 | UTF-8 TXT | 自写 IO 工具，见第 11 节 |
| HTTP（预留） | Python 标准库 `http.server`（v0.5） | 第一版不引入 Web 框架；以后需要时再评估 FastAPI |
| 并发/异步 | 标准库 `asyncio` | 单线程事件循环；模拟器为唯一写者；LLM 用 `asyncio.to_thread` 异步化 |
| 日志 | 标准库 `logging` | 分级日志 |
| 随机数 | 标准库 `random.Random(seed)` | 世界种子可复现 |
| 资源打包 | 普通 `configs/`、`assets/` 目录读取 | 第一版不追求单二进制；后期可用 PyInstaller 打包 |
| 测试 | 标准库 `unittest` 或 `pytest`（开发依赖） | 单元测试 + 模拟器集成测试 |
| LLM（预留） | `urllib.request`（在 asyncio 线程池中调用）+ `json` | 兼容 DeepSeek（OpenAI 格式）接口 |

**第一版依赖清单：**

```text
Python 3.12+
PyYAML            # 运行时唯一第三方依赖
pytest            # 仅开发测试用（可选）
```

其余全部使用 Python 标准库。后续若需要 WebSocket、FastAPI、MongoDB、Redis 等，再按版本引入，不提前引入。

---

## 5. 总体架构

### 5.1 运行形态

```text
第一版推荐运行方式（VSCode 内）：
  cd /home/wyh/project/MyGame
  python -m mygame
  # 或：python main.py

程序启动后：
  1. 加载 configs/ 配置并校验；
  2. 打开 SQLite（不存在则初始化并生成世界）；
  3. 恢复世界时间、人物、地图实体、动作队列；
  4. 启动 asyncio 事件循环并创建模拟器异步任务；
  5. 进入终端文字菜单循环。
```

程序通过命令行参数切换模式：

| 参数 | 作用 |
|---|---|
| `--http :8080` | 额外启动 HTTP API（为 GM 与后续图形前端预留） |
| `--gm` | 启动后直接进入 GM 菜单 |
| `--config ./configs` | 指定配置目录（默认 `./configs`） |
| `--db ./data/save.db` | 指定存档路径 |

### 5.2 进程内数据流

```text
终端输入 / HTTP 请求 / GM 命令
        │
        ▼
   Command 通道（模拟器唯一入口）
        │
        ▼
┌────────────────────────────────┐
│   Simulator（唯一写者异步任务）  │
│  1. 校验动作合法性               │
│  2. 推进 1 个月（内部 30 天）     │
│  3. 月末钩子（世界事件/出生）     │
│  4. 单事务写 SQLite             │
│  5. 返回结果快照与事件日志        │
└────────────────────────────────┘
        │
        ▼
   终端渲染 / HTTP JSON 响应
```

### 5.3 并发规则（“协程化机制”的正确实现）

用户要求“动作可以异步实现，实行协程化机制”，同时要求“所有结算操作都在模拟器中进行”。Python 版采用 **asyncio 单线程事件循环 + 单写者** 实现：

1. **世界状态单写者**：模拟器是事件循环中唯一允许修改 `WorldState` 的异步任务。
2. **长动作不靠线程/定时器驱动**：动作是状态机 `waiting → running → done/cancelled/interrupted`，进度存在 `ActionQueueEntry` 中，由模拟器在日循环中推进。这样“等待 30 天”不会占用任何线程或定时器。
3. **允许异步的部分**：
   - LLM 请求通过 `asyncio.to_thread` 放到线程池执行，超时或失败返回后由模拟器继续处理；
   - LLM 只能返回“动作提案”，模拟器通过 `can_execute()` 校验后才入队；
   - 终端输入/HTTP 请求通过 `asyncio.Queue` 向模拟器发送 Command，不直接改状态。
4. **查询**：外部只读取 `snapshot()` 返回的不可变快照，不直接访问模拟器内部状态。
5. **关服安全**：收到 `Shutdown` 命令后，模拟器完成当前结算、强制落盘，再停止事件循环。
6. **禁止事项**：模拟器结算过程不得中途 `await` 打断月结算；LLM/IO 的异步结果只能在结算开始前或结束后被消费。

```text
终端/HTTP 请求 ──asyncio.Queue──► 模拟器异步任务 ──事务──► SQLite
LLM 线程池 ──ActionProposal──► 模拟器异步任务（校验后入队）
查询方 ──SnapshotRequest──► 模拟器异步任务 ──Snapshot──► 查询方
```

---

## 6. 项目目录结构

```
MyGame/
├── Order.md                         # 需求原始文件
├── SPEC.md                          # 本文件
├── main.py                          # 程序入口：python main.py
├── requirements.txt                 # PyYAML
├── requirements-dev.txt             # pytest（可选）
├── pyproject.toml                   # 项目元数据与 pytest 配置（可选）
├── .gitignore                       # data/、__pycache__/ 等
├── mygame/
│   ├── __init__.py
│   ├── __main__.py                  # 支持 python -m mygame
│   ├── app.py                       # 依赖装配、启动/关闭流程
│   ├── idgen.py                     # ID 类
│   ├── namegen.py                   # 名称文件加载与随机姓名
│   ├── world/
│   │   ├── __init__.py
│   │   ├── map.py                   # Map：统一管理 Tile/Region/寻路/边界
│   │   ├── tile.py                  # Tile 类
│   │   ├── region.py                # Region 类 + kind 分发
│   │   ├── shape.py                 # Shape 接口 + rect/circle/polygon
│   │   └── generation.py            # 按 world.yaml 生成地图、校验覆盖
│   ├── person/
│   │   ├── __init__.py
│   │   ├── person.py                # 人物类（玩家与 NPC 统一）
│   │   └── history.py               # 最近 5 条动作记录（传记）
│   ├── action/
│   │   ├── __init__.py
│   │   ├── base.py                  # Action 抽象基类、队列条目、进度百分比
│   │   ├── registry.py              # 动作注册表（新增动作只在此注册）
│   │   ├── move.py                  # 移动动作
│   │   ├── cultivate.py             # 修炼动作
│   │   ├── breakthrough.py          # 突破动作
│   │   ├── entertain.py             # 娱乐动作
│   │   ├── hunt.py                  # 狩猎动作
│   │   ├── gather.py                # 采集动作
│   │   └── emotion.py               # 情绪动作
│   ├── gametime/
│   │   ├── __init__.py
│   │   ├── year.py                  # 年类
│   │   ├── month.py                 # 月类
│   │   ├── day.py                   # 日类
│   │   ├── timestamp.py             # 时间戳类：年-月-日
│   │   └── calendar.py              # day_index 与年/月/日互转
│   ├── simulator/
│   │   ├── __init__.py
│   │   ├── simulator.py             # 模拟器类：单写者、asyncio 队列
│   │   ├── settlement.py            # 月结算 = 30 天日循环 + 月末钩子
│   │   ├── lifespan.py              # 延寿事件字典应用与死亡判定
│   │   ├── checkpoint.py            # 存档检查点与事务提交
│   │   └── command.py               # Command/Result/Snapshot 类型
│   ├── cultivation/
│   │   ├── __init__.py
│   │   ├── realm.py                 # 修仙等级类
│   │   ├── linggen.py               # 灵根类（五行占比）
│   │   ├── technique.py             # 功法（本版读配置）
│   │   └── formula.py               # 修炼/突破公式（配置驱动）
│   ├── ecology/
│   │   ├── __init__.py
│   │   ├── animal.py                # 动物类
│   │   ├── plant.py                 # 植物类
│   │   └── growth.py                # 生长/重生推进
│   ├── item/
│   │   ├── __init__.py
│   │   ├── item.py                  # Item 类（模板 + 实例）
│   │   └── spiritstone.py           # 灵石类（独立货币）
│   ├── event/
│   │   ├── __init__.py
│   │   ├── event.py                 # 事件类（世界记录）
│   │   ├── template.py              # 事件模板
│   │   └── log.py                   # 事件写入 event_log
│   ├── ai/
│   │   ├── __init__.py
│   │   ├── base.py                  # NPC AI 接口
│   │   ├── rule.py                  # 规则类 AI
│   │   └── llm.py                   # LLM 类 AI（动作链 + 突发状况）
│   ├── personality/
│   │   └── personality.py           # 角色性格类（五维）
│   ├── config/
│   │   ├── __init__.py
│   │   ├── loader.py                # 配置文件类：YAML 导入
│   │   ├── validate.py              # 边界/引用/数值校验
│   │   └── types.py                 # 配置结构体（dataclass）定义
│   ├── io_utils/
│   │   └── textfile.py              # IO 文件：只读 UTF-8 TXT
│   ├── llmclient/
│   │   └── deepseek.py              # DeepSeek 异步调用（预留）
│   ├── store/
│   │   ├── __init__.py
│   │   ├── sqlite.py                # 连接、PRAGMA、事务
│   │   ├── migrations.py            # 迁移执行
│   │   ├── backup.py                # 自动备份
│   │   ├── repo_world.py            # 世界/时间存取
│   │   ├── repo_person.py           # 人物/动作队列/历史存取
│   │   ├── repo_ecology.py          # 动物/植物存取
│   │   ├── repo_item.py             # 物品/灵石存取
│   │   └── repo_event.py            # 事件日志存取
│   ├── frontend/
│   │   ├── __init__.py
│   │   ├── terminal.py              # 前端类：终端文字客户端
│   │   ├── menu.py                  # 菜单与输入处理
│   │   └── render.py                # ASCII 地图与状态渲染
│   └── server/
│       ├── __init__.py
│       ├── http.py                  # HTTP API（预留）
│       └── gm.py                    # GM 管理接口（预留）
├── configs/
│   ├── world.yaml                   # 地图、Region、Shape、灵气、出生点
│   ├── realms.yaml                  # 境界、寿元、突破公式
│   ├── techniques.yaml              # 5 本功法
│   ├── spirit_roots.yaml            # 五行灵根
│   ├── actions.yaml                 # 动作耗时与效果参数
│   ├── animals.yaml                 # 20 种动物
│   ├── plants.yaml                  # 20 种植物
│   ├── items.yaml                   # 物品模板
│   ├── events.yaml                  # 事件模板
│   ├── ai.yaml                      # 规则 AI 与性格权重
│   └── llm.yaml                     # DeepSeek 配置（默认关闭）
├── assets/
│   ├── names/
│   │   ├── surnames.txt
│   │   ├── male_names.txt
│   │   └── female_names.txt
│   ├── prompts/
│   │   ├── system_prompt.txt
│   │   ├── action_chain_prompt.txt
│   │   └── emergent_event_prompt.txt
│   └── sprites/
│       └── README.md                # 第一版不加载图片，仅预留 sprite 键目录
├── data/                            # 运行时数据（gitignore）
│   ├── save.db                      # SQLite 主存档
│   ├── backups/                     # 自动备份
│   └── logs/                        # 运行日志（可选）
└── tests/
    ├── unit/                        # 纯逻辑单测
    └── integration/                 # 模拟器长流程测试
```

---

## 7. 领域类详细设计（对应 Order.md 的每一个类）

### 7.1 人物类 `Person`

- 文件：`mygame/person/person.py`
- 玩家与 NPC 统一使用 `Person`，通过 `kind` 区分；玩家额外属性放 `extra`，NPC 决策由 `ai.Controller` 负责。

```python
from dataclasses import dataclass, field

@dataclass
class Person:
    id: str                       # P0001 / N0001
    kind: str                     # "player" | "npc"
    name: str
    gender: str                   # "male" | "female"
    birth_day_index: int          # 出生日（day_index）
    age_days: int = 0             # 年龄（天），内部权威值
    alive: bool = True
    death_day_index: int | None = None
    pos_x: int = 0
    pos_y: int = 0
    realm_id: str = "lianqi"      # lianqi / zhuji / jindan / yuanying
    stage_index: int = 0          # 0 初期 1 中期 2 后期 3 圆满
    progress_percent: float = 0.0 # 当前小层修为进度 0-100
    technique_id: str | None = None
    base_lifespan_days: int = 36000  # 当前境界基础寿元快照
    mood: float = 80.0            # 0-100
    spirit_root: LingGen = field(default_factory=LingGen)
    personality: Personality = field(default_factory=Personality)
    status: Status = field(default_factory=Status)  # 轻伤/突破冷却等，序列化为 status_json
    action_queue: ActionQueue = field(default_factory=ActionQueue)
    history: History = field(default_factory=History)  # 最近 5 条动作记录
    stones: int = 0               # 灵石独立存放
    items: list[ItemInstance] = field(default_factory=list)
    extra: dict = field(default_factory=dict)  # extra_json，后续新属性先放这里
```

**方法（第一版必须实现）：**

| 方法 | 说明 |
|---|---|
| `age_string()` | 年龄按“X年X月X天”显示 |
| `get_cultivation_progress()` | 获取修仙进度（境界/小层/百分比） |
| `is_old_and_dead(current_day)` | 是否因寿元耗尽死亡（见寿命类） |
| `current_lifespan_days(current_day)` | 基础寿元 + 已生效延寿事件之和 |
| `get_action_history(limit=5)` | 获取历史动作，默认最近 5 条 |
| `can_do(act)` | 调用动作的 `can_execute()` 做合法性判断 |
| `queue_action(act)` | 向动作队列追加动作 |
| `add_lifespan_event(...)` | 添加一条延寿事件（写字典） |
| `apply_death(day, reason)` | 玩家保留存档置死；NPC 移除 |
| `set_spirit_root(...)` | 仅新建人物时允许调用 |

**约束：**
- `kind == "player"` 在系统中只能有一个；
- NPC 死亡后实体移除，玩家死亡后实体保留（`alive=False`）。

---

### 7.2 动作类 `Action`

- 文件：`mygame/action/base.py`
- 所有动作实现同一抽象基类；新增动作 = 新文件 + 在 `registry.py` 注册一行，不改旧代码。

```python
from abc import ABC, abstractmethod

class Action(ABC):
    action_type: str = ""   # move/cultivate/breakthrough/entertain/hunt/gather/emotion_xxx
    kind: str = "physical"  # "physical" 实际动作 | "emotion" 情绪动作
    duration_days: int = 0  # 0 = 即时

    @abstractmethod
    def can_execute(self, ctx: SimContext, person: Person) -> None:
        """不合法时 raise ValueError，合法返回 None"""

    def on_start(self, ctx: SimContext, person: Person) -> None: ...
    def on_complete(self, ctx: SimContext, person: Person) -> None: ...
    def on_interrupt(self, ctx: SimContext, person: Person, reason: str) -> None: ...
```

**动作队列条目：**

```python
@dataclass
class ActionQueueEntry:
    person_id: str
    seq: int
    action_type: str
    params: dict = field(default_factory=dict)   # 例如移动方向、修炼功法
    total_days: int = 0
    elapsed_days: int = 0
    state: str = "waiting"   # waiting | running | done | cancelled | interrupted

    def progress_percent(self) -> float:
        """未到 100% 不结算"""
        return 100.0 if self.total_days == 0 else self.elapsed_days / self.total_days * 100.0
```

**第一版七类动作：**

| 动作 | 类型 | 耗时 | 说明 |
|---|---|---|---|
| `move` | 实际 | 3 天 | 移动到相邻方格；完成后更新坐标并触发到达事件 |
| `cultivate` | 实际 | 30 天 | 完成后按公式增加修为；100% 才结算 |
| `breakthrough` | 实际 | 7 天 | 完成后判定是否晋升大境界 |
| `entertain` | 实际 | 3 天 | 完成时少量修为 + 恢复心情 |
| `hunt` | 实际 | 15 天 | 完成后按境界/危险度判定是否获得猎物 |
| `gather` | 实际 | 5 天 | 完成后判定是否采到当前格成熟植物 |
| `emotion_*` | 情绪 | 0 天 | 即时动作，只改变心情/性格，不改变世界 |

**动作执行条件**（`can_execute()`）在 Python 代码中直接编写函数，第一版检查项包括：
- 人物存活、状态正常；
- 大限将至者不能 `cultivate` / `breakthrough`；
- 突破需要当前小层为“圆满”且无突破冷却；
- 移动目标在边界内、可进入、无境界限制；
- 狩猎/采集要求当前 Region 允许；
- 同一人物同一时刻只有一个动作在运行，其余排队。

**协程化说明：**
- 长动作通过 `ActionQueueEntry` 状态机 + 模拟器日循环推进，不使用线程/定时器计时；
- 效果只在 `progress_percent() == 100.0` 时通过 `on_complete()` 结算；
- 中断/死亡只调用 `on_interrupt()`，不重复结算。

---

### 7.3 Tile 类

- 文件：`mygame/world/tile.py`

```python
from dataclasses import dataclass

@dataclass(frozen=True)
class Tile:
    x: int
    y: int
    region_id: str
```

- Tile 只记录坐标和所属 Region，不记录地形、灵气等派生数据；
- 行为合理性（能否进入、能做什么动作）由 `Map.region_at(x, y)` 返回的 Region 决定；
- 后续若要加“地块修饰”（如灵泉眼），通过 `world.yaml` 的 `tile_modifiers` 层或 Region 子类实现，不修改 Tile 结构。

### 7.4 Region 类与 Shape 类

- 文件：`mygame/world/region.py`、`mygame/world/shape.py`

```python
from dataclasses import dataclass, field
from mygame.world.shape import Shape

@dataclass
class Region:
    id: str
    name: str
    kind: str                  # "normal" | "city" | "cultivation" | "ruin"
    shape: Shape
    aura_base: float           # 灵气基础值
    danger: int                # 危险度
    enterable: bool = True
    min_realm: str = "lianqi"
    event_pool: list[str] = field(default_factory=list)

class Shape(ABC):
    @abstractmethod
    def contains(self, x: int, y: int) -> bool: ...
    @abstractmethod
    def bounds(self) -> tuple[int, int, int, int]: ...
```

Shape 实现：
- `RectShape{x, y, w, h}`：矩形；
- `CircleShape{cx, cy, r}`：圆形；
- `PolygonShape{points}`：多边形。

**约束：**
- 启动时校验地图每个 Tile 都被恰好一个 Region 覆盖；
- 第一版禁止 Region 重叠；如重叠，配置校验直接失败（后续可加 priority 字段）；
- Region 后续子类（城市、修炼区）先通过 `kind` 分支实现，不写类继承。

### 7.5 时间类

- 文件：`mygame/gametime/`

```python
from typing import NewType

Year  = NewType("Year", int)    # 年类
Month = NewType("Month", int)   # 月类，1-12
Day   = NewType("Day", int)     # 日类，1-30

@dataclass(frozen=True)
class Timestamp:                # 时间戳类
    year: Year
    month: Month
    day: Day
```

- 权威存储为 **day_index**（自第 1 年 1 月 1 日以来的天数，起点为 0）；
- 转换：`day_index = (year-1)*360 + (month-1)*30 + (day-1)`；
- `str(Timestamp)` 输出 `第X年X月X日`；
- 年龄按天存储，显示为 `X年X月X天`；
- 提供 `add_days / compare / to_day_index / from_day_index` 方法。

### 7.6 模拟器类 `Simulator`

- 文件：`mygame/simulator/simulator.py`
- **唯一世界推进者与状态写者。**

```python
import asyncio
import random

class Simulator:
    def __init__(self, state: WorldState, store: SQLiteStore, cfg: Config):
        self.state = state
        self.cmds: asyncio.Queue[Command] = asyncio.Queue()
        self.llm_proposals: asyncio.Queue[ActionProposal] = asyncio.Queue()
        self.store = store
        self.cfg = cfg
        self.rng = random.Random(cfg.world.seed)

    async def run(self) -> None: ...
    async def advance_one_step(self) -> StepResult: ...
    async def submit(self, cmd: Command) -> None: ...
    def snapshot(self) -> WorldSnapshot: ...
```

- `advance_one_step()` 是“移动一格”触发的唯一入口；
- 内部先推一个月，再按第 9 节日循环结算，最后月末钩子与单事务落盘；
- 同时负责：NPC 出生、人物死亡、世界时间、世界事件、世界变化记录；
- 一次 `advance_one_step()` = 一个事务 = 要么整月生效，要么整月回滚。

### 7.7 前端类

- 文件：`mygame/frontend/`
- 第一版实现终端文字客户端；
- 接口预留，后续图形化实现同一接口：

```python
class Frontend(ABC):
    def render(self, state: WorldSnapshot) -> None: ...
    async def next_command(self) -> Command: ...
    def show_events(self, events: list[Event]) -> None: ...
```

终端菜单：

```text
【青云修士】炼气·初期 | 修为 12.3% | 寿元 18年3月 | 心情 80
当前位置：青云山脚(12,8)  灵气：10

[1] 移动（上下左右）   [2] 突破        [3] 娱乐
[4] 开始修炼           [5] 狩猎        [6] 采集
[7] 查看状态           [8] 查看传记    [9] 查看地图
[G] GM 菜单            [0] 保存并退出
```

- **重要交互规则**：入队动作不会推进世界；只有执行“移动”才触发 `advance_one_step()` 推进一个月。GM 菜单提供 `advance N months` 用于无移动测试。

### 7.8 灵气类

- 文件：`mygame/world/region.py`（灵气作为 Region 属性）
- 第一版算法：`实际灵气 = region.aura_base × tile_modifier_at(x,y)`；
- `tile_modifier` 默认全图 1.0，由 `world.yaml` 可选配置，第一版可全用默认；
- 灵气影响两项：修炼速度、突破成功率；其余影响后续加公式即可。

### 7.9 修仙等级类

- 文件：`mygame/cultivation/realm.py`
- 境界表（默认值，全部来自 `realms.yaml`）：

| 顺序 | id | 名称 | 小层 | 基础寿元 |
|---|---|---|---|---|
| 0 | lianqi | 炼气 | 初期/中期/后期/圆满 | 100 年 |
| 1 | zhuji | 筑基 | 同上 | 200 年 |
| 2 | jindan | 金丹 | 同上 | 400 年 |
| 3 | yuanying | 元婴 | 同上 | 800 年 |

- 小层晋升：`progress_percent` 满 100 后**自动**进入下一小层；
- 大境界晋升：当前大境界“圆满”且小层修为满时，由玩家/NPC 手动发起 `breakthrough` 动作；
- 突破成功率第一版公式（权重全部配置化）：

```text
P = clamp(base_probability
    × (1 + w_linggen × 灵根契合度)
    × (1 + w_technique × 功法加成)
    × (1 + w_lifespan × 剩余寿元比例)
    × (1 + w_aura × 灵气系数)
    × (1 + w_mood × 心情系数)
    × (1 + w_item × 物品加成), 0, 1)
```

- 突破失败惩罚（默认值，可配）：
  - 当前小层修为损失 10%；
  - 轻伤 30 天（期间修炼速度减半）；
  - 突破冷却 5 天；
  - 大境界突破失败额外损失 1 年寿元（写入延寿事件字典，值为负）。

### 7.10 灵根类

- 文件：`mygame/cultivation/linggen.py`
- 采用**五行占比**模型：

```python
@dataclass
class LingGen:
    metal: float = 0.0  # 金 0-100
    wood: float = 0.0   # 木
    water: float = 0.0  # 水
    fire: float = 0.0   # 火
    earth: float = 0.0  # 土
```

- 约束：五项之和为 100（容差 0.01）；
- 新建人物时按 `spirit_roots.yaml` 的分布随机生成；GM 可在创建时指定；
- 运行中不允许修改灵根；
- 修炼时按“主属性契合度”计算加成，公式在配置中；
- 存档格式为 JSON：`{"metal":50,"wood":20,"water":10,"fire":15,"earth":5}`。

### 7.11 寿命类与延寿事件字典

- 文件：`mygame/simulator/lifespan.py`
- 当前寿命计算：

```text
剩余寿元(天) = 当前境界基础寿元(天) + Σ(所有 effective_day_index ≤ 当前日 的延寿事件天数) - 年龄(天)
```

- 延寿事件字典即数据库表 `lifespan_events`，同时以 `dict[str, list[LifespanEvent]]` 缓存在内存；
- 事件在 `on_complete()` 中创建时，生效日 = 当日，立即加入字典并落库；
- 死亡检查在“延寿生效 → 年龄+1天”之后执行，保证突破成功延寿当天不会误死；
- 已死亡人物未开始的队列动作清空，并写 `interrupted` 记录。

### 7.12 事件类

- 文件：`mygame/event/`

```python
@dataclass
class Event:
    id: int
    day_index: int
    scope: str                  # person | tile | region | world
    event_type: str             # move/cultivate/breakthrough/birth/death/world_event/...
    tile_x: int | None = None
    tile_y: int | None = None
    region_id: str | None = None
    person_id: str | None = None
    template_id: str | None = None
    params: dict = field(default_factory=dict)
    result_text: str = ""
```

- 所有事件统一写 `event_log` 表；
- 世界事件第一版固定每 6 个月触发一次，模板来自 `events.yaml`；
- 地块到达事件在 `move.on_complete()` 时触发；
- 以后按 Region 属性、随机时间触发时，只改模板与调度器，不改事件结构。

### 7.13 NPC AI 类

- 文件：`mygame/ai/base.py`、`rule.py`、`llm.py`

```python
class Controller(ABC):
    @abstractmethod
    def decide(self, snap: WorldSnapshot, person: Person) -> list[ActionProposal]: ...
```

**规则类 AI（第一版实际启用）：**
- 每月 `advance_one_step()` 开始时，为每个无当前动作的存活 NPC 决策一次；
- 按性格五维从 `ai.yaml` 读取动作权重并加权随机；
- 达到突破条件则尝试突破；
- 所在 Region 灵气低于阈值时优先移动到相邻高灵气 Region；
- 决策结果同样通过 `can_execute()` 校验。

**LLM 类 AI（第一版仅接口与配置，默认关闭）：**
- 配置 DeepSeek 接口（见第 11.4 节）；
- 两类返回：
  1. **动作链**：一次返回多条有序动作提案；
  2. **突发状况响应**：世界/NPC 发起事件时生成应激动作提案；
- LLM 返回的 JSON 必须经过 `can_execute()` 校验，非法动作丢弃并记录；
- 调用在 asyncio 线程池中执行，超时或失败自动回退规则 AI；
- 每个 NPC 每月 LLM 调用次数有限制（`llm.yaml`）。

### 7.14 角色性格类

- 文件：`mygame/personality/personality.py`

```python
@dataclass
class Personality:
    caution: float = 50.0      # 谨慎 0-100
    benevolence: float = 50.0  # 仁善
    solitude: float = 50.0     # 孤僻
    greed: float = 50.0        # 贪婪
    tenacity: float = 50.0     # 坚韧
```

- 影响 NPC 动作选择权重与事件选项；
- 对玩家只作展示；
- 重大事件可按 `events.yaml` 中的 `personality_effects` 修改五维；
- 所有变化必须由模拟器结算，禁止外部直接赋值。

### 7.15 ID 类

- 文件：`mygame/idgen.py`
- 格式：`前缀 + 4 位序号`，例如 `P0001`、`N0001`、`A0001`、`T0001`、`I0001`；
- 前缀含义：P=玩家，N=NPC，A=动物，T=植物，I=物品实例；
- 序号持久化在 `id_counters` 表，进程重启后继续递增；
- 生成逻辑简单、易扩展；以后需要雪花算法时只替换本类。

### 7.16 动物类

- 文件：`mygame/ecology/animal.py`

```python
@dataclass
class Animal:
    id: str
    species_id: str
    pos_x: int
    pos_y: int
    alive: bool = True
    age_days: int = 0
    respawn_day: int | None = None
    attrs: dict = field(default_factory=dict)
```

- 第一版行为：每天随机向相邻可进入格移动 1 格；
- 狩猎成功率 = 查 `animals.yaml` 的 `hunt_difficulty` 与人物境界对照表；
- 动物死亡后按配置时间重生；位置持久化；
- 繁衍后续版本再加。

### 7.17 Item 类

- 文件：`mygame/item/item.py`

```python
@dataclass(frozen=True)
class ItemTemplate:           # 配置定义
    id: str
    name: str
    category: str
    stackable: bool = True
    effects: dict = field(default_factory=dict)
    sprite: str = ""          # 预留图片键

@dataclass
class ItemInstance:           # 存档只存实例
    id: str
    template_id: str
    owner_id: str | None
    quantity: int = 1
    attrs: dict = field(default_factory=dict)  # 随机词条/动态属性，JSON 存储
```

- 普通素材堆叠；带随机词条的装备不堆叠；
- 第一版物品用途：药材可服用（提高突破/恢复伤势），狩猎产物只作素材；
- 后续新增词条只改 `attrs` 与配置，不改表结构。

### 7.18 灵石类

- 文件：`mygame/item/spiritstone.py`
- 独立货币，**不放入 Item**；
- `SpiritStones(amount: int)`，恒 ≥ 0；
- 第一版获取：狩猎、采集按概率掉落；
- 消费入口（坊市/炼丹）后续版本补充。

### 7.19 植物类

- 文件：`mygame/ecology/plant.py`

```python
@dataclass
class Plant:
    id: str
    species_id: str
    pos_x: int
    pos_y: int
    stage: str = "seedling"   # seedling | mature | withered
    stage_days: int = 0       # 进入当前阶段已过天数
    alive: bool = True
    respawn_day: int | None = None
    attrs: dict = field(default_factory=dict)
```

- 阶段：幼苗 → 成熟 → 枯萎；每天推进生长；
- 只能采集“成熟”植株；采集后按配置天数重生；
- 按 Region 类型随机分布，位置持久化。

---

## 8. 时间系统规范

### 8.1 历法

- 1 年 = 12 个月；
- 1 月 = 30 天；
- 1 年 = 360 天；
- 纪元从 `第1年1月1日` 开始，对应 `day_index = 0`；
- 第一版不使用现实时间推进世界。

### 8.2 世界推进规则

| 行为 | 世界时间 |
|---|---|
| 玩家移动 1 格 | +1 个月（内部 30 天逐日回放） |
| 入队修炼/突破/娱乐等动作 | 不推进，等待下次移动结算 |
| 服务器关闭 | 世界暂停，存档保留 |
| 服务器重启 | 从存档的 `day_index` 继续 |
| GM `advance N months` | +N 个月，用于测试 |

---

## 9. 模拟器结算规范（本文件最重要的算法）

### 9.1 一次月结算 = 30 天日循环 + 月末钩子

`advance_one_step()` 伪代码：

```text
begin transaction

// 0. 推进世界时间
day_index += 30

// 1. 本月 NPC 决策（规则 AI / LLM 提案校验后入队）
for each alive NPC without running action:
    proposals = ai.decide(...)
    validate can_execute()
    enqueue

// 2. 30 天日循环
for day = 1 .. 30:
    current_day = 本月起始日 + day

    // 2.1 人物动作推进（先动作，后其他）
    for each alive person:
        if has running action:
            action.elapsed_days += 1
            if action.elapsed_days >= action.total_days:
                validate can_execute() again
                action.on_complete(ctx, person)   // 效果、延寿事件、日志
                mark done; write action_history
                dequeue
        // 队列下一个动作从“下一日”开始
        // （0 天情绪动作：当日完成效果，下一动作同样从下一日开始）

    // 2.2 动物行为
    for each alive animal:
        move 1 格（每天）

    // 2.3 植物生长
    for each alive plant:
        growth += 1 day; 按阶段推进/枯萎

    // 2.4 老化与死亡（先加年龄，后判死；死亡最后）
    for each alive person:
        person.age_days += 1
        lifespan_now = base_lifespan_days + sum(延寿事件 effective_day <= current_day)
        if person.age_days >= lifespan_now:
            mark death at current_day
            clear action queue (not-started → interrupted)
            if player: keep entity, alive=false, status=大限将至
            if npc: remove entity
            write death event

// 3. 月末钩子
if month_number % 6 == 0:
    trigger world event (模板来自 events.yaml)
    write world event to event_log

if month_number == 12:
    NPC 出生（概率、人口上限 100）
    write birth events

// 4. 检查点
update world_state
write all changed entities
commit transaction
return StepResult(logs=logs, snapshot=snapshot)
```

### 9.2 同一天内的顺序（定死）

```text
当天动作完成/效果 → 延寿事件按生效日应用 → 年龄 +1 天 → 死亡检查
```

死亡检查永远是当天最后一步，保证：
- 突破成功当天延寿不会误死；
- 已经死亡的人不会继续执行当天之后任何动作；
- 死亡日之前已完成动作的收益全部保留。

### 9.3 月中死亡与“做了一半的行为”

| 场景 | 规则 |
|---|---|
| 死亡日前已完成的动作 | 效果保留，记录正常 `done` |
| 死亡时正在进行的动作 | 推进到死亡当天为止，不再推进，不结算效果，记录 `interrupted` |
| 死亡时尚未开始的动作 | 从队列清空，记录 `interrupted`，原因“寿终中断” |
| 死亡后队列新增 | 禁止 |

### 9.4 延寿事件字典

- 结构：`(person_id, effective_day_index, days_added, reason, source_id)`；
- `days_added` 可为负（突破失败减寿）；
- 内存缓存 + `lifespan_events` 表双写；
- 每次死亡检查前实时求和，不缓存“剩余寿元”的陈旧值；
- GM 加减寿元同样写该字典，保留审计。

### 9.5 事务与回滚

- 一次 `advance_one_step()` 对应一个 SQLite 写事务：结算开始执行 `BEGIN IMMEDIATE`，成功 `COMMIT`，失败 `ROLLBACK` 并恢复内存快照；
- 结算过程中任何错误：数据库回滚，内存状态恢复到本次结算前快照，返回错误给调用方；
- 成功提交后更新 `world_state.checkpoint_day_index`；
- 崩溃场景：SQLite 自动回滚未提交事务，上次已提交月份为有效存档。

---

## 10. 配置系统

### 10.1 加载与校验

- 所有 YAML 位于 `configs/` 目录，启动时由 `mygame/config/loader.py` 加载并绑定到 dataclass 模型；
- `--config` 参数可指定其他配置目录（默认 `./configs`）；
- 启动时执行：
  1. YAML 解析；
  2. 字段类型与取值范围校验；
  3. ID 唯一性校验；
  4. 引用完整性校验（功法引用的灵根元素存在、事件引用的模板存在等）；
  5. 地图 Region 覆盖校验（每格恰好属于一个 Region）；
  6. 校验失败直接终止启动并打印错误位置。

### 10.2 `configs/world.yaml`

```yaml
map:
  width: 64
  height: 64
  seed: 20250101
  auto_generate: false        # 第一版只按 regions 生成；true 为后续随机生成预留
  player_start: {x: 12, y: 8}

regions:
  - id: qingyun_foothill
    name: 青云山脚
    kind: normal              # normal | city | cultivation | ruin
    shape: {type: rect, x: 0, y: 0, w: 64, h: 24}
    aura_base: 10
    danger: 1
    enterable: true
    min_realm: lianqi
    events: []                # 区域事件模板 ID（第一版可空）

  - id: cuizhu_forest
    name: 翠竹林
    kind: cultivation
    shape: {type: rect, x: 0, y: 24, w: 40, h: 20}
    aura_base: 25
    danger: 2
    enterable: true
    min_realm: lianqi

  - id: lingquan_cave
    name: 灵泉洞
    kind: cultivation
    shape: {type: circle, cx: 20, cy: 44, r: 8}
    aura_base: 50
    danger: 4
    enterable: true
    min_realm: lianqi

tile_modifiers: {}            # {(x,y): 倍率}，默认 1.0；第一版可空
```

### 10.3 `configs/actions.yaml`

```yaml
actions:
  move:
    kind: physical
    duration_days: 3
    interruptible: true
  cultivate:
    kind: physical
    duration_days: 30
    interruptible: true
  breakthrough:
    kind: physical
    duration_days: 7
    interruptible: false
  entertain:
    kind: physical
    duration_days: 3
    interruptible: true
  hunt:
    kind: physical
    duration_days: 15
    interruptible: true
  gather:
    kind: physical
    duration_days: 5
    interruptible: true
  emotion_generic:
    kind: emotion
    duration_days: 0
    interruptible: false
```

### 10.4 `configs/realms.yaml`

```yaml
realms:
  - id: lianqi
    name: 炼气
    order: 0
    stages: [初期, 中期, 后期, 圆满]
    base_lifespan_years: 100
  - id: zhuji
    name: 筑基
    order: 1
    stages: [初期, 中期, 后期, 圆满]
    base_lifespan_years: 200
  - id: jindan
    name: 金丹
    order: 2
    stages: [初期, 中期, 后期, 圆满]
    base_lifespan_years: 400
  - id: yuanying
    name: 元婴
    order: 3
    stages: [初期, 中期, 后期, 圆满]
    base_lifespan_years: 800

breakthrough:
  base_probability: 0.6
  weights:
    linggen: 0.20
    technique: 0.10
    lifespan_remaining_ratio: 0.20
    aura: 0.20
    mood: 0.10
    item: 0.20
  failure:
    progress_loss_percent: 10
    injury_days: 30
    cooldown_days: 5
    lifespan_loss_days: 360     # 大境界突破失败 -1 年
```

### 10.5 `configs/techniques.yaml`（5 本占位功法）

```yaml
techniques:
  - {id: tuna_shu,     name: 吐纳术, speed: 1.0, breakthrough_bonus: 0.00}
  - {id: qingyun_jue,  name: 青云诀, speed: 1.2, breakthrough_bonus: 0.03}
  - {id: ningyuan_gong,name: 凝元功, speed: 1.4, breakthrough_bonus: 0.05}
  - {id: hunyuan_jing, name: 混元经, speed: 1.6, breakthrough_bonus: 0.07}
  - {id: taixu_yin,    name: 太虚引, speed: 1.8, breakthrough_bonus: 0.10}
```

### 10.6 `configs/spirit_roots.yaml`

```yaml
elements: [metal, wood, water, fire, earth]
generation:
  method: random_dirichlet   # 随机占比，总和 100
  concentration: 2.0
affinity:
  cultivate_speed_weight: 1.0
  breakthrough_weight: 1.0
```

### 10.7 `configs/animals.yaml` / `configs/plants.yaml`

每种实体字段模板：

```yaml
animals:
  - id: wild_rabbit
    name: 野兔
    tier: 1
    danger: 1
    hunt_min_realm: lianqi
    hunt_success_base: 0.8
    drops:
      - {item: rabbit_fur, weight: 80, qty_min: 1, qty_max: 2}
      - {item: spirit_stone, weight: 5, qty_min: 1, qty_max: 3}
    respawn_days: 90
    sprite: animals/wild_rabbit.png

plants:
  - id: ling_cao
    name: 灵草
    tier: 1
    stage_days: {seedling: 90, mature: 180, wither: 360}
    gather_min_realm: lianqi
    gather_success_base: 0.9
    drops:
      - {item: ling_cao, weight: 100, qty_min: 1, qty_max: 3}
    respawn_days: 180
    sprite: plants/ling_cao.png
```

第一版默认提供 20 种动物、20 种植物（示例见附录 B），用户可按需增删。

### 10.8 `configs/events.yaml`

```yaml
world_events:
  - id: spirit_tide
    name: 灵潮爆发
    period_months: 6          # 第一版固定每 6 个月
    effects:
      - {type: all_regions_aura_mult, value: 1.5, duration_months: 3}
    result_text: "天地灵气涌动，全图灵气暂时提升。"

tile_events: []              # 地块到达事件，v0.3 填充
```

### 10.9 `configs/ai.yaml`

```yaml
npc:
  initial_count: 30
  max_count: 100
  birth_probability_per_year: 0.10
rule_ai:
  decision_interval_months: 1
  action_weights:
    cultivate:    {base: 50, caution: 0.0, benevolence: 0.0, solitude: 0.3, greed: -0.1, tenacity: 0.2}
    move:         {base: 20, caution: 0.2, benevolence: 0.0, solitude: 0.0, greed: 0.0, tenacity: 0.0}
    entertain:    {base: 15, caution: 0.0, benevolence: 0.1, solitude: -0.2, greed: 0.0, tenacity: 0.0}
    hunt:         {base: 10, caution: -0.2, benevolence: -0.1, solitude: 0.0, greed: 0.2, tenacity: 0.1}
    gather:       {base: 5,  caution: 0.0, benevolence: 0.0, solitude: 0.2, greed: 0.0, tenacity: 0.0}
  breakthrough_when_ready: true
  move_if_aura_below: 10
```

### 10.10 `configs/llm.yaml`

```yaml
enabled: false                 # 第一版默认关闭
provider: deepseek
base_url: https://api.deepseek.com
api_key_env: DEEPSEEK_API_KEY  # 密钥只从环境变量读取
model: deepseek-chat
timeout_seconds: 30
max_calls_per_npc_per_month: 5
max_chain_length: 3
fallback: rule                 # 失败回退规则 AI
```

---

## 11. IO、名称、图片、LLM 文件规范

### 11.1 IO 文件

- 文件：`mygame/io_utils/textfile.py`
- 职责边界：**只负责读取 UTF-8 文本文件**，不负责存档读写（存档在 `store` 包）；
- 功能：
  - 读入并去掉 BOM（`utf-8-sig`）；
  - 每行去空白，忽略空行与 `#` 注释行；
  - 去重；
  - 非 UTF-8 或空文件报错；
  - 返回 `list[str]`。

### 11.2 名称文件

| 文件 | 内容 | 示例 |
|---|---|---|
| `assets/names/surnames.txt` | 姓氏表，每行一个 | 赵、钱、孙、李、陈、林 |
| `assets/names/male_names.txt` | 男性名表 | 青玄、子昂、无尘 |
| `assets/names/female_names.txt` | 女性名表 | 若雪、清瑶、素心 |

- NPC 出生时按性别随机 `姓 + 名`；
- 姓名查重：第一版不强制唯一，通过 ID 区分人物。

### 11.3 图片文件

- 第一版**不加载任何图片**；
- 配置中所有实体预留 `sprite` 字符串字段；
- `assets/sprites/` 目录与 `README.md` 预留，后续图形化时按 `sprite` 键查找资源。

### 11.4 LLM 文件

- 文件：`mygame/llmclient/deepseek.py`
- 三个提示词模板：
  1. `system_prompt.txt`：世界规则、NPC 人设、输出 JSON 格式；
  2. `action_chain_prompt.txt`：生成动作链；
  3. `emergent_event_prompt.txt`：突发状况响应。
- 模板占位符：`{{name}}`、`{{gender}}`、`{{personality}}`、`{{realm}}`、`{{options}}`、`{{event}}`；
- LLM 输出约定：

```json
{"actions":[{"type":"cultivate","params":{},"reason":"灵气充足，闭关修炼"}]}
```

- 返回动作必须全部通过 `can_execute()`；非法动作丢弃并记 `llm_audit_log`；
- 第一版 `llm.enabled=false`，实现接口与 Mock，不实际联网；
- 真实调用时使用 `asyncio.to_thread` 在线程池执行 `urllib.request`，避免阻塞事件循环。

---

## 12. 数据库设计

### 12.1 基础设置

- 文件：`data/save.db`（路径可由 `--db` 指定）；
- 连接代码（Python 标准库 `sqlite3`）：

```python
import sqlite3
conn = sqlite3.connect("data/save.db", isolation_level=None)  # 手动管理事务
conn.execute("PRAGMA journal_mode = WAL;")
conn.execute("PRAGMA synchronous = FULL;")
conn.execute("PRAGMA foreign_keys = ON;")
conn.execute("PRAGMA busy_timeout = 5000;")
```

- 动态属性一律存 JSON 文本列，应用层负责序列化/反序列化；
- 未来迁移 PostgreSQL 时，JSON 列可平滑替换为 JSONB，无需改业务结构。

### 12.2 建表 DDL（v1 全量）

```sql
-- 迁移记录
CREATE TABLE IF NOT EXISTS schema_migrations (
    version     INTEGER PRIMARY KEY,
    description TEXT NOT NULL,
    applied_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

-- ID 序号
CREATE TABLE IF NOT EXISTS id_counters (
    kind      TEXT PRIMARY KEY,          -- P / N / A / T / I
    next_seq  INTEGER NOT NULL DEFAULT 1
);

-- 世界状态（单行，id 恒为 1）
CREATE TABLE IF NOT EXISTS world_state (
    id                     INTEGER PRIMARY KEY CHECK (id = 1),
    day_index              INTEGER NOT NULL DEFAULT 0,
    schema_version         INTEGER NOT NULL DEFAULT 1,
    map_seed               INTEGER NOT NULL DEFAULT 1,
    map_config_hash        TEXT NOT NULL DEFAULT '',
    checkpoint_day_index   INTEGER NOT NULL DEFAULT 0,
    created_at             TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at             TEXT NOT NULL DEFAULT (datetime('now'))
);

-- 人物（玩家 + NPC）
CREATE TABLE IF NOT EXISTS persons (
    id                  TEXT PRIMARY KEY,
    kind                TEXT NOT NULL CHECK (kind IN ('player','npc')),
    name                TEXT NOT NULL,
    gender              TEXT NOT NULL CHECK (gender IN ('male','female')),
    birth_day_index     INTEGER NOT NULL,
    age_days            INTEGER NOT NULL DEFAULT 0,
    alive               INTEGER NOT NULL DEFAULT 1 CHECK (alive IN (0,1)),
    death_day_index     INTEGER,
    pos_x               INTEGER NOT NULL,
    pos_y               INTEGER NOT NULL,
    realm_id            TEXT NOT NULL DEFAULT 'lianqi',
    stage_index         INTEGER NOT NULL DEFAULT 0 CHECK (stage_index BETWEEN 0 AND 3),
    progress_percent    REAL NOT NULL DEFAULT 0 CHECK (progress_percent >= 0 AND progress_percent <= 100),
    technique_id        TEXT,
    base_lifespan_days  INTEGER NOT NULL DEFAULT 36000,
    mood                REAL NOT NULL DEFAULT 80 CHECK (mood >= 0 AND mood <= 100),
    spirit_root_json    TEXT NOT NULL DEFAULT '{}',
    personality_json    TEXT NOT NULL DEFAULT '{}',
    status_json         TEXT NOT NULL DEFAULT '{}',
    extra_json          TEXT NOT NULL DEFAULT '{}',
    created_at          TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_persons_alive   ON persons(alive);
CREATE INDEX IF NOT EXISTS idx_persons_pos     ON persons(pos_x, pos_y);
CREATE INDEX IF NOT EXISTS idx_persons_kind    ON persons(kind);

-- 动作队列（等待/进行中）
CREATE TABLE IF NOT EXISTS action_queue (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    person_id         TEXT NOT NULL REFERENCES persons(id),
    seq               INTEGER NOT NULL DEFAULT 0,
    action_type       TEXT NOT NULL,
    params_json       TEXT NOT NULL DEFAULT '{}',
    total_days        INTEGER NOT NULL DEFAULT 0,
    elapsed_days      INTEGER NOT NULL DEFAULT 0,
    state             TEXT NOT NULL DEFAULT 'waiting'
                      CHECK (state IN ('waiting','running','done','cancelled','interrupted')),
    created_day_index INTEGER NOT NULL,
    started_day_index INTEGER,
    finished_day_index INTEGER,
    result_json       TEXT NOT NULL DEFAULT '{}',
    UNIQUE (person_id, seq)
);

CREATE INDEX IF NOT EXISTS idx_action_queue_person ON action_queue(person_id, seq);

-- 动作历史（每人物只保留最近 5 条；全量轨迹在 event_log）
CREATE TABLE IF NOT EXISTS action_history (
    id                INTEGER PRIMARY KEY AUTOINCREMENT,
    person_id         TEXT NOT NULL REFERENCES persons(id),
    action_type       TEXT NOT NULL,
    params_json       TEXT NOT NULL DEFAULT '{}',
    result_json       TEXT NOT NULL DEFAULT '{}',
    started_day_index INTEGER NOT NULL,
    finished_day_index INTEGER NOT NULL
);

CREATE INDEX IF NOT EXISTS idx_action_history_person ON action_history(person_id, finished_day_index);

-- 延寿事件字典（寿命变更的唯一权威记录）
CREATE TABLE IF NOT EXISTS lifespan_events (
    id                  INTEGER PRIMARY KEY AUTOINCREMENT,
    person_id           TEXT NOT NULL REFERENCES persons(id),
    effective_day_index INTEGER NOT NULL,
    days_added          INTEGER NOT NULL,     -- 正数延寿、负数减寿
    reason              TEXT NOT NULL,        -- breakthrough / item / event / gm
    source_type         TEXT,
    source_id           TEXT,
    note                TEXT NOT NULL DEFAULT '',
    created_at          TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_lifespan_person ON lifespan_events(person_id, effective_day_index);

-- 动物
CREATE TABLE IF NOT EXISTS animals (
    id               TEXT PRIMARY KEY,
    species_id       TEXT NOT NULL,
    pos_x            INTEGER NOT NULL,
    pos_y            INTEGER NOT NULL,
    alive            INTEGER NOT NULL DEFAULT 1 CHECK (alive IN (0,1)),
    age_days         INTEGER NOT NULL DEFAULT 0,
    death_day_index  INTEGER,
    respawn_day_index INTEGER,
    attrs_json       TEXT NOT NULL DEFAULT '{}',
    created_at       TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_animals_pos   ON animals(pos_x, pos_y);
CREATE INDEX IF NOT EXISTS idx_animals_alive ON animals(alive);

-- 植物
CREATE TABLE IF NOT EXISTS plants (
    id               TEXT PRIMARY KEY,
    species_id       TEXT NOT NULL,
    pos_x            INTEGER NOT NULL,
    pos_y            INTEGER NOT NULL,
    stage            TEXT NOT NULL DEFAULT 'seedling'
                     CHECK (stage IN ('seedling','mature','withered')),
    stage_days       INTEGER NOT NULL DEFAULT 0,
    alive            INTEGER NOT NULL DEFAULT 1 CHECK (alive IN (0,1)),
    respawn_day_index INTEGER,
    attrs_json       TEXT NOT NULL DEFAULT '{}',
    created_at       TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_plants_pos   ON plants(pos_x, pos_y);
CREATE INDEX IF NOT EXISTS idx_plants_alive ON plants(alive);

-- 物品实例
CREATE TABLE IF NOT EXISTS items (
    id               TEXT PRIMARY KEY,
    template_id      TEXT NOT NULL,
    owner_id         TEXT REFERENCES persons(id),
    quantity         INTEGER NOT NULL DEFAULT 1 CHECK (quantity >= 1),
    stackable        INTEGER NOT NULL DEFAULT 1 CHECK (stackable IN (0,1)),
    attrs_json       TEXT NOT NULL DEFAULT '{}',
    acquired_day_index INTEGER NOT NULL,
    created_at       TEXT NOT NULL DEFAULT (datetime('now')),
    updated_at       TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_items_owner ON items(owner_id);

-- 灵石（独立货币）
CREATE TABLE IF NOT EXISTS spirit_stones (
    person_id  TEXT PRIMARY KEY REFERENCES persons(id),
    amount     INTEGER NOT NULL DEFAULT 0 CHECK (amount >= 0),
    updated_at TEXT NOT NULL DEFAULT (datetime('now'))
);

-- 事件日志（个人/地块/区域/世界）
CREATE TABLE IF NOT EXISTS event_log (
    id          INTEGER PRIMARY KEY AUTOINCREMENT,
    day_index   INTEGER NOT NULL,
    scope       TEXT NOT NULL CHECK (scope IN ('person','tile','region','world')),
    event_type  TEXT NOT NULL,
    tile_x      INTEGER,
    tile_y      INTEGER,
    region_id   TEXT,
    person_id   TEXT,
    template_id TEXT,
    params_json TEXT NOT NULL DEFAULT '{}',
    result_text TEXT NOT NULL DEFAULT '',
    created_at  TEXT NOT NULL DEFAULT (datetime('now'))
);

CREATE INDEX IF NOT EXISTS idx_event_day    ON event_log(day_index);
CREATE INDEX IF NOT EXISTS idx_event_person ON event_log(person_id, day_index);
CREATE INDEX IF NOT EXISTS idx_event_scope  ON event_log(scope, day_index);

-- Region 运行态覆盖（世界事件改灵气等）
CREATE TABLE IF NOT EXISTS region_states (
    region_id         TEXT PRIMARY KEY,
    aura_override     REAL,
    flags_json        TEXT NOT NULL DEFAULT '{}',
    updated_day_index INTEGER NOT NULL
);

-- LLM 调用审计（v0.4 启用）
CREATE TABLE IF NOT EXISTS llm_audit_log (
    id            INTEGER PRIMARY KEY AUTOINCREMENT,
    day_index     INTEGER NOT NULL,
    npc_id        TEXT NOT NULL,
    prompt_hash   TEXT NOT NULL,
    request_json  TEXT NOT NULL DEFAULT '{}',
    response_json TEXT NOT NULL DEFAULT '{}',
    validated     INTEGER NOT NULL DEFAULT 0 CHECK (validated IN (0,1)),
    error_text    TEXT NOT NULL DEFAULT '',
    created_at    TEXT NOT NULL DEFAULT (datetime('now'))
);
```

### 12.3 存档安全与备份

| 措施 | 说明 |
|---|---|
| WAL | 崩溃后保留最后一个已提交检查点 |
| synchronous=FULL | 每次提交落盘 |
| 单事务结算 | 一次移动=一个事务，要么全生效要么全不生效 |
| 启动备份 | 打开数据库前，把上次 `save.db` 复制为 `data/backups/save-<时间戳>.db` |
| 定期备份 | 每 12 个游戏月检查点后，调用 `VACUUM INTO` 生成备份 |
| 备份保留 | 最近 20 份，自动清理更早备份 |
| 完整性检查 | 启动时执行 `PRAGMA integrity_check`，失败则提示从备份恢复并拒绝启动 |
| 配置哈希 | `world_state.map_config_hash` 记录生成地图的配置哈希，配置变更时告警并先备份 |

### 12.4 数据恢复

```text
1. 停止程序；
2. 从 data/backups/ 选择最近备份；
3. 用备份文件覆盖 data/save.db（同时删除 -wal、-shm 文件）；
4. 重新启动，程序执行 integrity_check 后继续。
```

---

## 13. HTTP API 与 GM 权限（本版范围）

### 13.1 GM 需求

用户明确要求“需要更改数据的权限”。第一版提供：

- 终端 GM 菜单（默认隐藏，按 `G` 进入，要求本地管理口令或开发模式）；
- HTTP GM 接口在 `--http` 模式开放（v0.5 实施），需要请求头 `X-Admin-Token`。

### 13.2 GM 命令清单

| 命令 | 参数 | 说明 |
|---|---|---|
| `add_cultivation` | `points` | 增加当前小层修为百分比 |
| `set_realm` | `realm_id, stage_index` | 设置境界与小层 |
| `set_progress` | `percent` | 设置修为进度 |
| `add_lifespan` | `days` | 延寿/减寿（写 lifespan_events） |
| `set_age` | `days` | 设置年龄 |
| `set_mood` | `value` | 设置心情 |
| `teleport` | `x, y` | 传送（必须合法且可进入） |
| `add_spirit_stones` | `amount` | 增减灵石 |
| `give_item` | `template_id, quantity` | 发放物品 |
| `spawn_npc` | `name?` | 生成 NPC |
| `advance` | `months` | 模拟器直接推进 N 个月（不移动） |
| `force_save` | - | 立即检查点 |
| `backup` | - | 立即备份 |
| `list_events` | `scope, limit` | 查询事件日志 |
| `reset_world` | `confirm` | 删除存档重新生成世界 |
| `show_state` | - | 输出完整世界状态 |

### 13.3 权限规则

- GM 修改同样走模拟器 Command 通道，由模拟器统一结算，不允许绕过；
- 所有 GM 修改写 `event_log`（`event_type=gm`）；
- 灵根：GM 只能在新建人物时指定，运行中修改被拒绝。

---

## 14. 错误处理、日志与可观测性

- 所有错误用 Python 异常链携带上下文（`raise ConfigError(...) from err`，如 `config realms.yaml: unknown technique id: xxx`）；
- 模拟器结算错误：事务回滚 + `logging.error()` + 返回调用方，进程不崩溃；
- 配置错误：启动即失败，打印具体文件与字段；
- 数据库错误：停止本次结算，必要时进入只读保护模式；
- 运行日志输出 stdout，可选写 `data/logs/game.log`；
- 所有世界变化写 `event_log`，关键操作（突破、死亡、延寿、GM）必须落库后才能向前端展示成功。

---

## 15. 测试方案

### 15.1 单元测试

| 模块 | 用例 |
|---|---|
| gametime | 年月日与 day_index 互转、跨年跨月、加减天数 |
| world/shape | rect/circle/polygon 的 `contains()` 与边界 |
| world/generation | 每格恰好一个 Region、无覆盖空洞、配置错误报错 |
| idgen | 重启后序号连续、前缀正确 |
| namegen | 随机姓名格式、空表报错 |
| config | 引用完整性、取值范围、未知字段 |
| action | 进度百分比、未到 100% 不结算 |
| cultivation | 小层自动晋升、突破公式 clamp、失败惩罚 |
| lifespan | 延寿事件求和、死亡判定顺序 |
| item | 堆叠规则、灵石非负 |

### 15.2 模拟器集成测试（固定种子）

1. **一步一月**：移动一次后世界时间恰好 +1 个月；
2. **动作推进**：移动动作 3 天完成、坐标更新、剩余 27 天分给下一动作；
3. **修炼结算**：修炼 30 天整月完成，修为只加一次；
4. **突破延寿不误死**：角色剩余 1 天寿命时，第 1 天突破成功加寿元，当日不死；
5. **月中死亡**：角色第 3 天死亡，第 1-2 天已完成动作保留，第 5 天动作被中断；
6. **延寿字典**：先减寿后延寿按生效日顺序结算，死亡检查使用正确值；
7. **世界事件**：第 6、12、18 个月触发；
8. **NPC 出生**：年末按概率出生且不超过 100 上限；
9. **动物植物**：每日移动/生长、采集成熟株、狩猎重生；
10. **存档恢复**：结算后强制退出，重启后 `day_index`、队列进度、实体位置完全一致；
11. **事务回滚**：注入第 20 天数据库错误，确认整月回滚、内存状态恢复；
12. **GM**：GM 修改写事件日志、非法传送被拒绝。

### 15.3 验收冒烟命令

```bash
cd /home/wyh/project/MyGame
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
pip install -r requirements-dev.txt   # 运行测试需要
python -m pytest tests/ -q
python -m mygame
# 终端内依次验证：查看状态 → 移动 → 世界+1月 → 入队修炼 → 再移动 → 修为增加
```

---

## 16. 实施清单（按版本迭代）

### v0.1 世界骨架与“一步一月”闭环

**目标：能创建人物、在地图上移动、世界按月结算、老死判定、存档恢复。**

| 任务 | 交付物 | 验收标准 |
|---|---|---|
| 初始化 Python 项目 | `requirements.txt`、`pyproject.toml`、`mygame/` 包目录 | `python -m mygame --help` 通过 |
| gametime 包 | Year/Month/Day/Timestamp/Calendar | 单测通过 |
| world 包 | Tile/Region/Shape/Map/generation | YAML 生成 64×64，无空洞 |
| config 包 | 全部 YAML dataclass 模型 + loader + validate | 坏配置启动即失败 |
| io + namegen | txt 读取、随机姓名 | 中英文姓名正常 |
| idgen | 五种 ID 生成 | 重启连续 |
| person 基础 | 人物字段、出生、历史 5 条 | 玩家初始创建成功 |
| action 接口 | Action/Queue/registry | 新动作可注册 |
| simulator | `advance_one_step()` 月/日循环、死亡最后、单事务 | 集成用例 1-6、11 通过 |
| lifespan | 延寿字典与死亡判定 | 用例 5、6 通过 |
| store | 迁移、仓储、备份 | 重启存档一致 |
| frontend | 终端菜单 + 10×10 ASCII 地图 | 可交互移动 |

### v0.2 修炼与突破

| 任务 | 交付物 | 验收标准 |
|---|---|---|
| cultivation 包 | Realm/LingGen/Technique/formula | 灵根占比校验通过 |
| cultivate 动作 | 30 天修炼、100% 结算、灵气/心情/功法加成 | 修为按配置增长 |
| breakthrough 动作 | 7 天突破、公式、成功/失败、冷却、轻伤 | 成功升大境界并延寿；失败减寿 |
| 小层晋升 | 进度满自动晋升 | 4 小层顺序正确 |
| 延寿联动 | 突破成功写 lifespan_events | 突破当天不死 |
| 5 本功法 | 配置加载与选择 | 速度/突破加成生效 |

### v0.3 其余动作与生态

| 任务 | 交付物 | 验收标准 |
|---|---|---|
| entertain/emotion | 娱乐 + 情绪动作 | 心情/性格正确变化 |
| hunt/gather | 狩猎、采集 | 掉落、灵石、失败分支正确 |
| item 包 | 模板/实例/堆叠/JSON 词条 | 物品发放与消耗正确 |
| spiritstone | 独立货币 | 非负、掉落、GM 增减 |
| ecology | 20 动物/20 植物配置、每日移动/生长 | 阶段推进、重生正确 |
| 地块事件 | 到达触发 Region/Tile 事件 | 事件日志正确 |

### v0.4 NPC 与 AI

| 任务 | 交付物 | 验收标准 |
|---|---|---|
| personality | 五维生成与事件修改 | 范围 0-100 |
| 规则 AI | 每月决策、性格权重、突破/移动策略 | NPC 可自主修炼突破 |
| 人口系统 | 年末出生、100 上限、死亡移除 | 人口曲线合理 |
| DeepSeek 客户端 | `llm.yaml` + 提示词 + 异步调用 | `enabled=false` 时零网络请求 |
| LLM 动作链 | JSON 解析 + `can_execute()` 校验 + 回退 | 非法动作被丢弃 |
| 突发状况 | 世界事件触发 LLM 提案 | 校验通过才执行 |

### v0.5 GM、HTTP 与收尾

| 任务 | 交付物 | 验收标准 |
|---|---|---|
| GM 终端菜单 | 第 13.2 节全部命令 | 修改有权限、有日志 |
| HTTP API | `--http` 模式 + `/api/state`、`/api/command` | curl 可读状态、发指令 |
| HTTP GM | `X-Admin-Token` + 管理接口 | 无 token 拒绝 |
| 事件查询 | `list_events` 终端/HTTP | 可按 scope 过滤 |
| 文档 | README、运行说明、配置说明 | 新手可按文档跑通 |
| 全量测试 | 15 节用例 | `python -m pytest tests/ -q` 全绿 |

---

## 17. 验收标准（本版完成定义）

- [ ] `python -m mygame` 在 VSCode 中可直接启动；
- [ ] 玩家移动一格，世界时间恰好推进一个月，事件按第 9 节顺序结算；
- [ ] 移动、突破、娱乐、修炼、狩猎、采集六类实际动作 + 情绪动作全部注册且可执行判定正确；
- [ ] 动作进度未到 100% 不产生效果；
- [ ] 延寿事件字典按生效日参与死亡判定，死亡永远是当天最后判定；
- [ ] 玩家寿终保留存档，可 GM 回滚；
- [ ] SQLite 开启 WAL/FULL，一次移动一个事务，备份可恢复；
- [ ] 重启进程后世界时间、人物、队列、动物、植物、物品完全一致；
- [ ] 所有 YAML 配置通过校验，所有动态属性走 JSON 列；
- [ ] 20 种动物、20 种植物可从配置文件增删；
- [ ] NPC 规则 AI 每月决策，人口不超上限；
- [ ] DeepSeek 配置默认关闭，接口可异步调用并校验动作；
- [ ] GM 修改有权限控制并写事件日志；
- [ ] `python -m pytest tests/ -q` 全部通过。

---

## 18. 后续扩展预留（不实现，只留口子）

| 未来功能 | 预留方式 |
|---|---|
| 动画/小程序/2D | 前端接口 + `sprite` 字段 + HTTP API |
| 多玩家/PVP | ID 体系、`asyncio.Queue` 命令入口天然支持多请求源 |
| 登录 | `persons.kind=player` 外接账号表即可 |
| 离线收益 | `world_state` 加 `last_real_seen_at`，按 `time_scale` 补算 |
| ECS 化 | 如 Person 属性爆炸，把灵根/寿命/背包拆为组件，保持领域方法不变 |
| Mongo/PG/Redis | JSON 列 → JSONB；store 接口化替换驱动 |
| 战斗系统 | 新增 `Combat` 动作与结算系统，动作注册表直接扩展 |
| 任务系统 | 地块事件模板加 `quest` 类型 |
| 动物繁衍 | animals 表加 `gender/partner_id` 字段 |
| 配置热更 | config 加版本号与安全重载白名单 |

---

## 附录 A：第一版默认数值总表

| 参数 | 默认值 |
|---|---|
| 地图 | 64×64 |
| 时间 | 1年=12月，1月=30天 |
| 移动 | 3 天/格，触发 1 个月结算 |
| 修炼 | 30 天，完成才结算修为 |
| 突破 | 7 天；基础成功率 0.6 |
| 娱乐 | 3 天 |
| 狩猎 | 15 天 |
| 采集 | 5 天 |
| 情绪动作 | 0 天，即时 |
| 境界 | 炼气→筑基→金丹→元婴；各 4 小层 |
| 基础寿元 | 100/200/400/800 年 |
| 突破失败 | 修为-10%、轻伤30天、冷却5天、减寿1年 |
| 灵根 | 五行占比，总和 100 |
| 性格 | 谨慎/仁善/孤僻/贪婪/坚韧，各 0-100 |
| NPC | 初始 30，上限 100 |
| 世界事件 | 每 6 个月一次 |
| 动作历史 | 每人显示最近 5 条 |
| 心情 | 0-100，默认 80 |
| 备份 | 启动备份 + 每 12 游戏月备份，保留 20 份 |
| 存档 | 单存档 `data/save.db` |

---

## 附录 B：20 种动物与 20 种植物初始配置建议

### 动物（可按需删改）

| id | 名称 | tier | danger | 主要掉落 |
|---|---|---|---|---|
| wild_rabbit | 野兔 | 1 | 1 | 兔皮、少量灵石 |
| pheasant | 山鸡 | 1 | 1 | 鸡羽 |
| wild_boar | 野猪 | 1 | 2 | 兽皮、獠牙 |
| fox | 狐狸 | 1 | 1 | 狐皮 |
| deer | 鹿 | 1 | 1 | 鹿角、兽皮 |
| wolf | 狼 | 2 | 3 | 狼皮、狼牙 |
| bear | 熊 | 2 | 4 | 熊皮、熊胆 |
| snake | 蛇 | 1 | 2 | 蛇胆 |
| eagle | 鹰 | 2 | 3 | 鹰羽、鹰爪 |
| monkey | 猴 | 1 | 1 | 猴儿酒（稀有） |
| carp | 鲤鱼 | 1 | 1 | 鱼鳞 |
| tortoise | 龟 | 1 | 1 | 龟甲 |
| crane | 鹤 | 2 | 2 | 鹤羽 |
| spirit_rabbit | 灵兔 | 2 | 1 | 灵兔毛 |
| spirit_deer | 灵鹿 | 3 | 2 | 鹿茸、灵血 |
| spirit_crane | 灵鹤 | 3 | 2 | 灵鹤羽 |
| demon_wolf | 妖狼 | 4 | 5 | 妖丹（稀有） |
| demon_snake | 妖蛇 | 4 | 5 | 妖丹（稀有） |
| red_fox | 赤狐 | 3 | 3 | 赤狐尾 |
| white_ape | 白猿 | 4 | 4 | 白猿骨、灵果 |

### 植物（可按需删改）

| id | 名称 | tier | 阶段天数（幼苗/成熟/枯萎） | 主要产物 |
|---|---|---|---|---|
| ling_cao | 灵草 | 1 | 90/180/360 | 灵草 |
| huichun_cao | 回春草 | 1 | 90/180/360 | 回春草（疗伤） |
| juling_hua | 聚灵花 | 2 | 180/360/720 | 聚灵花粉 |
| ironwood | 铁木 | 1 | 180/360/720 | 铁木 |
| qingzhu | 青竹 | 1 | 120/300/600 | 青竹 |
| zhuguo | 朱果 | 4 | 360/720/1440 | 朱果（突破材料） |
| xueshen | 血参 | 3 | 270/540/1080 | 血参 |
| lingzhi | 灵芝 | 3 | 270/540/1080 | 灵芝 |
| yuejian_cao | 月见草 | 2 | 180/360/720 | 月见草 |
| zisu | 紫苏 | 1 | 60/150/300 | 紫苏 |
| hanyan_cao | 寒烟草 | 2 | 180/360/720 | 寒烟草 |
| shihu | 石斛 | 2 | 150/300/600 | 石斛 |
| fuling | 茯苓 | 2 | 180/360/720 | 茯苓 |
| huangjing | 黄精 | 1 | 120/240/480 | 黄精 |
| tianma | 天麻 | 2 | 180/360/720 | 天麻 |
| xuelian | 雪莲 | 4 | 360/720/1440 | 雪莲 |
| longdan | 龙胆 | 2 | 150/300/600 | 龙胆 |
| gouqi | 枸杞 | 1 | 120/240/480 | 枸杞 |
| heshouwu | 何首乌 | 3 | 270/540/1080 | 何首乌 |
| biluo_teng | 碧罗藤 | 3 | 270/540/1080 | 碧罗藤 |

> 以上数值均为初始占位配置，最终以 `configs/animals.yaml` 与 `configs/plants.yaml` 为准。

---

## 附录 C：事件类型枚举（第一版）

| event_type | scope | 触发时机 |
|---|---|---|
| move | person | 移动动作完成 |
| cultivate | person | 修炼动作完成 |
| breakthrough_success | person | 突破成功 |
| breakthrough_fail | person | 突破失败 |
| entertain | person | 娱乐完成 |
| hunt_success / hunt_fail | person | 狩猎完成 |
| gather_success / gather_fail | person | 采集完成 |
| emotion | person | 情绪动作执行 |
| birth | person | NPC 出生 |
| death | person | 人物死亡 |
| action_interrupted | person | 动作因死亡/中断取消 |
| tile_arrive | tile | 到达新地块 |
| world_event | world | 每 6 个月世界事件 |
| gm | person/world | GM 修改 |

---

## 附录 D：关键运行命令

```bash
cd /home/wyh/project/MyGame

# 初始化虚拟环境（首次）
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate

# 安装依赖
pip install -r requirements.txt
pip install -r requirements-dev.txt   # 测试依赖，可选

# 运行
python main.py
# 或
python -m mygame

# 测试
python -m pytest tests/ -q

# 带 HTTP 模式
python main.py --http :8080

# 查看存档
sqlite3 data/save.db "select * from event_log order by id desc limit 20;"
```
