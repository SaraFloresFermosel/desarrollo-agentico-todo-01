# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Proyecto

Lista de tareas web en Go con [Buffalo](https://gobuffalo.io) (HTML renderizado en servidor con plantillas Plush) y PostgreSQL vía Pop. Los textos de la interfaz, los mensajes de validación, los comentarios y los mensajes de commit están en español. Los commits siguen Conventional Commits (`feat:`, `build:`, `ci:`…).

## Comandos

La base de datos de desarrollo se levanta con Docker; `database.yml` lee el host de `DB_HOST` (por defecto `127.0.0.1`) y el entorno de `GO_ENV` (por defecto `development`).

```bash
docker compose up -d db                 # solo Postgres (puerto 5432)
docker compose up -d --build            # app + Postgres, en http://localhost:3000
go run ./cmd/app                        # app en local (aplica migraciones al arrancar)
buffalo dev                             # recarga en caliente (.buffalo.dev.yml); el CLI se instala en el dev container
```

Tests (necesitan la base `todo_test` creada y migrada):

```bash
docker compose exec -T db createdb -U postgres todo_test
GO_ENV=test go run github.com/gobuffalo/pop/v6/soda@v6.4.1 migrate -e test
GO_ENV=test go test -p 1 ./...
GO_ENV=test go test -p 1 ./actions -run Test_ActionSuite -testify.m Test_TasksUpdate_Valid   # un solo test
```

`-p 1` es obligatorio: los paquetes `actions` y `models` comparten la misma base de datos y la suite la vacía entre tests.

El CI (`.github/workflows/ci.yaml`) ejecuta: `gofmt -l .` (debe salir vacío), `go vet ./...`, `soda migrate -e test`, `go test -p 1 ./...`, `go build ./...` y `docker build`.

Si el puerto 5432 del host lo ocupa otro Postgres, se puede ejecutar Go dentro de la red de compose, donde el host es `db`:

```bash
docker run --rm --network <proyecto>_default -v "$PWD:/src" -w /src -e GO_ENV=test -e DB_HOST=db golang:1.26-alpine go test -p 1 ./...
```

En Windows, con `core.autocrlf=true`, `gofmt -l` lista todos los ficheros por los CRLF; comprobar el formato sobre ficheros con LF.

## Arquitectura

- `cmd/app/main.go`: aplica las migraciones (`models.Migrate()`) y arranca `actions.App()`. No hay paso de migración aparte en producción.
- `models/models.go`: conecta `models.DB` en un `init()` según `GO_ENV`, así que importar `models` (o `actions`) ya exige una configuración de base de datos válida.
- `actions/app.go`: todas las rutas y el middleware. Cada petición se envuelve en una transacción (`popmw.Transaction`); los handlers la obtienen con `c.Value("tx").(*pop.Connection)`. Hay protección CSRF activa.
- `migrations/`, `templates/` y `public/` se embeben con `go:embed` (cada paquete expone `FS()`). `templates/embed.go` solo incluye `* */*`: las plantillas pueden estar como mucho un subdirectorio por debajo. Las migraciones son `.fizz` (esquema) y `.sql` (datos de ejemplo idempotentes con `ON CONFLICT DO NOTHING`).

Patrones que siguen los handlers de `actions/tasks.go`:

- Buscar por `task_id` con `tx.Find`; si falla, `c.Error(http.StatusNotFound, err)` (también cubre IDs que no son UUID).
- Recortar el título con `strings.TrimSpace` y guardar con `tx.ValidateAndCreate`/`ValidateAndUpdate`. La validación vive en `Task.Validate` (`models/task.go`; `TitleMaxLength` = 200, también en la columna y en el `maxlength` de los formularios).
- Si hay errores de validación: volver a renderizar la plantilla con `task` y `errors` y estado 422. Si todo va bien: mensaje flash `success` y redirección 303 a `/`.
- `Task` solo permite enlazar `title` desde formularios (el resto de campos lleva `form:"-"`).
- Los formularios incluyen `authenticity_token` (CSRF), y los PUT/DELETE se envían como POST con un campo oculto `_method`.

Tests: suites de testify de `gobuffalo/suite` (`ActionSuite` en `actions/actions_test.go`, `ModelSuite` en `models/models_test.go`), con escenarios de datos en `fixtures/*.toml` que se cargan con `as.LoadFixture("tres tareas")`. Los tests de acciones usan `as.HTML(...).Get/Post/Put/Delete`.
