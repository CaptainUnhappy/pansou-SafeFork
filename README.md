# pansou 安全同步

[查看 pansou 代码（main）](https://github.com/CaptainUnhappy/pansou-SafeFork/tree/main) · [运行同步](https://github.com/CaptainUnhappy/pansou-SafeFork/actions/workflows/safe-fork-sync.yml) · [上游仓库](https://github.com/fish2018/pansou)

`main` 跟随 `fish2018/pansou` 的 `main`。`sync-control` 是默认分支，只保存同步工作流和使用说明；回滚 `main` 不会删除同步工作流。克隆代码时请指定 `git clone --branch main https://github.com/CaptainUnhappy/pansou-SafeFork.git`。

同步每小时第 17 分钟运行，也可以在 Actions → Safe Fork Sync → Run workflow 手动执行，分支选择 `sync-control`。勾选 `dry_run` 会检查同步条件但不更新 `main`。GitHub 调度可能延迟；公共仓库连续 60 天没有仓库活动时，定时任务可能自动停用。

工作流通过 GitHub API 把 `main` 快进到已检查的上游提交，不生成合并提交，不重写已有提交，也不创建标签。上游自身原有的合并提交会正常保留。上游不可访问、历史分叉、文件数量不足 3 个或降至 Fork 的 40% 以下、文件列表不完整时，工作流停止，并在运行日志和摘要中说明原因。文件数量只是异常检测，不能判断全部删除是否合理或代码是否安全。

默认使用仓库的 `GITHUB_TOKEN`。若上游更新涉及 `.github/workflows/`，GitHub 可能要求额外的 workflow 写权限。遇到该权限错误时，在本仓库 Actions secrets 中设置 `FORK_SYNC_PAT`：fine-grained PAT 只授权本仓库的 Contents 和 Workflows 写权限，或 classic PAT 使用 `repo` 和 `workflow` scopes。不要将 token 写入文件。使用 PAT 更新分支还可能触发上游带来的其他工作流。

## 手动回滚并测试同步

下面命令适用于已登录 `gh` 的 PowerShell，目标仅为本 Fork 的 `main`。它会先备份远端当前提交，然后将 `main` 回退到第一父提交。该回退是为测试而主动重写分支指针；正常同步工作流不执行回退。

先暂停定时同步，等待已有同步运行结束。暂停变量不影响手动执行：

```powershell
$repo = 'CaptainUnhappy/pansou-SafeFork'
gh variable set FORK_SYNC_PAUSED --repo $repo --body true
gh run list --repo $repo --workflow safe-fork-sync.yml --limit 10
```

确认列表中没有 `queued`、`in_progress` 或 `waiting` 的同步任务，再逐段执行以下命令。若某条命令报错，停止执行后续步骤。

```powershell
# 新建一个独立的 bare 仓库，不修改现有工作目录。
$testDir = Join-Path $PWD ('pansou-rollback-' + (Get-Date -Format 'yyyyMMdd-HHmmss') + '.git')
git clone --bare --single-branch --branch main "https://github.com/$repo.git" $testDir
if ($LASTEXITCODE -ne 0) { throw '克隆失败' }

$before = git --git-dir=$testDir rev-parse refs/heads/main
if ($LASTEXITCODE -ne 0) { throw '读取当前提交失败' }
$rollback = git --git-dir=$testDir rev-parse "$before^"
if ($LASTEXITCODE -ne 0) { throw '读取上一提交失败' }
$backup = 'rollback-backup-' + (Get-Date -Format 'yyyyMMdd-HHmmss')

# 备份必须成功后才允许回退。
git --git-dir=$testDir push origin "${before}:refs/heads/$backup"
if ($LASTEXITCODE -ne 0) { throw '备份失败，停止回退' }

Write-Host "回退前：$before"
Write-Host "目标提交：$rollback"
Write-Host "备份分支：$backup"
```

确认显示的回退目标后，执行实际回退。`--force-with-lease` 要求远端仍然等于刚才记录的提交，避免覆盖期间发生的新更新：

```powershell
git --git-dir=$testDir push "--force-with-lease=refs/heads/main:$before" origin "${rollback}:refs/heads/main"
if ($LASTEXITCODE -ne 0) { throw '回退失败，请重新检查远端状态' }

gh api "repos/$repo/git/ref/heads/main" --jq '.object.sha'
```

此时远端 `main` 应等于 `$rollback`；`sync-control` 不变。2026-09-09 部署前，`main` 为 `cc0087a5991a869739a9d07e516be6c1d8b7051f`，第一父提交为 `beaa56133755a548ebc51b090b3816e2ae044aa6`。测试时使用命令实时读取的值。

先只检查，打开生成的运行详情，确认显示“可以快进”且 `main` 仍为回退后的 SHA：

```powershell
gh workflow run safe-fork-sync.yml --repo $repo --ref sync-control -f dry_run=true
gh run list --repo $repo --workflow safe-fork-sync.yml --limit 3
```

随后实际同步：

```powershell
gh workflow run safe-fork-sync.yml --repo $repo --ref sync-control -f dry_run=false
gh run list --repo $repo --workflow safe-fork-sync.yml --limit 3
```

等待这次运行成功。运行摘要中的上游 SHA 应等于同步后的 `main`；没有新增的同步提交。如果此时上游又发布了提交，本次只同步到摘要中已检查的那个版本，下次继续同步。

```powershell
gh api "repos/$repo/git/ref/heads/main" --jq '.object.sha'
gh api repos/fish2018/pansou/git/ref/heads/main --jq '.object.sha'

# 测试结束后恢复每小时定时同步。
gh variable set FORK_SYNC_PAUSED --repo $repo --body false
```

若同步失败且想恢复回退前的状态，在同一个 PowerShell 会话中执行以下命令。它只在 `main` 仍处于此次回退点时恢复到备份提交；备份分支继续保留：

```powershell
git --git-dir=$testDir push "--force-with-lease=refs/heads/main:$rollback" origin "${before}:refs/heads/main"
if ($LASTEXITCODE -ne 0) { throw 'main 已变化，请人工检查，不要改用 --force' }
gh variable set FORK_SYNC_PAUSED --repo $repo --body false
```

API 行为与调度规则：[更新 Git 引用](https://docs.github.com/en/rest/git/refs#update-a-reference)、[定时工作流](https://docs.github.com/en/actions/reference/workflows-and-actions/events-that-trigger-workflows#schedule)。
