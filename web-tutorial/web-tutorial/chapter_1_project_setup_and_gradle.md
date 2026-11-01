# Chapter 1: Project Setup and Gradle Configuration

In this chapter, you will configure a build environment leveraging **Java 25**, **Gradle 9.8.2** with the **Kotlin DSL**, and **Quarkus 3.40**.

---

## 1. Directory Structure

Run the following commands in your terminal to set up the project scaffold:

```bash
mkdir -p order-service
cd order-service
mkdir -p src/main/java/com/example/orders
mkdir -p src/main/resources/db/migration
mkdir -p src/main/openapi
```

---

## 2. Gradle Settings

Create `settings.gradle.kts`:

```kotlin:settings.gradle.kts
rootProject.name = "order-service"

pluginManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
}

dependencyResolutionManagement {
    repositoriesMode.set(RepositoriesMode.PREFER_PROJECT)
    repositories {
        mavenCentral()
    }
}
```

---

## 3. Build Script (`build.gradle.kts`)

Create `build.gradle.kts`. This configuration defines the Java 25 compilation target, Quarkus 3.40 platform, jOOQ plugin, and OpenAPI generator:

```kotlin:build.gradle.kts
plugins {
    java
    id("io.quarkus") version "3.40.0"
    id("org.openapi.generator") version "7.10.0"
}

group = "com.example.orders"
version = "1.0.0-SNAPSHOT"

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
}

val quarkusPlatformGroupId = "io.quarkus.platform"
val quarkusPlatformArtifactId = "quarkus-bom"
val quarkusPlatformVersion = "3.40.0"
val jooqVersion = "3.19.16"

dependencies {
    // Quarkus Platform BOM
    implementation(enforcedPlatform("$quarkusPlatformGroupId:$quarkusPlatformArtifactId:$quarkusPlatformVersion"))

    // Quarkus Web & Validation
    implementation("io.quarkus:quarkus-rest")
    implementation("io.quarkus:quarkus-rest-jackson")
    implementation("io.quarkus:quarkus-hibernate-validator")
    implementation("io.quarkus:quarkus-smallrye-openapi")

    // Database & jOOQ
    implementation("io.quarkus:quarkus-jdbc-postgresql")
    implementation("io.quarkus:quarkus-agroal")
    implementation("io.quarkiverse.jooq:quarkus-jooq:2.2.0")
    implementation("org.jooq:jooq:$jooqVersion")

    // Flyway for DB migrations
    implementation("io.quarkus:quarkus-flyway")

    // Jackson Annotations & Nullability
    implementation("com.fasterxml.jackson.core:jackson-annotations")
    implementation("jakarta.validation:jakarta.validation-api")
    implementation("jakarta.annotation:jakarta.annotation-api")

    // Observability & Health
    implementation("io.quarkus:quarkus-smallrye-health")

    // Testing
    testImplementation("io.quarkus:quarkus-junit5")
    testImplementation("io.rest-assured:rest-assured")
}

tasks.withType<JavaCompile> {
    options.encoding = "UTF-8"
    options.compilerArgs.addAll(listOf("-parameters", "--enable-preview"))
}

tasks.withType<Test> {
    jvmArgs("--enable-preview")
}
```

---

## 4. Gradle Wrapper Setup

Generate Gradle wrapper version 9.8.2:

```bash
gradle wrapper --gradle-version 9.8.2 --distribution-type bin
```

Verify that the wrapper works and picks up Java 25:

```bash
./gradlew --version
```