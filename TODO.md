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

- [ ] **升级 Maven 插件版本（根 pom）**
  - 现状：enforcer 3.1.0 → 3.6.3、gpg 3.0.1 → 3.2.8、deploy 3.0.0 → 3.1.4、install 3.1.0 → 3.1.4、clean 3.2.0 → 3.5.0、antrun 3.1.0 → 3.2.0、assembly 3.4.2 → 3.8.0、dependency 3.5.0 → 3.11.0、project-info-reports 3.4.2 → 3.9.0、versions 2.14.2 → 2.21.0、nexus-staging 1.6.13 → 1.7.0。
  - 验收：`mvn validate` 与 `mvn verify` 通过；README 版本表格同步更新。

- [ ] **升级 Maven 插件版本（pure 模块）**
  - 现状：compiler 3.10.1 → 3.15.0、jar 3.3.0 → 3.5.1、javadoc 3.4.1 → 3.12.0、resources 3.3.0 → 3.5.0、source 3.2.1 → 3.4.0、checkstyle-plugin 3.2.1 → 3.6.0、jxr 3.3.0 → 3.6.0、pmd 3.20.0 → 3.28.0、jacoco 0.8.8 → 0.8.15、jdepend 2.0 → 2.2.0、taglist 3.0.0 → 3.2.2。
  - 验收：`mvn validate` 通过；README 版本表格同步更新。

- [ ] **milestone 插件版本替换为正式版**
  - 现状：maven-release-plugin 3.0.0-M7（正式版 3.3.1）、surefire / surefire-report 3.0.0-M8（正式版 3.2.5+）、maven-site-plugin 4.0.0-M4。
  - 注意：maven-site-plugin 4.x 至今只有 milestone 版，需确认下游站点构建兼容性后再定版本；release 插件升级会改变发布行为，需单独验证 `mvn release:prepare`。
  - 验收：`mvn validate` 通过；发布流程不受影响。

- [ ] **升级 checkstyle 与 junit-jupiter**
  - 现状：checkstyle 9.3（主线已 10.x/11.x）、junit-jupiter 5.9.2。
  - 注意：checkstyle 大版本升级可能使现有规则文件 `sun_checks-120.xml` 下的检查变严，需评估对下游的影响。
  - 验收：`mvn validate` 通过；README 版本表格同步更新。

- [ ] **findbugs 迁移到 spotbugs**
  - 现状：findbugs-maven-plugin 3.0.5 已停止维护（2018 年最后发布），对新版 JDK 字节码支持差。
  - 方案：替换为 `com.github.spotbugs:spotbugs-maven-plugin`，属性名由 `version.findbugs-maven-plugin` 调整为 `version.spotbugs-maven-plugin`。
  - 注意：属于接口变更，会改变下游的插件配置继承方式，需评估。
  - 验收：`mvn validate` 通过；README 插件清单同步更新。

- [ ] **评估 spring-boot 版本基线**
  - 现状：`boot` 模块 `version.spring-boot = 2.7.7`，2.7.x 开源支持已于 2023 年 11 月结束。
  - 注意：升到 3.x 要求 Java 17 与 Jakarta EE 迁移，与 `pure` 默认编译级别 1.8 冲突；可能需要先升级 2.7.18（2.7 线最后一个版本），3.x 另议。
  - 验收：明确结论并记录；若升级则 `mvn validate` 通过、README 同步更新。

## 已完成

- [x] **README 与 pom 版本不一致**：`README.md` 中 `version.checkstyle` 由 10.6.0 修正为与 pom 一致的 9.3。

## 通用约束

- 每项修正后同步更新 `README.md` 对应版本表格。
- 版本一律改 `<properties>` 中的 `version.*` 属性，不硬编码。
- 本项目是父 POM，任何改动影响所有下游项目，提交前需评估兼容性。
- `pom.xml.versionsBackup` 为工具生成文件，不手工编辑、不提交。
