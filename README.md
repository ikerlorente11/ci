# ci

Workflows comunes de GitHub Actions para todos mis proyectos, para que todos sigan el mismo patrón.

| Workflow | Qué hace |
|---|---|
| `python.yml` | `pip install ".[dev]"` + `pytest` (Python 3.12, caché de pip) |
| `node.yml` | `npm ci` + la comprobación que se le pase (por defecto `npm test`) |
| `image.yml` | Imagen arm64 a `ghcr.io/ikerlorente11/<name>:latest` y `:<sha>`, que es lo que descargan las Pis |

`image.yml` usa el runner ARM nativo en los repos públicos (gratis solo para ellos) y x86 + QEMU en los privados.
Con `schedule` construye sin caché, para que entren las actualizaciones de la imagen base y de las dependencias.

## Plantilla de un servicio de las Pis

```yaml
name: CI
on:
  push:
    branches: [main]
  pull_request:
  workflow_dispatch: {}
  schedule:
    - cron: "0 5 * * 1"   # lunes: imagen sin caché

concurrency:
  group: ci-${{ github.ref }}
  cancel-in-progress: true

jobs:
  tests:
    uses: ikerlorente11/ci/.github/workflows/python.yml@main
  image:
    if: github.ref_name == github.event.repository.default_branch || github.event_name == 'schedule'
    needs: tests
    uses: ikerlorente11/ci/.github/workflows/image.yml@main
    permissions:
      contents: read
      packages: write
    with:
      name: <imagen>
```

## Dockerfile: las dependencias en su propia capa

Para que un cambio de código no reinstale todas las dependencias (bajo QEMU eso son minutos):

```dockerfile
COPY pyproject.toml ./
RUN python -c "import tomllib; print(*tomllib.load(open('pyproject.toml', 'rb'))['project']['dependencies'], sep=chr(10))" > /tmp/deps.txt \
    && pip install --no-cache-dir -r /tmp/deps.txt
COPY src ./src
ENV PYTHONPATH=/srv/src   # el código no se instala: corre desde src
```

En Node, lo mismo con `COPY package.json package-lock.json` + `npm ci` antes de copiar el código, y la etapa que
compila con `FROM --platform=$BUILDPLATFORM` (el resultado es igual en todas las plataformas, así que se compila nativo).

## En la Pi

El `compose.yaml` del servicio apunta a la imagen de GHCR; para desplegar: `docker compose pull && docker compose up -d`.

Licencia: PolyForm Noncommercial 1.0.0.
