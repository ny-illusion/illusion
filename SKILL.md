---
name: vscode-cpp-troubleshooting
description: Diagnose and fix VS Code C/C++ run/debug failures on Windows, including Code Runner running code in the read-only Output panel so cin/scanf cannot receive keyboard input, PowerShell parsing user input as commands, cd without /d failing to switch drives, and Code Runner's $dir variable trailing-backslash escaping breaking quoted paths. Also covers read-only "health check" requests asking whether VS Code can compile, run, and debug C/C++ at all (toolchain / extension / settings inspection plus a throwaway end-to-end compile + gdb breakpoint test, without modifying anything). This skill should be used when a user reports symptoms such as "code runs but cannot debug", "input then Enter does nothing", "表达式或语句中包含意外的标记", "ParserError / UnexpectedToken", "Invalid argument" from ld.exe, "No such file or directory" from cc1.exe, or "'70' 不是内部或外部命令" in VS Code with C/C++, or asks to check whether their VS Code environment can compile / run / debug normally.
agent_created: true
---

# VS Code C/C++ 运行调试故障排查（Windows）

## 适用范围

诊断 Windows 平台上 VS Code 运行/调试 C/C++ 程序时的故障。核心排查四类根因：

1. **输入通道错误** —— 程序跑在只读的「输出」面板，`cin`/`scanf` 无法接收键盘输入
2. **Shell 解析错误** —— PowerShell 把用户输入当命令解析
3. **跨盘符切换失败** —— `cd` 未加 `/d`，静默失败
4. **路径变量转义** —— Code Runner 的 `$dir` 末尾反斜杠转义引号，生成坏路径

## 症状 → 根因对照表

| 用户描述 / 报错原文 | 根因 | 处理 |
|---|---|---|
| 只能运行不能调试、找不到调试按钮 | 用的是 Code Runner 而非原生调试 | 见「故障 A」 |
| 输入数据后回车无反应、卡住 | 程序跑在只读输出面板 | 见「故障 A」 |
| `表达式或语句中包含意外的标记"x.xx"`<br>`ParserError` / `FullyQualifiedErrorId: UnexpectedToken` | 终端是 PowerShell，输入未进程序 | 见「故障 B」 |
| `cc1.exe: fatal error: xxx.c: No such file or directory` | 工作目录与文件不同盘；`cd` 缺 `/d` | 见「故障 C」 |
| `ld.exe: cannot open output file d:"xxx.exe : Invalid argument` | `$dir` 末尾反斜杠转义引号 | 见「故障 D」 |
| `'70' 不是内部或外部命令` | 上一步编译已失败，无 exe 可运行 | 见「故障 D」 |
| 断点是灰色空心圈 | 调试器与源码路径未绑定 | 用「打开文件夹」方式打开项目 |

## 排查流程

### 第 0 步：环境体检（先做，避免误判）

```bash
gcc --version; g++ --version; gdb --version
ls ~/.vscode/extensions   # 确认 cpptools 与 code-runner 是否安装
```

读取用户设置确认当前配置：

```
C:\Users\<用户名>\AppData\Roaming\Code\User\settings.json
```

### 第 1 步：定位真实源文件（不要假设文件不存在）

`cc1.exe: No such file or directory` 常常是路径问题而非文件缺失。先搜索确认：

```bash
find /c /d -maxdepth 4 \( -name "*.c" -o -name "*.cpp" \) \
  -not -path "*/\[Rr\]ecycle.bin/*" -not -path "*/Windows/*" \
  -not -path "*/Program Files*" -not -path "*mingw64*" 2>/dev/null | head -20
```

> **必须排除回收站**：`$Recycle.Bin` 里堆着历史删除的 `.c` 文件（形如 `$Ixxxxxx.c` / `$Rxxxxxx.c`），
> 不排除会瞬间刷满结果、淹没真实源文件，还会误判「用户有 20 个源文件」。
> 同理排除 `Windows`、`Program Files`、`mingw64`（工具链自带头文件）。
>
> 若已知项目根目录（如 `D:\CppProjects`），**直接搜那个目录**，不要全盘扫。

### 第 2 步：按症状对照表定位根因

### 第 3 步：实测验证（必做）

不要只改配置就交付。亲自编译运行一次，确认链路通：

```bash
cd <项目目录> && gcc main.c -g -Wall -o main.exe && printf "输入内容\n" | ./main.exe
```

## 故障 A：Code Runner 在只读输出面板运行

**判定**：程序能跑但没有交互；或用户以为"VS Code 不能调试"。

**修复**：设置 `code-runner.runInTerminal` 为 `true`，让程序跑在终端而非输出面板。

同时建议提醒：Code Runner 的右键 `Run Code` 永远不带调试，调试必须用 `F5`。

## 故障 B：PowerShell 解析用户输入

**判定**：报错含 `ParserError`、`CategoryInfo`、`FullyQualifiedErrorId`、`意外的标记`。
这些格式**只可能由 PowerShell 本体抛出**，C++ 程序绝不会产生。

**修复**：将默认终端切换为 Command Prompt：

```json
"terminal.integrated.defaultProfile.windows": "Command Prompt"
```

**注意**：`codeManager.js` 第 349-359 行 `changeExecutorFromCmdToPs()` 会在检测到
PowerShell 时把 `&&` 改写为 `; if ($?) { ... }`，这也是命令看起来怪异的原因。

## 故障 C：跨盘符 cd 失败

**判定**：终端提示符所在盘与文件所在盘不同，且命令是 `cd "d:\"` 这种形式。

**根因**：cmd 的 `cd` 默认只在当前盘内切换，跨盘必须用 `cd /d`。

**修复**：命令中统一使用 `cd /d <路径>`。

## 故障 D：Code Runner 路径变量转义（最隐蔽，必须查源码确认）

**判定**：命令行出现引号嵌套如 `""d:\""`，或 `ld.exe: Invalid argument`。

**根因**（已阅读 `formulahendry.code-runner` 源码确认）：

- 文件 `out/src/codeManager.js` 第 303-305 行：

  ```javascript
  quoteFileName(fileName) {
      return '"' + fileName + '"';   // 仅包引号，不处理末尾反斜杠
  }
  ```

- 第 338 行：`$dir` → `quoteFileName(codeFileDir)`，而 `codeFileDir` 实际值为 `d:\`
- `"d:\"` 中末尾 `\` 转义后引号 → 路径损坏为 `d:"`
- 第 336 行提供了专用变量 `$dirWithoutTrailingSlash`，**会先去末尾反斜杠再包引号**

**修复**：`executorMap` 统一使用 `$dirWithoutTrailingSlash` 替代 `$dir`：

```json
"code-runner.executorMap": {
    "c": "cd /d $dirWithoutTrailingSlash && gcc $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe",
    "cpp": "cd /d $dirWithoutTrailingSlash && g++ $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe"
}
```

要点：
- `$dirWithoutTrailingSlash` —— 规避末尾反斜杠转义
- `cd /d` —— 跨盘符切换
- 改用 `$fileName` / `$fileNameWithoutExt` 相对名 —— 已先 `cd` 过去，路径越短越不易出错

**教训**：诊断 Windows 下 Code Runner 路径问题，**必须查源码确认变量替换行为**，
不能凭猜测配置。源码位于
`~/.vscode/extensions/formulahendry.code-runner-<版本>/out/src/codeManager.js`。

## 标准交付配置

完整可用的 `settings.json`（用户级）：

```json
{
    "code-runner.runInTerminal": true,
    "code-runner.saveFileBeforeRun": true,
    "code-runner.clearPreviousOutput": true,
    "code-runner.executorMap": {
        "c": "cd /d $dirWithoutTrailingSlash && gcc $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe",
        "cpp": "cd /d $dirWithoutTrailingSlash && g++ $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe"
    },
    "terminal.integrated.defaultProfile.windows": "Command Prompt"
}
```

项目级 `.vscode/` 模板见 `assets/` 目录，直接复制到项目根目录使用。

> `assets/launch.json` 与 `assets/tasks.json` **已预填本机真实路径**：
> - `miDebuggerPath` 指向本机 WinGet 安装的 gdb 绝对路径（换机器需改）
> - `tasks.json` 同时含 `gcc`（默认）与 `g++` 两条构建任务，C / C++ 都能编译
>
> 因此第 171 节「结构性问题 C」在实际交付时，直接用 `assets/tasks.json` 覆盖即可修好。

## 环境体检（只读诊断）流程

当用户要求「看看我的 VS Code 能不能用 / 能不能编译 / 能不能调试」时，先做**只读**体检，
**不改任何文件**，出报告，等用户下指令再动手。

### 体检五步

1. **本体**：`code --version`，记录安装路径
2. **工具链**：`gcc --version; g++ --version; gdb --version`
3. **扩展**：`ls ~/.vscode/extensions`
4. **配置**：读用户级 `settings.json` + 各项目的 `.vscode/launch.json`、`tasks.json`
5. **实测（必做，不能省）**：在**系统临时目录**建测试程序，编译 → 运行 → gdb 断点，
   全程不碰用户任何文件；测完立刻删除

```bash
TMPD=$(mktemp -d); cd "$TMPD"
gcc test.c -g -Wall -o test.exe && printf "3 4\n" | ./test.exe     # 编译+运行+模拟输入
env -u PYTHONPATH -u PYTHONHOME gdb --batch -ex "break main" -ex "run" -ex "print a" -ex quit ./test.exe
rm -rf "$TMPD"
```

> 只改配置不实测 = 未验证。必须亲自跑通「编译 → 运行 → 断点命中 → 变量可读」才算通过。

### 体检中容易漏掉的 3 个结构性问题

**A. 源码被放进 `.vscode/` 目录**
`.vscode` 只应放配置。若发现 `*.c` / `*.exe` 躺在里面（多见于「模板」文件夹），
把源码移到项目根目录并规范命名为 `main.c`，编译产物直接删（可再生）。

**B. `miDebuggerPath` 用相对名 `"gdb.exe"`**
它依赖 PATH。Windows 下 MinGW 经 WinGet 安装时，PATH 条目形如
`...\WinGet\Packages\<包名>_<源>_<哈希>\mingw64\bin`。
**注意该哈希目录本身就是 PATH 条目的一部分**，所以「PATH 被其他安装程序清空」与
「WinGet 升级换了哈希目录」这两种场景下，相对名和绝对路径同样脆弱；
改绝对路径是为了防前者，且失败时报错更明确。JSON 中反斜杠必须双写：

```json
"miDebuggerPath": "C:\\Users\\<用户>\\AppData\\Local\\Microsoft\\WinGet\\Packages\\<包目录>\\mingw64\\bin\\gdb.exe"
```

**C. `tasks.json` 只有 gcc 任务**
模板只配 gcc 时，用户写 `.cpp` 会因 `gcc` 链接不到 C++ 标准库而失败。
体检要提醒：需要 C++ 就补一条 `g++` 构建任务。

### 读持久 PATH 的正确姿势

`reg query` 可能被安全策略拉黑（本机已拉黑 `reg.exe`）。改用 PowerShell：

```powershell
[Environment]::GetEnvironmentVariable('Path','User')    -split ';'
[Environment]::GetEnvironmentVariable('Path','Machine') -split ';'
```

若 PowerShell 工具 stdout 不回显，用 `Out-File` 落盘后再 Read。

### 陷阱：`gdb --version` 报 `_ctypes` / `sitecustomize` 错误

在**被宿主注入了 `PYTHONPATH` 的 shell** 里（如智能体自身运行环境），
`gdb --version` 会打印
`Error in sitecustomize; ... No module named '_ctypes'`。
**这是宿主环境产物，与用户的 VS Code 无关**，不要据此判定 gdb 损坏。
用 `env -u PYTHONPATH -u PYTHONHOME gdb ...` 复测即可证实 gdb 本身正常，
报告里应明确告知用户「无需处理」。

## 文件保存位置规范（用户常问）

当用户问"文件该保存在哪里"时，要点是：**位置不是根本问题，"打开方式"才是**
（双击单个文件会导致 `.vscode/` 不加载）。但仍应引导到规范结构，从源头规避路径类故障。

**推荐结构：一个题目 = 一个文件夹 = 一个项目**

```
D:\CppProjects\           ← 所有作业的根目录
├── template\             ← 模板文件夹，复制它来新建题目
│   ├── main.c
│   └── .vscode\{launch.json, tasks.json}
├── bmi\                  ← 每题一个文件夹
│   └── main.c
└── score\
    └── main.c
```

**三条铁律**：
1. 路径全用英文，不出现中文、空格、特殊符号
2. 用 `main.c` 等有意义的名字，不用 `Untitled-1.c`（该名字意味着未保存过）
3. 永远用「文件 → 打开文件夹」打开项目，不可双击单个文件

**位置评价**：
- ✅ 推荐 `D:\CppProjects\`（路径短、全英文）
- ⚠️ 桌面不推荐（可能含中文、易误删）
- ❌ 避免盘符根目录（如 `D:\main.c`，必触发反斜杠转义）
- ❌ 避免 OneDrive 同步目录（同步冲突可致编译失败）

**自检清单**（交付时附给用户）：
- [ ] 用「打开文件夹」打开了整个项目文件夹
- [ ] 左侧资源管理器能看到 `.vscode` 目录
- [ ] 源文件命名为 `main.c`，非 `Untitled-1.c`
- [ ] 路径无中文、无空格
- [ ] 文件不在盘符根目录

## 交付时必须包含的操作指引

改完配置后，用户仍需手工操作，必须明确告知：

1. **彻底重启 VS Code**（关闭所有窗口），不是 Reload Window，否则设置不生效
2. **用「文件 → 打开文件夹」打开项目**，不要直接打开单个源文件，否则 `.vscode/` 不加载
3. **建议将未命名文件规范命名**（如 `Untitled-1.c` → `main.c`）并移出盘符根目录

## 关键判断技巧（教给用户）

- **有闪烁光标 = 程序在等待输入；回到提示符 = 程序已结束**
- 报错含 `ParserError`/`CategoryInfo` → 一定是 PowerShell 抛的，输入没进程序
- `scanf("%lf %lf", ...)` 要求同一行空格分隔输入两个数
- `scanf` 后接 `getline`/`fgets` 会读到残留换行符，需 `cin.ignore()` 或 `getchar()` 吃掉

## 输入法陷阱

Windows 中文输入法在终端中可能吃掉回车或字符。输入前提醒切换英文输入法。

## 详细诊断报告模板

交付给用户的完整诊断报告模板见 `references/diagnosis-template.md`。
文件组织规范的完整说明模板见 `references/file-location-guide.md`。
