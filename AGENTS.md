# AGENTS.md

本文件面向 AI 编码代理，描述本项目的架构、构建方式与开发约定。

## 项目概述

本项目是一组 **Maven 父 POM**（parent pom），groupId 为 `io.github.dbstarll.parent`，当前版本 `1.4.0-SNAPSHOT`。它不包含任何 Java 源代码，所有模块的 `<packaging>` 均为 `pom`。其用途是作为其他 Maven 项目的 `<parent>`，通过继承复用预定义的依赖版本、插件配置、Profile 和发布流程。

所有真实项目约定（预定义属性、依赖、插件、Profile 的完整表格）记录在 `README.md` 中，修改 pom 后应同步更新 README。

## 模块结构与继承关系

仓库根 `pom.xml` 聚合了 7 个模块，模块之间的**继承链**（注意：不是平级的兄弟模块）如下：

```
pure → parent
base → pure
docker → base
assembly → docker
mode → assembly
obscure → mode
boot → docker        （注意：boot 继承自 docker，与 assembly 分支平行）
```

各模块职责：

| 模块 | 职责 |
| --- | --- |
| `parent`（根） | 继承根。编码设置（UTF-8）、git 仓库相关属性（`project.git.*`）、基础插件版本（enforcer/release/gpg/site 等）、发布 Profile |
| `pure` | 纯 Java 父项目。Java 编译级别（默认 8，属性 `project.java.version`，必须为纯数字格式）、按构建 JDK 分组的编译参数 Profile（jdk8 用 source/target，jdk9+ 用 release，jdk23+ 追加 proc=full）、打包插件、测试相关 Profile（JUnit 5 + JaCoCo）。代码质量/安全报告不在 pom 中预定义，统一由外部 `org.sonarsource.scanner.maven:sonar-maven-plugin:sonar` 生成 |
| `base` | 引入基准依赖：slf4j（api + jul/jcl/log4j 桥接，2.0.6）和 Apache Commons（lang3/io/codec/beanutils/collections4） |
| `docker` | 存在 `Dockerfile` 时激活 `docker` Profile，用 git-commit-id-plugin 生成 git.properties、用 dockerfile-maven-plugin 构建 Docker 镜像 |
| `assembly` | 存在 `src/main/assembly` 时激活，用 maven-assembly-plugin 打 `jar-with-dependencies` 可执行 jar，主类由 `project.mainClass` 属性指定（子项目必须设置） |
| `mode` | 存在 `src/main/mode` 时激活，按运行模式（`project.mode`，默认 `dev`）用 `src/main/mode/${project.mode}.properties` 做资源过滤 |
| `obscure` | 用 proguard-maven-plugin 做代码混淆；根据 `src/main/assembly`、`src/main/webapp` 是否存在自动切换混淆策略（工具 jar / 可执行 jar / Web 项目跳过） |
| `boot` | 存在 `src/main/java` 时激活 `spring-boot` Profile，用 spring-boot-maven-plugin（2.7.7）repackage 可执行 jar |

## 构建与测试

- 构建要求：Maven ≥ 3.6.3（enforcer 插件强制），JDK 8 及以上（默认编译级别 8；JDK 8 构建用 `maven.compiler.source`/`target`，JDK 9+ 用 `maven.compiler.release`，JDK 23+ 追加 `maven.compiler.proc=full`，由 `jdk8`/`jdk9+`/`jdk23+` 三个 profile 按 JDK 自动切换）。
- 全量构建：`mvn clean install`
- 校验：`mvn verify`
- 本项目自身没有 Java 代码和单元测试；「测试」本质上是验证各 pom 能正确解析。常用检查：
  - `mvn validate` 或 `mvn clean install` 确认所有 pom 无语法/继承错误
  - `mvn help:effective-pom` 查看某个模块的有效 POM
  - `mvn versions:display-plugin-updates` / `mvn versions:display-dependency-updates` 检查版本更新（versions-maven-plugin 配置了排除 beta 的规则，指向 `https://dbstarll.github.io/exclude-beta-rule.xml`）
- Profile 大多按**文件/目录存在性自动激活**（如 `src/main/java`、`src/test/java`、`Dockerfile`、`src/main/assembly`），本仓库各模块没有这些目录，因此这些 Profile 在本仓库构建时不会触发，只在下游子项目中生效。

## 发布与部署

- 发布使用 maven-release-plugin，命令：`mvn release:prepare -P release`（`release` Profile 启用 GPG 签名）。tag 格式为 `v@{project.version}`，子模块自动统一版本（`autoVersionSubmodules=true`）。
- 部署目标由 Profile 选择：
  - `distribution-ossrh`：发布到 Maven 中央仓库（Sonatype OSSRH，`s01.oss.sonatype.org`，autoReleaseAfterClose=true）
  - `distribution-github`：发布到 GitHub Packages（`maven.pkg.github.com/dbstar-org/parent`）
  - `site-local`：在本地 `~/.m2/sites` 生成站点报告
- CI 在 `.github/workflows/`：
  - `maven.yml`：push 到任意分支时触发（忽略 README 和 release.yml 的变更），复用 `dbstar-org/general/.github/workflows/maven.yml@v1` 做 verify + deploy。
  - `release.yml`：手动触发（workflow_dispatch），复用 `dbstar-org/general/.github/workflows/maven-release.yml@v1` 做发布。

## 代码风格与约定

- 文档与注释主要使用**中文**（README 全中文），提交信息使用英文。
- pom.xml 使用 2 空格缩进，UTF-8 编码。
- 版本号一律通过 `<properties>` 中的 `version.*` 属性集中管理，属性命名规则：`version.<artifactId>`（如 `version.maven-compiler-plugin`、`version.slf4j`）。修改版本时改属性，不要在插件/依赖声明里硬编码版本。
- 插件版本放 `<pluginManagement>`，依赖版本放 `<dependencyManagement>`；只有需要默认生效的插件/依赖才放进 `<plugins>`/`<dependencies>`。
- 可被下游覆盖的定制项以 `project.*` 属性暴露（如 `project.java.version`、`project.mainClass`、`project.mode`、`project.docker.image.*`）。
- 仓库根和各模块目录下存在 `pom.xml.versionsBackup`，是 versions-maven-plugin 运行后的备份文件，不要手工编辑，也不要提交其改动。
- 下游子项目直接继承这些父 POM 时，必须覆盖 `<description>`、`<url>`、`<scm>` 和 `site-local` Profile 的 `<site>` 配置，否则 URL 会错误叠加（详见 README「代码仓库相关配置」一节）。

## 测试策略

- 本仓库自身无测试代码。
- 预定义的测试体系（在下游子项目中生效）：JUnit 5（junit-jupiter-engine 5.9.2）+ maven-surefire-plugin + JaCoCo 覆盖率（0.8.8），由 `java-test` Profile（存在 `src/test/java` 目录时）激活。
- 代码质量/安全报告不在 pom 中预定义：findbugs/pmd/checkstyle/jdepend/taglist/jxr/changelog 等老旧报告插件已于 1.4.0 移除，统一改用外部 Sonar 扫描（`mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`）；`java-main` Profile 的站点报告仅保留 javadoc。

## 安全注意事项

- 发布需要 GPG 密钥（`release` Profile 签名）和 OSSRH / GitHub Packages 凭证（本机 `~/.m2/settings.xml` 中的 `ossrh` / `github` server 配置），CI 通过 `secrets: inherit` 注入。不要在 pom 或仓库中写入任何凭证。
- `base` 模块中对 slf4j 桥接包排除了 `slf4j-api` 传递依赖、对 commons-beanutils 排除了 commons-logging，修改这些 exclusion 会影响所有下游项目的日志体系，需谨慎。
- 本项目的任何改动都会被所有下游继承项目感知，修改共享配置（插件版本、依赖版本、Profile 激活条件）前应评估对下游的兼容性影响。
