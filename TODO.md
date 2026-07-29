# TODO

本项目当前存在的问题清单，按优先级排序，逐一修正后勾选。

## 待办

- [ ] **升级 commons-beanutils 修复 CVE**
  - 现状：`base` 模块 `version.commons-beanutils = 1.9.4`，存在 CVE-2025-48734，1.11.0 修复。
  - 影响：`base` 被所有下游项目继承，影响面大，需评估兼容性。
  - 验收：`mvn validate` 通过；README 版本表格同步更新。

- [ ] **升级 slf4j 与 Apache Commons 基准依赖**
  - 现状：slf4j 2.0.6 → 2.0.18、commons-lang3 3.12.0 → 3.20.0、commons-io 2.11.0 → 2.22.0、commons-codec 1.15 → 1.22.0、commons-collections4 4.4 → 4.5.0。
  - 注意：commons-beanutils 对 commons-logging 的 exclusion、slf4j 桥接包对 slf4j-api 的 exclusion 需保持原样。
  - 验收：`mvn validate` 通过；README 版本表格同步更新。

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

- [ ] **升级 junit-jupiter**
  - 现状：junit-jupiter 5.9.2；5.x 最新 5.14.4（Java 8 兼容）。
  - 注意：JUnit 6.x 要求 Java 17，与本项目 JDK 8 基线冲突，只能升 5.x 线。
  - 验收：`mvn validate` 通过；README 版本表格同步更新。

- [ ] **评估 spring-boot 版本基线**
  - 现状：`boot` 模块 `version.spring-boot = 2.7.7`，2.7.x 开源支持已于 2023 年 11 月结束。
  - 注意：升到 3.x 要求 Java 17 与 Jakarta EE 迁移，与 `pure` 默认编译级别 8 冲突；可能需要先升级 2.7.18（2.7 线最后一个版本），3.x 另议。
  - 验收：明确结论并记录；若升级则 `mvn validate` 通过、README 同步更新。

- [ ] **评估 ProGuard 7.x 升级**
  - 现状：`obscure` 模块 `version.proguard = 6.2.2`，只支持 Java 8 字节码，混淆 Java 11+ 编译的 class 会失败。
  - 注意：ProGuard 7.x 配置项有变化，需单独验证 obscure 的三种混淆策略（工具 jar / 可执行 jar / Web 跳过）。
  - 验收：明确结论并记录；若升级则 `mvn validate` 通过、README 同步更新。

## 已完成

- [x] **README 与 pom 版本不一致**：`README.md` 中 `version.checkstyle` 由 10.6.0 修正为与 pom 一致的 9.3。
- [x] **移除老旧报告插件**：从 `pure` 模块 `java-main` Profile 删除 findbugs、pmd、checkstyle、jdepend、taglist、jxr、changelog 共 7 个报告插件及 8 个版本属性，代码质量/安全报告统一改用 Sonar（`org.sonarsource.scanner.maven:sonar-maven-plugin:sonar`，不纳入 pom 管理）。findbugs→spotbugs 迁移项随之作废。
- [x] **升级 Maven 插件版本（根 pom）**：enforcer 3.6.3、antrun 3.2.0、clean 3.5.0、dependency 3.11.0、deploy 3.1.4、install 3.1.4、project-info-reports 3.9.0、versions 2.21.0、assembly 3.8.0、gpg 3.2.8、nexus-staging 1.7.0（均已实测 JDK 8 可运行）；enforcer 的 requireMavenVersion 同步从 3.2.5 提升到 3.6.3（新插件的 requiredMavenVersion 均为 3.6.3）。release 3.0.0-M7 留待 milestone 转正批次。
- [x] **maven-site-plugin 改用稳定主线**：4.x 里程碑线停滞于 4.0.0-M16（2024-07，始终未出正式版），由 4.0.0-M4 改用活跃维护的 3.x 正式版 3.22.0（JDK 8 / Maven 3.6.3 要求均满足）。
- [x] **升级 Maven 插件版本（pure 模块）**：compiler 3.15.0、jar 3.5.1、javadoc 3.12.0、resources 3.5.0、source 3.4.0、jacoco 0.8.15，surefire / surefire-report 由 3.0.0-M8 转正 3.5.6（均已实测 JDK 8 可运行，requiredMavenVersion 3.6.3 与 enforcer 门槛一致）。junit-jupiter 保持 5.9.2 留待单独处理。
- [x] **按构建 JDK 分组的编译参数 Profile**：`project.java.version` 默认值 1.8 → 8（纯数字，`--release` 不兼容 `1.8` 格式）；删除死配置 `maven.compiler.compilerVersion`；新增 `jdk8`（source/target）、`jdk9+`（release）、`jdk23+`（proc=full，JDK 23 起 javac 不再默认执行注解处理）三个 profile；`maven.compiler.parameters=true` 全版本启用（保留方法参数名供反射使用）。已在 JDK 8/11/25 三种环境下实测属性解析与 `mvn verify`。
- [x] **milestone 插件版本替换为正式版**：maven-release-plugin 3.0.0-M7 → 3.3.1（JDK 8 可运行，requiredMavenVersion 3.6.3）；本地 `release:prepare -DdryRun=true` 完整跑通 17 个阶段（版本映射 v1.4.0 → 1.4.1-SNAPSHOT、tag v1.4.0 均正确）。至此 milestone 版本全部清零。

## 通用约束

- 每项修正后同步更新 `README.md` 对应版本表格。
- 版本一律改 `<properties>` 中的 `version.*` 属性，不硬编码。
- 本项目是父 POM，任何改动影响所有下游项目，提交前需评估兼容性。
- `pom.xml.versionsBackup` 为工具生成文件，不手工编辑、不提交。
