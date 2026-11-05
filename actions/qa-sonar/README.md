# Action: SonarCloud Analysis

Esta acción de GitHub realiza un análisis de código utilizando SonarCloud.

## Entradas

- `app-name` (requerido): Nombre de la aplicación.
- `build-timestamp` (requerido): Marca el tiempo de la compilación.
- `app-version` (requerido): Version de la aplicación
- `tek` (opcional): Tek. Valor predeterminado: una cadena vacía.
- `sonar-url` (opcional): URL de sonarCloud. Valor predeterminado.
- `sonar-token` (opcional): Token de SonarCloud.

## Ejemplo de Uso

```yml
name: SonarQube Analysis
on:
  push:
    branches:
      - main

jobs:
  sonar-analysis:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout repository
        uses: actions/checkout@v3
      - name: SonarQube Analysis
        uses: ./
        with:
          app-name: '....'
          app-version: '1.0.0'
          tek: 'my-tek'
```

## Cobertura en Android (`tek: android`)

El input `coverage-tool` define cómo se interpreta el reporte de cobertura para proyectos Android:

| `coverage-tool` | Properties generadas | Uso recomendado |
|---|---|---|
| `auto` (default) / `jacoco` | `sonar.java.coveragePlugin=jacoco` + `sonar.coverage.jacoco.xmlReportPaths` | Apps con reportes JaCoCo (comportamiento original) |
| `kover` | `sonar.kotlin.coverage.reportPaths` | Apps **100% Kotlin** que generan XML de Kover |
| `both` | Las de JaCoCo (código Java) **+** `sonar.kotlin.coverage.reportPaths` (código Kotlin) | Apps **híbridas** Java + Kotlin |

> En modo `both`, el input `coverage-path` se reserva para el XML de **Kover**; JaCoCo usa sus rutas por defecto.

### Ejemplo Android + Kover

```yml
- name: Generate Kover coverage report
  run: ./gradlew koverXmlReport

- name: SonarQube Analysis
  uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-sonar@main
  with:
    app-name: 'mi-app-android'
    app-version: '1.0.0'
    tek: 'android'
    coverage-tool: 'kover'
    coverage-path: 'app/build/reports/kover/report.xml'
```

### Requisitos en el proyecto Android

1. Aplicar el plugin de Kover en el módulo (por ejemplo `app/build.gradle.kts`):

```kotlin
plugins {
    id("org.jetbrains.kotlinx.kover") version "0.9.1"
}
```

2. Correr `./gradlew koverXmlReport` **antes** del análisis. El reporte XML queda en
   `app/build/reports/kover/report.xml` (pasar esa ruta en `coverage-path`, o confiar en el default del action).
3. Las exclusiones de cobertura de código se definen en el bloque `kover { }` del proyecto;
   el input `coverage-exclusions` solo alimenta `sonar.coverage.exclusions` (a nivel scanner).

### Limitaciones de Kover

- Kover cubre **tests unitarios JVM** (`testDebugUnitTest`/`testReleaseUnitTest`).
- Los **tests instrumentados** (Espresso / `connectedAndroidTest`) no son cubiertos por Kover;
  en ese caso usar `coverage-tool: both` o `jacoco`.

