# pansou SafeFork

[代码（main）](https://github.com/CaptainUnhappy/pansou-SafeFork/tree/main) · [同步记录](https://github.com/CaptainUnhappy/pansou-SafeFork/actions/workflows/safe-fork-sync.yml) · [上游](https://github.com/fish2018/pansou) · [SafeFork 规范](https://github.com/CaptainUnhappy/SafeFork)

本仓库按 SafeFork 规范保留上游 Git 引用，同时拒绝破坏性镜像：

- 上游分支不存在于 Fork 时创建；已存在时只允许 fast-forward。
- 上游标签不存在于 Fork 时复制；同名不同 SHA 时停止。
- Fork 独有的分支和标签保留。
- 不执行 merge、force update 或删除。

`sync-control` 是默认分支，只保存工作流和说明，避免回滚代码分支时删掉定时任务。`main` 是代码入口；克隆代码请使用：

```powershell
git clone --branch main https://github.com/CaptainUnhappy/pansou-SafeFork.git
```

工作流每小时第 17 分钟运行。也可在 Actions → Safe Fork Sync → Run workflow 手动执行；选择 `sync-control`，勾选 `dry_run` 可只查看计划、零写入。

同步前会验证 Fork 的上游关系、所有来源引用快照、分支快进关系和并发变化。主分支文件数不足 3 个或降至当前的 40% 以下时会停止。任一分支分叉、标签冲突或 API 异常都会在写入前终止。

工作流用只读 `GITHUB_TOKEN` 做检查，用仅能写入本仓库的 `SAFEFORK_DEPLOY_KEY` 推送引用，因此上游提交涉及 `.github/workflows/` 时也不会要求广域 PAT。私钥只存于 Actions Secret。

回滚测试、冲突处理与新仓库接入步骤见 [SafeFork 规范](https://github.com/CaptainUnhappy/SafeFork/blob/main/skills/safefork/references/spec.md)。
