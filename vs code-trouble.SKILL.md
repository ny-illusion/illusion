---
name: vscode-cpp-troubleshooting
description: Diagnose and fix VS Code C/C++ run/debug failures on Windows, including Code Runner running code in the read-only Output panel so cin/scanf cannot receive keyboard input, PowerShell parsing user input as commands, cd without /d failing to switch drives, Code Runner's $dir variable trailing-backslash escaping breaking quoted paths, and Chinese text printed by printf/cout showing up as mojibake such as 「璇疯緭鍏ヤ竴涓暣鏁帮細」 because a UTF-8 source file is rendered through the console's GBK code page (936). Also covers read-only "health check" requests asking whether VS Code can compile, run, and debug C/C++ at all (toolchain / extension / settings inspection plus a throwaway end-to-end compile + gdb breakpoint test, without modifying anything). This skill should be used when a user reports symptoms such as "code runs but cannot debug", "input then Enter does nothing", "表达式或语句中包含意外的标记", "ParserError / UnexpectedToken", "Invalid argument" from ld.exe, "No such file or directory" from cc1.exe, or "'70' 不是内部或外部命令", or "printf/cout 输出的中文变成乱码 / 璇疯緭鍏ヤ竴涓暣鏁帮細 / 终端中文全是问号或方块" in VS Code with C/C++, or asks to check whether their VS Code environment can compile / run / debug normally.
agent_created: true
---

# VS Code C/C++ 运行调试故障排查（Windows）

## 适用范围

诊断 Windows 平台上 VS Code 运行/调试 C/C++ 程序时的故障。核心排查五类根因：

1. **输入通道错误** —— 程序跑在只读的「输出」面板，`cin`/`scanf` 无法接收键盘输入
2. **Shell 解析错误** —— PowerShell 把用户输入当命令解析
3. **跨盘符切换失败** —— `cd` 未加 `/d`，静默失败
4. **路径变量转义** —— Code Runner 的 `$dir` 末尾反斜杠转义引号，生成坏路径
5. **编码不对称** —— 源文件 UTF-8、控制台码页 GBK(936)，中文输出变乱码

## 症状 → 根因对照表

| 用户描述 / 报错原文 | 根因 | 处理 |
|---|---|---|
| 只能运行不能调试、找不到调试按钮 | 用的是 Code Runner 而非原生调试 | 见「故障 A」 |
| 输入数据后回车无反应、卡住 | 程序跑在只读输出面板 | 见「故障 A」 |
| `表达式或语句中包含意外的标记"x.xx"`<br>`ParserError` / `FullyQualifiedErrorId: UnexpectedToken` | 终端是 PowerShell，输入未进程序 | 见「故障 B」 |
| `cc1.exe: fatal error: xxx.c: No such file or directory` | 工作目录与文件不同盘；`cd` 缺 `/d` | 见「故障 C」 |
| `ld.exe: cannot open output file d:"xxx.exe : Invalid argument` | `$dir` 末尾反斜杠转义引号 | 见「故障 D」 |
| `'70' 不是内部或外部命令` | 上一步编译已失败，无 exe 可运行 | 见「故障 D」 |
| printf/cout 的中文变成 `璇疯緭鍏ヤ竴涓暣鏁帮細` 这类怪字（码位对不上） | 源文件 UTF-8，控制台码页 GBK(936) | 见「故障 E」 |
| 中文显示成 `????` 或方块 | 同上，或终端字体不含中文字形 | 见「故障 E」 |
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

## 故障 E：中文输出乱码（编码不对称，最容易被误判成"代码写错了"）

**判定**：程序能编译、能运行、逻辑正常，唯独 `printf` / `cout` 里的中文变成怪字，
形如 `璇疯緭鍏ヤ竴涓暣鏁帮細`（本案例原文是「请输入一个整数：」）。

**关键辨别点**：乱码是**有规律的、成组的汉字**，不是 `?` 或方块。
`??` / 方块才是字体或宽窄字符问题；成组怪字一定是指纹级证据 —— **字节没错，是解释方式错了**。

### 根因

UTF-8 与 GBK 的**字节 → 字符**映射不一致：

| 环节 | 编码 | 说明 |
|---|---|---|
| 源文件 `.c` | UTF-8 | VS Code 默认保存为 UTF-8（无 BOM） |
| 编译后的字符串常量 | 原样照搬源文件字节 | 编译器不改编码，仍是 UTF-8 字节 |
| 程序 stdout | 输出上述 UTF-8 字节 | 程序本身完全正确 |
| cmd 控制台 | 码页 **936 / GBK** | 拿 GBK 去拆 UTF-8 字节 → 乱码 |

所以「代码没写错、编译器没出错、程序也没出错」，坏在终端如何解释这串字节。

### 第 1 步：三条命令锁定根因（不要靠猜）

```bash
# ① 看源文件真实编码：中文字节应以 e8/e4/e5... 开头（UTF-8 三字节序列）
python -c "print(repr(open('no.c','rb').read()))"
# 期望看到 b'\xe8\xaf\xb7\xe8\xbe\x93\xe5\x85\xa5...' = 「请输入」

# ② 直接看程序吐出的原始字节（绕开控制台渲染）
cmd //c "no.exe" | python -c "import sys;d=sys.stdin.buffer.read();print(repr(d))"
# 若字节同样是 e8 af b7 ... → 程序输出没问题，确认是终端渲染环节的锅

# ③ 反查乱码：把看到的乱码字符按 GBK 编码，应能还原出源文件的 UTF-8 字节
python -c "print(repr('璇疯緭鍏ヤ竴涓暣鏁帮細'.encode('gbk')))"
# 得到 b'\xe8\xaf\xb7\xe8\xbe\x93...' ← 与 ① 完全吻合，闭环
```

第 ③ 步能否对上，是**判定本故障的铁证**。注意：手工敲的乱码字符常有一两个字抄错
（本例就出现过），只要大部分字节吻合即可确认，不必纠结单字节差异。

### 第 2 步：修复

**方案 1（首选，全局生效，不动源码）** —— 让终端用 UTF-8 码页。
在 Code Runner 的 `executorMap` 每条命令**最前面**加 `chcp 65001 >nul &&`：

```json
"code-runner.executorMap": {
    "c": "chcp 65001 >nul && cd /d $dirWithoutTrailingSlash && gcc $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe",
    "cpp": "chcp 65001 >nul && cd /d $dirWithoutTrailingSlash && g++ $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe"
}
```

优点：一次配置修好**该用户所有**含中文的源文件（学员文件夹里往往躺着十几个）。
`chcp` 在同一个终端会话内持续有效，后续运行也不会退回 GBK。

**方案 2（改源码，适合只交单个文件）** —— 让程序自己切码页。
在 `main` 开头加一行（`system` 需 `#include <stdlib.h>`）：

```c
system("chcp 65001 >nul");
```

**方案 3（改文件编码，适合课程/评委环境固定为 GBK）** —— 把源文件另存为
GB2312/GBK：VS Code 右下角点编码 → `Save with Encoding` → 选 `GB2312`。
好处是**任何 GBK 终端都能正确显示**，不依赖配置；代价是换到 UTF-8 环境又要转回来。
⚠️ 注意 `files.autoSave` 开启时，个别版本会把文件重新保存回 UTF-8，改用方案 1 更稳。

### 第 3 步：实测验证（必做）

```bash
cd <项目目录> && cmd //c "chcp 65001 >nul && gcc no.c -g -Wall -o no.exe && no.exe" \
  | python -c "import sys;d=sys.stdin.buffer.read();print(d.decode('utf-8','replace'))"
```

看到正确中文才算通过。**注意**：若只用管道读字节来判断，无论码页对错字节都一样，
测不出来 —— 必须按上面这样显式 `chcp 65001` 后在真实 cmd 里跑，或让用户肉眼看终端。

### 坑：`F5` 原生调试走的是另一条路

`chcp` 只加在 Code Runner 命令里，**不影响 `cpptools` 的 F5 调试**。
若用户改用 F5，调试控制台的乱码要另修：`launch.json` 里设
`"externalConsole": true` 并在 `preLaunchTask` / 程序入口补 `system("chcp 65001>nul")`，
或改用 `"console": "integratedTerminal"`。交付时**主动问一句用户用不用 F5**，
避免"Code Runner 修好了、F5 还是乱码"的返工。

**教训**：乱码类问题永远先分清「**字节错**」还是「**解释错**」。
用 `repr()` 打原始字节是 10 秒定位法，比反复改代码、改编码试错高效得多。

## 标准交付配置

完整可用的 `settings.json`（用户级）：

```json
{
    "code-runner.runInTerminal": true,
    "code-runner.saveFileBeforeRun": true,
    "code-runner.clearPreviousOutput": true,
    "code-runner.executorMap": {
        "c": "chcp 65001 >nul && cd /d $dirWithoutTrailingSlash && gcc $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe",
        "cpp": "chcp 65001 >nul && cd /d $dirWithoutTrailingSlash && g++ $fileName -g -Wall -o $fileNameWithoutExt.exe && $fileNameWithoutExt.exe"
    },
    "terminal.integrated.defaultProfile.windows": "Command Prompt"
}
```

> 每条命令开头的 `chcp 65001 >nul` 是**故障 E 的防线**：中文输出环境一步到位，
> 免去逐个文件改编码。国内教学场景（源文件多为 UTF-8 + 学员写中文提示）建议**默认带上**。

项目级 `.vscode/` 模板见 `assets/` 目录，直接复制到项目根目录使用。

> `assets/launch.json` 与 `assets/tasks.json` **已预填本机真实路径**：
> - `miDebuggerPath` 指向本机 WinGet 安装的 gdb 绝对路径（换机器需改）
> - `tasks.json` 同时含 `gcc`（默认）与 `g++` 两条构建任务，C / C++ 都能编译
>
> 因此下文「体检中容易漏掉的 3 个结构性问题 → C」在实际交付时，直接用 `assets/tasks.json` 覆盖即可修好。

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
- **输出乱码先分清：成组怪字 = 编码/码页问题（故障 E）；`?`/方块 = 字体问题**
- 报错含 `ParserError`/`CategoryInfo` → 一定是 PowerShell 抛的，输入没进程序
- `scanf("%lf %lf", ...)` 要求同一行空格分隔输入两个数
- `scanf` 后接 `getline`/`fgets` 会读到残留换行符，需 `cin.ignore()` 或 `getchar()` 吃掉

## 输入法陷阱

Windows 中文输入法在终端中可能吃掉回车或字符。输入前提醒切换英文输入法。

## 详细诊断报告模板

交付给用户的完整诊断报告模板见 `references/diagnosis-template.md`。
文件组织规范的完整说明模板见 `references/file-location-guide.md`。
