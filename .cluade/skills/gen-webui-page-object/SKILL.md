---
name: gen-webui-page-object
description: "根据 webui-reference-builder 采集的真实 WebUI 参考资料，生成或增量更新同步 Playwright Python 页面对象、业务 Facade、能力清单、API 文档和安全只读自检。适用于页面对象、POM、WebUI Facade、页面能力清单和 locator 证据生成；不得猜测未观察到的页面、控件或路由。"
argument-hint: "[可选：参考资料目录、页面标题目录或生成范围]"
user-invocable: true
disable-model-invocation: true
---

# 通用 WebUI 页面对象生成器

读取项目内由skill: `webui-reference-builder` 采集的真实资料，按产品/站点边界生成可复用的
Python 页面对象和业务 Facade。所有页面、Tab、控件、定位器和能力都必须有参考资料
证据；本 skill 不负责浏览器探索，也不把静态分析伪装成运行时验证。

## 适用场景

- 根据 `webui-references/` 生成或更新 POM 页面对象；
- 为已观察页面生成面向测试意图的业务 Facade；
- 生成 `webui-api.md` 和 `webui-capabilities.yaml`；
- 增量同步页面、Tab、控件或定位器证据；
- 为已实现且安全的只读 API 生成最小 pytest 自检。

默认执行离线生成和本地验证，不自动连接设备。用户明确要求真实页面探索、登录或 locator
验证时，按第 6 节交接给适用的浏览器探索技能；获取并核对实测证据后再更新本 Skill
生成的代码与清单。

## 硬性边界

1. 必须使用同步 Playwright Python API。
2. 不得根据 URL、厂商习惯、页面标题、前端框架或示例值猜测 DOM、路由、菜单、控件、
   参数、默认值或断言。
3. 账号、密码、Cookie、Token、Authorization header、设备地址等由用户通过安全配置提供；
   不得将真实值写入生成的代码、测试、资料、日志或文档。
4. Facade 只暴露有业务含义的 API，不暴露通用的 `click_button()`、`fill_input()`、
   `locator()` 等测试封装。
5. 第一阶段只实现安全的只读查询。配置、保存、应用、重启、恢复出厂、升级、注销、
   连接中断等操作默认只登记能力，不自动执行；除非用户明确授权且证据足够。
6. 已访问且证据充分的页面可离线实现完整 API，并标记 `implemented: true,
   verified: false`；未访问、失败、跳过或缺少必要证据的能力不得声称已实现或已验证。
   未经实测的 locator 可作为候选实现的一部分，但不得声称已验证。

## 1. 前置检查和资料发现

先读取当前项目结构和编码规则，不得假设包名、Fixture、测试框架布局或已有页面对象。
按以下优先级查找并读取：

1. `webui-references/` 下的页面标题目录；
2. 每个标题目录的 `README.md`、`manifest.yaml`、`controls.yaml`；
3. `pages/<page_id>/snapshot.txt`、`outer.html` 和 Tab/状态子目录中的对应文件；
4. `controls/` 下代表控件的 HTML；
5. 项目已有的 `conftest.py`、页面对象、Facade、测试、配置和编码规范。

以每个标题目录中的 `manifest.yaml` 所在目录为路径根，解析其中所有相对路径。
存在多个标题目录时分别处理，禁止把不同产品/站点的控件或页面混在同一模块。

对 `manifest.yaml` 中的导航递归遍历 `children`，只为同时满足以下条件的页面生成代码：

- `visited: true`；
- 状态不是 `skipped` 或 `failed`；
- manifest 明确引用的必要文件存在且非空；无页面文件路径字段的旧版 manifest，
  检查本节约定的 `pages/<page_id>/snapshot.txt` 和 `outer.html` 均存在且非空，
  并记录使用的是约定路径，不能声称 manifest 显式引用了它们。

`visited: false`、`skipped`、`failed` 或引用缺失的条目必须保留为待处理/失败状态，
不得生成虚构页面对象。

若没有可解析的参考资料，只创建或更新本 skill，并报告缺少的目录、manifest、页面或
控件资料；不得生成虚构代码。

### 证据来源与归档包装

建立逐页面、Tab、业务字段和 locator 的证据映射：来源文件、原始设备 DOM 节点、
选择器、作用域及验证状态。区分设备原始 DOM、采集工具的归档包装和采集元数据：

- `outer.html` 的归档包装元素及其 `data-reference`、`data-page` 等属性不是设备
  DOM 证据；不得直接把包装节点或属性写成设备 locator。
- `data-source-selector` 等标注是来源选择器的**声明**，不是证明该节点一定存在、
  可见或唯一；需与原始 DOM、Snapshot 或运行时证据交叉核对。来源或归属不明确时
  标记 `needs_review`，不得猜测。
- 若采集资料提供 `source_dom`、`archive_wrapper`、`source_selector` 等来源标注，
  按其来源解释而非一律视为设备属性；旧资料不假设存在这些字段。需要补充证据时
  向采集工作流交接，不得擅自将推测写回参考资料。

## 2. 输出布局

根据项目已有命名规则确定 `<webui-package>`，不得固定使用厂商或设备名称。除非项目已有
明确布局，至少使用：

```text
3-scripts/<webui-package>/
├── pages/
│   ├── base_page.py
│   └── <page_id>_page.py
├── tests/
│   ├── test_manifest_pages.py
│   └── test_control_elements.py
├── conftest.py
├── webui_facade.py
├── webui_api.md
└── webui-capabilities.yaml
```

不要无条件创建示例目录；实际模块、类和方法名必须来自已观察的 `page_id`、菜单路径、
Tab 或业务元素，并转换为合法且稳定的 Python 名称。不同标题目录中相同页面名必须用
产品/站点上下文区分。

逐页面/Tab 建立交付矩阵：

| 交付状态 | 判定 |
|---|---|
| 已实现业务查询 | 至少一个有业务含义、证据充分的完整只读 API，包含返回类型、错误行为和对应测试 |
| 仅骨架 | 只有页面类、路由、状态方法或候选 locator；不得称为已实现业务能力 |
| 待采集 | 缺少页面、字段语义或可靠定位依据；列出缺失证据和后续动作 |
| 跳过/失败 | 未访问、被跳过、采集失败或必要文件缺失；保留具体原因 |

不机械要求每页都有业务 API；证据不足时保留待采集状态，不生成无业务意义的方法。

## 3. 页面对象

页面对象只负责：

- 进入页面或 Tab、等待稳定、检查页面状态；
- 在页面/Tab/表格行/业务容器作用域内定位真实控件；
- 只读查询和明确授权的交互；
- 将底层 Playwright 异常转换为清晰的领域错误。

推荐保持显式类型和页面边界：

```python
from playwright.sync_api import Page


class BaseWebUiPage:
    def __init__(self, page: Page) -> None:
        self.page = page


class ObservedPage(BaseWebUiPage):
    def is_open(self) -> bool:
        ...
```

不得把设备配置策略、保存/应用流程、跨页面业务流程或测试层断言写入页面对象。

## 4. Facade 和 API

Facade 组合页面对象并统一登录态、页面切换、等待和错误语义。公开方法必须表达真实的
业务意图，例如查询连接状态、客户端列表或无线设置，但只能使用参考资料中实际观察到
的名称和结构，不能照抄示例为未观察能力创建方法。

每个公开 API 在代码或 `webui-api.md` 中明确：

- 所属标题目录、页面和 Tab；
- 参数和类型、返回值和类型；
- 前置条件和页面/设备状态要求；
- 异常行为；
- 是否只读、是否修改设备、是否需要保存/应用；
- 是否可能中断连接；
- `implemented`、`verified`、`status` 及证据路径。

`implemented: true` 表示该具体 API 的业务逻辑、返回结构、错误处理及对应测试代码
完整存在，而非仅有类、路由或定位器；其定位证据必须充分。离线实现可标记
`implemented: true, verified: false, status: unverified`。`verified: true` 必须
有该 API 在实际浏览器中的运行证据；页面已打开或代表 locator 已匹配不能证明整个 API。
页面导航、各 locator 和业务 API 分别记录验证状态，不相互继承。未知字段使用空值或
`unknown`，不得用默认值掩盖未知状态。

每个查询明确字段的 `attached` / `visible` / `enabled` 要求及依据、空数据与加载失败
的区别、是否包含隐藏内容、文本/数值和单位、条件控件缺失时的行为。区分页面渲染行数
与设备总数；解析开关时对未知、缺失或冲突状态显式报错，不用默认值伪造结果。

## 5. Locator 证据规则

所有定位器必须来自当前页面或 Tab 的 Snapshot、outerHTML、`controls.yaml` 和控件
证据。优先级如下：

1. 运行时确认唯一的 role + accessible name；
2. 真实关联的 label；
3. 观察到且稳定的 `id`、`name`、`data-testid` 或语义属性；
4. 有明确容器上下文的可见文本；
5. 有证据支持的 CSS；
6. 仅在无更可靠选择且已有证据时使用 XPath。

必须使用页面、Tab、表格行或业务容器限制作用域，并保留同名控件的实际区分依据。
禁止为了通过测试而添加未经观察的：

- `nth()`、位置索引或固定 DOM 层级；
- 脆弱 XPath；
- 猜测的 id、class、route、`data-*` 属性；
- 采集时的动态示例值作为默认参数或断言。

`controls.yaml` 中 `verified: false` 的候选 locator 只能标记为 `candidate` 或
`unverified`，不得在文档中宣称已验证。没有可靠定位器时保留待实现状态，并说明
需要补充的运行时证据。
实测时对每个 locator 在目标页面/Tab 和业务容器内分别检查匹配数量、可见性和业务
关联；重复匹配不得当成唯一。仅当该次检查确实验证了相应 locator 与范围时，才可
更新该 locator 的验证记录，不将结果传播给同类元素或其他设备状态。

## 6. Fixture、安全和危险操作

浏览器生命周期、登录状态和设备连接必须由已有 Fixture 或测试基础设施管理。新增
Fixture 时复用项目模式，并将浏览器启动、登录态、页面对象和断言职责分离。

自检 Fixture 文件必须位于 `<webui-package>/conftest.py`，至少提供
`fxt_dut_web` 作为已登录 WebUI 测试对象的唯一入口。没有浏览器或登录态时必须显式
跳过并说明原因，不能伪造成功。禁止绕过登录、验证码或认证拦截。

生成代码时不执行设备配置变更。危险 API 必须由测试层显式调用，并由测试层决定是否
允许执行；默认测试只调用安全的只读 API。

### 离线与真实设备交接

1. 离线模式只使用现有证据，不自动登录或访问设备；可运行本地静态和离线行为测试，
   将缺失证据逐项列为待采集或待验证。
2. 用户明确要求实测时，先确认目标设备、已登录会话或用户提供的安全配置，以及只读
   范围；调用或交接给适用的浏览器探索/locator 验证技能，遵守其调用限制。若资料采集
   技能要求显式调用，先请用户显式调用；不得自行绕过。必须先检查登录脚本和 fixture
   是否适配当前设备，禁止使用其他设备的默认地址、脚本或凭据。
3. 实测仅限用户授权的登录、只读导航、读取和无副作用 locator 检查；不得点击保存、
   应用、连接、重启等按钮。获取脱敏的逐项结果后，再更新代码、证据和清单；不得凭
   静态结果或一次代表性匹配宣称全部能力已验证。
4. 设备不可达、登录失败、会话过期或跳回登录页应明确报错，不得视为空业务数据。
   真实设备测试须显式 opt-in，测试结束关闭由本次测试创建的浏览器上下文。

凭据优先使用环境变量或用户管理的未跟踪本地配置，仅提供无真实值的示例模板。检查
本地敏感配置是否可能被 Git 纳入，发现风险时提醒用户，不擅自修改现有凭据；不在
命令行参数、代码、日志、截图、快照、测试失败输出、文档或能力清单中泄露敏感值。
TLS 校验默认开启；证书异常时停止发送凭据，先让用户确认设备身份，再取得针对该设备
和本次会话的明确授权，方可临时忽略证书错误；优先使用可信 CA。

## 7. 最小自检

在项目已有 pytest 配置和浏览器 Fixture 的条件下，生成或更新以下自检；缺少基础设施
时明确报告或按项目约定补充，不能把未运行的测试当成通过：

### `tests/test_manifest_pages.py`

- 读取对应标题目录的 `manifest.yaml`；
- 递归遍历所有导航项及 `children`；
- 仅检查已访问且引用文件存在的页面；
- 通过 Facade 或页面对象验证页面可进入、页面标识或状态可识别；
- 对未访问、跳过、失败或引用缺失项明确跳过或失败，不得静默通过。

### `tests/test_control_elements.py`

- 读取 `controls.yaml`；
- 对每个已访问且有业务元素记录的页面检查至少一个代表性控件；
- 检查已访问页面是否有 controls 覆盖；缺失时报告 `needs_review` 或失败；
- 使用 controls 记录的 locator candidate 和证据；
- `verified: false` 不得报告为已验证，可显式 skip 或报告 `needs_review`；
- 不得点击保存/应用、改变配置或执行其他危险操作。

测试分层，分别报告执行、通过、跳过和未覆盖的页面/API：

1. 静态测试：解析清单、检查真实文件和实际页面类/API 映射，不以文件存在或硬编码
   ID 列表代替行为验证。逐项报告未访问、失败、缺文件和无 controls 覆盖。
2. 离线行为测试：使用受控 HTML 或项目现有测试基础设施覆盖路由/容器、可见和隐藏、
   未登录、空表、未知或冲突开关状态、重复 locator 及错误处理；这不等于设备验证。
3. 显式 opt-in 的设备只读测试：逐页面验证导航、locator 匹配数和作用域、业务 API
   的返回结构及类型，并记录每个项目结果。测试仅通过 Facade 或业务 API 操作，
   不直接写 CSS、XPath 或页面内部定位器；不把采集时动态值写成固定断言。

隔离的页面对象本地行为测试可为构造受控 DOM 使用选择器，但不能借此声称设备 locator
已验证。不能以测试通过总数替代逐页/API 覆盖情况。

## 8. 能力清单和文档

按页面/Tab 更新 `webui-api.md`，区分已实现、待实现、跳过和失败能力。同步更新
`webui-capabilities.yaml`，至少包含：

```yaml
- name: observed_capability
  page_id: observed-page
  tab: ""
  purpose: read_only
  implemented: false
  verified: false
  status: candidate
  side_effect: none
  requires_login: true
  source:
    manifest: webui-references/<title-directory>/manifest.yaml
    controls: webui-references/<title-directory>/controls.yaml
  verification:
    scope: ""                 # navigation、locator 或 business_api
    observed_at: null
    environment: unknown      # 不包含设备地址或标识
    action: ""
    match_count: null
    result: unknown
    test: ""
    limitations: ""
  notes: "仅填写参考资料实际支持的内容。"
```

能力字段必须与代码和证据一致。推荐状态包括 `observed`、`candidate`、`verified`、
`unverified`、`skipped`、`failed` 和 `needs_review`；不得把代表控件的验证结果传播
给同类的其他元素。
对每个验证项独立记录上述字段：运行时间、脱敏的设备/固件范围、动作、匹配数、测试
名称、结果和适用限制；不可获得的字段用 `null` 或 `unknown`。不要通过 YAML 锚点或
批量操作继承 `verified: true`；须基于每项独立结果。`status: verified` 与
`verified: true` 一致；API 的 `verified: true` 要求 API 已实现且实际运行成功。

## 9. 增量更新

再次运行时先读取现有 skill、页面对象、Facade、文档和能力清单，然后按标题目录、
`page_id`、Tab 标识和相对路径匹配已有实现：

1. 只更新新增或结构发生变化的页面；
2. 保留人工备注、异常说明和未变化代码；
3. 参考资料、DOM、路由、设备/固件范围或验证前置条件变化可能影响已有结论时，
   对受影响的 locator、导航和 API 分别标记 `verified: false, status: needs_review`，
   保留历史证据，并在重新验证成功后更新；
4. 不传播页面、Tab 或代表控件的验证结果；
5. 报告重复映射、跳过/失败页面和仍需人工确认的能力。

## 10. 验证和交付报告

完成后运行项目已有的最小格式化、类型检查和测试命令。至少验证：

1. 新增 Python 文件可导入；
2. manifest、controls 和能力清单可解析；
3. manifest 引用路径存在且非空；
4. 页面对象没有未观察到的路由或控件；
5. 代码、HTML、Snapshot、日志、文档和 YAML 不含敏感值；
6. `implemented`、`verified`、`status` 和 `side_effect` 语义一致。

真实设备或登录态不可用时完成可行的静态及离线行为校验，明确说明设备运行时验证未完成，
不能把静态或模拟结果报告为设备 locator 或 API 已验证。交付报告用简洁表格列出：

- 创建/更新的 Skill、页面对象、Facade、文档和测试；
- 使用的标题目录和已访问页面；
- 每页交付矩阵，以及导航、locator、API 各自的 `implemented`/`verified`/`status`；
- 未实现、跳过或失败页面及原因；
- 验证命令和结果；
- 仍需真实浏览器、登录态或用户授权的事项。
