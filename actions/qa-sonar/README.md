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
