

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





# 2. 分支管理及命名规范





## 2.1. 分支分类



一般情况下，Git 分支可以分成 主分支 和 临时分支。

- 主分支一般为 Master 和 Develop，前者用于正式发布，后者用于日常开发。

- 临时分支也叫辅助分支，临时性分支用于应对一些特定目的的版本开发。临时分支一般分为三种，feature、release 和 fixbug。

以上三种临时分支都属于临时性需要，使用完以后，应该删除，使得代码库的常设分支始终只有 Master 和 Develop。



### 2.1.1. 开发分支

开发分支，顾名思义就是开发使用的分支，这个分支可以用来生成代码的最新隔夜版本（nightly）。如果想要正式对外发布，就在 Master 分支上，对 Develop 分支进行合并。

- 创建分支

```bash
git branch develop

git push -u origin develop
```



- 拉取 feature

```bash
git checkout -b feature/add-new-feature develop

git push -u origin feature
```



- 合并 feature

```bash
git pull origin develop

git checkout develop

git merge --no-ff feature/add-new-feature

git push origin develop
```



### 2.1.2. 预发布分支

预发布分支，是指发布正式版本之前（即合并到 Master 分支之前），我们可能需要一个预发布的版本进行测试。预发布分支是从 Develop 分支上面分出来的，预发布结束之后，必须合并进 Develop 和 Master 分支。它的命名可以采用 release-* 的形式

- 创建分支

```bash
git checkout -b release-1.2.1 develop
```



- 合并分支

```bash
# 合并到 master 分支
git checkout master

git merge --no-ff release-1.2.1

git tag -a 1.2.1

# 合并到 develop 分支
git checkout develop

git merge --no-ff release-1.2.1
```



- 删除分支

```bash
git branch -d release-1.2.1
```



