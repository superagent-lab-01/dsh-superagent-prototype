# Windows support for this prototype (unofficial)

These commits remove four POSIX assumptions that block this prototype on Windows. They are published here because the upstream repository has pull requests disabled, so there is no PR to open.

## Verified end to end

Environment: Windows 11, Node 22.19, DSH `0.2.1-rc.5`, plugin built with `pnpm build`, service started with `node lib/bin.js --profile superagent --no-open` (`scripts/dsh-superagent` needs tmux).

With these changes a request went all the way through: GitHub notification -> Ticket -> Router assessment (`strength: explicit`) -> isolated Worker session with its own worktree directory -> commit -> pull request -> merged.

## The four blockers

| # | Where | Symptom | Fix |
| --- | --- | --- | --- |
| 1 | `src/deployment.ts` (`absolute`) | `$.stateDir expect string to match regexp /^\//`; `/E:/x` passes but resolves to `E:\E:\x` and mkdir fails | accept a drive-letter path |
| 2 | `src/deployment.ts` (`workerProbe`) | every setup probe fails with `spawn env ENOENT`, reported misleadingly as `agentCliSignedOut` | pass PATH through the subprocess environment |
| 3 | `src/executor/dsh.ts` (`prepareDirectory`) | every Ticket fails in prepare when a prepare command is set (`sh -c` not found) | `cmd.exe /d /s /c` on win32 |
| 4 | `src/app/sessions.ts` | `tools.restrict() names unknown global tool "bash"` - orchestrator session cannot resume, nothing runs | filter the allow list against registered tools (`pwsh` on Windows) |

## Known remaining issues

- The Feishu probe in `src/deployment.ts` (`const run = ...`) still uses POSIX `env` for `lark-cli`; fixing it changes the `Probe` signature.
- An unclean shutdown leaves `<stateDir>.lock` behind. The next start logs `dsh: warning: 2 entries did not activate` + `Lock file is already being held` and every `/orchestrator/api/*` route returns 404 until the lock is removed. Operationally: stop the process gracefully, then delete `<stateDir>.lock`.
- The setup check also verifies `git` credentials for `github.com`. On networks where `github.com:443` is unreachable, mapping `https://github.com/` to `ssh://git@ssh.github.com:443/` through `git config url.<base>.insteadOf` lets everything work over SSH.