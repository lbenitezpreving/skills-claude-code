# Configuración Gradle + JaCoCo

## build.gradle (Groovy DSL)

```groovy
plugins {
    id 'java'
    id 'org.springframework.boot' version '3.2.0'
    id 'io.spring.dependency-management' version '1.1.4'
    id 'jacoco'
}

dependencies {
    // JUnit 5 + Mockito + AssertJ incluidos en starter-test
    testImplementation 'org.springframework.boot:spring-boot-starter-test'
}

test {
    useJUnitPlatform()               // obligatorio para JUnit 5
    finalizedBy jacocoTestReport     // genera reporte siempre tras los tests
}

jacoco {
    toolVersion = '0.8.11'
}

jacocoTestReport {
    dependsOn test
    reports {
        xml.required = true          // para SonarQube / CI
        html.required = true
        html.outputLocation = layout.buildDirectory.dir('reports/jacoco/html')
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/dto/**',
                '**/config/**',
                '**/model/**',
                '**/entity/**',
                '**/*Application.class',
                '**/exception/**'
            ])
        }))
    }
}

jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = 'CLASS'
            limit {
                counter = 'LINE'
                value   = 'COVEREDRATIO'
                minimum = 0.80
            }
        }
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.collect {
            fileTree(dir: it, exclude: [
                '**/dto/**',
                '**/config/**',
                '**/model/**',
                '**/entity/**',
                '**/*Application.class',
                '**/exception/**'
            ])
        }))
    }
}

check.dependsOn jacocoTestCoverageVerification
```

---

## build.gradle.kts (Kotlin DSL)

```kotlin
plugins {
    java
    id("org.springframework.boot") version "3.2.0"
    id("io.spring.dependency-management") version "1.1.4"
    jacoco
}

dependencies {
    testImplementation("org.springframework.boot:spring-boot-starter-test")
}

tasks.test {
    useJUnitPlatform()
    finalizedBy(tasks.jacocoTestReport)
}

jacoco {
    toolVersion = "0.8.11"
}

val excludedClasses = listOf(
    "**/dto/**",
    "**/config/**",
    "**/model/**",
    "**/entity/**",
    "**/*Application.class",
    "**/exception/**"
)

tasks.jacocoTestReport {
    dependsOn(tasks.test)
    reports {
        xml.required.set(true)
        html.required.set(true)
        html.outputLocation.set(layout.buildDirectory.dir("reports/jacoco/html"))
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.map {
            fileTree(it) { exclude(excludedClasses) }
        }))
    }
}

tasks.jacocoTestCoverageVerification {
    violationRules {
        rule {
            element = "CLASS"
            limit {
                counter = "LINE"
                value   = "COVEREDRATIO"
                minimum = "0.80".toBigDecimal()
            }
        }
    }
    afterEvaluate {
        classDirectories.setFrom(files(classDirectories.files.map {
            fileTree(it) { exclude(excludedClasses) }
        }))
    }
}

tasks.check {
    dependsOn(tasks.jacocoTestCoverageVerification)
}
```

---

## Comandos de referencia

```bash
# Tests + reporte + verificación 80% (recomendado en CI)
./gradlew clean check

# Tests + reporte HTML
./gradlew clean test jacocoTestReport

# Solo tests
./gradlew clean test

# Solo reporte (si los tests ya se ejecutaron)
./gradlew jacocoTestReport

# Solo verificar cobertura mínima
./gradlew jacocoTestCoverageVerification
```

## Rutas de salida

| Artefacto          | Ruta                                          |
|--------------------|-----------------------------------------------|
| Reporte HTML       | `build/reports/jacoco/html/index.html`        |
| Reporte XML (CI)   | `build/reports/jacoco/test/jacocoTestReport.xml` |
