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

## 已知 backlog(低/中風險,之後慢慢補)

- `-X` / `--debug` 應強制 passthrough(目前 debug 行會污染失敗詳情)
- 截斷的 surefire XML 會丟失失敗細節(只剩計數)
- 招牌節省率(test 99% 等)目前無測試實證
- `stack_trace.rs:190 / 222` 兩個 `expect()` 違反 no-`expect` 規則(目前不變式成立、不會 panic)
- Windows 未優先選 `mvnw.cmd`
- 無 `mvnd` 專屬 fixture/測試

## 與上游同步

```fish
git remote -v                       # origin = rtk-ai/rtk(上游), fork = 你的 fork
git fetch origin
git rebase origin/develop           # 把本地 mvn/mvnd 修改 rebase 到最新 develop
# 解完衝突後重跑品質閘,再 push 到 fork
cargo fmt --all && cargo clippy --all-targets && cargo test --all
```
