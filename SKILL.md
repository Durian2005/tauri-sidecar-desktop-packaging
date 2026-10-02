---
name: tauri-sidecar-desktop-packaging
agent_created: true
description: "把 Tauri v2 桌面壳 + 后端 sidecar（.NET / Node / Python 可执行文件）打包成可覆盖安装的 Windows 安装包，并对安装后的程序做真实的界面级端到端验证。触发词：打包桌面安装包、重新打包、生成安装包、NSIS、Tauri 打包、sidecar、覆盖安装、桌面端验证、WebView2 调试、package desktop installer、Tauri sidecar, NSIS, verify installed desktop app。"
---
# Tauri v2 + Sidecar 桌面交付：打包与实测

面向"Vue/前端产物 + 本地后端进程"这类桌面应用（Tauri v2 壳把后端当 sidecar 子进程拉起）。
两类任务：**① 打包成安装包**、**② 对安装后的程序做真实界面的端到端验证**。

---

## 一、打包链路

### 1. 发布后端（以 .NET 为例）

```bash
dotnet publish -c Release -r win-x64 --self-contained false \
  -p:PublishSingleFile=true -p:IncludeNativeLibrariesForSelfExtract=true \
  -p:DebugType=none -o <staging>
```

要点：

- `--self-contained false` + `PublishSingleFile=true` → 单个 exe（体积小），但**目标机需装对应运行时**。
  要在无运行时机器上跑，改用 `--self-contained true`（体积大涨）。
- `PublishSingleFile` 必须指定 RID，否则报错。
- **静态资源目录（`wwwroot` 之类）不会被打进单文件**，会作为普通文件夹留在发布目录，需一并随包分发。

### 1b. 用 `tauri build` 打包时：必须有 `bundle.resources`（否则安装包装完是白屏）

如果不用手写 NSIS、直接跑 `tauri build`，**只声明 `externalBin` 是不够的**。
`externalBin` 只把 sidecar 后端 exe 复制过去，`wwwroot` 与 `appsettings.json`
**不会自动随行**，装完必然缺文件。

典型症状：窗口先全白、后全黑或空白，但后端进程确实起来了——因为桌面壳
通常会把窗口导航到 sidecar 的 HTTP 地址，由后端的 `wwwroot` 提供页面：

```rust
window.navigate(url.parse()?)?;   // url = http://127.0.0.1:<动态端口>
```

所以 `wwwroot` 缺失 = 页面 404 = 白屏；`appsettings.json` 缺失 = 后端连不上数据库。

**正确配置**（`src-tauri/tauri.conf.json`）：

```json
"bundle": {
  "externalBin": ["binaries/backend"],
  "resources": {
    "../Frontend/dist": "wwwroot",
    "../Backend/appsettings.json": "appsettings.json",
    "../Backend/appsettings.Local.json": "appsettings.Local.json"
  }
}
```

**`resources` 是一份清单：sidecar 运行所需的一切都要列进去。**
最常漏的两类：

| 漏掉的东西 | 后果 | 为什么难发现 |
|---|---|---|
| 静态资源目录 | 页面 404 / 白屏 | 后端进程活着，界面却空白 |
| 本地覆盖配置（`appsettings.Local.json` 等） | 功能**静默降级**，界面一切正常 | 源码目录有该文件，只有安装版才缺失 |

第二类特别阴险：典型的"配置不全就自动降级"设计（如邮件从 SMTP 退到控制台），
表现是"功能提示成功了但对方收不到"。排查手法：**去安装目录找降级产物**
（如 `email-codes.log`），有新增记录就说明走了兜底分支。

给配置文件里的凭证单独留一条检查：文件存在 ≠ 配置已填。
写个 `verify_install.py` 之类的脚本，把"必填文件清单 + 关键字段非空"都验一遍。

关键细节：

- **`resources` 的源目录要指向与本次构建同批产出的目录。**
  若前端有两套产物（普通模式 → 后端 `wwwroot`，desktop 模式 → `Frontend/dist`），
  而 `tauri build` 只跑 desktop 那一套，那么指向后端 `wwwroot` 会**打包进上一次的旧前端**，
  表现为"代码改了但安装包行为没变"。看到产物里的 JS 文件名哈希与 `dist/` 对不上就是这个原因。
- 映射是 `源: 目标`，目标是**相对安装目录**的路径。
- 源是 git 跟踪的目录更稳（避免 `publish/` 这类被 gitignore 的目录在干净检出时不存在）。
- 打完后**务必核对**：安装目录里的 JS 文件名 == `dist/assets/` 里的文件名。

排查手法：`strings <主exe> | grep single-process` 查不到是正常的
（`additionalBrowserArgs` 是运行时配置，不在主 exe 字符串里），
要查的是**安装目录里有没有 wwwroot 目录**。

### 1c. 两套前端产物：**两份都构建**，再把"等价"当断言去校验

只要前端同时要服务「浏览器直连后端」和「打包进桌面端」，就存在**同一份源码、两个输出目录**：

| 链路 | 产物目录 | 构建命令 |
|---|---|---|
| 后端静态托管（浏览器访问，同源 API） | `Backend/wwwroot` | `vite build`（默认模式） |
| 桌面端（`bundle.resources` / 热替换） | `Frontend/dist` | `vite build --mode desktop` |

**别用「构建一次再复制」来省事**，除非你确认两套产物确实等价。判断依据只有两条：
源码里**有没有用 `import.meta.env` / `process.env`**，以及**环境判定是不是运行期**做的
（例如 `'__TAURI_INTERNALS__' in window`）。这两条一旦变化，两套产物就会**合法地分叉**，
复制会悄悄给其中一条链路塞错产物。

更稳的做法：**两套各按自己的 mode 构建**，然后把"等价"当断言去校验：

```bash
npm --prefix Frontend run build:desktop   # -> Frontend/dist
npm --prefix Frontend run build           # -> Backend/wwwroot
```

校验 = 两个目录的**文件名集合 + 每个文件的 md5** 对一遍，不一致直接非零退出。要点：

- **挂进每次构建**（统一走 `build_all.py` 之类），而不是写进文档靠人记得手动跑
  —— 实际发生过 `wwwroot` 落后 `dist` **15 天**、浏览器直连后端看到旧界面的事故。
- 另留一个**只校验不构建**的入口，改完产物后好快速复查。
- **务必做反例验证**：手动加一个多出来的文件、再改一个文件的内容，
  确认两种偏差都能被检出。**从不报警的检查器等于没测过。**
- 报告输出用 `OK`/`x`/`!`，别用 `✓`/`✗` —— 非 GBK 字符在管道里会 `UnicodeEncodeError`。

配套的必知坑：**`dotnet build` 不把 `wwwroot` 复制到 `bin/`**（只有 `dotnet publish` 才复制）。
所以直接跑 `bin/Release/net8.0/Backend.exe` 会**页面 404** —— 要验证"后端托管的界面是不是新的"，
必须以**后端项目目录为 cwd** 启动（ContentRoot 才指向项目里的 `wwwroot`），或跑 `publish/` 那一份。
想刷新 `publish/wwwroot` 时**只拷 `wwwroot`，别重跑 publish**：重跑会换掉 exe，
破坏「publish 产物 md5 == sidecar 二进制 md5」这条在用的不变量。

验证手法：起一次后端，断言 `GET /` 200 且**引用新产物的 hash 文件名**、
**旧产物的 hash 名返回 404**、静态资源的字节数与本地文件一致。
看文件时间戳不算数 —— 要 ASP.NET Core 自己把文件吐出来才算。

### 2. 整理成最小文件集

发布目录里通常有这些**不需要**的东西，删掉可让安装目录干净：

| 文件 | 为什么可以删 |
|---|---|
| `web.config` | IIS 部署才用 |
| `appsettings.Development.json` | 生产不读 |
| `*.staticwebassets.endpoints.json` | 静态资源 manifest，生产环境不加载，静态文件直接按目录服务 |
| 发布目录里嵌套的 `publish/` | 源码目录残留的旧发布结果被当成内容又复制了一遍 |

**踩坑**：如果后端工程目录下存在历史 `publish/` 文件夹，它会被内容通配符带进发布输出，
产生 `输出/publish/publish/...` 递归。先删源目录里的 `publish/` 再发布。

让后端 exe 名称与 Tauri sidecar 名称一致（例如源码产出 `App.exe`，sidecar 期望 `app.exe`）。

### 3. 桌面壳二进制

`src-tauri/target/release/<app>.exe`。**Rust 源码没改就不必重建**——重建一次往往数分钟，
且需要 `-j 2` 之类的内存限制。用 `ls` 比对 exe 与 `src/*.rs` 的时间戳即可判断。

### 4. NSIS 打包

```powershell
& "<nsis>\makensis.exe" "<path>\installer.nsi"
```

**必须用 PowerShell 调用**：Git Bash 直接调用 makensis 常常静默失败（退出码 0 但不产出文件）。
生成后用 `ls` 核对产物的时间戳与体积，别只看命令退出码。

### 5. 安装器里两条必备的保护逻辑

```nsis
; ① 用户本机敏感配置：已存在则保留，覆盖安装不冲掉（例如 SMTP 授权码）
IfFileExists "$INSTDIR\appsettings.Local.json" localcfg_keep
  File "${SRCDIR}\appsettings.Local.json"
localcfg_keep:

; ② 前端产物文件名带哈希：先清空旧目录，否则历史文件越堆越多
RMDir /r "$INSTDIR\wwwroot"
File /r "${SRCDIR}\wwwroot"
```

卸载段记得删掉运行期生成的日志文件，否则残留会让 `RMDir "$INSTDIR"` 静默失败。

### 6. 静默安装验证

```bash
<installer>.exe /S          # 静默安装，默认装到 %LOCALAPPDATA%\Programs\<App>
```

安装前先 `taskkill /F /IM <app>.exe` 与 sidecar 进程名，避免文件占用导致覆盖失败。

**四个必踩的坑：**

1. **静默安装会"检测到已安装同版本"而跳过替换**。
   `/S` 退出码 0、耗时很短、但 exe 时间戳没变 —— 就是被跳过了。
   想验证新构建的包，**必须先卸载再装**。

   **但卸载器必须用 Python 同进程调用** —— 从 Bash 工具直接跑会**静默失败**：

   ```python
   # 从 shell 直接跑 `uninstall.exe /S`：调用方 shell 退出会**回收它派生的子进程**，
   # 症状是"rc=0、但一个文件都没删、注册表项也还在"，极易误判成"卸载成功"。
   # 实测与权限无关（沙箱读写 %LOCALAPPDATA% 都正常），换 /S _?=<dir> 等参数也是白费功夫。
   # 另：不要带 _?=<路径> 参数，带了对不上退出码 2。
   subprocess.Popen([r"<install>\uninstall.exe", "/S"]).wait()
   ```

   **卸载器删不干净 `deploy_hotswap.py` 造的 `.bak-*`**：它只删自己清单里的文件，
   最后 `RMDir "$INSTDIR"` 因目录非空静默失败。实测残留 13 项 / 84MB，重装后与新文件
   混在同一目录 —— 要么手动清，要么别指望卸载器。

2. **新版安装路径可能和旧版不一样**。
   例如旧版在 `%LOCALAPPDATA%\Programs\<App>`，新版 NSIS 装到 `%LOCALAPPDATA%\<App>`。
   猜目录会一直看"没装成功"。**查注册表最可靠**：

   ```powershell
   $p=@('HKCU:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*',
        'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\*')
   Get-ItemProperty $p -EA SilentlyContinue |
     Where-Object { $_.DisplayName -like '*<App>*' } |
     Select-Object DisplayName, DisplayVersion, InstallLocation
   ```

   用 `$_.InstallLocation` 去 `ls`，比遍历 `AppData` 快得多。

3. **后台静默安装要等够时间**：`nohup installer.exe /S &` 之后至少 sleep 25-30 秒
   （sidecar 有 20MB+ 要落盘），且**必须核对 `find <dir> -type f` 列出的文件数**，
   只有 sidecar 而没有主 exe，说明安装被中断了（常见于调用方 shell 提前退出）。

判断"新包是否真的生效"，看**四点**（第 4 条 2026-09-25 补）：主 exe 大小/时间戳、
`wwwroot` 里 JS 的哈希文件名、`appsettings.json` 是否存在、**sidecar 的 md5 是否等于
`src-tauri/binaries/` 里刚同步的那份** —— 最容易漏的一条：忘了跑 `sync:sidecar` 时，
主程序是新的、sidecar 还是旧版本，而前三项全都正常。

### 6b. 改了 `identifier` 之后，到底什么会变

常见动机是"去掉 `identifier` 里的本机用户名"。实测结论：

**改 `identifier` 不会改变安装身份** —— Tauri NSIS 的卸载注册表键名
（`HKCU\...\Uninstall\<productName>`）与 `InstallLocation` 都来自 **`productName`**。
新旧版本是同一个安装身份，装新版**覆盖**旧版，不会共存。

真正跟着 `identifier` 走的只有两处：

| 项 | 变化 |
|---|---|
| 注册表 `Publisher` | 由 identifier 派生（`com.acme.myapp` → `myapp`） |
| WebView2 用户数据目录 | `%LOCALAPPDATA%\<identifier>` ⇒ **localStorage 里的登录票据会丢**，需重新登录 |

⇒ 别为了"避免两版共存"去卸载重装；要卸载的真实理由只有坑 1（同版本被跳过）。
但**改了 identifier 后仍要重打包** —— 否则安装包生成的数据目录名与源码不一致。

### 6c. 清理卸载残留（`.bak` 备份 + 孤儿目录 + 旧 WebView2 数据）

卸载器只会删它自己清单里的文件，`deploy_hotswap.py` 造的 `.bak-*` **会全部留下**
（实测 13 项 / 84MB）。清理时**走回收站**（可恢复），Windows 上最稳的做法是
Python 调 `SHFileOperationW` —— 这正是资源管理器"删除"走的同一条路：

```python
FO_DELETE, FOF_ALLOWUNDO, FOF_NOCONFIRMATION, FOF_NOERRORUI, FOF_SILENT = 3, 0x40, 0x10, 0x400, 0x04

class SHFILEOPSTRUCTW(ctypes.Structure):
    _fields_ = [("hwnd", wintypes.HWND), ("wFunc", ctypes.c_uint),
                ("pFrom", ctypes.c_wchar_p), ("pTo", ctypes.c_wchar_p),
                ("fFlags", ctypes.c_ushort), ("fAnyOperationsAborted", wintypes.BOOL),
                ("hNameMappings", ctypes.c_void_p), ("lpszProgressTitle", ctypes.c_wchar_p)]

# pFrom 必须是**双 NUL 结尾**的多字符串；即使只传一个路径也要补两个 NUL
pfrom = "\0".join(paths) + "\0\0"
```

三条实测纪律：

- **一次只传一个顶级路径。** 把 5 个路径塞进一个调用 → 返回 2（`FILE_NOT_FOUND`），
  部分项目只删掉一小半（实测 WebView2 目录 78MB 只掉了几个 MB）。
  拆成单路径逐个调用后返回 0、一次干净。
- **返回码不可信。** 成功删掉 10 项的批次也返回过 2。
  判断依据一律是"**目标是否还存在**"，不是返回值。
- 删 WebView2 这类目录前先 `os.chmod(f, 0o666)` 清属性，并把目录**改短名**
  （如 `_wv2old`）给内部 MAX_PATH 拼接留余量；失败要能改回原名，不留半拉子状态。

⚠️ 有些机器的 PowerShell 安全策略会拦 `Add-Type`（"compiles and loads .NET code at runtime"），
而 `Microsoft.VisualBasic.FileIO.FileSystem` 也未默认加载 —— 别在那条路上耗时间。
送回收站**不等于释放空间**：要真正腾出磁盘得清空回收站，收尾时提醒用户一句。

### 6d. 应用自己的**持久数据**绝不能落在安装目录里

Tauri NSIS 的安装目录由 `productName` 决定，默认就是 `%LOCALAPPDATA%\<productName>`。
这个位置**看起来**很像该放应用数据的地方（`%LOCALAPPDATA%` 本来就是放用户数据的地方），
于是很容易顺手把密钥 / 证书 / 本地库写进安装目录里 —— **这是错的**：

- 卸载器删完清单文件后还会尝试 `RMDir "$INSTDIR"`。目录空得下来时，**它会连同你的数据一起消失**。
  实测一次干净的卸载（目录里没有 `.bak` 之类的额外文件）之后，安装目录被**完全清空**。
- 更常见的中间状态是：目录里只剩你那份数据、其余文件都被删了 —— 它变成"空楼里孤零零一个文件"，
  此后任何"清理安装残留"的操作（这类脚本/工具很常见）都会顺手带走它。
- 覆盖安装时，你的数据与安装器要写的文件混在同一层，边界也不清晰。

⇒ 数据目录**取同级但不同名**的路径：app 装 `%LOCALAPPDATA%\MyApp`，数据放 `%LOCALAPPDATA%\MyAppData\`。
这样安装、卸载、清残留都碰不到它。实测做法：把密钥文件从 `%LOCALAPPDATA%\<productName>\key.dat`
搬到 `%LOCALAPPDATA%\<productName>Data\key.dat` 后，**卸载 + 覆盖安装全程 md5 未变**。

若旧版本已经发布过，就得带**一次性搬迁**（老位置有、新位置无则搬过去，搬完删老文件），
并且搬迁入口要有**守卫**：目标不是默认位置、用户也没显式指定老位置时不要搬 ——
否则"指向临时目录"的自动化测试会把真实数据复制进测试环境，
让"首次生成"这类断言**静默失效**（测试反而更绿）。

### 6e. 覆盖安装的完整验证循环（全部放在一次进程内）

「卸载 → 安装 → 启动 → 查产物」四步都别指望 shell：

```python
Popen([uninstall_exe, "/S"], cwd=install_dir).wait()      # 见坑 1：必须同进程
wait_until(lambda: not os.path.exists(install_dir + r"\sidecar.exe"), 90)
Popen([setup_exe, "/S"]).wait()
wait_until(lambda: os.path.exists(install_dir + r"\sidecar.exe"), 120)
```

- **别用固定 sleep 判断完成**：Tauri 的 NSIS 安装器/卸载器会先把自身复制到 `%TEMP%` 再重启执行，
  因此**原进程立刻返回 0**，真正的工作还在后面。轮询"产物出现/消失"才可靠。
- 卸载后目录可能**整个消失**，下一次 `listdir` 会抛 `FileNotFoundError` ⇒ 用 `try/except` 包住。
- 验证"程序真的活着"看三件事：桌面壳进程在不在、`tasklist` 里有没有 sidecar、
  再用 `netstat -ano` 按 sidecar 的 PID 找到它监听的 `127.0.0.1:<随机端口>`，
  对该端口发一次 HTTP GET（200 且响应含前端挂载点）。
  **后端起得来本身就是强证据** —— 对 fail-closed 的程序（如 pepper 缺失即退出 1），
  只要它在监听，就说明关键配置都已加载成功。
- 启动 GUI 要带 `creationflags=DETACHED_PROCESS | CREATE_NEW_PROCESS_GROUP`，
  否则脚本退出会把它一起带走。

### 7. 免重装热替换（改代码后要立刻在真机验证时的快路径）

只改后端 / 前端时**不必重打安装包**：把新 exe 覆盖到安装目录的 sidecar 位置、
把新前端覆盖到安装目录的 `wwwroot`，功能立刻生效。打一次 `tauri build` 要几分钟，
热替换只要十几秒，迭代期应该默认走这条路。

```python
# 顺序不能变，每一步都有坑：
1. taskkill /F /IM <桌面壳>.exe  与  /IM <sidecar>.exe
2. time.sleep(2)                 # 进程退出后句柄释放有几秒延迟
3. 旧文件先改名备份（<name>.bak-YYYYMMDD-HHMM），不要直接覆盖
4. shutil.copy2 带重试（WinError 32 会偶发），前端整目录用 copytree 替换
5. 校验 md5：源产物 == 安装目录
```

要点：

- **`tasklist /FI "IMAGENAME eq ..."` 不能省**。若进程还在，复制会失败或写出半截文件。
- **备份名加时间戳后缀**，别用固定名 —— 固定名会被下一次热替换覆盖掉，
  等你真需要回滚时发现备份已经是坏的那一版。
- **前端必须整目录替换**。产物文件名带哈希，只覆盖 `index.html` 会留下旧 assets，
  也可能堆积历史文件；`wwwroot` 整目录用 `copytree` 重建最干净。
- **md5 校验要覆盖到 `index.html` 与 `assets/*`**，不能只看 exe。
- 前端由 sidecar 从磁盘按请求读取，但 **WebView 会缓存**，所以还是要把桌面壳一起重启。
- 依赖的"构建 → 同步 → 替换"链路建议脚本化（如 `build_all.py` + `deploy_hotswap.py`），
  把过滤编译输出的逻辑放在 Python 里 —— 见文末「工具链环境」。

---

## 二、对安装后的程序做界面级验证

难点：桌面端后端端口是**动态分配**的，且 GUI 程序没有控制台，日志看不见。

### 1. 打开 WebView2 调试端口

```bash
WEBVIEW2_ADDITIONAL_BROWSER_ARGUMENTS="--remote-debugging-port=9333 --remote-allow-origins=*" ./app.exe
```

### 2. 取到动态地址

```bash
curl -s http://127.0.0.1:9333/json/list
```

返回里 `type == "page"` 的那一项，其 `url` 就是当前页面地址（即动态后端端口），
`webSocketDebuggerUrl` 用于 CDP 驱动。

### 3. 用 CDP 做真实点击

`Runtime.evaluate` 注入 JS 操作 DOM / 触发 Vue 的 `input` 事件即可，无需 Playwright。
把脚本参数化，同一份脚本既能驱动浏览器（9222）也能驱动桌面（9333）：

- `E2E_CDP`：调试端口地址
- `E2E_APP`：不填则**自动从调试目标读取页面地址**（桌面端端口动态，必须自动取）
- `E2E_LOG`：验证码等日志位置

**反模式**：按文本模糊匹配按钮会点错元素（例如页面顶部有个同名的 tab）。
表单提交一律用 `form button[type=submit]` 这类结构选择器。

### 4. 只截应用窗口，别截整块桌面

用户桌面上可能有私密内容。用 `EnumWindows` 找到窗口 + `PrintWindow(hwnd, hdc, 2)`
（`PW_RENDERFULLCONTENT`）——**即使窗口被遮挡也能正确渲染**，且只含目标窗口。
不要用全屏截图（`ImageGrab.grab()`）。

### 5. 清理测试数据

自动化验证会往数据库写测试账号。收尾时按前缀（如 `ui######`）识别并删除，
保留管理员账号，别把测试数据留给用户演示。

**用独立测试库，且脚本必须自己收尾清库。** 端到端脚本一律 `drop_database(测试库)` 起步、
跑完再 `drop_database(测试库)` 收尾，全程别碰真实库。踩过的坑：脚本只在**开头**清库、
结尾不管，于是每跑一次就留下一个壳，最后库里攒出 `xxxRoleTest` 这种"3 个账号 + 28 条审计"
的残留，还得回头人工清。推荐策略：**成功即清、失败保留现场（便于排查）、加 `--keep` 强制保留**。

```python
# 开头：保证从全新库起步（后端会在此播种初始管理员）
if DB in _cli.list_database_names():
    _cli.drop_database(DB)
# 结尾：
if FAILED or KEEP:          # 有失败项就留现场
    print("测试库保留：%s" % DB)
else:
    _cli.drop_database(DB)
```

### 6. 判断"画面到底渲染出来没有"

`PrintWindow` 抓到图不代表渲染成功——白屏/黑屏也是一张图。加两个数值判据：

- **颜色数**：`len(img.getcolors(maxcolors=w*h) or [])`，正常界面通常 > 500，纯白/纯黑只有个位数；
- **非黑像素占比**：灰度直方图前几档之和，正常界面 > 30%。

经验值：修好渲染管线后颜色数会从 ~330 跳到 ~5700。

两个实现细节：

- 用 `getcolors()` 而不是 `len(set(img.getdata()))` —— 后者对 1942x1286 要建一个
  250 万元素的集合，慢，且 `getdata()` 在 Pillow 14 起被弃用（会刷 DeprecationWarning）。
- **截图前先判 `IsIconic(hwnd)`**。窗口最小化时 `GetWindowRect` 返回的是标题栏那条窄条
  （实测形如 237x39），抓出来的图毫无内容，却极易被误读成"界面渲染失败"。
  正确做法：`ShowWindow(SW_RESTORE)` + `BringWindowToTop` + 等 1 秒；
  若高度仍 < 200px 就再等一轮。触发场景：`real_click` 里为解除前台锁定会做
  最小化→还原，还原没生效就会留下这个状态。

**从截图反查控件坐标**：要点击的按钮若不便估算坐标，可以在上一张截图里按颜色聚类定位
（例如危险按钮是红色：`r>140 and g<110 and b<110 and r-g>60`），按行聚类取出按钮的
中心点，再把这个像素坐标直接当点击坐标用 —— `PrintWindow` 抓的是整个窗口矩形，
与"窗口相对坐标"是 1:1，不需要任何缩放换算。

### 7. 驱动 GUI + 查后端数据要放在同一进程里跑

本机沙箱环境里，**调用方 shell 退出会回收它派生的所有子进程**
（`nohup ... &` 也拦不住）。所以：

- "先起后端，再另开一条命令打接口" → 必然 `ConnectionRefused`；
- 正确做法是写一个脚本，在**同一个 Python 进程里** `Popen` 起服务 →
  轮询等端口可用 → 调接口/驱动界面 → `terminate()` 收尾。

同理，桌面端端到端验证（启动 exe → 找窗口 → 截图 → 点击）也必须在**一次进程内**完成。

另外 `urllib` 在本机对 `127.0.0.1` 可能被代理拦截返回 502，
**用裸 `socket.create_connection` 发 HTTP/1.1 请求最稳**（记得带 `Content-Length`，
并处理 `Transfer-Encoding: chunked` 响应）。

### 8. sidecar 走 HTTPS 之后，界面级验证要改的三处

打包好的程序里 sidecar 一旦启用 TLS（`--urls https://127.0.0.1:<随机端口>`），
原先那套"拿到端口就发 http 请求"的验证会**以误导人的方式失败**，务必改这三处：

1. **取到端口后必须沿用页面 URL 的 scheme**，不能写死 `http://`。
   用 `http` 去打 `https` 端口只会得到连接重置 / `RemoteDisconnected`，
   症状极像"后端没起来"，很容易白查半天。页面地址从 CDP 的 `/json/list` 里取，
   `url.split('//')[0]` 就是它当前真实的 scheme（sidecar 可能被配置降级成明文）。
2. **"不带信任锚必须失败"这条断言会变成假阳性。**
   自签 CA 一旦写进系统根存储（生产路径就是这么做的），
   `ssl.create_default_context()` 会把这张 CA 也读进来 ⇒ 不带 `cacert` 照样连得上。
   正确的两条对照必须**分开**：
   - **B** 信任锚为空的上下文（或只加载 `server.crt`）→ 必须失败（证明真在验链，不是装饰）；
   - **E** 默认上下文（= Schannel / WebView2 真正走的判定路径）→ 必须成功（证明界面能渲染）。

   混在一起写，就得到一条"怎么都通过"的断言。另：Windows 自带 curl 是 **Schannel 后端**，
   `--cacert` 不足以让它信任自签 CA ⇒ 这套断言请用 Python(OpenSSL) 实现。
3. **徽记 / 状态类断言要先轮询等它落定，再采样。**
   界面上的"已连接 / 加密"来自一次异步请求，页面刚挂载时还没回来 ——
   直接断言会得到"有时过有时不过"的假阴性，而错误结论会被当成功能缺陷去查。
   实测：修好后徽记 0ms 就在；但没加等待时**第一次跑就误报过一次**。

> 顺带一条界面设计上的坑：这类"链路状态"徽记若只挂在**业务请求成功后的回调**里，
> 就会等到登录成功才显示 —— 而登录本身就走在这条链路上，
> 最需要提示的时刻反而空着。应当在启动探活成功后单独取一次（该接口要设计成**匿名可取**）。

---

## 三、GUI 程序的一个隐形陷阱

桌面壳在 Windows 上通常带 `#![cfg_attr(not(debug_assertions), windows_subsystem = "windows")]`，
**没有控制台**。若后端把关键信息（例如登录验证码）只 `print` 到 stdout，
在安装版里用户永远看不到。

**规则**：凡是"离线/降级模式下用户必须看到"的信息，一律**同时落盘到文件**
（用 `AppContext.BaseDirectory` 定位到 exe 所在目录），不能只依赖 stdout。

---

## 四、工具链环境（Windows 本机沙箱）

本机 Bash 工具的 PATH 时常是断的：`ls` / `grep` / `head` / `dirname` / `cat` 全部
`command not found`，连重设 `PATH` 指向 PortableGit 也救不回来。
**唯一稳定的执行通道**是：

```bash
MSYS_NO_PATHCONV=1 C:/Users/<user>/python.exe <脚本或 -c "...">
```

再在 Python 里用绝对路径 + `subprocess` 干所有事（调 dotnet / node / taskkill / 过滤输出）。
`MSYS_NO_PATHCONV=1` 必须有，否则路径里的 `/` 会被 MSYS 改写。

固定资源位置（避免每次重新探测）：

| 工具 | 路径 |
|---|---|
| dotnet | `C:\Program Files\dotnet\dotnet.exe` |
| node / npm | `C:\Users\<user>\.workbuddy\binaries\node\versions\<ver>\node.exe` / `npm.CMD` |
| 系统 Python（装过 pymongo 等第三方库） | `C:\Users\<user>\python.exe` |
| Rust 工具链（cargo / rustc） | `C:\Users\<user>\.cargo\bin\cargo.exe`（可能**没装 rustup**，bin 下直接是 cargo.exe / rustc.exe） |

**`tauri build` 报 `failed to run 'cargo metadata' ... program not found`** ⇒
就是 cargo 不在 PATH 里。两条容易白折腾的点：

- 若装的是**独立工具链**（`.cargo/bin` 下直接躺着 cargo.exe，没有 `.rustup` 目录），
  它**不会自动进 PATH**，得自己加。
- Git Bash 里**别用 `C:\Users\...` 这种反斜杠写法**拼 PATH —— MSYS 解析不了，
  `command not found` 照旧。要用 `/c/Users/<user>/.cargo/bin`：

```bash
export PATH="/c/Users/<user>/.cargo/bin:$PATH" && npm run build
```

两条容易踩的纪律：

- **不要把输出管到 `head` / `grep`** —— 它们在 Bash 工具里不存在，整条命令会 127 退出
  且看不到任何输出，看起来像"脚本没跑"。过滤逻辑写进 Python。
- **`subprocess` 抓长输出不要用 `PIPE`**。后端每请求打一行日志，几百个请求会撑满
  64KB 管道缓冲区，子进程阻塞在写 stdout，后续请求全部超时 —— 表现为
  "接口突然不通"，实则管道堵死。**重定向到文件或 `DEVNULL`**。

`dotnet publish` 的输出目录若落在工程目录内，小心出现 `输出/publish/publish/` 递归 ——
先删源目录里的历史 `publish/`，且别把 `-o` 指到已有的 `publish` 子目录上。
