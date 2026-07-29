# TODO

本项目当前存在的问题清单，按优先级排序，逐一修正后勾选。

## 待办

- [ ] **迁移中央仓库发布到 Central Portal（本仓库改造已完成，剩人工准备工作）**
  - 已完成：`distribution-ossrh` 重写为 `distribution-central`（`central-publishing-maven-plugin` 0.11.0，server id `central`，autoPublish + waitUntil=published，含 SNAPSHOT 支持），nexus-staging 插件与属性已移除。注意该插件要求 Maven ≥ 3.9.2（仅发布时激活）。
  - 人工准备工作：
    1. 登录 [central.sonatype.com](https://central.sonatype.com)，Namespaces → Migrate Namespace 迁移 `io.github.dbstarll`；
    2. 命名空间菜单开启 "Enable SNAPSHOTs"；
    3. Account → Generate User Token；
    4. 本地 `~/.m2/settings.xml` 增加 id 为 `central` 的 server（token 用户名/密码）；
    5. GitHub 仓库 secrets 新增 Central token（替换 OSS_USERNAME / OSS_PASSWORD）；
    6. 改造 dbstar-org/general 的 maven-release.yml：OSSRH 的 `release:perform` 段改为 `-Prelease,distribution-central`，env 换用新 secrets；
    7. 触发一次真实 release 验证中央仓库发布成功。
  - 验收：一次真实 release 成功发布到中央仓库。

- [ ] **评估 spring-boot 3.x 版本基线**
  - 现状：`boot` 模块 `version.spring-boot` 已升至 2.7.18（2.7 线最后一个版本，JDK 8 基线），2.7.x 开源支持已于 2023 年 11 月结束。
  - 注意：升到 3.x 要求 Java 17 与 Jakarta EE 迁移，与 `pure` 默认编译级别 8 冲突，需连同 `project.java.version` 基线一起决策。
  - 验收：明确结论并记录；若升级则 `mvn validate` 通过、README 同步更新。

## 已完成

- [x] **依赖版本升级到 JDK 1.8 适配的最新版**：slf4j 2.0.6→2.0.18、commons-lang3 3.12.0→3.20.0、commons-io 2.11.0→2.22.0、commons-codec 1.15→1.22.0、commons-beanutils 1.9.4→1.11.0（修复 CVE-2025-48734）、commons-collections4 4.4→4.5.0、spring-boot 2.7.7→2.7.18（2.7 线最后一版）；8 个新构件全部实测字节码 version 52.0（Java 1.8），无需回退；slf4j 桥接包对 slf4j-api、commons-beanutils 对 commons-logging 的 exclusion 保持原样（1.11.0 传递依赖面与 1.9.4 一致，commons-logging 仍在）；junit-bom 5.14.4 已是 5.x 最新线（JUnit 6 需 Java 17）不动。
- [x] **git-commit-id-plugin 提升到 base 层**：从 boot 的 spring-boot Profile（spring 专属）提升到 base 新增的 `java-main` Profile（与 pure 同名 Profile 自动合并，存在 `src/main/java` 即激活），所有 Java 项目均可生成 git.properties，不再局限于 spring；新增 `failOnNoGitDirectory=false`，无 .git 目录的构建环境（源码包、CI 浅克隆）宽容跳过；版本属性 `version.git-commit-id-plugin` 同步移至 base，boot 中相关配置（插件、filters）全部拆除。
- [x] **boot 导入 spring-boot-dependencies BOM**：在 boot 的 spring-boot Profile（src/main/java 门控）内 import BOM，版本复用 `version.spring-boot`（2.7.18），插件与 BOM 天然同版；logback（1.2.12）、junit（5.8.2）等 base 未直接钉版的构件由 spring 策展版接管（slf4j 的接管未达预期，见下条修正）。
- [x] **修正 slf4j 版本接管，钉回 1.7.36**：经临时项目实测发现 Maven 优先级规则为「父 pom 直接钉版 > 子 pom BOM > 父 pom BOM」——base 直接钉版的 slf4j 2.0.18 胜过 BOM 的 1.7.36，与 logback 1.2.12 组成破碎组合（logback 1.2 的 StaticLoggerBinder 对 slf4j 2.0 无效，运行期日志静默丢失）。修复：boot 覆盖 `version.slf4j=1.7.36`（利用属性在子项目上下文插值的机制，base 的 slf4j 钉版随之解析为 1.7.36）；复测确认 spring 项目得到 slf4j 1.7.36（api+全桥接包）+ logback 1.2.12 官方自洽组合，commons-lang3/codec 保留 base 新版（无害），junit 5.8.2 由 BOM 接管符合预期。
- [x] **boot 前置导入 junit-bom，确立子项提前升级机制**：为支持基线组件漏洞修复抢在 spring-boot 发版前升级子项，在 boot 的 spring-boot Profile 中将 junit-bom 排在 spring-boot-dependencies 之前导入（同 pom 内先声明的 BOM 胜），占住位置防 BOM 接管；版本由 boot 覆盖的 `version.junit-jupiter=5.8.2` 显式决定（覆盖 base 的 5.14.4，与 spring-boot 2.7.18 策展版一致，junit 版本成为 boot 的显式决策而非被动跟随 BOM），临时项目实测 junit 5.8.2 / platform 1.8.2、slf4j 1.7.36、logback 1.2.12 同时成立；通用机制（直接声明钉版胜过一切 BOM、BOM 内部属性覆盖无效）已写入 AGENTS.md 安全注意事项。
- [x] **README 与 pom 版本不一致**：`README.md` 中 `version.checkstyle` 由 10.6.0 修正为与 pom 一致的 9.3。
- [x] **移除老旧报告插件**：从 `pure` 模块 `java-main` Profile 删除 findbugs、pmd、checkstyle、jdepend、taglist、jxr、changelog 共 7 个报告插件及 8 个版本属性，代码质量/安全报告统一改用 Sonar（`org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`，不纳入 pom 管理）。findbugs→spotbugs 迁移项随之作废。
- [x] **升级 Maven 插件版本（根 pom）**：enforcer 3.6.3、antrun 3.2.0、clean 3.5.0、dependency 3.11.0、deploy 3.1.4、install 3.1.4、project-info-reports 3.9.0、versions 2.21.0、assembly 3.8.0、gpg 3.2.8、nexus-staging 1.7.0（均已实测 JDK 8 可运行）；enforcer 的 requireMavenVersion 同步从 3.2.5 提升到 3.6.3（新插件的 requiredMavenVersion 均为 3.6.3）。release 3.0.0-M7 留待 milestone 转正批次。
- [x] **maven-site-plugin 改用稳定主线**：4.x 里程碑线停滞于 4.0.0-M16（2024-07，始终未出正式版），由 4.0.0-M4 改用活跃维护的 3.x 正式版 3.22.0（JDK 8 / Maven 3.6.3 要求均满足）。
- [x] **升级 Maven 插件版本（pure 模块）**：compiler 3.15.0、jar 3.5.1、javadoc 3.12.0、resources 3.5.0、source 3.4.0、jacoco 0.8.15，surefire / surefire-report 由 3.0.0-M8 转正 3.5.6（均已实测 JDK 8 可运行，requiredMavenVersion 3.6.3 与 enforcer 门槛一致）。junit-jupiter 保持 5.9.2 留待单独处理。
- [x] **按构建 JDK 分组的编译参数 Profile**：`project.java.version` 默认值 1.8 → 8（纯数字，`--release` 不兼容 `1.8` 格式）；删除死配置 `maven.compiler.compilerVersion`；新增 `jdk8`（source/target）、`jdk9+`（release）、`jdk23+`（proc=full，JDK 23 起 javac 不再默认执行注解处理）三个 profile；`maven.compiler.parameters=true` 全版本启用（保留方法参数名供反射使用）。已在 JDK 8/11/25 三种环境下实测属性解析与 `mvn verify`。
- [x] **milestone 插件版本替换为正式版**：maven-release-plugin 3.0.0-M7 → 3.3.1（JDK 8 可运行，requiredMavenVersion 3.6.3）；本地 `release:prepare -DdryRun=true` 完整跑通 17 个阶段（版本映射 v1.4.0 → 1.4.1-SNAPSHOT、tag v1.4.0 均正确）。至此 milestone 版本全部清零。
- [x] **junit-jupiter 下沉 base 并改用 junit-bom**：`version.junit-jupiter` 5.9.2 → 5.14.4 并移至 `base`；base 新增与 pure 同名的 `java-test` Profile（自动合并），导入 junit-bom 统一版本，注入构件由 junit-jupiter-engine（仅运行期）换成 junit-jupiter 聚合构件（api+params+engine，开箱即可编写+运行测试）；pure 的 java-test 只保留 surefire/jacoco 等测试插件。
- [x] **取消 obscure 层**：确认不再需要代码混淆，删除 `obscure` 模块（叶子节点，不影响继承链），根 pom modules 摘除，README/AGENTS.md 中 obscure 及 proguard 相关的模块、插件、版本属性、Profile 说明全部清除，「评估 ProGuard 7.x 升级」待办项随之作废。
- [x] **取消 assembly 和 mode 层**：可执行 jar 打包全面由 spring-boot（boot 模块）取代，删除 `assembly`、`mode` 模块（死胡同分支，仅 mode 以 assembly 为 parent，删除后继承链 pure→parent、base→pure、docker→base、boot→docker 无断链）；根 pom 的 maven-assembly-plugin pluginManagement 条目与 `version.maven-assembly-plugin` 属性（死配置）一并删除；README/AGENTS.md 中相关模块、配置项、插件、Profile 说明全部清除；`project.mainClass` 约定改由 boot 承接（boot 的 spring-boot-maven-plugin 仍以 `${project.mainClass}` 指定主类，下游子项目设置方式不变），`project.mode` 随 mode 模块移除。
- [x] **取消 docker 层，git-commit-id-plugin 移入 boot**：实际从不通过 mvn 构建镜像（由集团 CI 凭 Dockerfile 构建），dockerfile-maven-plugin 及 `project.docker.image.*` 属性均为死配置，删除整个 `docker` 模块；git-commit-id-plugin 从 Dockerfile 门控中解耦，移入 boot 的 `spring-boot` Profile（存在 `src/main/java` 即激活，spring 项目不再需要为拿 `git.commit.id.describe` 而放置 Dockerfile），配置原样保留（revision goal、generateGitPropertiesFile、3 个 includeOnlyProperty、resource filter）；boot 改继承 base，继承链收敛为 pure→parent、base→pure、boot→base；boot 中覆盖 `project.docker.image.jar` 的 docker Profile 随之删除。

## 通用约束

- 每项修正后同步更新 `README.md` 对应版本表格。
- 版本一律改 `<properties>` 中的 `version.*` 属性，不硬编码。
- 本项目是父 POM，任何改动影响所有下游项目，提交前需评估兼容性。
- `pom.xml.versionsBackup` 为工具生成文件，不手工编辑、不提交。
