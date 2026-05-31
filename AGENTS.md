# AGENTS.md

## 项目概况

Minecraft NeoForge Mod 项目，使用 `net.neoforged.gradle.userdev` 插件构建。

## CI/CD 配置

### 配置文件

| 文件 | 用途 |
|------|------|
| `.github/workflows/ci.yml` | CI 流水线：`gradle check` + `gradle build` |
| `.github/workflows/publish.yml` | Publish 流水线：发布 jar + sourcesJar 到自建 Maven |
| `build.gradle` | 定义 sourcesJar 任务和 publishing 仓库配置 |

### Secrets 配置

在 GitHub 仓库 **Settings → Secrets and variables → Actions** 中添加：

| Secret | 用途 |
|--------|------|
| `MAVEN_USERNAME` | Maven 仓库用户名 |
| `MAVEN_PASSWORD` | Maven 仓库密码（或 Token） |

### sourcesJar 配置 (build.gradle)

```groovy
tasks.register('sourcesJar', Jar) {
    archiveClassifier = 'sources'
    manifest {
        attributes([
                "Specification-Title": archivesBaseName,
                "Specification-Vendor": "${author}",
                "Specification-Version": "1",
                "Implementation-Title": archivesBaseName,
                "Implementation-Version": "${version}",
                "Implementation-Vendor" :"${author}",
                "Implementation-Timestamp": new Date().format("yyyy-MM-dd'T'HH:mm:ssZ")
        ])
    }
    from sourceSets.main.allJava
}
```

### Publishing 配置 (build.gradle)

```groovy
publishing {
    publications {
        mavenJava(MavenPublication) {
            artifact jar
            artifact sourcesJar
        }
    }
    repositories {
        maven {
            name = "MyMaven"
            url = uri("https://maven.example.com/")
            credentials {
                username = System.getenv("MAVEN_USERNAME")
                password = System.getenv("MAVEN_PASSWORD")
            }
        }
    }
}
```

> **注意**：`maven-publish` 插件默认使用项目的 `group` 和 `version` 作为 `groupId` 和 `version`，无需显式指定。

### Publish Workflow 参考

```yaml
name: Publish

on:
  push:
    branches:
      - main
      - master

jobs:
  publish:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Run Data
        run: ./gradlew runData

      - name: Publish
        run: ./gradlew publish
        env:
          MAVEN_USERNAME: ${{ secrets.MAVEN_USERNAME }}
          MAVEN_PASSWORD: ${{ secrets.MAVEN_PASSWORD }}
```

### CI Workflow 参考

```yaml
name: CI

on:
  push:
    branches:
      - main
      - master
  pull_request:

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Setup JDK
        uses: actions/setup-java@v4
        with:
          java-version: 21
          distribution: 'temurin'
          cache: 'gradle'

      - name: Grant execute permission for gradlew
        run: chmod +x gradlew

      - name: Check
        run: ./gradlew check

      - name: Build
        run: ./gradlew build
```

## GitHub Secrets 类型说明

| 类型 | 作用域 |
|------|--------|
| **Repository secrets** | 仅当前仓库可用 |
| **Environment secrets** | 关联特定 Environment，可限制分支和审批 |
| **Organization secrets** | 该组织下所有仓库可用 |

优先级：Environment > Repository > Organization

## 常见问题

- **Repository name 不合法**：Maven 仓库名只能包含 `A-Za-z0-9_\-.`，不能用单引号或空格
- **sourcesJar 任务不存在**：`sourcesJar` 不会自动创建，必须用 `tasks.register('sourcesJar', Jar)` 显式定义
