# tauri-sidecar-desktop-packaging

把 **Tauri v2 桌面壳 + 后端 sidecar**（.NET / Node / Python 可执行文件）打包成可覆盖安装的
Windows 安装包，并对**安装之后**的程序做真实的界面级端到端验证。

这是一份给 AI 编码助手（WorkBuddy / Claude Code / CodeBuddy 等）加载的 Skill，
核心就 `SKILL.md` 一个文件。当然它也能当排错手册直接读。

## 它管两件事

| 链路 | 内容 |
|---|---|
| **一、打包** | 后端发布 → 最小文件集 → 桌面壳二进制 → NSIS 安装包 → 静默安装验证 → 免重装热替换 |
| **二、验证** | 给装好的程序开 WebView2 调试端口 → 取动态地址 → 用 CDP 做真实点击/断言/截图 → 清理测试数据 |

## 为什么需要它

这类应用有个结构性陷阱：桌面壳通常只是把窗口导航到 sidecar 的 HTTP 地址，
页面由后端的 `wwwroot` 提供。所以**只声明 `externalBin` 是不够的**——
`wwwroot` 与 `appsettings.json` 不会自动随行，装完必然白屏，
而且症状具有迷惑性：窗口全白，但后端进程其实是起来的。

文档里记的都是真机实测结论，不只是配置写法。几个典型：

- `bundle.resources` 是一份清单，sidecar 运行所需的一切都要列进去
- 改 `identifier` 不会改变安装身份（覆盖与否看 `productName`），但会让 WebView2
  用户数据目录改名，localStorage 里的登录票据全丢
- 卸载器只删自己清单里的文件，热替换造的 `.bak-*` 会全部留下
- 只截应用窗口、不要截整块桌面；驱动 GUI 与查后端数据要放在同一进程里跑

## 安装

Clone 到用户级 skills 目录：

```bash
git clone https://github.com/Durian2005/tauri-sidecar-desktop-packaging.git \
  ~/.workbuddy/skills/tauri-sidecar-desktop-packaging
```

其他助手请把目录放到各自的 skills 加载路径下，起作用的只有 `SKILL.md`。

## 适用环境

Windows + Tauri v2 + WebView2。示例以 .NET 后端为主，其他 sidecar 同理。
