---
layout: title
title: gitlab ci/cd与docker构建
date: 2021-08-17 12:35:09
tags: [运维]
categories: 技术
---

## 背景
目前BT是没有「运维」这一专门的职能岗存在的，这是由两个因素共同决定的：

1. 第三方提供的云服务越来越完善，传统的运维工作已经大量被替代(比如我们不需要自己搭集群，不需要维护自己的机房)
2. 我们从一开始推崇的是devOps，开发即运维，底层体系搭建好之后，通过大部分自动化+少部分开发的手动操作，由开发完成之前需要运维去做的事情。(这不代表我们不需要运维，只不过需求比较弱)

没有运维不代表我们没有运维的工作，我们目前的运维工作主要是由华仔、宇智和我共同在完成，当然我们也是站在了前人的肩膀上，才能完成这个工作。

至于为什么做这一个分享，有以下两个原因：

1. 理解运维的工作，知道我们这一套系统背后是怎么运行的，在一定程度上大家的日常工作会有帮助。比如为什么同样一个接口，在测试环境正常，走正式环境的时候却会出错。
2. 随着业务的发展，我们还需要不断更新、完善我们的运维工具，这部分仅靠我们三个人是不够的。所以希望大家了解到我们的运维工作之后，有人会对它感兴趣并主动尝试。

## 不包含什么
目前打算总共做三次分享，这一系列的分享，不包含docker的底层原理和基础知识，也不会包含k8s的基础知识与搭建，同样的也不会包含HTTP协议的内容、网络传输的基础知识。不是因为这些不重要，只是这些都有很完善的学习资料，相信大家都有自己去学习、了解的能力。另外很重要的一点，就是又很多我也不够了解，我没办法去做分享，大家如果对某一块感兴趣的话，后面可以一起学习。

所以，我的这次分享，更多的像是我们运维系统的使用手册，希望你们能了解到这个系统有什么，应该怎么去使用它，最后怎么去完善它。

## 分享提纲

1. 前言
   1. 开发、运维模式的总览
   2. CI/CD，从提交代码到打包到上线
   3. 从DockerFile看Docker
2. 从 镜像 到 提供服务
   1. k8s 简述（configMap / deployment / svc / ingress）
   2. 一个请求，从发起到响应，经过哪些环节，出现问题我们应该怎么定位
3. 我们目前的运维工具
   1. prometheus + grafana 的简单展示 
   2. splunk 能怎么用
   3. 阿里云提供的数据看板(NAT | SLB | RDS)等等
   4. sentry
   5. bt-api
   6. 我们还能做什么

## 正文
### 开发、运维模式的总览
![](http://cdn.jsblog.site/16690915015062.jpg)

### CI/CD，从提交代码到打包到上线
> Git的开发上线流程就不讲了，新人手册里有收录([快去看看](https://www.yuque.com/bttp/ogln0t/gdrfug))
> Gitlab CI/CD文档：[自己看](https://docs.gitlab.com/12.10/ee/ci/index.html)

以whale的CI文件为例：

```yaml
image: registry-vpc.cn-shenzhen.aliyuncs.com/openbt/docker:stable
stages:
  - build
  - deploy
  - job_failure
  - job_success

variables:
  BRANCH_VERSION: ${CI_COMMIT_REF_NAME}
  IMAGE_NAME: "${REGISTRY_URL}/btclass/whale:"
  KUBECONFIG: /etc/deploy/config
  CACHE: ""
  PROJECT_ENV: test
```

**image:** CI runner会在指定的image启动的container里执行，上面使用docker镜像，是因为我们这个CI需要在docker环境中运行。这里容易造成误导，可以看下[web-link](https://gitlab.bt-tp.group/lib/web/bt-web-link/-/blob/master/.gitlab-ci.yml)里的CI配置。

**stages:** CI的阶段，每个阶段之间互相独立。上面例子分成了四个阶段，分别为构建、部署、失败和成功。

**variables:** 定义变量，这些变量在所有阶段里都可以使用。其中`${}`包裹的是预定义的CI变量，有些是gitlab CI预定义的，比如`${CI_COMMIT_REF_NAME}`是当前构建所在的branch或tag([更多变量请看](https://docs.gitlab.com/ee/ci/variables/predefined_variables.html#predefined-variables-reference))。有些是我们在gitlab后台手动配置的，比如`${REGISTRY_URL}`是镜像仓库地址。

```yaml
before_script:
  - if [ "$CI_COMMIT_TAG" != "" ]; then PROJECT_ENV=production; fi
  - VERSION=${BRANCH_VERSION//./-}
  - echo "before change image name " $IMAGE_NAME
  - if [ "$CI_COMMIT_TAG" = "" ]; then IMAGE_NAME=${IMAGE_NAME}test-${VERSION}; else IMAGE_NAME=${IMAGE_NAME}v${CI_COMMIT_TAG:3}; fi
  - CACHE="--cache-from $IMAGE_NAME"
  - echo $IMAGE_NAME $CACHE $PROJECT_ENV
```
**before_script:** 在所有job运行之前执行的命令，这一段没什么特别难理解的，都是些基本的shell语法

```yaml
build_image:
  stage: build
  services:
    - registry-vpc.cn-shenzhen.aliyuncs.com/openbt/docker:18-dind
  variables:
    DOCKER_DRIVER: overlay
    DOCKER_HOST: tcp://localhost:2375
  only:
    - tags
    - /^v{1}\d+([.]\d+){2}$/
    - playground
  tags:
    - k8s-runner
  script:
    - docker login -u $REGISTRY_USERNAME -p $REGISTRY_PASSWORD $REGISTRY_URL
    - if [ "$PROJECT_ENV" = "test" ]; then docker pull $IMAGE_NAME || true; fi
    - docker build $CACHE --build-arg NPM_AUTH="$NPM_AUTH" --build-arg WEB_ENV="$PROJECT_ENV" -t $IMAGE_NAME .
    - docker push $IMAGE_NAME
    - docker rmi $IMAGE_NAME
```
**build_image:** 该key值为job的名字，下面为该job的内容

**stage:** 所属阶段，这里说明该job会在build阶段执行

**services:** 用docker镜像启动对应的服务，在当前CI中可以访问该服务。举个比较简单的例子，假如在你的CI脚本中，需要创建一个mysql，用于执行测试用例，services需要加一个mysql，在当前CI环境中可以直接连接该数据库进行操作。
它跟前面的image的区别是，image是当前CI所在容器的镜像，services是另外开一个容器，当前CI容器可以访问该容器。
dind全称是docker in docker，简单理解，`image: docker`运行了一个docker客户端，dind服务则允许连接至宿主的docker server。

**variables:** 当前job内的变量，这里两个变量都是为了连接docker服务。

**only:** 符合条件的时候，才执行该job。这里其实是`only: refs`的缩写，即当前是通过打标签、或是playground分支、或是v2.2.2这样的版本分支，才会执行该job。[相关文档](https://docs.gitlab.com/ee/ci/yaml/#only--except)

**tags:** gitlab支持多个runner，这里就是指定拥有`k8s-runner`标签的runner执行当前job。这里一个比较常见的操作，就是runner分正式环境和测试环境，以将构建后的容器部署到对应的环境。

**script:** 当前job执行的内容，同样是shell语法。简单描述下，这里做了几件事情

1. docker登录到我们的私有仓库
2. 判断是否测试环境，测试环境的话，拉取当前的镜像，这一步是为了利用docker的缓存机制，提高打包效率，具体的在后面的docker打包再讲。
3. docker构建镜像
4. 推送镜像到仓库
5. 移除本地镜像(这里是为了避免宿主机镜像越来越多，这句是否有意义需要考究)。

```yaml
deploy_dev_version:
  image: registry-vpc.cn-shenzhen.aliyuncs.com/openbt/kubectl:latest
  stage: deploy
  tags:
    - k8s-runner
  only:
    - /^v{1}\d+([.]\d+){2}$/
  except:
    - tags
  script:
    - mkdir -p /etc/deploy
    - echo $kube_config | base64 -d > $KUBECONFIG
    - kubectl config use-context bt-test
    - sed -i "s#EGG_SERVER_ENV_VALUE#test#g" deployment.yaml
    - sed -i "s#IMAGE_NAME#$IMAGE_NAME#g" deployment.yaml
    - sed -i "s#CI_LAST_UPDATED#$(date +'%s')#g" deployment.yaml
    - sed -i "s#BRANCH_VERSION#$VERSION-whale#g" deployment.yaml
    - cat deployment.yaml
    - kubectl apply -f deployment.yaml

deploy_test:
  image: registry-vpc.cn-shenzhen.aliyuncs.com/openbt/kubectl:latest
  stage: deploy
  tags:
    - k8s-runner
  only:
    - playground
  script:
    - mkdir -p /etc/deploy
    - echo $kube_config | base64 -d > $KUBECONFIG
    - kubectl config use-context bt-test
    - sed -i "s#EGG_SERVER_ENV_VALUE#test#g" deployment.yaml
    - sed -i "s#IMAGE_NAME#$IMAGE_NAME#g" deployment.yaml
    - sed -i "s#CI_LAST_UPDATED#$(date +'%s')#g" deployment.yaml
    - sed -i "s#BRANCH_VERSION#whale#g" deployment.yaml
    - cat deployment.yaml
    - kubectl apply -f deployment.yaml
```
**deploy_dev_version和deploy_test:** 另外两个job名

**image:** kubectl是k8s的客户端

**stage和only:** 这两个job都是在deploy这个阶段执行的，但是通过`only`做了区分，一个只在版本分支如v2.2.2会执行，另一个在playground分支执行

**except:** 与only相反，符合该条件的话，不执行该job。这里是只有分支名为v2.2.2是才执行，当标签名为v2.2.2是，不执行。

**script:** 这一段主要是做了k8s的部署动作，前三句是设置了k8s的配置信息，使kubectl能正确访问我们的测试k8s集群。中间四句sed，是对deployment做了内容的替换，最后通过kubectl apply更新。(关于k8s的介绍，在后续讲)

```yaml
job_failure:
  stage: job_failure
  image: registry-vpc.cn-shenzhen.aliyuncs.com/openbt/curl:latest
  tags:
    - k8s-runner
  only:
    - playground
    - /^v{1}\d+([.]\d+){2}$/
    - tags
  script:
    - title="项目：[${CI_PROJECT_NAME}](${CI_PROJECT_URL})"
    - text="### ${title}\n#### 运行分支：${CI_PROJECT_NAMESPACE}/${CI_COMMIT_REF_NAME}\n#### 运行状态：失败！\n#### 提交者：${GITLAB_USER_NAME}\n#### gitlab版本号：${CI_SERVER_VERSION}\n#### 提交信息：${CI_COMMIT_MESSAGE}  \n\n\n##### [流水线 Pipeline#${CI_PIPELINE_ID}](${CI_PROJECT_URL}/pipelines/${CI_PIPELINE_ID}) \n"
    - curl POST "$DING_TOKEN_URL" -H "Content-Type:application/json" -d "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"$title\",\"text\":\"$text\"}}"
  when: on_failure

job_success:
  stage: job_success
  image: registry-vpc.cn-shenzhen.aliyuncs.com/openbt/curl:latest
  tags:
    - k8s-runner
  only:
    - playground
    - /^v{1}\d+([.]\d+){2}$/
    - tags
  dependencies:
    - deploy_test
  script:
    - title="项目：[${CI_PROJECT_NAME}](${CI_PROJECT_URL})"
    - text="### ${title}\n#### 运行分支：${CI_PROJECT_NAMESPACE}/${CI_COMMIT_REF_NAME}\n#### 运行状态：部署成功！\n#### 提交者：${GITLAB_USER_NAME}\n#### gitlab版本号：${CI_SERVER_VERSION}\n#### 提交信息：${CI_COMMIT_MESSAGE}  \n\n\n##### [流水线 Pipeline#${CI_PIPELINE_ID}](${CI_PROJECT_URL}/pipelines/${CI_PIPELINE_ID}) \n"
    - curl POST "$DING_TOKEN_URL" -H "Content-Type:application/json" -d "{\"msgtype\":\"markdown\",\"markdown\":{\"title\":\"$title\",\"text\":\"$text\"}}"
  when: on_success
```

**when:** 什么时候执行这个job，on_failure为只要其中一个为执行失败就会执行，on_success为前面所有job都成功了才会执行。[相关文档](https://docs.gitlab.com/ee/ci/yaml/#when)

**dependencies:** 这里翻阅文档后，感觉这里是不需要这一项的，之前我们应该对这一项有误解。关于这一项的配置，我们直接看[文档](https://docs.gitlab.com/ee/ci/yaml/#dependencies)就比较清晰了，其和`artifacts`是搭配使用的，`artifacts`指定job的产物，`dependencies`则指定当前job基于前面哪个job的产物运行。

---

我们目前关于CI/CD用到的配置就这些，实际上看Gitlab CI的文档就知道，它还有很多其他的能力，大家可以发挥自己的想象力，想想我们有哪些动作是可以被自动化的。随便举个例子，git commit msg里如果带上tapd任务的地址，在CI里可以做到提交自动关闭任务(msg带tapd地址，有不少好处，之前有强调过，但是目前基本没人会这么操作)。
### 从DockerFile看Docker
> 前面CI里面有很重要的一步，`docker build`是我们目前自动化的核心，大部分项目经过自动化构建后，得到的都是一个个的docker镜像，我们通过这一个切入口来了解docker。
> docker的文档资料我是看的[【Docker 从入门到实践】](https://yeasy.gitbook.io/docker_practice/)，如前面所说，基础内容还是要靠自己看，搬运过来的东西是主观加工的

#### 什么是Docker 以及 为什么需要Docker
感觉很难用一句话描述什么是Docker，从使用角度可以简单地将Docker理解成轻量级的虚拟机，但从实现角度这种理解是不太正确的。它跟虚拟机有本质上的区别，比如没办法在linux Docker上跑windows操作系统，在Docker容器内查看当前操作系统内核，会发现是宿主机的内核，也就是它没有自己的内核。

我现在是这么理解Docker的，Docker容器是一组资源隔离的进程，这里的资源包括根目录、网络栈、挂载点、进程ID、用户和组ID等等。它比虚拟机更快，是因为它不需要做硬件虚拟也不需要运行完整的操作系统。

为什么需要使用Docker：

1. 拥有一致的运行环境。无论是开发环境、测试环境、生产环境，都可以保证运行环境一致。(这在我这运行正常，为什么上线就报错了)
2. 通过DockerFile，可以实现将运行环境也加入到代码的版本管理中。(这个项目1.0版本是运行在php5.6版本的，2.0版本是运行在php5.7版本上的)
3. 大幅降低运维成本，项目的上线和部署简单很多。

#### 基本概念
镜像：将目标的程序 以及 运行所依赖的底层资源、库打包到一起，就是镜像。比如现在想要打包一个node服务镜像，除了我们自己实现的业务代码，还需要将Node.js程序、libuv库等等打包到里面。镜像叫打包不太贴切，应该称为构建，因为镜像实际上是一层一层存储的而不是一个独立的打包文件，这个后面会讲到。

容器：有点像程序与进程的关系，镜像运行起来就是一个容器。镜像是一层一层存储的，容器会在镜像上面再建一层作为当前容器的存储层，所以在容器内我们可以创建、修改文件，但一旦容器消亡，这个存储层就会销毁。所以容器是无状态的。

#### 从Dockerfile了解Docker
> 知道什么是镜像，什么是容器，基本就可以开始使用Docker了，无非就是把代码构建成镜像，运行起来。

Docker镜像有两种来源，一种是从Docker仓库里获取别人构建好的镜像，像`mysql、nginx、wordpress`等等比较出名的软件都有现成的镜像了，另一种就是自己构建了。
自己构建镜像有两种方式：
一种是`docker commit`，刚才说了，容器会在镜像上面再建一层临时的存储层，`docker commit`就是将该存储层保存下来作为新的镜像。但我们实际构建镜像并不会用这方式，一个是构建出来的镜像可能会很臃肿，一个是无法追踪到这个镜像是怎么生成的。
第二种就是用`Dockerfile`进行镜像定制。

我们从whale项目的Dockerfile为例：

```dockerfile
FROM registry-vpc.cn-shenzhen.aliyuncs.com/openbt/node:v12.18.3-alpine
```
FROM：FROM是Dockerfile的必备指令，而且必须在第一句，作用是指定基础镜像。比如这里指定的是一个我们私有镜像仓库里的`node:v12.18.3-alpine`镜像，这里代表了这个基础镜像基于alpine系统安装了`node:v12.18.3`（[Dockerhub](https://hub.docker.com/_/node?tab=tags&page=1&ordering=last_updated&name=12.18.3)，[node-Dockerfile](https://github.com/nodejs/docker-node/blob/3047652162a4f83f68260aabfdbb688e58e7b152/12/alpine3.11/Dockerfile)，[alphine-Dockerfile](https://github.com/alpinelinux/docker-alpine/blob/48093276e3960aeccc4db369c11ea0460b90f349/x86_64/Dockerfile)）。
所有镜像通过基础镜像往前追，最后都会是`scratch`这个基础镜像，这是一个虚拟的空白镜像。

```dockerfile
ARG APP_PATH=/app/whale/
ARG WEB_PATH=/app/whale/app/web/
ARG NPM_AUTH
ARG WEB_ENV
```

ARG：`ARG`作用是定义变量，`A=xxx`代表了A变量的默认值是xxx，变量值可以在构建时通过--build-arg修改。
比较类似的还有一个`ENV`指令，用来指定环境变量。二者的区别是，ARG在容器中无法访问到，ENV在容器中可以访问。

```dockerfile
# egg npm install
COPY ./package.json ./.npmrc ${APP_PATH}
RUN echo "${NPM_AUTH}" >> ~/.npmrc && cd ${APP_PATH} && npm install

# web npm install
COPY ./app/web/package.json ./app/web/package-lock.json ./app/web/.npmrc ${WEB_PATH}
RUN cd ${WEB_PATH} && npm install

# web npm build
COPY ./ ${APP_PATH}
RUN cd ${WEB_PATH} \
    && npm run build -- --mode ${WEB_ENV} \
    && rm -rf ${WEB_PATH}
```
> 这里需要插入一个知识点，上面的FROM、ARG、COPY、RUN每一行都是一个指令，在构建镜像的时候，每一个指令都会构建镜像的一层，作为下一行指令的基础。而且前一层的所有文件都是固定的，即使在后一层将所有文件都删除了，最后镜像的大小并不会变小，只不过通过障眼法将文件隐藏起来而已。

![](http://cdn.jsblog.site/16690975225896.jpg)

COPY：从构建上下文目录中的文件复制到镜像的目标位置。比如上面第一句COPY是将package.json和.npmrc移到/app/whale目录下。类似的指令还有`ADD`，ADD支持源文件是一个URL，还会自动解压压缩包，比如上面的alpine。

RUN：RUN指令是用来执行命令行命令的，基本上我们所有的构建操作都需要用RUN来完成，比如上面的`npm install`和`npm run build`。

可以看到，RUN其实是可以一次性执行多句命令行命令的，但是在whale的Dockerfile里还是将整个构建过程拆成了3个COPY和3个RUN。根据我们前面说的，每一句指令都会生成一个镜像层，而且在后面层里删除文件也不会减小镜像的总大小，为什么我们还这么做。
主要的目的是为了缓存`npm install`这个比较耗时的操作，因为package.json很少会变动，在文件内容不变动的情况下，如果这一层执行的指令是一样的话，则不会重新构建，而是直接用之前的缓存。
这样拆分会带来另一个问题，就是前面的构建产物没办法删除，比如`npm run build`之后，就不再需要node_modules了，但这个时候删除并不会减小镜像的大小。([这有个解决方案，后面可以研究下可行性](https://yeasy.gitbook.io/docker_practice/buildx/buildkit#run-mount-type-cache))

```dockerfile
WORKDIR ${APP_PATH}

EXPOSE 7001

HEALTHCHECK --interval=30s --timeout=3s CMD [ "sh", "-c", "wget --spider localhost:7001/api/alive || exit 1" ]

CMD [ "sh", "-c", "npm start" ]
```
WORDIR：指定工作目录，后面所有相对路径都是以该工作目录为准。
EXPOSE：暴露端口声明，我们目前主要还是起一个声明作用。在用随机端口映射的时候，会自动随机映射该端口。
HEALTHCHECK：用于检查容器的健康状态，只会有一句生效，后一句会覆盖前一句。这里用wget alive接口来判断服务是否存活，间隔30秒判断一次，响应超过3秒就说明服务不健康。
CMD：容器运行的默认启动命令，比如这里就是执行npm start。这个启动命令运行起来的进程就是容器的主进程，当主进程退出容器就会退出。

#### 启动容器
> 构建完镜像，接下来只需要将镜像运行起来，就大功告成。

运行容器的命令是`docker run ubuntu`，加上`-it`参数可以在当前终端直接输入指令以及看到容器的输出，还可以加上`-d`参数让容器以守护态运行。

上面的运行命令没有指定启动命令，所以会运行镜像的CMD指令。我们还可以通过指定命令来覆盖CMD，比如`docker run whale sh -c "npm start"`。

为了让容器能对外提供网络服务，一般还需要指定端口`docker run -d -p 8888:7001 whale`，即本地的8888端口会映射至容器内的7001端口。

通过`-v`可以挂载本地文件或文件目录到容器内，比如`docker run -v /usr/data:/var/www/html whale`。

#### 总结
基本上了解以上的内容，就已经简单地使用docker了，但如果要深入还有不少进阶的内容需要去了解，比如容器之间怎么互联，docker compose、docker swarm、k8s这些docker集群的部署与使用。如果大家感兴趣，可以平时一起交流下。

## 其他资料

- [我是虚拟机内核我困惑？！](https://www.cnblogs.com/popsuper1982/p/8516543.html)
- [进程、容器和虚拟机之间的区别是什么](https://mp.weixin.qq.com/s/dVB3uNsiErn8p0RAvwSe2g)



