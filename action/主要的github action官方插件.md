# 主要的github action官方插件

## 核心和常用的，不按类型区分
checkout

cache

container-toolkit-action：制作镜像并推送(这个需要重点关照)

## github操作、workflow操作
[add-to-project]([https://](https://github.com/actions/add-to-project))

[create-github-app-token]([https://](https://github.com/actions/create-github-app-token))

reusable-workflows：可以在workflow工作过程中调用其他workflow

### 代码仓体验和自动化
stale：可以自动巡检issue，可以选择关闭或者清理

labeler：可以自动对pr打标签

first-interaction：新贡献者初次提交时，会触发的动作。可以改善新人体验

action-versions：用于自动检查并更新你的工作流文件中使用的其他 Action 的版本。

delete-package-versions：自动删除产物包版本

## 执行机操作
[runner]([https://](https://github.com/actions/runner))

runner-container-hooks：可以在执行启动和清理容器前后，做一些动作

actions-runner-controller：用于在k8s环境上部署、管理和自动缩放自托管 GitHub Actions Runner的控制器。

### 容器相关
runner-images

runner-images-check：镜像检查

runner-images-temp：提供执行机镜像版本

container-prebuilt-action：容器相关，作用未知

## 产物管理
[upload-pages-artifact]([https://](https://github.com/actions/upload-pages-artifact))

[actions/versions-package-tools]([https://](https://github.com/actions/versions-package-tools))：可以维护其他actin的版本号

download-artifact

upload-artifact

### 依赖分析
attest-sbom：生成并签署软件物料清单（SBOM）

component-detection-dependency-submission-action：依赖检测工具基础依赖

maven-dependency-submission-action：maven依赖检测基础依赖

dependency-review-action：超期依赖门禁基础依赖

[attest]([https://](https://github.com/actions/attest))：生成sbom

[attest-build-provenance]([https://](https://github.com/actions/attest-build-provenance))：生成sbom

go-dependency-submission ：专门给go语言制作的依赖分析

## 语言类(部分action由三方维护)
setup-dotnet

setup-node

setup-java

setup-go

setup-python

node-versions：负责管理和提供node版本

python-versions：负责管理和提供python版本

go-versions：版本管理

gradle-build-tools-actions：gradle构建工具辅助

## github pages
jekyll-build-pages、configure-pages：github pages的前置相关插件

deploy-pages：发布github page

## 工具类
toolkit

javascript-action：action的基础依赖

container-action：容器action的基础依赖

typescript-action：ts的action的基础依赖

publish-action：用于自动化和发布action

languageservices：开发工具包，提供actions语法解析

publish-immutable-action：发布一个不可变的action

gh-actions-cache：github cache的一个ctl管理工具

github-script：允许直接在普通action中执行script脚本

### cicd迁移工具
importer-issue-ops：基于issue-ops，提供其他cicd的自动化迁移

importer-labs：自动迁移cicd基础类

### 示例
alpine_nodejs：从构建到测试到release的一个示例

starter-workflows：包含大量的示例

hello-world-docker-action：如何插件docekr容器action的示例

hello-world-javascript-action：一个javascript-action使用的简单事例

### AI接口调用
[ai-inference]([https://](https://github.com/actions/ai-inference))：可以调用ai接口，支持promot文件等AI接口基本调用

## 一些不太重要的action
itgmania212121：ITGmania（一个 StepMania 音游模拟器的分支）项目相关

opensearch-pyd：与 OpenSearch（AWS 开源的搜索引擎）相关的 Action

humans.txt：列示网站或项目背后贡献者的文件

actions-sync：在不同的github实例(GitHub.com、GitHub Enterprise Server)同步action

partner-runner-images：与第三方镜像提供相关

runner-images-sangeeth：与第三方镜像提供相关

runner-images-PK1：与第三方镜像提供相关

runner-images-PK：与第三方镜像提供相关

buildtypes

.github

anno-test


