# 本 fork 携带的本地补丁

本 fork 服务于 calico-studio（`lrbmike/calico-studio`）等本地依赖开发模式：
桌面项目通过 `go mod replace` 引用本仓库检出，`main` 分支只做上游
（`egoist/mygo`）的镜像，**永不承载改动**；所有为桌面需求做的修改都走
`fix/*` 主题分支，再由主题分支进入集成面 `local`。

## 分支拓扑

```
upstream/main ──ff-only──> main（镜像）        local（项目 replace 的稳定面）
                          └──cherry-pick──> fix/* 主题分支（单个改动）
```

**主 checkout 常驻 `local`**：`replace` 认目录不认分支，桌面项目读的就是
`~/workspaces/mine/mygo` 当前检出的内容。停在 `main` 会丢掉全部补丁（中文占位
下沉重新出现），停在某个 `fix/*` 上会丢掉别的主题。

写主题分支时用 worktree，别动主 checkout：

```sh
git worktree add ../mygo-topic -b fix/<主题> upstream/main
# 在 ../mygo-topic 里改、提交
git checkout local && git cherry-pick fix/<主题>   # 主 checkout 只做这一步
git worktree remove ../mygo-topic
git push -u origin fix/<主题>
```

只读地看上游代码不必切分支：`git show upstream/main:<path>`、`git log -p`。

## 补丁准入规则

1. **可选参数优先**：加字段、加选项、加回调，零值等于旧行为（例：
   `TextInputOptions` 之类的新选项）。这类改动对其他使用方无感。
2. **不改共享默认值**：桌面项目嫌某处排版不对，优先在应用侧覆盖已有参数；把
   `inputHeight` 这类全局计算改成"更适合某个 app"的算法，会同时改掉所有 mygo
   使用者，只有确认是上游 bug 时才动。
3. **项目独有逻辑留在项目仓库**：界面编排、业务流程、文案在 app 里组装，不进
   `ui/`。
4. **确实无法参数化时才考虑分叉**：先改抽象（插槽 / hook / 回调）；仍做不到才
   开项目专属分支，且必须配独立 worktree 并在下方清单记明合并责任。

## 动手前与落地后的检查

主题分支从 `upstream/main` 切、集成面才装全部补丁，两边因此可能错开版本：

1. **先对齐上游再切分支**：

   ```sh
   git remote add upstream https://github.com/egoist/mygo.git  # 只配一次
   git fetch upstream
   git checkout main && git merge --ff-only upstream/main
   git checkout local && git merge main
   git log --oneline local..upstream/main   # 必须为空
   ```

   本 fork 目前只配了 `origin`（`lrbmike/mygo`），`upstream` 需要按上面第一行补。
   在落后的上游上写补丁，cherry-pick 只按文本判断冲突：上游改了同一段逻辑但行
   位置错开时，它会"干净通过"，语义却是错的。

2. **看目标文件已被哪些补丁动过**，避免与已有主题踩在同一段代码上：

   ```sh
   git diff upstream/main <集成面分支> -- ui/editor_layout.go
   ```

3. **落进集成面后，每个使用本 fork 的项目各自跑全量测试**——文本冲突 Git 会报，
   语义回归只有项目自己的测试能发现（Calico Studio：`go test -count=1
   ./internal/app/`，另加 `-race`；mygo 自身：`go test ./ui/`）。

## 同步与冲突规则

1. 上游推进后：`main` 用 `git merge --ff-only upstream/main` 快进。
2. 主题分支同步上游：`git merge main`（用 merge 不用 rebase，保留冲突
   解决的历史），冲突按语义裁决——
   - 纯上游演进相关 → 取**上游**；
   - 与桌面需求相关 → 取**本 fork**。
3. 新增补丁：先从 `upstream/main` 切主题分支提交；优先"加文件/加可选
   参数"而不是改热点路径，把冲突面压到底。
4. 上游以其他写法吸收了某个主题后：从 `upstream/main` + 剩余主题重建，
   作废对应主题分支。

## 补丁清单

| 主题分支 | 内容 | 为什么桌面需要 | 上游状态 |
|---|---|---|---|
| `fix/cjk-placeholder-centering` | `ui/editor_layout.go` 两处：`inputHeight` 让单行输入框的高度把占位文字的行高也算进去（取两者较大者）；`paintInput` 让 placeholder 按其自身行高在内容框垂直居中 | Linux/Pango 下纯中文文本经 fontconfig 回退到 CJK 字体（如 Noto Sans CJK），行框比拉丁界面字体（如 Ubuntu Sans）高 ~3px。原实现只用空文本（纯拉丁字体）的行高定输入框高度和占位顶部，CJK 占位行框（20px）装进拉丁行框（17px）放不下，整体下沉 ~2px，与输入框图标错位；仅居中而不同时补高度会被 `(17-20)/2 < 0` 钳到 0，两处缺一不可。英文/数字无回退不受影响 | 未提 PR；已实测复现（`Metrics` 对比：Ubuntu Sans EN 17 / CJK 20）。上游若以其他写法修复，本主题作废并删除 |

## 已知与本 fork 无关的测试失败

- `go test ./ui/ -run TestTableRows`：在本机字体环境（无上游 CI 的字体集）
  修补丁前就失败（"a row of one line is 34 high"），与补丁无关，不阻塞。
