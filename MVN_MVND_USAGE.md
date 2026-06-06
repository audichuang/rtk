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

## 已知 backlog(低/中風險,之後慢慢補)

- `-X` / `--debug` 應強制 passthrough(目前 debug 行會污染失敗詳情)
- 截斷(truncated)的 surefire XML 仍會丟失失敗細節(只剩計數)— 與下方已修的 self-closing 是不同情況
- Windows 未優先選 `mvnw.cmd`
- 無 `mvnd` 專屬 fixture/測試(mvnd 與 mvn 共用過濾邏輯,以 `rtk mvn` 實證即可涵蓋)

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
