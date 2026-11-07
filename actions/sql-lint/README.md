# Action: SQL Lint (sqlfluff)

Linteo estatico de archivos y migraciones SQL usando [sqlfluff](https://docs.sqlfluff.com/), con:

- Soporte multi-dialecto (`postgres`, `mysql`, `mariadb`, `ansi`, `tsql`, `oracle`, `bigquery`, `snowflake`, etc.)
- Version pinneada de `sqlfluff` para consistencia entre repositorios
- Omision segura (`skipped=true`) cuando el repositorio aun no tiene las rutas SQL indicadas
- Configuracion por defecto optimizada para CI (`large_file_skip_byte_limit = 0` y `max_parse_nodes = 0`) si el repo no define su propio `.sqlfluff`
- Resumen en `$GITHUB_STEP_SUMMARY` con tabla de parametros y listado desplegable de violaciones

## Entradas

| Input | Default | Descripcion |
|---|---|---|
| `sql-paths` | `src/main/resources/db/migration` | Rutas (archivos o directorios) a lintear, separadas por coma o espacio |
| `dialect` | `postgres` | Dialecto SQL de sqlfluff (`postgres` \| `mysql` \| `mariadb` \| `ansi` \| `tsql` \| `oracle` \| ...) |
| `sqlfluff-version` | `4.4.0` | Version de sqlfluff a instalar (pinneada) |
| `config-path` | `""` | Ruta a archivo `.sqlfluff` propio; vacio autodectecta `.sqlfluff` en la raiz del repo |
| `fail-on-error` | `true` | `true` bloquea el job si existen violaciones; `false` solo emite warning |
| `extra-args` | `""` | Argumentos extra para `sqlfluff lint` |

## Salidas

- `violations`: numero total de violaciones encontradas
- `skipped`: `true` si se omitio el linteo porque ninguna de las rutas en `sql-paths` existe

## Ejemplo basico (migraciones Flyway / PostgreSQL)

```yml
name: SQL Lint
on: [push, pull_request]

permissions:
  contents: read

jobs:
  sql-lint:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@11d5960a326750d5838078e36cf38b85af677262

      - name: Lint SQL migrations
        uses: DevSecOps-Engineer-Org/reusable-workflows/actions/sql-lint@v1.0.2
        with:
          sql-paths: "src/main/resources/db/migration"
          dialect: postgres
          fail-on-error: "true"
```

## Ejemplo con multiples rutas y dialecto MySQL/MariaDB

```yml
      - name: Lint SQL scripts
        uses: DevSecOps-Engineer-Org/reusable-workflows/actions/sql-lint@v1.0.2
        with:
          sql-paths: "db/migrations,db/seeds,scripts/schema.sql"
          dialect: mariadb
          fail-on-error: "true"
```

## Configuracion personalizada en el repositorio (`.sqlfluff`)

Si tu proyecto necesita ajustar reglas (por ejemplo longitud de linea o excluir reglas especificas), agrega un archivo `.sqlfluff` en la raiz del repositorio y el action lo respetara automaticamente:

```ini
[sqlfluff]
dialect = postgres
exclude_rules = LT05, AM04
max_line_length = 120
large_file_skip_byte_limit = 0
max_parse_nodes = 0
```

## Notas importantes

- Si el repositorio no tiene un `.sqlfluff` propio ni se pasa `config-path`, el action inyecta una configuracion temporal con `large_file_skip_byte_limit = 0` y `max_parse_nodes = 0` para evitar que `sqlfluff` omita migraciones mayores a 20 KB en CI.
- Si Defines tu propio `.sqlfluff`, se recomienda incluir `large_file_skip_byte_limit = 0` en el para que todas las migraciones grandes sean evaluadas.
- Antes de ejecutar el linteo, el action valida que el `dialect` exista en la version instalada de `sqlfluff` y muestra la lista completa de dialectos soportados si hay un error de tipeo.
