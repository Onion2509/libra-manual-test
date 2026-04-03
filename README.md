# 第一批命令人工验证测试方案

## 测试环境准备

### 前置条件

| 项目 | 要求 |
|------|------|
| Rust 工具链 | `stable` + `nightly`（格式化） |
| 编译 | `cargo build` 通过，产物路径 `target/debug/libra` |
| GitHub 账号 | 有 Personal Access Token（需 `repo` scope） |
| GitHub 测试仓库 | 在测试 namespace 下创建 `libra-manual-test`（public，含 `main` 分支和至少一个 commit） |
| 环境变量 | `LIBRA_TEST_GITHUB_TOKEN`、`LIBRA_TEST_GITHUB_NAMESPACE` |
| SSH 密钥 | 本机 `~/.ssh/id_ed25519` 或 `id_rsa` 已添加到 GitHub |
| jq | 用于验证 JSON 输出（`brew install jq`） |

### 约定

```bash
# 统一别名
alias libra='cargo run --'

# 测试工作目
export TEST_ROOT=$(mktemp -d)

# GitHub 变量
export GH_TOKEN="$LIBRA_TEST_GITHUB_TOKEN"
export GH_NS="$LIBRA_TEST_GITHUB_NAMESPACE"
export GH_REPO="libra-manual-test"
export GH_URL="https://github.com/$GH_NS/$GH_REPO.git"
export GH_AUTH_URL="https://x-access-token:$GH_TOKEN@github.com/$GH_NS/$GH_REPO.git"
```

### GitHub 测试仓库初始化（一次性）

```bash
cd $(mktemp -d)
git init && git config user.name "Test" && git config user.email "test@test.dev"
echo "# libra manual test" > README.md
git add . && git commit -m "initial commit"
git branch -M main
git remote add origin "$GH_AUTH_URL"
git push -u origin main

# 创建 dev 分支（用于 pull 测试）
echo "dev content" > dev.txt
git add . && git commit -m "dev commit"
git push origin main:dev
git push origin main # main 也推上去
```

---

## 验证记录模板

每条测试用例按以下格式记录：

| 字段 | 说明 |
|------|------|
| **ID** | 唯一编号 |
| **命令** | 被测命令 |
| **场景** | 测试什么 |
| **步骤** | 具体执行的命令 |
| **预期结果** | 期望的 stdout/stderr/exit code |
| **实际结果** | 填写 PASS / FAIL + 备注 |

---

## 一、config 命令

### C-01 基本 set/get

```bash
cd $TEST_ROOT && mkdir config-test && cd config-test
libra init
libra config set user.name "Test User"
libra config set user.email "test@example.com"
libra config get user.name
```

**预期**: stdout 输出 `Test User`，exit 0

**实际结果：**PASS

### C-02 list 所有配置

```bash
libra config list
```

**预期**: 列出 `user.name`、`user.email` 以及 init 写入的 seed keys（`core.*`、`libra.repoid`、`vault.signing` 等），exit 0

**实际结果：**PASS。但未输出exit 0。

### C-03 list --json

```bash
libra --json config list | jq '.ok'
libra --json config list | jq '.data[0].key'
```

**预期**: `ok: true`，`data` 为数组，每项含 `key`、`value`、`encrypted`、`scope` 字段

**实际结果：**FAIL。jq: error (at <stdin>:71): Cannot index object with number

### C-04 list --machine

```bash
libra --machine config list
```

**预期**: NDJSON 格式输出，每行可被 `jq` 解析

**实际结果：**FAIL。不是NDJSON输出的是普通JSON结构。

### C-05 unset 配置

```bash
libra config set test.key "to-remove"
libra config unset test.key
libra config get test.key; echo "exit=$?"
```

**预期**: get 返回 key not found 错误，exit 129，stderr 含 `LBR-CLI-003`

**实际结果：**PASS。但输出的是exit=1。stderr未包含`LBR-CLI-003`

### C-06 加密存储（vault）

```bash
libra config set --encrypt user.secret.token "my-secret-value"
libra config get user.secret.token
```

**预期**: get 输出 `[ENCRYPTED]`（不显示原文），exit 0

**实际结果：**PASS。但get输出的是<REDACTED>，未输出exit 0。

### C-07 --reveal 查看加密值

```bash
libra config get --reveal user.secret.token
```

**预期**: 输出 `my-secret-value`（明文），exit 0

**实际结果：**PASS。但未输出exit 0。

### C-08 敏感 key 自动加密

```bash
libra config set remote.origin.password "hunter2"
libra config get remote.origin.password
```

**预期**: 匹配 `password` 模式，自动加密，get 输出 `[ENCRYPTED]`

**实际结果：**PASS。但get实际输出<REDACTED>

### C-09 --show-origin

```bash
libra config list --show-origin
```

**预期**: 每条配置显示 `scope`（local/global）来源

**实际结果：**PASS

### C-10 --global scope

```bash
libra config set --global user.name "Global User"
libra config get --global user.name
```

**预期**: 输出 `Global User`，exit 0

**实际结果：**PASS。但没有exit 0。

### C-11 无效 key 格式

```bash
libra config get ""; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-002`

**实际结果：**PASS。但输出的是exit=1。stderr中未包含`LBR-CLI-002`。

### C-12 JSON 错误输出

```bash
libra --json config get nonexistent.key 2>&1 | jq '.ok'
```

**预期**: stderr JSON 含 `"ok": false`、`"error_code": "LBR-CLI-003"`

**实际结果：**FAIL。仅输出false。

### C-13 generate-ssh-key

```bash
libra config generate-ssh-key
```

**预期**: 生成 SSH 密钥对并存入 vault，显示公钥，exit 0

**实际结果：**FAIL。

error: the following required arguments were not provided:
-remote <REMOTE>

Usage: libra config generate-ssh-key --remote <REMOTE>

For more information, try '--help'.

### C-14 --encrypt 与 --plaintext 互斥

```bash
libra config set --encrypt --plaintext test.val "x"; echo "exit=$?"
```

**预期**: exit 129，stderr 提示互斥

**实际结果：**FAIL。

xit=$?"
error: --encrypt and --plaintext are mutually exclusive
exit=128

---

## 二、init 命令

### I-01 默认初始化

```bash
cd $TEST_ROOT && mkdir init-test && cd init-test
libra init
ls -la .libra/
```

**预期**: 创建 `.libra/` 目录，含 `libra.db`、`objects/`、`refs/` 等，stdout 显示 "Initialized empty Libra repository"，exit 0

**实际结果：**PASS。但stdout没有显示"Initialized empty Libra repository"，exit 0

### I-02 --json 输出

```bash
cd $TEST_ROOT && mkdir init-json && cd init-json
libra --json init | jq '.'
```

**预期**: `ok: true`，`data` 含 `path`、`bare`、`initial_branch`（"main"）、`object_format`、`repo_id`、`vault_signing`、`warnings`

**实际结果：**PASS

### I-03 --machine 输出

```bash
cd $TEST_ROOT && mkdir init-machine && cd init-machine
libra --machine init | jq '.'
```

**预期**: 单行 JSON，可被 jq 解析，含与 --json 相同字段

**实际结果：**PASS

### I-04 --bare 裸仓库

```bash
cd $TEST_ROOT && mkdir init-bare && cd init-bare
libra init --bare
ls objects/ refs/
```

**预期**: 当前目录即仓库根（无 `.libra/` 子目录），exit 0

**实际结果：**PASS。但未输出exit 0

### I-05 --initial-branch 自定义分支

```bash
cd $TEST_ROOT && mkdir init-branch && cd init-branch
libra --json init --initial-branch develop | jq '.data.initial_branch'
```

**预期**: 输出 `"develop"`

**实际结果：**PASS

### I-06 重复初始化报错

```bash
cd $TEST_ROOT/init-test
libra init; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-003`（already initialized）

**实际结果：**PASS。但stderr不含 `LBR-REPO-003`

### I-07 重复初始化 JSON 错误

```bash
cd $TEST_ROOT/init-test
libra --json init 2>&1 | grep error_code
```

**预期**: stderr JSON 含 `"error_code": "LBR-REPO-003"`

**实际结果：**PASS

### I-08 --quiet 静默

```bash
cd $TEST_ROOT && mkdir init-quiet && cd init-quiet
libra init --quiet
```

**预期**: stdout 无输出，exit 0

**实际结果：**PASS。

### I-09 --object-format sha256

```bash
cd $TEST_ROOT && mkdir init-sha256 && cd init-sha256
libra --json init --object-format sha256 | jq '.data.object_format'
```

**预期**: `"sha256"`

**实际结果：**PASS

### I-10 --object-format 拼写纠错

```bash
cd $TEST_ROOT && mkdir init-typo && cd init-typo
libra init --object-format sha265 2>&1
echo "exit=$?"
```

**预期**: exit 129，stderr 含 "did you mean 'sha256'?"

**实际结果：**PASS

### I-11 --vault false

```bash
cd $TEST_ROOT && mkdir init-novault && cd init-novault
libra --json init --vault false | jq '.data.vault_signing'
```

**预期**: `false`

**实际结果：**PASS

### I-12 vault 默认 true

```bash
cd $TEST_ROOT && mkdir init-vault && cd init-vault
libra --json init | jq '.data.vault_signing'
```

**预期**: `true`

**实际结果：**PASS

---

## 三、clone 命令

### CL-01 HTTPS clone

```bash
cd $TEST_ROOT
libra clone "$GH_AUTH_URL" clone-https
ls clone-https/README.md
```

**预期**: 成功 clone，README.md 存在，exit 0

**实际结果：**PASS

### CL-02 --json 输出

```bash
cd $TEST_ROOT
libra --json clone "$GH_AUTH_URL" clone-json | jq '.'
```

**预期**: `ok: true`，`data` 含 `path`、`remote_url`、`branch`（"main"）、`repo_id`、`vault_signing`、`shallow`（false）、`warnings`

**实际结果：**PASS

### CL-03 --machine 输出

```bash
cd $TEST_ROOT
libra --machine clone "$GH_AUTH_URL" clone-machine | jq '.'
```

**预期**: 单行 NDJSON，字段同 CL-02

**实际结果：**PASS

### CL-04 --bare clone

```bash
cd $TEST_ROOT
libra --json clone --bare "$GH_AUTH_URL" clone-bare.git | jq '.data.bare'
```

**预期**: `true`，clone-bare.git 目录下有 `objects/`、`refs/`

**实际结果：**FAIL。没有`refs/`

### CL-05 --branch 指定分支

```bash
cd $TEST_ROOT
libra --json clone --branch dev "$GH_AUTH_URL" clone-dev | jq '.data.branch'
```

**预期**: `"dev"`，工作目录含 `dev.txt`

**实际结果：**PASS

### CL-06 目标目录已存在且非空

```bash
cd $TEST_ROOT && mkdir clone-exists && echo "x" > clone-exists/file.txt
libra clone "$GH_AUTH_URL" clone-exists; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-003`（destination exists/not empty）

**实际结果：**PASS。但是stderr 未含 `LBR-CLI-003`

### CL-07 无效 URL

```bash
libra clone "https://github.com/nonexistent/repo-does-not-exist-12345.git" clone-bad 2>&1
echo "exit=$?"
```

**预期**: exit 128，stderr 含网络/认证错误码
**实际结果**FAIL。让输入账号密码。

### CL-08 JSON 错误输出

```bash
libra --json clone "https://invalid-url-xxx" clone-err 2>&1 | grep error_code
```

**预期**: stderr JSON 含 `"ok": false`，带 `error_code`

**实际结果：**PASS。但stderr只有  "error_code": "LBR-NET-001",

### CL-09 --quiet 静默

```bash
cd $TEST_ROOT
libra clone --quiet "$GH_AUTH_URL" clone-quiet
echo "exit=$?"
```

**预期**: stdout 无进度输出，exit 0，目录已正确 clone

**实际结果：**PASS

### CL-10 自动推断目标目录名

```bash
cd $TEST_ROOT
libra clone "$GH_AUTH_URL"
ls "$GH_REPO/README.md"
```

**预期**: 自动创建 `libra-manual-test/` 目录

**实际结果：**PASS。

---

## 四、add 命令

### A-01 基本添加

```bash
cd $TEST_ROOT && mkdir add-test && cd add-test
libra init
echo "hello" > file1.txt
echo "world" > file2.txt
libra add file1.txt file2.txt
libra status
```

**预期**: status 显示 file1.txt、file2.txt 在 staged/new 中，exit 0

**实际结果：**PASS。但没有exit 0

### A-02 --json 输出

```bash
echo "new" > file3.txt
libra --json add file3.txt | jq '.'
```

**预期**: `ok: true`，`data.added` 含 `"file3.txt"`

**实际结果：**PASS

### A-03 --all 全部暂存

```bash
echo "modified" >> file1.txt
rm file2.txt
echo "brand-new" > file4.txt
libra --json add --all | jq '.'
```

**预期**: `data` 含 `modified`（file1.txt）、`removed`（file2.txt）、`added`（file4.txt）

**实际结果：**PASS

### A-04 --dry-run 预览

```bash
echo "dry" > dry.txt
libra --json add --dry-run dry.txt | jq '.data.dry_run'
libra status | grep dry.txt
```

**预期**: `dry_run: true`，status 中 dry.txt 仍为 untracked

**实际结果：**PASS

### A-05 --verbose 详细输出

```bash
echo "verbose" > verbose.txt
libra add --verbose verbose.txt
```

**预期**: 输出含 `add(new): verbose.txt` 前缀

**实际结果：**PASS

### A-06 pathspec 不匹配

```bash
libra add nonexistent.txt; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-003`

**实际结果：**PASS。但stderr中不含`LBR-CLI-003`

### A-07 无参数无标志

```bash
libra add; echo "exit=$?"
```

**预期**: exit 129，错误提示需要路径或标志

**实际结果：**PASS

### A-08 --update 仅更新已跟踪

```bash
echo "untracked" > untracked.txt
echo "update" >> file1.txt
libra --json add --update | jq '.data'
```

**预期**: `modified` 含 file1.txt，`added` 不含 untracked.txt

**实际结果：**PASS

### A-09 --force 添加忽略文件

```bash
echo "*.log" > .gitignore
libra add .gitignore && libra commit -m "add gitignore"
echo "test" > test.log
libra add test.log; echo "exit=$?"
libra add --force test.log; echo "exit=$?"
```

**预期**: 第一次 add 失败（被 ignore），--force 成功

**实际结果：**FAIL。第一次就成功。

### A-10 --quiet 静默

```bash
echo "quiet" > quiet.txt
libra add --quiet quiet.txt
```

**预期**: stdout 无输出，exit 0

**实际结果：**PASS。但是也没有exit 0

### A-11 --machine 输出

```bash
echo "machine" > machine.txt
libra --machine add machine.txt | jq '.'
```

**预期**: 单行 NDJSON

**实际结果：**PASS

### A-12 --exit-code-on-warning

```bash
echo "warn" > warn.log
libra --exit-code-on-warning add warn.log; echo "exit=$?"
```

**预期**: .log 被 ignore，触发 warning，exit 9

**实际结果：**FAIL。被成功add

### A-13 --refresh 刷新索引

```bash
libra --json add --refresh | jq '.data.refreshed'
```

**预期**: 返回刷新的文件列表，exit 0

**实际结果：**FAIL。输出了[]

---

## 五、status 命令

### S-01 干净工作树

```bash
cd $TEST_ROOT && mkdir status-test && cd status-test
libra init
echo "init" > file.txt && libra add file.txt && libra commit -m "init"
libra status
```

**预期**: 显示 "nothing to commit, working tree clean"，exit 0

**实际结果：**PASS。但是没有exit 0

### S-02 --json 干净

```bash
libra --json status | jq '.data.is_clean'
```

**预期**: `true`

**实际结果：**PASS

### S-03 有修改的 --json

```bash
echo "changed" >> file.txt
echo "new" > new.txt
libra --json status | jq '.'
```

**预期**: `is_clean: false`，`unstaged.modified` 含 file.txt，`untracked` 含 new.txt

**实际结果：**PASS

### S-04 有暂存的 --json

```bash
libra add file.txt
libra --json status | jq '.data.staged'
```

**预期**: `staged.modified` 含 file.txt

**实际结果：**PASS

### S-05 --short 短格式

```bash
libra status --short
```

**预期**: 类似 Git 短格式（`M file.txt`、`?? new.txt`）

**实际结果：**PASS

### S-06 --porcelain v2

```bash
libra status --porcelain v2
```

**预期**: porcelain v2 格式，含 `# branch.oid`、`# branch.head`

**实际结果：**FAIL。输出为 porcelain v2 记录格式，但未包含预期中的 # branch.oid 和 # branch.head 头信息。

### S-07 --exit-code（dirty）

```bash
libra status --exit-code; echo "exit=$?"
```

**预期**: exit 1（有未提交更改）

**实际结果：**PASS

### S-08 --exit-code（clean）

```bash
libra add --all && libra commit -m "clean up"
libra status --exit-code; echo "exit=$?"
```

**预期**: exit 0

**实际结果：**PASS

### S-09 --branch 分支信息

```bash
libra status --branch
```

**预期**: 显示当前分支名和 tracking 信息

**实际结果：**PASS

### S-10 detached HEAD

```bash
COMMIT=$(libra --json log 2>/dev/null | jq -r '.data[0].commit // empty' 2>/dev/null || libra log --oneline | head -1 | awk '{print $1}')
libra switch --detach "$COMMIT" 2>/dev/null || true
libra --json status | jq '.data.head'
```

**预期**: `head.type` 为 `"detached"`，含 `oid`

**实际结果：**PASS

### S-11 --machine 输出

```bash
libra --machine status | jq '.'
```

**预期**: 单行 NDJSON，字段完整

**实际结果：**PASS

### S-12 --quiet 静默

```bash
libra status --quiet
```

**预期**: stdout 无输出，exit 0

**实际结果：**PASS。但没有exit 0

### S-13 --untracked-files no

```bash
echo "untracked" > ut.txt
libra --json status --untracked-files no | jq '.data.untracked'
```

**预期**: `untracked` 为空数组

**实际结果：**PASS

### S-14 非仓库目录

```bash
cd /tmp
libra status; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-001`

**实际结果：**PASS。但是stderr中未包含`LBR-REPO-001`

---

## 六、commit 命令

### CM-01 基本提交

```bash
cd $TEST_ROOT && mkdir commit-test && cd commit-test
libra init
libra config set user.name "Test" && libra config set user.email "test@test.dev"
echo "hello" > file.txt && libra add file.txt
libra commit -m "initial commit"
```

**预期**: 输出 `[main <short_id>] initial commit`，exit 0

**实际输出：**PASS。但是并没有输出exit 0

### CM-02 --json 输出

```bash
echo "more" > file2.txt && libra add file2.txt
libra --json commit -m "add file2" | jq '.'
```

**预期**: `ok: true`，`data` 含 `head`、`branch`、`commit`、`short_id`、`subject`、`root_commit`（false）、`files_changed`

**实际结果：**PASS

### CM-03 root-commit 标识

```bash
cd $TEST_ROOT && mkdir commit-root && cd commit-root
libra init
libra config set user.name "Test" && libra config set user.email "test@test.dev"
echo "first" > f.txt && libra add f.txt
libra --json commit -m "first" | jq '.data.root_commit'
```

**预期**: `true`
**实际结果：**PASS

### CM-04 nothing to commit

```bash
cd $TEST_ROOT/commit-test
libra commit -m "empty"; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-003`

**实际结果：**PASS。但是stderr中未含`LBR-REPO-003`

### CM-05 nothing to commit --json

```bash
libra --json commit -m "empty" 2>&1 | grep error_code
```

**预期**: stderr 含 `"error_code": "LBR-REPO-003"`

**实际结果：**PASS

### CM-06 --amend

```bash
echo "amend" >> file.txt && libra add file.txt
libra --json commit --amend -m "amended message" | jq '.data.amend'
```

**预期**: `true`

**实际结果：**PASS

### CM-07 --amend --no-edit

```bash
echo "amend2" >> file.txt && libra add file.txt
libra --json commit --amend --no-edit | jq '.data.subject'
```

**预期**: subject 保持 "amended message"（复用上次提交信息）

**实际结果：**FAIL。实际输出"gpgsig -----BEGIN PGP SIGNATURE-----"

### CM-08 -a 自动暂存

```bash
echo "auto" >> file.txt
libra --json commit -a -m "auto stage" | jq '.data.files_changed'
```

**预期**: `total >= 1`，`modified >= 1`

**实际结果：**FAIL。实际输出：

{
  "deleted": 0,
  "modified": 1,
  "new": 0,
  "total": 1
}

### CM-09 --signoff

```bash
echo "signoff" > s.txt && libra add s.txt
libra --json commit --signoff -m "signed commit" | jq '.data.signoff'
```

**预期**: `true`

**实际结果：**PASS

### CM-10 --conventional 合规

```bash
echo "conv" > c.txt && libra add c.txt
libra commit --conventional -m "feat(core): add feature"; echo "exit=$?"
```

**预期**: exit 0

**实际结果：**PASS

### CM-11 --conventional 不合规

```bash
echo "bad" > b.txt && libra add b.txt
libra commit --conventional -m "bad message"; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-002`

**实际结果：**PASS。但是stderr未含`LBR-CLI-002`

### CM-12 --author 覆盖

```bash
echo "author" > a.txt && libra add a.txt
libra --json commit --author "Custom Author <custom@test.dev>" -m "custom author" | jq '.'
```

**预期**: exit 0，提交成功

**实际结果：**PASS。但实际输出为：
{
  "command": "commit",
  "data": {
    "amend": false,
    "branch": "main",
    "commit": "308ab02d5068ed4fe57e3f83cefff5dbbc8a54fa",
    "conventional": null,
    "files_changed": {
      "deleted": 0,
      "modified": 0,
      "new": 2,
      "total": 2
    },
    "head": "main",
    "root_commit": false,
    "short_id": "308ab02",
    "signed": true,
    "signoff": false,
    "subject": "custom author"
  },
  "ok": true
}

### CM-13 --author 无效格式

```bash
echo "bad" > bad.txt && libra add bad.txt
libra commit --author "no-angle-brackets" -m "bad"; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-002`

**实际结果：**PASS。但stderr未含`LBR-CLI-002`

### CM-14 -F 文件读取消息

```bash
echo "message from file" > /tmp/msg.txt
echo "fromfile" > ff.txt && libra add ff.txt
libra --json commit -F /tmp/msg.txt | jq '.data.subject'
```

**预期**: `"message from file"`

**实际结果：**PASS

### CM-15 --allow-empty

```bash
libra --json commit --allow-empty -m "empty commit" | jq '.data.files_changed.total'
```

**预期**: `0`，exit 0

**实际结果：**PASS。但未输出exit 0

### CM-16 --quiet 静默

```bash
echo "q" > q.txt && libra add q.txt
libra commit --quiet -m "quiet commit"
```

**预期**: stdout 无输出，exit 0

**实际结果：**PASS。但未输出exit 0

### CM-17 --machine 输出

```bash
echo "m" > m.txt && libra add m.txt
libra --machine commit -m "machine" | jq '.'
```

**预期**: 单行 NDJSON

**实际结果：**PASS

### CM-18 signed 字段（vault 签名）

```bash
echo "signed" > signed.txt && libra add signed.txt
libra --json commit -m "signed" | jq '.data.signed'
```

**预期**: `true`（init 默认 vault signing = true）或 `false`（取决于 vault 状态）

**实际结果：**PASS。

---

## 七、push 命令

### P-01 基本推送

```bash
cd $TEST_ROOT && libra clone "$GH_AUTH_URL" push-test && cd push-test
libra config set user.name "Test" && libra config set user.email "test@test.dev"
echo "push-content" > pushed.txt && libra add pushed.txt
libra commit -m "push test"
libra push origin main
echo "exit=$?"
```

**预期**: exit 0，成功推送到 GitHub

**实际结果：**PASS

### P-02 --json 输出

```bash
echo "json-push" > jp.txt && libra add jp.txt && libra commit -m "json push"
libra --json push origin main | jq '.'
```

**预期**: `ok: true`，`data` 含 `remote`、`url`、`branch`、`pushed`（含 `updated_refs`）、`objects_sent`、`force`（false）

**实际结果：**FAIL。json中部分数据不一致

### P-03 --dry-run 预览

```bash
echo "dry" > dry.txt && libra add dry.txt && libra commit -m "dry"
libra --json push --dry-run origin main | jq '.data'
```

**预期**: exit 0，但不实际推送（后续 pull 看不到该 commit）

**实际结果：**PASS

### P-04 --set-upstream

```bash
libra switch -c feature-upstream
echo "upstream" > up.txt && libra add up.txt && libra commit -m "upstream"
libra --json push --set-upstream origin feature-upstream | jq '.data.set_upstream'
```

**预期**: `true`，后续 push/pull 不需要指定 remote/branch

**实际结果：**FAIL。输出null

### P-05 non-fast-forward 拒绝

```bash
# 在 GitHub 上直接修改，制造 non-fast-forward
cd $TEST_ROOT && libra clone "$GH_AUTH_URL" push-conflict && cd push-conflict
libra config set user.name "Test" && libra config set user.email "test@test.dev"

# 用另一个 clone 推送一个 commit
cd $TEST_ROOT && libra clone "$GH_AUTH_URL" push-other && cd push-other
libra config set user.name "Other" && libra config set user.email "other@test.dev"
echo "other" > other.txt && libra add other.txt && libra commit -m "other"
libra push origin main

# 回到原 clone，推送冲突
cd $TEST_ROOT/push-conflict
echo "conflict" > c.txt && libra add c.txt && libra commit -m "conflict"
libra push origin main; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-CONFLICT-002`（non-fast-forward）

**实际结果：**PASS

### P-06 --force 强制推送

```bash
cd $TEST_ROOT/push-conflict
libra push --force origin main; echo "exit=$?"
```

**预期**: exit 0，强制覆盖远程

**实际结果：**PASS

### P-07 远程不存在（fuzzy match）

```bash
cd $TEST_ROOT/push-test
libra push origni main 2>&1
echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-003`，可能含 "did you mean 'origin'?"

**实际结果：**PASS

### P-08 无效 refspec

```bash
libra push origin "a:b:c"; echo "exit=$?"
```

**预期**: exit 129，stderr 含 `LBR-CLI-002`

**实际结果：**PASS

### P-09 --quiet 静默

```bash
echo "quiet" > qp.txt && libra add qp.txt && libra commit -m "quiet push"
libra push --quiet origin main
```

**预期**: stdout 无进度输出，exit 0

**实际结果：**PASS

### P-10 JSON 错误输出

```bash
libra --json push nonexistent-remote main 2>&1 | grep error_code
```

**预期**: stderr 含 `"error_code": "LBR-CLI-003"`

**实际结果：**PASS

### P-11 detached HEAD 推送

```bash
libra switch --detach HEAD
libra push origin main; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-003`（detached HEAD）

**实际结果：**PASS

---

## 八、pull 命令

### PL-01 基本 pull（fast-forward）

```bash
cd $TEST_ROOT && libra clone "$GH_AUTH_URL" pull-test && cd pull-test
libra config set user.name "Test" && libra config set user.email "test@test.dev"

# 从另一个 clone 推送新内容
cd $TEST_ROOT && libra clone "$GH_AUTH_URL" pull-pusher && cd pull-pusher
libra config set user.name "Pusher" && libra config set user.email "pusher@test.dev"
echo "new-content" > pulled.txt && libra add pulled.txt && libra commit -m "to be pulled"
libra push origin main

# 回到 pull-test 执行 pull
cd $TEST_ROOT/pull-test
libra pull origin main
echo "exit=$?"
cat pulled.txt
```

**预期**: exit 0，`pulled.txt` 内容为 "new-content"

### PL-02 --json 输出

```bash
# 再推一个 commit
cd $TEST_ROOT/pull-pusher
echo "more" > more.txt && libra add more.txt && libra commit -m "more content"
libra push origin main

cd $TEST_ROOT/pull-test
libra --json pull origin main | jq '.'
```

**预期**: `ok: true`，`data` 含 `branch`、`upstream`、`fetch`（含 `refs_updated`、`objects_fetched`）、`merge`（含 `strategy: "fast-forward"`、`files_changed`、`up_to_date: false`）

### PL-03 already up-to-date

```bash
cd $TEST_ROOT/pull-test
libra --json pull origin main | jq '.data.merge.up_to_date'
```

**预期**: `true`

### PL-04 detached HEAD 拒绝

```bash
libra switch --detach HEAD
libra pull; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-003`，hint 提示 checkout 分支

### PL-05 无 tracking 信息

```bash
libra switch -c orphan-branch
libra pull; echo "exit=$?"
```

**预期**: exit 128，stderr 含 `LBR-REPO-003`，hint 提示设置 upstream

### PL-06 远程不存在（fuzzy match）

```bash
libra switch main
libra pull origni main 2>&1
echo "exit=$?"
```

**预期**: exit 129，含 "did you mean 'origin'?"

### PL-07 --quiet 静默

```bash
# 推送新内容
cd $TEST_ROOT/pull-pusher
echo "quiet" > quiet.txt && libra add quiet.txt && libra commit -m "quiet"
libra push origin main

cd $TEST_ROOT/pull-test
libra pull --quiet origin main
```

**预期**: stdout 无进度输出，exit 0

### PL-08 JSON 错误输出

```bash
libra --json pull nonexistent-remote 2>&1 | grep error_code
```

**预期**: stderr 含 `"error_code": "LBR-CLI-003"`

---

## 九、全局标志交叉验证

### G-01 --color never

```bash
cd $TEST_ROOT/status-test 2>/dev/null || (cd $TEST_ROOT && mkdir g-test && cd g-test && libra init)
libra status --color never 2>&1 | cat -v
```

**预期**: 无 ANSI 转义码

### G-02 --color always

```bash
libra status --color always 2>&1 | cat -v
```

**预期**: 含 ANSI 转义码（即使 stdout 不是 TTY）

### G-03 NO_COLOR 环境变量

```bash
NO_COLOR=1 libra status 2>&1 | cat -v
```

**预期**: 无 ANSI 转义码

### G-04 --no-pager

```bash
libra log --no-pager
```

**预期**: 直接输出到 stdout，不通过 less/pager

### G-05 --progress none

```bash
cd $TEST_ROOT && libra clone --progress none "$GH_AUTH_URL" clone-noprogress 2>&1
```

**预期**: 无进度条输出

### G-06 --progress json

```bash
cd $TEST_ROOT && libra clone --progress json "$GH_AUTH_URL" clone-jsonprogress 2>&1 | head -5
```

**预期**: stderr 输出 NDJSON 进度事件

### G-07 --exit-code-on-warning 全局

```bash
cd $TEST_ROOT/add-test 2>/dev/null || (cd $TEST_ROOT && mkdir g-warn && cd g-warn && libra init)
echo "*.log" > .gitignore && libra add .gitignore 2>/dev/null; libra commit -m "ignore" 2>/dev/null
echo "test" > test.log
libra --exit-code-on-warning add test.log; echo "exit=$?"
```

**预期**: exit 9

### G-08 --json 与 --quiet 组合

```bash
libra --json --quiet status | jq '.ok'
```

**预期**: JSON 仍正常输出（--json 优先于 --quiet）

### G-09 --machine 综合

```bash
libra --machine status | jq '.'
```

**预期**: 单行 NDJSON，无颜色，无进度

---

## 十、错误码一致性验证

### E-01 LBR-REPO-001 非仓库目录

```bash
cd /tmp
libra --json status 2>&1 | jq -r '.error_code'
echo "exit=$?"
```

**预期**: `LBR-REPO-001`，exit 128

### E-02 LBR-CLI-002 无效参数

```bash
cd $TEST_ROOT/commit-test
libra commit --conventional -m "bad"; echo "exit=$?"
libra --json commit --conventional -m "bad" 2>&1 | jq -r '.error_code'
```

**预期**: `LBR-CLI-002`，exit 129

### E-03 LBR-CLI-003 无效目标

```bash
libra add /nonexistent/path; echo "exit=$?"
libra --json add /nonexistent/path 2>&1 | jq -r '.error_code'
```

**预期**: `LBR-CLI-003`，exit 129

### E-04 LBR-REPO-003 状态不允许

```bash
libra commit -m "nothing"; echo "exit=$?"
```

**预期**: exit 128，`LBR-REPO-003`

### E-05 退出码三级模型

```bash
# exit 0: 成功
libra status; echo "SUCCESS=$?"

# exit 128: 运行时错误
cd /tmp && libra status; echo "RUNTIME=$?"

# exit 129: 参数错误
cd $TEST_ROOT/commit-test
libra commit --author "bad-format" -m "x"; echo "USAGE=$?"
```

**预期**: SUCCESS=0, RUNTIME=128, USAGE=129

### E-06 错误 JSON 信封格式一致性

```bash
cd /tmp
libra --json status 2>&1 | jq 'keys'
```

**预期**: 含 `ok`、`error_code`、`category`、`exit_code`、`message`

### E-07 hint 字段

```bash
cd /tmp
libra --json status 2>&1 | jq '.hints'
```

**预期**: hints 数组非空，含可操作建议（如 "run 'libra init'"）

---

## 十一、端到端工作流

### E2E-01 完整 init → add → commit → push → clone → pull 流程

```bash
# 创建临时 GitHub 仓库
REPO_NAME="libra-e2e-$(date +%s)"
curl -s -H "Authorization: Bearer $GH_TOKEN" \
-H "Accept: application/vnd.github+json" \
https://api.github.com/user/repos \
-d "{\"name\":\"$REPO_NAME\",\"auto_init\":false,\"private\":true}" | jq '.full_name'

E2E_URL="https://x-access-token:$GH_TOKEN@github.com/$GH_NS/$REPO_NAME.git"

# 1. init
cd $TEST_ROOT && mkdir e2e && cd e2e
libra init
libra config set user.name "E2E Test"
libra config set user.email "e2e@test.dev"
libra remote add origin "$E2E_URL"

# 2. add + commit
echo "hello e2e" > hello.txt
mkdir -p src && echo "fn main() {}" > src/main.rs
libra add hello.txt src/main.rs
libra --json commit -m "feat: initial e2e commit" | jq '.data.root_commit'
# 预期: true

# 3. push
libra --json push --set-upstream origin main | jq '.data.pushed'
# 预期: added_refs 含 refs/heads/main

# 4. clone（新目录）
cd $TEST_ROOT
libra --json clone "$E2E_URL" e2e-clone | jq '.data.branch'
# 预期: "main"

# 5. 修改 + push
cd $TEST_ROOT/e2e
echo "updated" >> hello.txt
libra add hello.txt
libra commit -m "fix: update hello"
libra push origin main

# 6. pull
cd $TEST_ROOT/e2e-clone
libra --json pull origin main | jq '.data.merge'
# 预期: strategy=fast-forward, up_to_date=false, files_changed>=1
cat hello.txt
# 预期: 含 "updated"

# 7. status 验证 clean
libra --json status | jq '.data.is_clean'
# 预期: true

# 清理
curl -s -X DELETE -H "Authorization: Bearer $GH_TOKEN" \
-H "Accept: application/vnd.github+json" \
"https://api.github.com/repos/$GH_NS/$REPO_NAME"
```

### E2E-02 config vault → push 认证流程

```bash
cd $TEST_ROOT && mkdir e2e-vault && cd e2e-vault
libra init
libra config set user.name "Vault Test"
libra config set user.email "vault@test.dev"
libra config set --encrypt remote.origin.token "$GH_TOKEN"
libra config get remote.origin.token
# 预期: [ENCRYPTED]
libra config get --reveal remote.origin.token
# 预期: 实际 token 值
```

### E2E-03 全 JSON 模式流水线

```bash
cd $TEST_ROOT && mkdir e2e-json && cd e2e-json
libra --json init | jq -e '.ok == true'
libra config set user.name "JSON Test"
libra config set user.email "json@test.dev"
echo "pipeline" > data.txt
libra --json add data.txt | jq -e '.data.added | length > 0'
libra --json commit -m "json pipeline" | jq -e '.data.commit != null'
libra --json status | jq -e '.data.is_clean == true'
echo "ALL JSON CHECKS PASSED"
```

### E2E-04 --machine 模式全链路

```bash
cd $TEST_ROOT && mkdir e2e-machine && cd e2e-machine
libra --machine init | jq -e '.ok'
libra config set user.name "Machine"
libra config set user.email "machine@test.dev"
echo "m" > m.txt
libra --machine add m.txt | jq -e '.ok'
libra --machine commit -m "machine test" | jq -e '.ok'
libra --machine status | jq -e '.data.is_clean'
echo "ALL MACHINE CHECKS PASSED"
```

---

## 测试结果汇总表

| 模块 | 测试编号范围 | 总数 | PASS | FAIL | 备注 |
|------|-------------|------|------|------|------|
| config | C-01 ~ C-14 | 14 | | | |
| init | I-01 ~ I-12 | 12 | | | |
| clone | CL-01 ~ CL-10 | 10 | | | |
| add | A-01 ~ A-13 | 13 | | | |
| status | S-01 ~ S-14 | 14 | | | |
| commit | CM-01 ~ CM-18 | 18 | | | |
| push | P-01 ~ P-11 | 11 | | | |
| pull | PL-01 ~ PL-08 | 8 | | | |
| 全局标志 | G-01 ~ G-09 | 9 | | | |
| 错误码 | E-01 ~ E-07 | 7 | | | |
| 端到端 | E2E-01 ~ E2E-04 | 4 | | | |
| **合计** | | **120** | | | |

---

## 清理

```bash
# 删除本地测试目录
rm -rf $TEST_ROOT

# 删除 GitHub 测试仓库（如果是临时创建的）
curl -s -X DELETE -H "Authorization: Bearer $GH_TOKEN" \
-H "Accept: application/vnd.github+json" \
"https://api.github.com/repos/$GH_NS/$GH_REPO"
```


