# Action: Trivy Security Scan

Escaneo de seguridad polivalente usando [Trivy](https://github.com/aquasecurity/trivy) (SCA de dependencias, IaC/Dockerfile, contenedores, licencias y secretos), con:

- Versiones pinneadas de la herramienta (consistencia entre repos)
- Multi-modo (`scan-type`): `fs` (dependencias + IaC del repo), `image` (contenedores Docker/OCI), `config` (solo IaC/Dockerfile/K8s/Terraform), `rootfs` y `repo`
- Autodeteccion de `.trivyignore` / `.trivyignore.yaml` y `trivy.yaml` / `.trivy.yaml` del repositorio
- Resumen detallado en `$GITHUB_STEP_SUMMARY` con desplegables de CVEs y misconfigurations
- Reporte opcional en `sarif` (GitHub Code Scanning), `json`, `cyclonedx` o `spdx-json` (SBOM) sin re-escanear (`trivy convert`)

## Entradas

| Input | Default | Descripcion |
|---|---|---|
| `version` | `v0.76.2` | Version de Trivy (pinneada) |
| `scan-type` | `fs` | Modo de escaneo: `fs` \| `image` \| `config` \| `rootfs` \| `repo` |
| `target` | `.` | Ruta a escanear (`fs`/`config`/`rootfs`) o referencia `imagen:tag` (obligatorio en `image`) |
| `scanners` | `vuln,misconfig` | Escaneadores separados por coma (`vuln,misconfig,secret,license`) |
| `severity` | `CRITICAL,HIGH` | Severidades evaluadas (`CRITICAL,HIGH,MEDIUM,LOW,UNKNOWN`) |
| `ignore-unfixed` | `false` | `true` ignora vulnerabilidades que aun no tienen version corregida |
| `vuln-type` | `os,library` | Tipos de paquetes en modo `image`/`rootfs` (`os,library`) |
| `ignorefile` | `""` | Ruta a archivo `.trivyignore`; vacio autodectecta `.trivyignore` / `.trivyignore.yaml` en la raiz |
| `config-path` | `""` | Ruta a `trivy.yaml` propio; vacio autodetecta `trivy.yaml` / `.trivy.yaml` en la raiz |
| `skip-dirs` | `node_modules,.next,dist,build,.git,coverage,vendor` | Directorios a omitir separados por coma |
| `skip-files` | `""` | Archivos a omitir separados por coma |
| `fail-on-error` | `true` | `true` falla el job si se detectan hallazgos de la severidad configurada |
| `report-format` | `""` | Formato opcional para el consumidor: `sarif` \| `json` \| `cyclonedx` \| `spdx-json` |
| `report-path` | `""` | Ruta del reporte; default `trivy-report.<ext>` en el workspace |
| `extra-args` | `""` | Argumentos extra para el CLI de Trivy |

## Salidas

- `vulnerabilities`: numero de vulnerabilidades (CVEs) detectadas
- `misconfigurations`: numero de misconfigurations (IaC/Dockerfile) detectadas
- `secrets`: numero de secretos detectados (si `secret` se incluyo en `scanners`)
- `licenses`: numero de problemas de licencia detectados (si `license` se incluyo en `scanners`)
- `total`: suma total de hallazgos
- `report-path`: ruta del archivo de reporte generado (si se definio `report-format`)
- `unblocked`: `true` si hubo hallazgos pero `fail-on-error=false`

## Ejemplo basico: dependencias (SCA) + IaC en repositorio (`fs`)

```yml
name: Security Scan
on: [push, pull_request]

permissions:
  contents: read

jobs:
  trivy-fs:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262

      - name: Trivy FS Scan
        uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-trivy@v1.0.2
        with:
          scan-type: fs
          severity: CRITICAL,HIGH
          ignore-unfixed: "true"
          fail-on-error: "true"
```

## Ejemplo: escaneo de imagen Docker recien construida (`image`)

```yml
      - name: Build Docker image
        run: docker build -t mi-app:${{ github.sha }} .

      - name: Trivy Image Scan
        uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-trivy@v1.0.2
        with:
          scan-type: image
          target: mi-app:${{ github.sha }}
          severity: CRITICAL,HIGH
          ignore-unfixed: "true"
          fail-on-error: "true"
```

## Ejemplo con reporte SARIF + GitHub Code Scanning

```yml
permissions:
  contents: read
  security-events: write

steps:
  - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262

  - name: Trivy Scan (SARIF)
    uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-trivy@v1.0.2
    with:
      scan-type: fs
      report-format: sarif
      report-path: trivy-report.sarif

  - name: Upload SARIF to GitHub Security
    uses: github/codeql-action/upload-sarif@v3
    if: always()
    with:
      sarif_file: trivy-report.sarif
```

## Exclusion de falsos positivos o riesgos aceptados (`.trivyignore`)

Para ignorar temporalmente un CVE o regla de IaC aceptada por el equipo, crea un archivo `.trivyignore` en la raiz del repositorio (el action lo detecta automaticamente):

```txt
# Ignorar CVE especifico con fecha de expiracion
CVE-2024-12345 exp:2026-12-31

# Ignorar regla de Dockerfile especifica
DS002
```

## Notas importantes

- En modo `fs`, Trivy analiza los archivos de lock (`package-lock.json`, `pnpm-lock.yaml`, `yarn.lock`, `pom.xml`, `gradle.lockfile`, `go.sum`, `requirements.txt`, `Cargo.lock`, etc.). Por eso `node_modules` se omite por defecto en `skip-dirs` para acelerar el escaneo sin perder cobertura de dependencias.
- En modo `config`, el action fuerza automaticamente `scanners: misconfig` para auditar Dockerfiles, manifiestos de Kubernetes, Helm charts y Terraform.
- Para generar un SBOM del proyecto o imagen, usa `report-format: cyclonedx` o `report-format: spdx-json`.
