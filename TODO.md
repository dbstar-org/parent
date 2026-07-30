# TODO

本项目当前存在的问题清单，按优先级排序，逐一修正。已完成的条目直接从本文件移除，历史见 git 提交记录。

## 待办

- [ ] **account-core 改继承 native**（后续独立议题，在 account-core 仓库执行）
  - parent 坐标 `io.github.dbstarll.parent:base` → `io.github.dbstarll.parent:native`（版本随 native 层首个发布版）；
  - 删根 pom 自带 `native-trace` Profile（pom.xml:136-149，已由 native 层提供）；
  - 自行覆盖 `version.slf4j` / `version.junit-jupiter` / `version.spring-boot` / `project.java.version`——继承 native 会经 boot 层带入 spring-boot 2.7.18 BOM 与 slf4j 1.7.36 等 2.7 适配覆盖，作为 SB3 库必须钉回自己的基线；
  - `dependencies` 模块「parent 必须是外部 POM」约束不动（它直接继承 `io.github.dbstarll.parent:base`，不经过 account-core-parent）；
  - 验收：`mvn verify` 通过，依赖版本与迁移前一致（`mvn dependency:list` 对比）。

- [ ] **spring-boot-use-core 改继承 native**（后续独立议题，在 spring-boot-use-core 仓库执行）
  - parent 坐标 `io.github.dbstarll.parent:boot` → `io.github.dbstarll.parent:native`；
  - 删自带 `native` / `native-trace` Profile（pom.xml:199-264，已由 native 层提供）；
  - 项目特定 buildArg（`-H:ExcludeResources`）改以同名 `native-build` Profile 合并追加；`-H:ConfigurationFileDirectories` 无需追加——覆盖 `project.native.agent.config.dir=${project.basedir}/native/agent-output-test` 即同时改变采集输出与编译输入；
  - 属性 `native.agent.config.dir` 改名为 `project.native.agent.config.dir`，采集输出覆盖为 `${project.basedir}/native/agent-output-test`；
  - 验收：`mvn clean test -Pnative-trace` 与 `mvn -Pnative-build package -DskipTests` 流程不变，二进制产出位置不变。

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

## 通用约束

- 每项修正后同步更新 `README.md` 对应版本表格。
- 版本一律改 `<properties>` 中的 `version.*` 属性，不硬编码。
- 本项目是父 POM，任何改动影响所有下游项目，提交前需评估兼容性。
- `pom.xml.versionsBackup` 为工具生成文件，不手工编辑、不提交。
