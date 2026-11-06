# Action: Gitleaks Scan

Escaneo de secretos (tokens, API keys, contrasenas, credenciales en codigo)
usando [gitleaks](https://github.com/gitleaks/gitleaks), con:

- Versiones pinneadas de la herramienta (consistencia entre repos)
- **Redact** por defecto: el valor del secreto nunca aparece en los logs ni en el summary
- Config propia del repo (`.gitleaks.toml`) o la default de gitleaks
- **Baseline** para adoptar la herramienta en repos con secretos historicos ya conocidos
- Reporte opcional SARIF/JSON (SARIF compatible con GitHub code scanning)

## Entradas

| Input | Default | Descripcion |
|---|---|---|
| `version` | `v8.30.1` | Version de gitleaks (pinneada) |
| `config-path` | `""` | Path a un `.gitleaks.toml` propio; vacio usa `.gitleaks.toml`/`gitleaks.toml` del repo si existe |
| `scan-history` | `true` | `true` = historial git completo (`gitleaks detect`), `false` = solo working dir (`--no-git`) |
| `fail-on-error` | `true` | `true` bloquea el build si encuentra secretos |
| `redact` | `true` | No imprime el valor del secreto en logs/summary |
| `baseline-path` | `""` | Reporte JSON previo con hallazgos aceptados que no bloquean |
| `report-format` | `""` | `sarif` o `json` (opcional) |
| `report-path` | `""` | Ruta del reporte; default `gitleaks-report.<format>` en el workspace |
| `extra-args` | `""` | Argumentos extra para `gitleaks detect` |

## Salidas

- `leaks`: numero de secretos detectados
- `report-path`: ruta del reporte (si se definio `report-format`)
- `unblocked`: `true` si hubo hallazgos pero `fail-on-error=false`

## Ejemplo basico (push/PR)

```yml
name: Secrets Scan
on: [push, pull_request]

permissions:
  contents: read

jobs:
  gitleaks:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@<sha>
        with:
          fetch-depth: 0 # historia completa para escanear commits previos
      - name: Gitleaks
        uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-gitleaks@<sha>
        with:
          fail-on-error: true
```

## Ejemplo con reporte SARIF + GitHub code scanning

```yml
permissions:
  contents: read
  security-events: write

steps:
  - uses: actions/checkout@<sha>
    with:
      fetch-depth: 0
  - uses: DevSecOps-Engineer-Org/reusable-workflows/actions/qa-gitleaks@<sha>
    with:
      report-format: sarif
      report-path: gitleaks-report.sarif
  - uses: github/codeql-action/upload-sarif@<sha>
    if: always()
    with:
      sarif_file: gitleaks-report.sarif
```

## Adopcion en un repo con secretos historicos (baseline)

1. Corre gitleaks con `fail-on-error: false` y guarda los hallazgos actuales
   como baseline (`report-format: json`, `report-path: .gitleaks-baseline.json`).
2. Comitea `.gitleaks-baseline.json` y en los siguientes runs pasalo como
   `baseline-path: .gitleaks-baseline.json` con `fail-on-error: true`.
3. Los hallazgos del baseline **no bloquean, pero siguen existiendo**: los
   secretos igualmente deben **rotarse y revocarse** en su plataforma
   (gitleaks detecta, no rota).

## Notas importantes

- El escaneo del historial requiere `fetch-depth: 0` en el checkout.
- Con checkouts shallow (PRs), el action hace fallback automatico a `--no-git`
  con un warning; para PRs normalmente basta el working directory.
- Si defines `redact: false`, el valor de los secretos podria pintarse en los
  logs del CI: normalmente NO quieres eso.
- La exclusion de falsos positivos se hace en el `.gitleaks.toml` del repo
  (allowlists con reglas por fingerprint), no en este action.
- Cuidado con reportes viejos en el working directory: en modo `--no-git`,
  gitleaks puede detectarse a si mismo (los reportes `.sarif`/`.json`
  contienen cadenas que parecen credenciales). Limpia los reportes antes del
  scan o ignoralos en el `.gitleaks.toml`.
