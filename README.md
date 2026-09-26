# @goxjs/goxjs

**English** | [简体中文](#简体中文)

> Gox JavaScript Runtime — built from scratch in Go: lexer → parser → bytecode compiler → stack VM. Single binary, zero external dependencies.

This package distributes Gox's platform binaries via npm. Installing it gives you the `goxjs` command.

## Install

```bash
npm i -g @goxjs/goxjs
```

## Usage

**REPL:**

```bash
goxjs
```

```
Gox REPL (ES6 subset, no var)
Type :exit to quit, :help for help

> let x = 10
> [1, 2, 3].map(v => v * 2)
  [2, 4, 6]
```

**Run a script:**

```bash
goxjs app.js
```

**Run without installing:**

```bash
npx goxjs app.js
```

Supports the ES6+ language subset (`let`/`const`, classes, `async`/`await`, destructuring, template literals, ES modules), standard objects such as `Array` / `Map` / `Set` / `Promise` / `BigInt` / `Temporal`, and Node-style host APIs including `fs` / `http` / `fetch` / `path` / `process`.

## Declarative UI (GUI)

`gx/gfx` is a pure-Go, self-drawn windowing layer; you write the UI in JSX. Bindings, lists, and conditionals are **element-level directives** (written on the element, no import needed):

```jsx
import { h, render, createSignal, Switch, Match } from "gox";

const [name, setName]   = createSignal("");
const [rows, setRows]   = createSignal([{ id: "a", title: "Alpha" }]);
const [open, setOpen]   = createSignal(true);
const [phase, setPhase] = createSignal("loading");

render(
  <window title="app" width={420} height={260}>
    <column gap={8} padding={12}>
      <input model={name} />                                                  {/* two-way binding */}
      <view each={rows} key="id">{(row) => <text>{row.title}</text>}</view>   {/* list: keyed reuse */}
      <view show={open} fallback={<text>Hidden</text>}>Visible content</view> {/* conditional: kept alive */}
      <Switch fallback={<text>Unknown state</text>}>                          {/* multi-branch */}
        <Match when={() => phase() === "loading"}><text>Loading…</text></Match>
      </Switch>
    </column>
  </window>
);
```

- `model` — two-way binding for controlled components (like Vue's `v-model`): `input` / `textarea` / `slider` / `select` read and write `value`, `checkbox` / `switch` read and write `checked`, and `radio` writes its `value` attribute into the model when selected. You can also pass a `[get, set]` pair to wire up a custom source.
- `each` — list rendering: rows with the **same key + same row reference + same index** are reused as-is (inputs, scroll positions, and local signals inside the row are preserved); only rows that actually changed are rebuilt. Supports `stable` / `fallback` / `key="id"`.
- `show` — conditional visibility with **keep-alive**: hiding merely removes the subtree from layout flow; showing it again is instantaneous (no destroy/rebuild).
- `view` — a layout-transparent container (Fragment); the directive repeats/hides whichever element carries it.

> **Breaking change since 0.3.0**: the `<For>` / `<Show>` components have been removed —
> `<For each={x} key={k}>…</For>` → `<view each={x} key={k}>…</view>`,
> `<Show when={x} fallback={f}>…</Show>` → `<view show={x} fallback={f}>…</view>`.
> `gx/view` now only exports `Switch` / `Match`.

GUI examples (40+ runnable demos live in the repo's `testdata/` directory):

```bash
goxjs testdata/view_demo2.js      # list reuse / conditional keep-alive / multi-branch
goxjs testdata/model_demo.js      # two-way binding across all eight control types
goxjs testdata/router_demo.js     # route table / param matching / three-level guards / lazy loading
```

## Router & Screens (`gx/router` / `gx/screen`)

**New in 0.4.0**: no more hand-rolled signal-based page switching —

```jsx
import { h, render } from "gx/gfx";
import { createRouter, RouterView, RouterLink } from "gx/router";

function HomePage()        { return <text font={15}>home</text>; }
function ListPage()        { return <text font={15}>list</text>; }
function DetailPage(props) { return <text font={15}>{"detail id=" + props.param.id}</text>; }

const router = createRouter({
  routes: [
    { path: "/",           name: "home",   component: HomePage },
    { path: "/list",       name: "list",   component: ListPage, keepAlive: true },
    { path: "/detail/:id", name: "detail", component: DetailPage },
    { path: "*",           name: "nf",     component: HomePage },
  ],
  initial: "/",
});

render(
  <window title="app" width={480} height={360}>
    <column gap={8} padding={12}>
      <text font={12}>{() => "route: " + router.currentRoute().path}</text>
      <row gap={10}>
        <RouterLink to="/list"><text font={13}>List</text></RouterLink>
        <RouterLink to={{ name: "detail", params: { id: 7 } }}><text font={13}>Detail</text></RouterLink>
      </row>
      <RouterView />
    </column>
  </window>
);
```

Out of the box: `:param` / `:param?` / `*` matching (priority independent of declaration order), three levels of guards
(`beforeEach` / `beforeEnter` / `beforeRouteLeave` …, may return a Promise for async), `push` returning a
Promise (`{ok:false, reason}` expresses interception), `Alt+←` / `Alt+→` back/forward, `lazy(() => import(…))`
lazy loading, **two tiers of state preservation** (route records with `keepAlive: true` preserve the whole subtree;
`useRouteState()` stores values),
per-window independent navigation stacks and `sync()` sync groups (mirror / share / follow).

`gx/screen` provides display enumeration, the window's screen, and fold-posture reporting; **when half-folded,
`RouterView` automatically switches to two panes**
(the previous entry in the left pane's history, typically the list).

> Three deliberate differences from browser routing: **no URL and no history mode** (desktop apps have no address
> bar; "history" is an in-memory stack); `to` is always resolved as an **absolute path**; regex path constraints /
> `alias` / relative paths are not supported — validate params in `beforeEnter`. Windows / X11 have no fold-posture
> query API — the framework only provides the `reportPosture()` channel and never guesses; with no reporter the
> screen stays permanently unfolded.
>
> Full manual: [`docs/gui-router.md`](https://github.com/14752222/Gox/blob/main/docs/gui-router.md).

## Native capability layer (`gx/device` · `gx/app` · `gx/geo` · `gx/media` · `gx/permission` · `gx/viewport`)

**New in 0.5.0**: device info, battery, network, location, camera / photo album, permissions, safe area and soft
keyboard — six modules share one host contract with only three call shapes:

```js
import { deviceInfo, battery, isOnline, canIUse } from "gx/device";   // pull-type + report-type
import { getLocation, watchLocation } from "gx/geo";
import { takePhoto } from "gx/media";                                  // action-type

const info = deviceInfo();                    // synchronously available
console.log(info.platform, info.model, battery().level + "%", isOnline());
// battery() / isOnline() read snapshots; useBattery() returns a getter that refreshes when the host reports

if (canIUse("camera")) {                      // check capability up front; don't rely on catch
  const photo = await takePhoto({ count: 1 });      // failures reject with an errCode
}

const stop = watchLocation((loc) => console.log(loc.latitude, loc.longitude));
try {
  await getLocation({ highAccuracy: true });
} catch (e) {
  if (e.errCode === "permission-denied") console.warn(e.message);
}
```

- **Three shapes**: pull-type (sync read) · action-type (`await`, rejects on failure) · report-type (host pushes
  updates, `useXxx()` refreshes reactively).
- **No soft fallback for missing capabilities**: action-type calls always reject; there are eight error codes
  (`unsupported` / `permission-denied` / `cancelled` / `timeout` / `busy` / `unavailable` / `platform-error` /
  `invalid-arg`) — check with `canIUse()` first. Only purely report-type states get explicit defaults when nobody
  reports.
- **Desktop vs mobile**: desktop backends implement what they can (battery / network / brightness / screen
  keep-awake / opening system settings) and honestly report `unsupported` for the rest (no fake data); a mobile
  native shell implements the same `NativeHost` contract to plug in, without touching the core.
- **`gx/screen` vs `gx/viewport`**: the former is the device (displays / posture / hinge), the latter is the window
  (safe area / soft keyboard / split-screen form).

Demo: `goxjs testdata/native_demo.js`. Semantics and export tables:
[`docs/gui-guide.md` §9.6](https://github.com/14752222/Gox/blob/main/docs/gui-guide.md#96-原生能力层).

## 0.6.0 Release Notes

- **GUI on all platforms**: the macOS window backend (cocoa, driving AppKit via purego in pure Go) has landed —
  desktop GUI now works on Windows / macOS / Linux, and the npm prebuilt binaries include it out of the box.
- **Developer workflow**: `gox dev [entry.js]` hot reload (watches `.js` changes under `src/`, rebuilds the VM and
  reruns the entry); `gox.json` project config; `gox build <android|ios|windows|macos>` unified build entry (with
  permission injection and icon generation built in); `gox icon` / `gox sync` also work standalone.
- **Language**: async arrow functions (`async () => {}`, `this` stays lexical); `for (const [a, b] of pairs)`
  destructuring bindings; trailing commas in argument/parameter lists.
- **Fixes**: circular-import stack overflow, JSX default factory missed in module mode, nested destructuring
  parsing, solid notifications re-triggering every pass, and more.

Full docs and language examples: [GitHub repository](https://github.com/14752222/Gox).

## License

[MIT License](https://github.com/14752222/Gox/blob/main/LICENSE)

---

## 简体中文

**[English](#goxjsgoxjs) | 简体中文**

> Gox JavaScript 运行时 — 用 Go 从零实现：词法分析 → 语法分析 → 字节码编译 → 栈式虚拟机执行。单二进制、零外部依赖。

本包把 Gox 的各平台二进制分发到 npm，安装后直接获得 `goxjs` 命令。

### 安装

```bash
npm i -g @goxjs/goxjs
```

### 使用

**REPL：**

```bash
goxjs
```

```
Gox REPL (ES6 subset, no var)
Type :exit to quit, :help for help

> let x = 10
> [1, 2, 3].map(v => v * 2)
  [2, 4, 6]
```

**运行脚本：**

```bash
goxjs app.js
```

**免安装直接跑：**

```bash
npx goxjs app.js
```

支持 ES6+ 语言子集（`let`/`const`、class、`async`/`await`、解构、模板字符串、ES 模块），内置 `Array` / `Map` / `Set` / `Promise` / `BigInt` / `Temporal` 等标准对象，以及 `fs` / `http` / `fetch` / `path` / `process` 等 Node 风格宿主 API。

### 声明式界面（GUI）

`gx/gfx` 是纯 Go 自绘的窗口层，界面用 JSX 写。绑定、列表、条件都是**元素级指令**（写在元素上，不需要 import）：

```jsx
import { h, render, createSignal, Switch, Match } from "gox";

const [name, setName]   = createSignal("");
const [rows, setRows]   = createSignal([{ id: "a", title: "Alpha" }]);
const [open, setOpen]   = createSignal(true);
const [phase, setPhase] = createSignal("loading");

render(
  <window title="app" width={420} height={260}>
    <column gap={8} padding={12}>
      <input model={name} />                                                  {/* 双向绑定 */}
      <view each={rows} key="id">{(row) => <text>{row.title}</text>}</view>   {/* 列表: keyed 复用 */}
      <view show={open} fallback={<text>已隐藏</text>}>可见时显示这里</view>  {/* 条件: 保活显隐 */}
      <Switch fallback={<text>未知状态</text>}>                               {/* 多分支 */}
        <Match when={() => phase() === "loading"}><text>加载中…</text></Match>
      </Switch>
    </column>
  </window>
);
```

- `model` —— 受控组件的双向绑定（等价 Vue 的 `v-model`）：`input` / `textarea` / `slider` / `select` 读写 `value`，`checkbox` / `switch` 读写 `checked`，`radio` 选中时把 `value` 属性写进 model；也可传 `[get, set]` 二元组接自定义来源。
- `each` —— 列表循环：**同 key + 同行引用 + 同下标**的行原样复用（行内输入框、滚动位置、局部 signal 都留着），只重建真正变了的行；支持 `stable` / `fallback` / `key="id"`。
- `show` —— 条件显隐，**保活**：隐藏只把子树摘出布局流，再显示瞬间切回（不是销毁重建）。
- `view` —— 布局透明的容器（Fragment）；指令写在哪个元素上就重复/显隐哪个元素。

> **0.3.0 起是破坏性变更**：`<For>` / `<Show>` 组件已移除 ——
> `<For each={x} key={k}>…</For>` → `<view each={x} key={k}>…</view>`，
> `<Show when={x} fallback={f}>…</Show>` → `<view show={x} fallback={f}>…</view>`。
> `gx/view` 现在只导出 `Switch` / `Match`。

GUI 示例（仓库 `testdata/` 下有 40+ 个可直接运行的演示）：

```bash
goxjs testdata/view_demo2.js      # 列表复用 / 条件保活 / 多分支
goxjs testdata/model_demo.js      # 八类受控组件的双向绑定
goxjs testdata/router_demo.js     # 路由表 / 参数匹配 / 三级守卫 / 懒加载
```

### 路由与屏幕（`gx/router` / `gx/screen`）

**0.4.0 新增模块**：路由不用再自己手写 signal 切页了 ——

```jsx
import { h, render } from "gx/gfx";
import { createRouter, RouterView, RouterLink } from "gx/router";

function HomePage()        { return <text font={15}>home</text>; }
function ListPage()        { return <text font={15}>list</text>; }
function DetailPage(props) { return <text font={15}>{"detail id=" + props.param.id}</text>; }

const router = createRouter({
  routes: [
    { path: "/",           name: "home",   component: HomePage },
    { path: "/list",       name: "list",   component: ListPage, keepAlive: true },
    { path: "/detail/:id", name: "detail", component: DetailPage },
    { path: "*",           name: "nf",     component: HomePage },
  ],
  initial: "/",
});

render(
  <window title="app" width={480} height={360}>
    <column gap={8} padding={12}>
      <text font={12}>{() => "route: " + router.currentRoute().path}</text>
      <row gap={10}>
        <RouterLink to="/list"><text font={13}>列表</text></RouterLink>
        <RouterLink to={{ name: "detail", params: { id: 7 } }}><text font={13}>详情</text></RouterLink>
      </row>
      <RouterView />
    </column>
  </window>
);
```

开箱就有：`:param` / `:param?` / `*` 匹配（优先级不依赖声明序）、三级守卫
（`beforeEach` / `beforeEnter` / `beforeRouteLeave` …，可返回 Promise 做异步）、`push` 返回
Promise（`{ok:false, reason}` 表达被拦截）、`Alt+←` / `Alt+→` 后退前进、`lazy(() => import(…))`
懒加载、**两档状态保留**（路由记录 `keepAlive: true` 保住整棵子树；`useRouteState()` 存住值）、
多窗口独立导航栈与 `sync()` 同步组（mirror / share / follow）。

`gx/screen` 提供显示器枚举、窗口所在屏、折叠姿态上报；**半折时 `RouterView` 自动变双栏**
（左栏历史的上一条，通常是列表）。

> 三处与浏览器路由的刻意差异：**没有 URL，也没有 history 模式**（桌面应用没有地址栏，"历史"
> 是内存里的一个栈）；`to` 一律按**绝对路径**解析；正则路径约束 / `alias` / 相对路径不做，
> 参数校验写在 `beforeEnter`。Windows / X11 没有折叠姿态查询 API —— 框架只提供 `reportPosture()`
> 通道而不猜姿态，没人上报就恒为平展屏。
>
> 完整手册：[`docs/gui-router.md`](https://github.com/14752222/Gox/blob/main/docs/gui-router.md)。

### 原生能力层（`gx/device` · `gx/app` · `gx/geo` · `gx/media` · `gx/permission` · `gx/viewport`）

**0.5.0 新增模块**：设备信息、电量、网络、定位、相机 / 相册、权限、安全区与软键盘 —— 六个模块
共用一个宿主契约，调用形态只有三种：

```js
import { deviceInfo, battery, isOnline, canIUse } from "gx/device";   // 拉取型 + 上报型
import { getLocation, watchLocation } from "gx/geo";
import { takePhoto } from "gx/media";                                  // 动作型

const info = deviceInfo();                    // 同步可得
console.log(info.platform, info.model, battery().level + "%", isOnline());
// battery() / isOnline() 读快照；useBattery() 返回取值函数，宿主上报时自动刷新

if (canIUse("camera")) {                      // 事前判断能力，不要靠 catch 兜底
  const photo = await takePhoto({ count: 1 });      // 失败会 reject，带 errCode
}

const stop = watchLocation((loc) => console.log(loc.latitude, loc.longitude));
try {
  await getLocation({ highAccuracy: true });
} catch (e) {
  if (e.errCode === "permission-denied") console.warn(e.message);
}
```

- **三种形态**：拉取型（同步读）· 动作型（`await`，失败 reject）· 上报型（宿主主动告知，
  `useXxx()` 响应式刷新）。
- **缺能力不软降级**：动作型一律 reject，错误码共八个（`unsupported` / `permission-denied` /
  `cancelled` / `timeout` / `busy` / `unavailable` / `platform-error` / `invalid-arg`），
  先用 `canIUse()` 判断即可。只有纯上报型状态在没人上报时给明确的缺省值。
- **桌面 vs 移动**：桌面后端实现能实现的部分（电量 / 网络 / 亮度 / 屏幕常亮 / 打开系统设置页），
  给不出的一律诚实报 `unsupported`（不返回假数据）；移动端原生壳实现同一个 `NativeHost`
  契约即可接入，不必改内核。
- **`gx/screen` 与 `gx/viewport` 分工**：前者是设备（显示器 / 姿态 / 折痕），后者是窗口
  （安全区 / 软键盘 / 分屏形态）。

演示：`goxjs testdata/native_demo.js`。语义与导出表见
[`docs/gui-guide.md` §9.6](https://github.com/14752222/Gox/blob/main/docs/gui-guide.md#96-原生能力层)。

### 0.6.0 更新

- **GUI 全平台**：macOS 窗口后端（cocoa，purego 纯 Go 驱动 AppKit）落地 —— 桌面 GUI 现在 Windows / macOS / Linux 三平台可用，npm 预编译二进制开箱即含。
- **开发工作流**：`gox dev [入口.js]` 热更新（监听 `src/` 的 `.js` 变更，自动重建 VM 重跑入口）；`gox.json` 项目配置；`gox build <android|ios|windows|macos>` 统一构建入口（自动完成权限注入与图标生成）；`gox icon` / `gox sync` 也可独立使用。
- **语言**：async 箭头函数（`async () => {}`，`this` 保持词法）；`for (const [a, b] of pairs)` 解构绑定；实参/形参列表尾逗号。
- **修复**：循环导入栈溢出、JSX 缺省工厂在模块模式漏补、嵌套解构解析、solid 通知每轮重复触发等。

完整文档与语言示例见 [GitHub 仓库](https://github.com/14752222/Gox)。

### License

[MIT License](https://github.com/14752222/Gox/blob/main/LICENSE)
