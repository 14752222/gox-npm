# @goxjs/goxjs-mobile-android-arm64-v8a

Gox 运行时在 **Android arm64-v8a** 上的预编译库（`libgox.so`，Go `-buildmode=c-shared`）。
本包是 [`@goxjs/goxjs`](https://www.npmjs.com/package/@goxjs/goxjs) 的**平台子包（M11）**——
主包只发桌面二进制，移动端按「平台 × ABI」拆成独立子包，避免把几十 MB 的 `.so` 塞给每个桌面用户。

## 里面是什么

| 文件 | 说明 |
|---|---|
| `libgox.so` | Gox 引擎（lexer → parser → 字节码 VM → gfx）编成的共享库 |
| `manifest.json` | `goxVersion` + 本产物 `sha256`/`size`（取库前校验，防版本错配） |

## 怎么用

壳工程（Kotlin 宿主）不直接 `require` 本包，而是用取库脚本把它落进
`app/src/main/jniLibs/arm64-v8a/`：

```bash
npm i @goxjs/goxjs-mobile-android-arm64-v8a
bash scripts/fetch-mobile-libs.sh     # 校验 goxVersion + sha256 后拷贝
```

- 版本必须与引擎一致（壳工程 ↔ 引擎错配是运行时才崩，编译期拦不住）。
- 完整分发策略见 Gox 仓库的 `app/MOBILE-DISTRIBUTION.md`。
