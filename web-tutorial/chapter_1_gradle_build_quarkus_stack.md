# Chapter 1: Scaffolding the Gradle Build & Quarkus Stack

**Milestone:** Establish a clean, reproducible project structure using Gradle 9.8.2 (Kotlin DSL), Java 25, and Quarkus 3.40.

---

## 1. Directory Structure

Create the project base:

```text
order-service/
├── gradle/
│   └── wrapper/
│       ├── gradle-wrapper.jar
│       └── gradle-wrapper.properties
├── src/
│   ├── main/
│   │   ├── java/
│   │   └── resources/
│   │       └── application.properties
├── build.gradle.kts
├── settings.gradle.kts
└── gradlew
```

---

## 2. Gradle Configuration

### `settings.gradle.kts`
```kotlin
pluginManagement {
    repositories {
        mavenCentral()
        gradlePluginPortal()
    }
}

rootProject.name = "order-service"
```

### `gradle/wrapper/gradle-wrapper.properties`
Ensure Gradle 9.8.2 is specified:
```properties
distributionBase=GRADLE_USER_HOME
distributionPath=wrapper/dists
distributionUrl=https\://services.gradle.org/distributions/gradle-9.8.2-bin.zip
zipStoreBase=GRADLE_USER_HOME
zipStorePath=wrapper/dists
```

### `build.gradle.kts`
Here we configure the Java 25 toolchain, Quarkus 3.40 plugins, and OpenAPI code generator plugins:

```kotlin
plugins {
    java
    id("io.quarkus") version "3.40.0"
    id("org.openapi.generator") version "7.10.0"
}

repositories {
    mavenCentral()
}

val quarkusPlatformGroupId = "io.quarkus.platform"
val quarkusPlatformArtifactId = "quarkus-bom"
val quarkusPlatformVersion = "3.40.0"

dependencies {
    // Import Quarkus BOM
    implementation(enforcedPlatform("$quarkusPlatformGroupId:$quarkusPlatformArtifactId:$quarkusPlatformVersion"))

    // REST and Serialization
    implementation("io.quarkus:quarkus-resteasy-reactive-jackson")
    implementation("io.quarkus:quarkus-hibernate-validator")

    // Database: jOOQ and PostgreSQL
    implementation("io.quarkiverse.jooq:quarkus-jooq:2.2.0")
    implementation("io.quarkus:quarkus-jdbc-postgresql")
    implementation("io.quarkus:quarkus-flyway")

    // Observability & Cloud-Native
    implementation("io.quarkus:quarkus-smallrye-health")
    implementation("io.quarkus:quarkus-micrometer-registry-prometheus")

    // Test dependencies
    testImplementation("io.quarkus:quarkus-junit5")
    testImplementation("io.rest-assured:rest-assured")
}

java {
    toolchain {
        languageVersion.set(JavaLanguageVersion.of(25))
    }
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

## 3. Application Configuration

Create `src/main/resources/application.properties`:

```properties
quarkus.application.name=order-service
quarkus.http.port=8080

# Health and Metrics
quarkus.smallrye-health.root-path=/health
quarkus.micrometer.export.prometheus.path=/metrics
```

---

## 4. Verification Checkpoint

Execute the build task to verify the Java 25 compiler toolchain and Gradle wrapper integration:

```bash
./gradlew build -x test
```

Expected output:
```text
BUILD SUCCESSFUL in 2s
```