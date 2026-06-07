# Maven (mvn) + Maven Daemon (mvnd) 本地支援說明

> 個人 fork 維護筆記。這個分支 `feat/mvn-mvnd-1089` 把上游 PR
> [rtk-ai/rtk#1089](https://github.com/rtk-ai/rtk/pull/1089) 吸收進來,自行維護、之後再慢慢往上游合。

## 這是什麼

上游 rtk 對 Maven 原本只有一個很淺的 TOML filter(只認 `compile/package/clean/install`、漏掉
`mvn test`),對 **Maven Daemon (`mvnd`)** 則完全沒有支援。

PR #1089 用一個 Rust 模組(`src/cmds/jvm/`)取代舊的 TOML filter,提供 **`rtk mvn` 與 `rtk mvnd`** 完整支援:

| 指令 | 大致節省 | 說明 |
|------|---------|------|
| `mvn test` / `mvnd test` | ~99% | 讀 `target/surefire-reports/*.xml`,只留結構化的失敗摘要 |
| `mvn verify` / `mvnd verify` | ~95% | 含 `failsafe-reports/` 整合測試 |
| `mvn compile` | ~85% | 砍 `[INFO]`/下載雜訊,保留錯誤 |
| `mvn checkstyle:check` | ~90% | 只留違規 |
| `mvn dependency:tree` | ~70% | 座標壓縮 |
| `mvn clean` | ~95% | |
| `mvn package` / `install` | ~0%(passthrough) | rtk 不過濾,原樣輸出 |

`mvnd` 與 `mvn` 走同一套過濾/XML 強化邏輯(binary-agnostic),`mvn_command(MvnBinary::Mvnd)` 會呼叫真正的
`mvnd` daemon,不是硬寫 `mvn`。`mvnw` / `./mvnw` wrapper 也會自動偵測。

## 狀態(本地已驗證)

- 合併方式:`develop` ← `merge 69aee99`(吸收 #1089)+ `fix ebb7080`(修掉深度檢查發現的阻斷)
- 品質閘:`cargo fmt --all && cargo clippy --all-targets && cargo test --all` **全綠**
- 深度檢查修掉的重點:clippy 阻斷、surefire 報告非決定性排序、`mvn test -q` 誤報 no tests、
  discover per-goal 節省率失效(capture group)、`package`/`install` 虛報節省率
- 版本:`0.42.2`

## 本地使用

> ⚠️ 共通前提:**binary 不能跨平台複製**(Linux 是 ELF、Mac 是 Mach-O)。每台機器都要當場
> `cargo build --release`。產出在 `target/release/rtk`。

### Linux(curl/install.sh 裝的,通常在 `~/.local/bin/rtk`)

```fish
cd <你 clone 的位置>
cargo build --release

# 方案 A:並行試用,不動全域(推薦先這樣)
./target/release/rtk mvnd test
abbr --add rtkdev (pwd)/target/release/rtk   # 之後 rtkdev mvn test

# 方案 B:覆蓋全域 active binary(用 which 動態取得路徑,別寫死)
set -l active (which rtk)
cp $active "$active.bak"                       # 先備份
cp (pwd)/target/release/rtk $active
rtk --version                                  # 應為 0.42.2
# 回復:cp "$active.bak" $active
```

### macOS(Homebrew 裝的,在 `/opt/homebrew/bin/rtk`,Intel 為 `/usr/local/bin/rtk`)

`/opt/homebrew/bin/rtk` 是 **symlink** 指向 Cellar,**不要 `cp` 直接蓋**(會被 `brew upgrade` 還原、
`brew doctor` 抱怨)。

```fish
cd <你 clone 的位置>
cargo build --release        # 一定要在 Mac 上 build

# 方案 A:並行試用(最乾淨,不碰 brew)
./target/release/rtk mvnd test
abbr --add rtkdev (pwd)/target/release/rtk

# 方案 B(brew 正確做法):unlink brew 版 + cargo install,可逆
brew unlink rtk
cargo install --path .       # 裝到 ~/.cargo/bin/rtk
which rtk                     # 確認指到 ~/.cargo/bin/rtk(否則把 ~/.cargo/bin 排到 PATH 前面)
rtk --version
# 回復原狀:brew link rtk
```

> macOS 的 Gatekeeper 不會擋你自己 build 的 binary(quarantine 只針對「下載」的檔),免 codesign。

### 讓 Claude Code 裡打 `mvn`/`mvnd` 自動路由(改 hook)

把 Claude Code 的 Bash hook 指到新 binary 的絕對路徑即可(不受 PATH 影響)。編輯
`~/.claude/settings.json` 的 `PreToolUse` → `Bash` hook:

```jsonc
"command": "/絕對路徑/到/rtk/target/release/rtk hook claude"
```

或在新 binary 已上 PATH 後執行 `rtk init -g --auto-patch`(會自動改成 `rtk hook claude` 形式)。

> 注意:目前(原本環境)hook 指向的 `~/.claude/hooks/rtk-rewrite.sh` 可能不存在,等於沒生效——
> 改成上面的絕對路徑即可修好。

## 驗證

```fish
set -l BIN (pwd)/target/release/rtk     # 或 install 後的 rtk
$BIN --version                          # rtk 0.42.2
$BIN hook check "mvn test"              # → rtk mvn test
$BIN hook check "mvnd test"             # → rtk mvnd test
$BIN hook check "./mvnw verify"         # → rtk mvn verify
$BIN gain                               # 有輸出 = 正版 Rust Token Killer(非 Rust Type Kit)
```

## 本地安裝實戰紀錄(2026-06-07 Linux 實測 + MacBook Pro 換裝指南)

> 把「實際從零裝起來」的完整流程記下來,換機(MacBook Pro)照著做即可。本次在 Linux(linuxbrew)
> 全程驗證通過;每段的「macOS」子項是換 MacBook Pro 時要改的地方。

### 1. 改裝 rtk(讓全域 `rtk` 換成本分支 binary)

原本 `rtk` 是 Homebrew symlink(`$(brew --prefix)/bin/rtk` → Cellar),**不能直接 `cp` 蓋**。
正確做法:cargo 裝到 `~/.cargo/bin`,再 `brew unlink` 讓 PATH 解析到它(可逆)。

```fish
cd ~/research/rtk                   # 你 clone 的位置
cargo install --path . --force      # 裝到 ~/.cargo/bin/rtk(release + LTO,約數分鐘)
brew unlink rtk                     # 移掉 brew symlink,PATH 改解析到 ~/.cargo/bin/rtk
which rtk                           # 應為 ~/.cargo/bin/rtk
rtk --version                       # rtk 0.42.2(本分支版本)
# 還原:brew link rtk
```

- **macOS**:步驟相同。Apple Silicon 的 brew 在 `/opt/homebrew`;若 `which rtk` 仍指到 brew,
  確認 `~/.cargo/bin` 在 PATH 比 `/opt/homebrew/bin` 前面,或就靠 `brew unlink rtk` 解決。
  Gatekeeper 不擋自己 build 的 binary,免 codesign。
- Claude Code hook 用 `rtk hook claude`,brew unlink 後新 shell 會自動解析到新 binary,
  hook 路由(`mvnd test` → `rtk mvnd test`)立即生效,**不必改 settings.json**。

### 2. 安裝 Maven

```fish
brew install maven        # 本次裝到 3.9.16
mvn -version
```

### 3. 安裝 mvnd(Maven Daemon)

- **macOS(推薦,最省事)**:官方 tap 有 formula
  ```fish
  brew install mvndaemon/mvnd/mvnd
  mvnd --version
  ```
- **Linux(brew 沒有 mvnd formula)**:從 Apache GitHub release 抓 tarball
  ```fish
  set V 1.0.6
  curl -sSL -o /tmp/mvnd.tgz \
    https://github.com/apache/maven-mvnd/releases/download/$V/maven-mvnd-$V-linux-amd64.tar.gz
  mkdir -p ~/.local/share/mvnd; tar -xzf /tmp/mvnd.tgz -C ~/.local/share/mvnd
  ln -sf ~/.local/share/mvnd/maven-mvnd-$V-linux-amd64/bin/mvnd ~/.local/bin/mvnd
  ```
  > ⚠️ **本次 Linux 踩雷**:Apache mvnd 1.0.6 的 daemon 端在這台 sandbox 噴
  > `Unable to load mvndnative native library`(`org.mvndaemon.mvnd.nativ.CLibrary` 靜態初始化失敗),
  > 任何 goal 都起不來。這是 **mvnd 自帶 native lib 在此環境載不起來,與 rtk 無關**;rtk 仍正確呼叫
  > mvnd、套同一過濾、用時間閘跳過舊報告(回報 `no tests run`,不誤報)。macOS 用 brew tap 裝的 mvnd
  > 是當平台 build,不會有這問題;Linux 若遇到,改用 `sdk install mvnd`(SDKMAN)或檢查 glibc 相容性。
- **JAVA_HOME**(mvnd/mvn 都要找得到 JDK):
  - Linux:`set -x JAVA_HOME (dirname (dirname (readlink -f (command -v java))))`
  - macOS:`set -x JAVA_HOME (/usr/libexec/java_home -v 21)`

### 4. 驗證 rtk × mvn/mvnd 正確運作

```fish
rtk hook check "mvnd clean test"   # → rtk mvnd clean test
rtk hook check "./mvnw verify"     # → rtk mvn verify
```

拿一個 Spring Boot 專案(本次:Spring Boot 4 / Java 21,含通過 + 故意失敗測試)實跑各 goal。
各 goal 路由與行為(2026-06-07 實測,以 `rtk mvn` 量;mvnd 共用同一 binary-agnostic 過濾):

| goal | 行為 | 穩態節省 |
|------|------|---------|
| `clean` | 一行摘要 + 刪除路徑 | ~98% |
| `compile` | 砍 INFO/下載雜訊,保留 BUILD SUCCESS / 錯誤 | ~96% |
| `test` | 讀 surefire XML,只留結構化失敗摘要 | 94–99% |
| `verify` | 同 test + failsafe 整合測試 | ~99% |
| `dependency:tree` | 座標壓縮 | ~99% |
| `clean test` / `clean install` | 多 goal 合併過濾(可過濾段過濾、passthrough 段原樣) | 視內容 |
| `package` / `install` | **passthrough**(原樣輸出,不過濾) | ~0% |

失敗路徑實測(`rtk mvn test`,8 測試 2 失敗):原始 23,337 字元 → rtk 521 字元(**−97.8%**),
正確還原 1 個 `[error] IllegalStateException` + 1 個 `AssertionFailedError`(`expected:<200> but was:<404>`)。
`rtk gain --history` 記錄:`rtk mvn test` −94%、成功路徑 `-Dtest=...` −99%。

> 量測注意:比較 `rtk proxy mvn <goal>`(raw)vs `rtk mvn <goal>`(filtered)時 **先各跑一次暖機**,
> 否則第一次的一次性下載(如 spring-boot repackage 拉 loader)會灌水節省率——package 冷測假性顯示 ~92%,
> 暖機後僅 ~4% ≈ passthrough。

## macOS(Apple Silicon)實測紀錄(2026-06-07,MacBook Pro)

> 上節 playbook 的 macOS 子項本次全程照做、**全部成立**。這也是第一次拿「真的 mvnd」實跑
> (Linux 那次 daemon 起不來,只能驗證 rtk 的呼叫路徑)。
>
> 環境:Apple Silicon,brew tap 裝的 mvnd 1.0.5(darwin-aarch64 native client,內含 Maven 3.9.14)、
> brew Maven 3.9.16、JDK 21(Zulu)。受測對象:一個單模組 Spring Boot / Java 21 專案,
> 565 個單元測試(無 failsafe 整合測試)。

### 安裝驗證(playbook 成立 + 一個新坑)

- `cargo install --path . --force` + `brew unlink rtk` 照做即成功(1m05s,`which rtk` → `~/.cargo/bin/rtk`)。
- **新坑:這台機器 `~/.cargo/bin` 原本不在 PATH**(rustup 有裝、shell 設定沒帶到),`brew unlink`
  之後 `rtk` 會直接消失。fish 解法:`fish_add_path -p ~/.cargo/bin`(永久、排最前)。連帶
  `cargo`/`rustc` 本來也只能用完整路徑 `~/.cargo/bin/cargo` 叫。
- brew tap 的 mvnd 是當平台 native build,**沒有** Linux tarball 的 native-lib 載入問題——上節預測成立。
- hook 路由換 binary 後立即生效(`rtk hook check "mvnd clean test"` → `rtk mvnd clean test`),
  不必動 settings.json。

### goal 矩陣(mvnd 實跑,非以 mvn 代測)

| goal | 過濾後輸出 | 實測節省(`rtk gain`) |
|------|-----------|---------------------|
| `clean` | 1 行(deleted 路徑 + 時間) | −92% |
| `compile` | BUILD SUCCESS + javac unchecked 警告(合理保留) | −76% |
| `test` | **1 行**:`mvn test: 565 passed (8.228 s (Wall Clock))` | **−100%**(75.9K → 1 行) |
| `verify` | **1 行** | **−100%**(76.0K) |
| `dependency:tree` | 39 行壓縮樹(`(23 transitive)` 摺疊正常) | 冷 −77% / 暖 −86% |
| `clean test`(多 goal) | 9 行(警告 + 測試摘要 + BUILD SUCCESS) | **−100%**(76.2K) |
| 失敗路徑(見問題 1) | BUILD FAILURE + tee log 路徑 | −73% |

exit code 傳遞(BUILD FAILURE 時 rtk 回 1)與 tee 兜底(`~/Library/Application Support/rtk/tee/`)皆正確。

### 新發現問題(依嚴重度排序,皆附複現方式)

> **更新(2026-06-07):以下 4 項皆已修復**(TDD + 真實 fixture,品質閘 `fmt && clippy && test`
> 全綠、token 節省率不退步),詳見下方「已修(field-test gaps)」小節。本節保留為當時的實測記錄
> 與複現方式。

**1. mojo 層級 BUILD FAILURE 的 `[ERROR]` 原因整段被砍(最值得修)**

當失敗不是「測試失敗」(沒有 surefire XML 可 enrich)而是 plugin/mojo 錯誤時,過濾後輸出只剩
`[INFO] BUILD FAILURE` + 時間,**完全看不到失敗原因**;原因只存在 tee log。exit code 仍正確回 1,
但 LLM/使用者必須多開一次 tee 檔才知道為什麼掛。

複現(任何用 surefire 的專案都行):

```fish
rtk mvnd test -Dtest=NoSuchTest         # 過濾後:只有 BUILD FAILURE,無原因
rtk proxy mvnd test -Dtest=NoSuchTest   # raw 對照:有完整 [ERROR] 區塊
```

raw 裡被砍掉的關鍵行(`<專案>` 為去識別化):

```
[ERROR] Failed to execute goal org.apache.maven.plugins:maven-surefire-plugin:3.5.5:test
        (default-test) on project <專案>: No tests matching pattern "NoSuchTest" were
        executed! (Set -Dsurefire.failIfNoSpecifiedTests=false to ignore this error.)
```

修法方向:BUILD FAILURE 且無 XML enrichment 時,保留開頭數行 `[ERROR]`(去掉「re-run with -e/-X」
「[Help 1] 連結」那段樣板尾巴)。未驗證 `mvn` 是否同病(推測會,過濾邏輯 binary-agnostic)。

**2. mvnd daemon 專屬雜訊未過濾(每次 3–5 行)**

每個 `rtk mvnd <goal>` 都會漏出 mvnd client/daemon 的固定開場白與 jline 終端警告:

```
[INFO] Processing build on daemon <id>
[INFO] BuildTimeEventSpy is registered.
[INFO] Using the SmartBuilder implementation with a thread count of N
WARNING: Unable to create a system terminal, creating a dumb terminal ...        (jline,stderr)
[main] WARNING org.jline - Unable to create a system terminal, ...
```

原因:噪音 pattern 是照 `mvn` 輸出寫的,mvnd 才有的行不在清單(jline 警告是 rtk 以 pipe 捕捉
輸出、無 tty 所致,必現)。複現:macOS brew tap mvnd 跑任一 goal,`rtk mvnd compile` 最明顯。
修法方向:上述固定前綴加進噪音表 + 收一份真 mvnd fixture。
**這也推翻了原 backlog「mvnd 與 mvn 共用過濾邏輯,以 rtk mvn 實證即可涵蓋」的假設——mvnd 輸出確實長得不一樣。**

**3. `dependency:tree` 冷啟動下載雜訊漏過(~20 行)**

首跑時 `Downloading from central:` / `Downloaded from central:`(抓 maven-metadata.xml)原樣輸出;
`compile` 路徑的下載噪音有被濾掉,dep-tree 路徑沒有。暖機第二跑即消失(節省 −77% → −86%)。
複現:新機器或清掉 `~/.m2` 的 metadata 快取後首跑 `rtk mvnd dependency:tree`。
未驗證 `mvn` 是否同病(推測會)。

**4. 多 goal 模式測試時間顯示 `(?)`(cosmetic)**

`rtk mvnd clean test` 的摘要行是 `mvn test: 565 passed (?)`;單 goal 跑得出 `(8.228 s (Wall Clock))`。
多 goal 分段路徑沒把 wall-clock 接過來。複現:任何會通過的測試集跑 `rtk mvnd clean test`。
同類 cosmetic:mvnd 跑出來的摘要前綴仍寫 `mvn test:`(顯示標籤未隨 binary 切換;
`rtk gain` 的 tracking 標籤倒是正確分開記 `rtk mvnd ...`)。

## 已知 backlog(低/中風險,之後慢慢補)

- `-X` / `--debug` 應強制 passthrough(目前 debug 行會污染失敗詳情)
- 截斷(truncated)的 surefire XML 仍會丟失失敗細節(只剩計數)— 與下方已修的 self-closing 是不同情況
- Windows 未優先選 `mvnw.cmd`

> 原 backlog 的「無 `mvnd` 專屬 fixture/測試」「mojo 失敗原因被砍」「多 goal wall-clock / mvnd 標籤」
> 三項已於 2026-06-07 修復(見下方「已修(field-test gaps)」),已移出本清單。

### 已修(field-test gaps,2026-06-07)

上節「新發現問題」的 4 項,皆以 TDD(真實 fixture → snapshot/regression test → 實作)修復,
品質閘全綠、token 節省率不退步(test/verify 仍 ~99%+):

- ✅ **問題 1 — mojo 層級 BUILD FAILURE 原因被砍**(commit `acebb1d`):`filter_mvn_compile` 在
  沒有其他 `[ERROR]` 內容被保留時,把 `Failed to execute goal …: <原因>` 行接到 `BUILD FAILURE`
  前(去掉 ` -> [Help N]` 與 re-run 樣板尾巴);compile / 測試失敗輸出維持不變。fixture
  `mvn_test_no_matching_tests.txt` + regression 測試。binary-agnostic,mvn/mvnd 同修。
- ✅ **問題 2 — mvnd daemon + jline 雜訊**(commit `cbb700b`):daemon 開場白(`Processing build on
  daemon`、`BuildTimeEventSpy`、`SmartBuilder`)與 jline 終端警告(`WARNING: Unable to create a
  system terminal` / `WARNING org.jline`)加進 `is_mvn_startup_noise`(compile/checkstyle 共用的
  跨指令噪音閘)。首個 mvnd 專屬 fixture `mvnd_compile_daemon_noise.txt` + snapshot 測試。
- ✅ **問題 3 — dep-tree 冷啟動下載雜訊**(commit `b187f84`):`filter_mvn_dep_tree` 補上
  `Downloading `/`Downloaded ` 過濾,並改先過 `is_mvn_startup_noise`(daemon 雜訊在 dep-tree 也一併
  清掉)。fixture `mvnd_dep_tree_cold_download.txt` + 測試。binary-agnostic。
- ✅ **問題 4 — 多 goal wall-clock `(?)` + mvnd 摘要標籤**(commit `7f890da`):多 goal 測試摘要新增
  `time_override`,把 build wall-clock 從 raw 接過來(單 goal 仍自行解析、傳 `None`);新增
  `relabel_summary` 在 run_* 路徑把顯示前綴 `mvn …`→`mvnd …`(含 `(multi-goal)` 標頭、`mvn: ok`、
  `rtk proxy mvn …` 提示)。過濾器仍輸出 `mvn …`,故既有 snapshot 與 binary 無關;`rtk gain` 的
  tracking 標籤本就分開記 `mvnd …`、不受影響。

### 已修(2026-06-07,commit `27dc44c`)

- ✅ **self-closing `<failure .../>` / `<error .../>`**(body-less 失敗)過去被 quick-xml 當成
  `Event::Empty`、只有 End 分支會 push,導致該筆失敗的 XML enrichment 整個遺失;已抽 `push_failure`
  helper 在 Empty 路徑也補 push,並在 self-closing 後與 testcase 結束時重置 `capture` 防狀態洩漏
  (加了 regression fixture + 測試)
- ✅ **三個 production `expect()`**(`stack_trace.rs:190 / 222`、`mvn_cmd.rs run_compile_like`)改為
  infallible(`if-let` / `let-else` / `unwrap_or`),符合 no-`expect` / no-panic 規則
- ✅ **招牌節省率已實證**(見上方「本地安裝實戰紀錄」量測表 + `rtk gain` 數據)

## 與上游同步

```fish
git remote -v                       # origin = rtk-ai/rtk(上游), fork = 你的 fork
git fetch origin
git rebase origin/develop           # 把本地 mvn/mvnd 修改 rebase 到最新 develop
# 解完衝突後重跑品質閘,再 push 到 fork
cargo fmt --all && cargo clippy --all-targets && cargo test --all
```
