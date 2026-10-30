---
name: webui-reference-builder
description: "仅在用户显式调用 /webui-reference-builder 时，调用名为 playwright-cli 的 Skill，根据指定 URL 使用真实浏览器运行时探索嵌入式设备 WebUI，按控件类别采集典型定位证据，并记录设备参数、状态和告警清单，支持页面资料增量更新。"
argument-hint: "<网址>"
disable-model-invocation: true
user-invocable: true
---

# 通用 WebUI 参考资料构建器

为嵌入式设备 WebUI 构建经过真实浏览器运行时观察的参考资料，供后续 Python、pytest、Playwright 和 POM 工作流生成稳定 locator 使用。重点记录可查询或配置的设备参数、状态、告警等业务元素，而非罗列所有通用页面控件。

本 skill **只能通过显式调用使用**。不得根据 URL、项目文件或上下文自动启动浏览器或猜测页面结构。

## 适用场景

使用本 skill 的场景包括：

- 为 WebUI 建立或更新页面对象参考资料；
- 探索真实存在的主菜单、子菜单、Tab、表单、表格、弹窗和只读状态区域；
- 为后续 locator 生成保存带上下文的 Snapshot、outerHTML 和控件证据；
- 对已有采集结果做增量更新，并标记结构变化或失效 locator。

本 skill 不生成完整 pytest 测试脚本，不执行设备配置变更，也不替代人工登录。

## 输入

用户必须提供目标 URL，例如：

```text
/webui-reference-builder https://example.com
```

如果未提供 URL，必须在打开浏览器前询问用户，不得猜测 URL。示例 URL 仅用于说明调用格式，不是无参数调用时的默认目标。

## 前置检查

开始页面采集前，按顺序检查并报告：

1. 当前工作目录和项目文件，确认不会无条件覆盖已有资料；
2. Claude Code 配置和可用相关 Skills；
3. 名为 `playwright-cli` 的 Skill 是否可用，以及其提供的浏览器操作能力；
4. 该 Skill 支持的页面快照、DOM/HTML 导出和浏览器会话能力。

## 参考资料输出

所有页面资料必须保存到项目根目录的 `./webui-references/` 下。打开目标 URL 并确认页面标题后，以页面标题创建独立的资料目录；目录名称应保留标题信息，并转换为安全、稳定的文件夹名称。后续该目标页面获取的 manifest、页面、Tab、控件、截图和 README 等资料，全部保存在该标题目录下。

例如，页面标题为 `Network Settings` 时，目录结构为：

```text
./webui-references/Network-Settings/manifest.yaml
./webui-references/Network-Settings/pages/<page_id>/snapshot.txt
./webui-references/Network-Settings/pages/<page_id>/outer.html
./webui-references/Network-Settings/pages/<page_id>/controls.yaml
./webui-references/Network-Settings/pages/<page_id>/controls/<control_id>.outer.html
./webui-references/Network-Settings/pages/<page_id>/screenshots/*
./webui-references/Network-Settings/README.md
```

如果项目已有明确的参考资料目录或 manifest 文件，应先沿用既有布局，并在对应的标题目录中更新；不得静默迁移、删除或覆盖人工维护内容。manifest 中的路径必须相对于该标题目录中的 manifest 文件解析。

所有资料只记录事实。未知属性使用空值或 `unknown`，不得根据框架约定补写路由、控件属性或验证结果。

## 执行流程

### 1. 检查并打开页面

前置检查通过后，调用名为 `playwright-cli` 的 Skill，使用其提供的浏览器能力打开用户提供的 URL。不得在本 Skill 中直接执行名为 `playwright-cli` 的命令行工具，也不得直接调用其他 Playwright 工具替代该 Skill。

随后获取 Snapshot，并记录页面是否可达、标题、当前 URL、可见文本摘要、登录状态和控制台错误。

确认页面标题后，立即确定本次采集使用的标题目录：

1. 在 `./webui-references/` 下查找与页面标题对应的现有目录；
2. 如果不存在，则创建该标题目录；
3. 后续所有资料只写入该标题目录，不得写入 `./webui-references/` 根目录或其他页面标题目录；
4. 页面标题为空或无法可靠获取时，暂停采集并报告，不能根据 URL 或产品类型猜测目录名称。

如果出现登录表单、认证拦截、验证码或其他人工步骤：

- 立即暂停自动探索；
- 提示用户在已打开的浏览器中自行完成授权；
- 不索取、读取或记录密码、Token、Cookie、Authorization header 或其他凭据；
- 只有用户确认访问已完成后，才能继续采集。

如果页面无法打开，保留失败记录并停止后续页面探索，不得生成虚构的页面或控件资料。

### 2. 探索导航层次

创建 manifest 前，必须先探索真实渲染页面。不得根据框架约定推测菜单名称或路由。

1. 使用 `playwright-cli` Skill 提供的 Snapshot，必要时使用该 Skill 提供的页面评估能力，识别可见的一级菜单项；
2. 逐个打开安全的一级菜单，等待页面渲染稳定；
3. 记录所有可见的嵌套菜单项、子菜单组和导航动作。采集运行时证据：
   - visible text；
   - `id`、`name`、`data-testid` 或其他稳定属性；
   - `href`、hash、观察到的 URL 或面板状态；
   - `onclick` 或其他实际观察到的动作。
4. 递归展开可见子菜单组；面板式导航要记录打开后的状态；
5. 对重复标签按菜单路径、ID、路由或容器分别建条目；
6. 只读导航优先。遇到保存、应用、确认修改、重启、恢复出厂、升级固件、注销或可能中断连接的控件，记录但不执行。

manifest 示例：

```yaml
version: 1

source:
  url: "http://192.168.197.1"
  title: "Observed page title"
  captured_at: "2026-10-29T13:00:00+08:00"
  traversal: main-menu
  login_required: true
  status: completed

navigation:
  - id: menu-example
    name: Example
    route: "#example"
    action: "selectPage('example')"
    type: main
    visited: true
    children: []

notes:
  - "Names and routes were observed in the live browser."
```

manifest 规则：

- 使用实际观察到的 URL、标题、名称、路由和动作；
- 使用实际观察到的 `type: main`、`group`、`page` 或其他明确类型；
- 只有菜单项成功打开后，才能设置 `visited: true`；
- 只有观察到子项时才添加 `children`；
- 不得为未访问内容虚构 `html`、`tabs`、路由或控件；
- 页面无法打开时，在 `notes` 记录真实原因，并将状态设为失败或待人工确认。

### 3. 采集页面与控件

对 manifest 中每个已访问页面：

1. 在可见 DOM 和 Snapshot 中识别 Tab、Tab 列表、类似 Tab 的链接和面板；可检查 `role="tab"`、`aria-controls`、`aria-selected`、`.tab`、`.el-tabs` 等语义，但不得假设特定前端框架；
2. 安全打开所有可检查的 Tab，等待内容稳定，并记录名称、稳定标识、路由/状态和采集时间；
3. 先分析页面中可供用户查询或配置的设备业务元素，按实际 DOM 结构和交互形式分类，例如只读参数、文本/数字输入框、下拉选择框、开关、复选框、告警状态或告警表格。上述仅为示例，以实际页面为准；同名控件若结构或定位方式不同，应分为不同类别；
4. 每类选择一个实际观察到、具有代表性的控件，采集包含其标签和定位上下文的最小稳定容器 outerHTML，生成该典型控件的 Playwright locator，并记录证据和验证状态。代表控件无法安全访问或定位时，记录原因，不得虚构；
5. 在该类别下列出当前页面或 Tab 中实际观察到的设备参数、状态、告警等业务元素，记录名称、所属页面/Tab、可查询或配置的用途，以及足以区分同类元素的实际标签或稳定标识。只列业务相关元素，不逐个罗列通用导航、布局、装饰性元素或无关按钮；保存、应用等危险操作仍须按安全约束记录，不执行；
6. 按类别写入 `controls.yaml`。保留现有文件和人工维护内容；每个类别至少包括典型控件、outerHTML 路径、locator 及证据，以及该类别的业务元素清单。例如：

```yaml
- control_id: profile-select
  page_id: network-settings
  tab: General
  description: Profile selector category
  control_type: combobox
  visible_text: Profile
  role: combobox
  accessible_name: Profile
  label: Profile
  attributes: {}
  representative_html: pages/network-settings/controls/profile-select.outer.html
  locator_candidates:
    - 'page.get_by_role("combobox", name="Profile")'
  locator_evidence:
    - "The representative control has role combobox and accessible name Profile."
  verified: false
  side_effect: unknown
  elements:
    - name: Profile
      purpose: configurable
      identifying_evidence: "Label: Profile"
  notes: "示例 locator 尚未单独验证；清单不代表同类元素均可直接套用该 locator。"
```

以上字段和值仅示例结构，采集时必须替换为实际观察结果。`elements` 清单按类别列出业务元素，不记录参数值、告警详情中的敏感内容或凭据。相同名称出现在不同容器时，记录容器或其他实际证据以区分；不能区分时标记待确认。典型 locator 只用于其代表控件，同类其他元素是否可沿用相同定位模式需分别依据其证据确认，不得把代表控件的验证状态继承给整类。

只有真实 DOM 观察到的属性才能写入 `attributes` 或定位证据；无法确认的字段留空。候选 locator 必须附证据，并优先使用可访问角色和名称、关联标签或实际观察到的稳定属性；仅在真实浏览器中确认匹配唯一且指向预期代表控件后，才能标记 `verified: true`。无副作用的匹配检查不能确认时，保留 `verified: false` 并说明原因，不为验证而执行配置操作。

7. 需要认证、会产生副作用或不可用的 Tab、类别或控件，记录跳过原因，不得猜测内容。

存在 Tab 时，在对应页面条目使用 `tabs` 列表：

```yaml
tabs:
  - name: General
    id: tab-general
    route: "#example"
    html: pages/example/general/outer.html
    snapshot: pages/example/general/snapshot.txt
    captured_at: "2026-10-29T13:05:00+08:00"
```

### 4. 保存 Snapshot 与 HTML 参考

对于每个已访问页面及其每个已访问 Tab，保存 Snapshot；需要定位上下文时保存最小稳定容器的 outerHTML。另为第 3 节中每个已识别类别的代表控件保存独立 outerHTML：

1. 页面稳定后，使用 `playwright-cli` Skill 提供的页面评估能力读取实时页面；
2. 容器选择优先级为：当前 Tab 的 `aria-controls` 面板、可见活动面板（如 `[role="tabpanel"]` 或 `.el-tab-pane.is-active`）、包含页面控件和标题的最小稳定容器；
3. 不保存 `document.documentElement.outerHTML`，也不只保存扁平控件列表；
4. 保留标题、标签、字段、表格行、按钮、Tab、Dialog 与业务上下文的关系；
5. 保留 `id`、标签、`name`、`placeholder`、`aria-*`、`data-*`、可见文本和相关 class 等定位证据；
6. 文件名使用确定性、无冲突的小写 ASCII 路径，例如：

```text
webui-references/Network-Settings/pages/network-settings/mobile-network/outer.html
webui-references/Network-Settings/pages/network-settings/mobile-network/snapshot.txt
```

7. HTML 必须是有效文本，并在当前容器外包含简短元数据包装：

```html
<!doctype html>
<html>
<body>
<section data-reference="tab-container" data-page="example" data-tab="general"
         data-source-selector="[role=tabpanel][aria-labelledby=tab-general]">
  <!-- outerHTML of the observed container -->
</section>
</body>
</html>
```

8. 将页面和 Tab 文件的相对路径添加到对应的 manifest 条目，将代表控件文件的相对路径添加到对应的 `controls.yaml` 条目；所有路径均相对于 manifest 所在目录解析。不同页面、Tab、类别及交互状态使用不同文件，不得互相覆盖。代表控件文件同样保留标签、控件与必要业务上下文，并使用上述元数据包装标明页面、Tab 和来源；
9. 仅在无法安全或可靠获取某类别代表控件时省略其文件，记录原因并将该类别标记为未完成；不得用其他类别或未经观察的 HTML 代替。

Snapshot 和 outerHTML 只反映采集时状态，不能证明 locator 唯一或已经通过运行时验证。

### 5. 采集典型交互状态

如果 UI 使用运行时创建或依赖状态的 DOM，只采集安全且有代表性的状态，例如：

- 打开 Element UI `el-select` 或 combobox；
- 打开日期选择器；
- 打开非破坏性 Dialog；
- 展开折叠面板、手风琴或表格行；
- 聚焦会改变定位相关标记的输入框。

一次只执行一个交互并等待稳定。若弹出层渲染在 Tab 外，单独保存其最小容器，并记录触发器和条件：

```yaml
states:
  - name: profile-select-open
    trigger: "#profile-select"
    condition: "dropdown expanded"
    html: pages/network-settings/profile-select-open.outer.html
    captured_at: "2026-10-29T13:10:00+08:00"
```

继续下一个状态前恢复基线状态，不保存凭据、已提交的值、Cookie 或授权数据。不要枚举所有值或交互组合。

### 6. WebUI 安全约束

默认只执行页面读取和无副作用导航。未经用户明确授权，禁止：

- 修改业务、网络、系统或安全相关参数；
- 修改 Wi-Fi 密码或安全模式；
- 点击保存、应用或确认修改的按钮；
- 重启设备、恢复出厂设置或升级固件；
- 执行可能中断当前管理连接的操作；
- 猜测、暴力破解或自动试探登录凭据。

无法判断副作用时，默认按有风险处理，只记录不执行。不得将密码、Cookie、Token、设备标识、个人信息或其他敏感值写入参考资料、日志或报告；出现敏感字段时只记录字段名称。

### 7. 增量采集

再次运行时：

1. 读取已有 manifest 和参考文件；
2. 按页面 ID、路径、Tab 标识比较运行时结构；
3. 尽量只更新变化页面；
4. 保留人工备注和验证记录，不静默覆盖人工维护字段；
5. 对失效或未重新验证的 locator 标记 `verified: false` 或 `needs_review`；
6. 比较各类别的代表控件和业务元素清单，更新发生变化的证据；不把代表控件验证结果传播给其他元素；
7. 对重复映射在报告中说明是否有意保留。

## 校验与完成标准

报告成功前必须完成：

1. 确认 manifest 可解析为 YAML；
2. 确认所有引用路径相对于 manifest 所在目录，且目标文件存在、非空；
3. 确认基线文件包含页面/Tab 元数据和内容容器；
4. 确认状态文件包含触发器、条件和对应文件；
5. 检查重复页面/Tab 映射，并报告有意保留项；
6. 扫描 manifest、HTML、Snapshot、控件清单和报告，确认未写入敏感值；
7. 检查每个已采集类别的代表控件 outerHTML 路径、locator、定位证据和业务元素清单；确认 locator 验证状态只属于代表控件，未采集的类别说明原因；
8. 明确区分 `observed`、`candidate`、`verified`、`skipped` 和 `failed` 状态。

只有以上检查通过，才能报告对应页面或 Tab“采集完成”。登录未完成、`playwright-cli` Skill 不可用、页面打不开或引用缺失时，必须报告为未完成。

## 输出报告格式

完成或暂停时，简要报告：

```text
source URL/title: ...
playwright-cli Skill: available | unavailable | partial
visited pages/menu items/tabs: ...
control categories and business element counts: ...
representative locators verified/unverified: ...
created or updated files: ...
skipped items and reasons: ...
validation: passed | partial | failed
unresolved issues: ...
```

报告不得声称未实际访问的路由、页面或控件已采集。

## 失败处理

- 如果后续页面失败，应保留已经有效的参考文件；
- 如果导航或页面稳定过程失败，不得声称页面或 Tab 已完成探索；
- 明确报告浏览器、超时、导航和解析错误；
- 如果 `playwright-cli` Skill 不可用或功能无法确认，先报告环境，不得直接使用命令行工具或切换到其他浏览器工具；
- 如果需要登录，应暂停并等待用户手动完成，不得尝试绕过认证；
- 如果 UI 只显示菜单组而没有可以安全打开的页面，应在 `notes` 中记录该菜单组及其限制。