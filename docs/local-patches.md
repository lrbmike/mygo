# 本 fork 携带的本地补丁

本 fork 服务于 calico-studio（`lrbmike/calico-studio`）等本地依赖开发模式：
桌面项目通过 `go mod replace` 引用本仓库检出，`main` 分支只做上游
（`egoist/mygo`）的镜像，**永不承载改动**；所有为桌面需求做的修改都走
`fix/*` 主题分支。

## 分支拓扑

```
upstream/main ──ff-only──> main（镜像）        fix/cjk-placeholder-centering
                          └──cherry-pick──> 其余 fix/* 主题分支（单个改动）
```

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
