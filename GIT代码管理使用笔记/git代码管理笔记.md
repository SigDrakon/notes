

# 1.代码提交流程


## 1.1.前置问题：为什么不直接在 develop 分支上提交代码？

1. 多人修改冲突风险大

- dev 是团队的开发主线，大家都在上面推代码。
- 一个人推的代码可能影响到别人的开发、测试或构建。
- 分支分开开发可以先在自己分支上测试、合并时统一解决冲突。

2. 代码质量控制困难

- 直接推 dev 让代码跳过 Code Review、测试验证等环节。
- 如果出现问题，dev 就不再稳定。

3. 缺少变更隔离

- 新功能、修复、实验性修改混在一起，不好追踪来源。
- 分支能清楚地表明变更的上下文（例如：feature-add-upf-logging、 fixbug-gtpu-parser）。


根据以上问题可以很容易的得知，独立分支存在以下好处：

- 清晰的开发流：每个功能/修复都在独立分支中进行。
- 方便代码评审：合并到 dev 之前可以通过 PR（或 MR）进行 review。
- 回滚更容易：如果某个功能有问题，可以整分支回滚，不影响其他开发。
- CI/CD 友好：可以针对分支触发自动化测试、构建。


## 1.2.前置问题2：新分支需不需要删除？

要删除，除非它仍在使用，当分支功能已经合并进 dev 并经过验证后，就可以：

```bash
git branch -d feature/my-feature        	 # 删除本地分支
git push origin --delete feature/my-feature  # 删除远程分支
```

这样可以保持远程仓库干净，避免出现上百个历史分支。

## 2.具体的代码提交示例


1. 在 dev 分支上拉取最新代码

```bash
git checkout dev

git pull origin dev
```

2. 创建一个新的功能分支

```bash
需要遵循命名规范：
- 新功能： feature-***
- 修复：   fixbug-***

git checkout -b feature-upf-li-support
```

3. 在新分支中修改代码

```
git add --all

git commit -m ""
```

4. 推送到远程仓库

```bash
# 推送到远程仓库（会自动在远程创建该分支）
git push origin feature/upf-li-support
```

5. MR 查看并合并代码

```bash
git fetch origin
git checkout feature/upf-li-support
git diff dev..feature/upf-li-support   # 查看改动


git checkout dev
git pull origin dev
git merge --no-ff feature/upf-li-support -m "Merge feature/upf-li-support" # --no-ff：保留分支历史记录（推荐用于团队开发） -m：添加合并说明
git push origin dev
```

6. 清理分支

```bash
git branch -d feature/upf-li-support
git push origin --delete feature/upf-li-support
```