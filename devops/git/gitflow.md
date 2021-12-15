## Gitflow 分支模型

### 介绍

​	Gitflow是一种分支开发模型，来源于Vinvent Driessen的文章"A Successful Git Branching Model"

- 主分支
  - master
    - master分支的head始终保持具备生产部署的状态
  - develop
    - 包含所有最新的，可发布的变更，也叫集成分支
    - 当从develop发布到release分支时，变更需要合并的master分支
- 功能分支
  - feature
    - 可能基于develop创建
    - 必须合并回归到develop
- Supporting 分支
  - release
    - 可能基于develop创建
    - 必须合并回归到develop, master
    - 分支命名：release-*

- 修复分支
  - hotfix
    - 分支命名：hotfix-*

### 适用场景

- 软件本身对稳定性要求高
- 团队少量资深研发，多数初，中级研发
- 开源项目





## 主干开发分支模型

​	facebook团队使用

### 适用场景

- 软件本身迭代较快
- 团队中级，资深研发较多
- 互联网产品

