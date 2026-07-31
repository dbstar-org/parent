# AGENTS.md

## 重要禁令

- **未经用户明确许可，绝对禁止执行 `git commit`、`git push` 或任何会改写 git 历史、影响远程仓库的操作。**
- 所有代码提交、推送、发布、tag 打取、分支回滚等必须由用户自己完成，或在用户逐条明确授权后执行。
- 不要因为"提交一次""先提交"等口语化指令就自动提交，必须等待用户明确的提交确认。

本文件面向 AI 编码代理，描述本项目的架构、构建方式与开发约定。

## 项目概述

本项目是一组 **Maven 父 POM**（parent pom），groupId 为 `io.github.dbstarll.parent`，当前版本 `1.4.1-SNAPSHOT`。它不包含任何 Java 源代码，所有模块的 `<packaging>` 均为 `pom`。其用途是作为其他 Maven 项目的 `<parent>`，通过继承复用预定义的依赖版本、插件配置、Profile 和发布流程。

**分工**：`README.md` 是面向下游使用者的**权威配置参考**（全部版本号、属性默认值、依赖/插件/Profile 清单以此为准）；本文件是面向 AI 代理的**操作指南**（构建/发布命令、开发约定、实测经验与坑），不重复记录版本数字。修改 pom 后同步更新 README。

## 模块结构与继承关系

仓库根 `pom.xml` 聚合了 4 个模块，模块之间的**继承链**（注意：不是平级的兄弟模块）如下：

```
pure → parent
base → pure
boot → base
native → boot
```

各模块职责（一句话版；版本号、属性默认值、激活条件的权威清单见 README 表格，此处不重复）：

| 模块 | 职责 |
| --- | --- |
| `parent`（根） | 继承根：编码、git 仓库属性（`project.git.*`）、基础插件版本、发布 Profile |
| `pure` | 纯 Java 父项目：编译级别与按构建 JDK 分组的编译参数 Profile（jdk8/jdk9+/jdk23+）、打包插件、测试插件（surefire + JaCoCo） |
| `base` | 基准依赖（slf4j + 桥接、Apache Commons、Lombok provided）、JUnit 5 测试环境（java-test Profile）、git-commit-id-plugin（java-main Profile 生成 git.properties） |
| `boot` | spring-boot 可执行 jar：**主干** dependencyManagement 导入 junit-bom（前置占位）+ spring-boot-dependencies BOM 做依赖管理（junit/slf4j 有属性钉版覆盖，见「安全注意事项」）；`spring-boot` Profile（file 激活）只做 spring-boot-maven-plugin repackage |
| `native` | GraalVM native image 构建支撑：`native-build` Profile（Spring AOT + native-maven-plugin 编译骨架，通用 buildArg 固化、项目特定 buildArg 由下游同名 Profile 合并追加）、`native-trace` Profile（surefire 挂 Tracing Agent 采集反射元数据）；`native-trace` 产出目录默认经 `project.native.agent.config.dir` 属性作为 `native-build` 的 `-H:ConfigurationFileDirectories` 输入（缺失目录被静默忽略）；**不动版本基线**，spring-boot/java 等基线由下游项目自定 |

## 构建与测试

- 构建要求：Maven ≥ 3.6.3（enforcer 插件强制），JDK 8 及以上（默认编译级别 8；jdk8/jdk9+/jdk23+ 三个 profile 按构建 JDK 自动切换编译参数，详见 README 配置项表）。
- 全量构建：`mvn clean install`
- 校验：`mvn verify`
- 本项目自身没有 Java 代码和单元测试；「测试」本质上是验证各 pom 能正确解析。常用检查：
  - `mvn validate` 或 `mvn clean install` 确认所有 pom 无语法/继承错误
  - `mvn help:effective-pom` 查看某个模块的有效 POM
  - `mvn versions:display-plugin-updates` / `mvn versions:display-dependency-updates` 检查版本更新（versions-maven-plugin 内联了 `ruleSet`，正则排除 rc/beta/alpha 预发布版，含 `beta-2` 连字符形式；`-M` 里程碑版不排除，扫描结果中需人工甄别）
- Profile 大多按**文件/目录存在性自动激活**（如 `src/main/java`、`src/test/java`），本仓库各模块没有这些目录，因此这些 Profile 在本仓库构建时不会触发，只在下游子项目中生效。

## 发布与部署

- 发布使用 maven-release-plugin，命令：`mvn release:prepare -P release`（`release` Profile 启用 GPG 签名）。tag 格式为 `v@{project.version}`，子模块自动统一版本（`autoVersionSubmodules=true`）。
- 部署目标由 Profile 选择：
  - `distribution-central`：发布到 Maven 中央仓库（Central Portal，`central-publishing-maven-plugin`，server id `central`，autoPublish + waitUntil=published；该插件要求 Maven ≥ 3.9.2，仅在发布时激活）
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
- 可被下游覆盖的定制项以 `project.*` 属性暴露（如 `project.java.version`、`project.mainClass`）。
- 仓库根和各模块目录下存在 `pom.xml.versionsBackup`，是 versions-maven-plugin 运行后的备份文件，不要手工编辑，也不要提交其改动。
- 下游子项目直接继承这些父 POM 时，必须覆盖 `<description>`、`<url>`、`<scm>` 和 `site-local` Profile 的 `<site>` 配置，否则 URL 会错误叠加（详见 README「代码仓库相关配置」一节）。

## 测试策略

- 本仓库自身无测试代码。
- 预定义的测试体系（在下游子项目中生效）：测试插件由 `pure` 的 `java-test` Profile（存在 `src/test/java` 目录时）激活，JUnit 5 依赖由 `base` 的同名 Profile 提供（junit-bom 统一版本 + junit-jupiter 聚合构件作为默认测试环境）；版本号见 README 表格。
- 代码质量/安全报告不在 pom 中预定义：老旧报告插件（findbugs/pmd/checkstyle 等）已于 1.4.0 移除，统一改用外部 Sonar 扫描（`mvn org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`）。

## 安全注意事项

- 发布需要 GPG 密钥（`release` Profile 签名）和 Central Portal / GitHub Packages 凭证（本机 `~/.m2/settings.xml` 中的 `central` / `github` server 配置；Central 使用 Portal 生成的 User Token），CI 通过 `secrets: inherit` 注入。不要在 pom 或仓库中写入任何凭证。
- `base` 模块中对 slf4j 桥接包排除了 `slf4j-api` 传递依赖、对 commons-beanutils 排除了 commons-logging，修改这些 exclusion 会影响所有下游项目的日志体系，需谨慎。
- 依赖版本接管优先级（已实测验证）：**父 pom 直接 dependencyManagement 钉版 > 子 pom 导入的 BOM > 父 pom 导入的 BOM**；同一 pom 内多个 BOM 按声明顺序先声明者胜。因此 boot 导入 spring-boot-dependencies 后，base 直接钉版的构件（slf4j、commons-lang3、commons-codec 等）仍胜 BOM；boot 通过覆盖 `version.slf4j` 属性钉回 spring 适配版本，是利用「属性在子项目上下文中插值」的机制，修改 base 的 `version.*` 属性名时需检查 boot 是否有同名覆盖。
- **抢在 spring-boot 发版前单独升级子项（如 CVE 修复）的通用方法**：在 boot 主干的 dependencyManagement 中**直接声明**该构件的新版本（配 `version.*` 属性）——直接声明胜过一切 BOM；有自家 BOM 的组件族（如 junit）可在 spring-boot-dependencies 之前前置导入对应 BOM（同一 pom 多 BOM 先声明者胜）。注意：覆盖 spring-boot-dependencies 的内部属性（如 `logback.version`）无效，BOM 内属性在 BOM 自己的上下文中插值。
- **file 激活的 Profile 不得放置影响下游消费的 dependencyManagement**（教训：boot 曾把 BOM import 放在按 `src/main/java` 激活的 `spring-boot` Profile，下游从仓库以部署 pom 消费构件时 Profile 不求值 → 消费方报 invalid POM、传递依赖丢失，1.4.1 已修复移到主干）。判定标准：Profile 的 dependencyManagement 只能管理该 Profile 内自声明的依赖（`base` 的 `java-test` 自闭包模式是正面范例）；下游无条件声明的依赖，其版本管理必须放主干。
- 本项目的任何改动都会被所有下游继承项目感知，修改共享配置（插件版本、依赖版本、Profile 激活条件）前应评估对下游的兼容性影响。
- surefire 的 `<argLine>` 约定（`native-trace` 等 Profile 遵守）：所有 `<argLine>` 必须以 `@{argLine}` 开头（晚期属性替换，保留 JaCoCo prepare-agent 写入的覆盖率 agent）；对应禁令——公共属性 `argLine` 严禁经 settings profile / pom `<properties>` / `-DargLine` 注入，会与 JaCoCo 争抢或被显式 `<argLine>` 静默覆盖（实测 fork JVM 启动失败）。
