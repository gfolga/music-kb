# Wiki schema

Este repo implementa el patrón de wiki mantenida por LLM descrito por Karpathy
(https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): en vez de
RAG clásico (recuperar de fuentes crudas en el momento de la consulta), el LLM
construye y mantiene incrementalmente una wiki persistente e interconectada
que vive entre el usuario y las fuentes crudas.

Es la wiki de **música** de Gustavo: artistas, bandas, canciones y playlists.
Mismo patrón que el repo padre `../` (personal-kb: finanzas, viajes,
inversiones), el hermano `../health-kb` (salud) y `../../research`
(investigación). Vive en GitHub como `gfolga/music-kb`.

## Estructura

- `raw/` — fuentes crudas e inmutables (exports de playlists, letras,
  reseñas, notas de escucha, capturas). El LLM las lee pero nunca las
  modifica. Son la fuente de verdad.
  - `raw/playlists/` — export original de una playlist. Dos archivos por playlist:
    `<slug>.csv` (columnas: name, artist, album, spotify_uri) para herramientas
    como Exportify o TuneMyMusic; `<slug>-uris.json` (array de URIs + metadata)
    para reimportar vía `spotify_cli` o la API de Spotify. La página de síntesis
    vive en `wiki/playlists/`, no acá.
  - `raw/listening-stats/` — snapshots de escuchas de Gustavo por artista.
    Un archivo por artista: `<slug>-stats.json` con campos `artist`,
    `spotify_uri`, `extracted` (fecha ISO), `period`, `total_plays` y
    `top_tracks` (array con `title`, `uri`, `plays`). Al actualizar, agregar
    un nuevo archivo con sufijo de fecha (`<slug>-stats-2026-10.json`) en
    vez de sobreescribir, para preservar histórico. La sección "Escuchas"
    de la wiki referencia el archivo raw en vez de duplicar los datos.
  - `raw/articles/` — reseñas, entrevistas, notas.
  - `raw/assets/` — tapas, capturas y otros adjuntos.
- `wiki/` — capa de síntesis, generada y mantenida enteramente por el LLM.
  - `wiki/index.md` — catálogo de todas las páginas, con un resumen de una
    línea cada una, organizado por categoría. Se actualiza en cada ingest.
  - `wiki/log.md` — registro cronológico append-only de ingests, queries y
    pasadas de mantenimiento (lint). Formato parseable con prefijo por
    entrada.
  - `wiki/entities/` — una página por artista o banda.
  - `wiki/songs/` — una página por canción. También son entidades
    (`type: entity`); viven en carpeta propia para no mezclarse con el
    artista del mismo nombre.
  - `wiki/albums/` — una página por álbum o compilación (`type: album`).
    Incluye tracklist, datos de grabación, y contexto del lanzamiento.
    URI de Spotify en el frontmatter cuando existe (`spotify_uri`).
  - `wiki/playlists/` — una página por playlist (`type: playlist`).
  - `wiki/concepts/` — síntesis que cruza varios artistas, canciones o
    playlists (un género, una época, un hilo de escucha).
  - `wiki/summaries/` — un resumen por fuente ingerida cuando el volumen lo
    justifica (una reseña larga, un libro). Un export de playlist no necesita
    summary propio: alcanza la página de `wiki/playlists/`.
  - `wiki/comparisons/` — análisis que sale de una pregunta y vale la pena
    conservar.
- `CLAUDE.md` — este archivo. El schema: convenciones, nombres y workflows.
  Co-evoluciona con el uso; si un patrón se repite, se agrega acá.

## Convenciones de nombres

- Archivos en `kebab-case.md`, sin fechas en el nombre (la fecha vive en el
  contenido y en `log.md`).
- Artista o banda: `wiki/entities/<slug>.md` (ej. `radiohead.md`).
- Canción: `wiki/songs/<titulo-slug>.md`. Si el título ya existe,
  desambiguar con el artista: `<titulo-slug>-<artista-slug>.md`.
- Playlist: `wiki/playlists/<slug>.md`.
- Álbum o compilación: `wiki/albums/<artista-slug>-<titulo-slug>.md`. Para
  compilaciones de varios artistas, omitir el slug de artista:
  `wiki/albums/<titulo-slug>.md`.
- Los links entre páginas usan rutas relativas de Markdown:
  `[texto](../entities/foo.md)`, `[texto](../songs/foo.md)`,
  `[texto](../albums/foo.md)`, `[texto](../playlists/foo.md)`.
- Cada página de wiki empieza con un frontmatter mínimo:

  ```markdown
  ---
  title: Título legible
  type: entity | album | playlist | concept | summary | comparison
  sources: [raw/playlists/foo.csv, ...]
  updated: YYYY-MM-DD
  ---
  ```

- Omitir secciones que todavía no tienen datos, en vez de dejarlas con
  placeholder.

## Workflows

### Ingest (agregar una fuente nueva)

1. Guardar la fuente sin modificar en `raw/`. Si trae imágenes, van a
   `raw/assets/`.
2. Leer la fuente completa y discutir los puntos clave con el usuario si hace
   falta contexto (es habitual: muchas entradas van a ser relatos de qué
   estaba escuchando, no un documento).
3. Escribir `wiki/summaries/<nombre>.md` solo si la fuente es rica. Un
   export de playlist no lleva summary: se sintetiza en `wiki/playlists/`.
4. Crear o actualizar las páginas de artista, canción y playlist que la
   fuente toca.
5. Agregar la entrada en `wiki/index.md`.
6. Agregar una línea en `wiki/log.md`:
   `## [YYYY-MM-DD] ingest | <título de la fuente>`

### Query (responder una pregunta usando la wiki)

1. Buscar en `wiki/` (no en `raw/`, salvo que haga falta verificar una cita)
   las páginas relevantes.
2. Sintetizar la respuesta citando las páginas de wiki usadas.
3. Si la respuesta es síntesis nueva y vale la pena conservarla, ofrecer
   archivarla en `wiki/comparisons/` o en la página de `concepts/`
   correspondiente.
4. Agregar una línea en `wiki/log.md`:
   `## [YYYY-MM-DD] query | <pregunta resumida>`

### Lint (mantenimiento periódico)

Pasada de salud de la wiki: contradicciones entre páginas, canciones
huérfanas (sin link desde un artista o una playlist), artistas mencionados
sin página, y referencias cruzadas faltantes. Registrar en `wiki/log.md`:
`## [YYYY-MM-DD] lint | <hallazgos>`

## Dominio: artistas y bandas

`wiki/entities/<slug>.md`, `type: entity`. Una página por artista o banda,
no una por álbum.

Secciones, en este orden, omitiendo las vacías:

- `## Datos` — si es banda o solista, origen, años activos, géneros,
  miembros cuando importan.
- `## Discografía` — álbumes y EPs que ya aparecieron en una fuente. No
  completar la discografía de oídas ni desde la web salvo que el usuario
  lo pida. Tabla `| Año | Título | Tipo | Notas |`.
- `## Canciones` — links a `wiki/songs/`. Solo las que ya tienen página.
- `## Playlists` — en qué playlists de esta wiki aparece.
- `## Notas`

Cuando una canción o playlist menciona un artista por primera vez, crear
la página aunque sea corta (`## Datos` + el link de vuelta). No dejar el
nombre solo como texto si ya hay una canción propia.

## Dominio: canciones

`wiki/songs/<slug>.md`, `type: entity`.

Secciones:

- `## Datos` — artista (link a `wiki/entities/`), álbum, año, y duración
  solo si la fuente la trae.
- `## Por qué está` — el motivo de Gustavo: cuándo la escuchó, con qué la
  asocia, qué versión prefiere. Es la sección que más valor acumula. Si
  todavía no hay motivo, no inventarlo: omitir la sección.
- `## Playlists` — links a las playlists donde figura.
- `## Notas`

No crear una página de canción por cada fila de una playlist. La tabla de
la playlist lista todos los temas; la página de canción se abre cuando hay
algo que decir (un recuerdo, una versión, un cruce con otra playlist) o
cuando el usuario pide trackearla.

## Dominio: playlists

`wiki/playlists/<slug>.md`, `type: playlist`.

El export crudo queda en `raw/playlists/`. Esta página es la síntesis.

Secciones:

- `## Datos` — de dónde salió (Spotify, Apple Music, armada a mano), URL
  si existe, fecha, para qué es.
- `## Temas` — orden de la playlist, tabla `| # | Canción | Artista | Página |`.
  La columna Página lleva el link a `wiki/songs/` cuando esa canción tiene
  página, y queda vacía cuando no.
- `## Notas` — cambios de orden, temas sacados, duplicados entre playlists.

Al ingerir una playlist nueva: crear o actualizar la página, crear la
página de artista para cada artista que ya tenga al menos una canción con
página, y no abrir una página de canción por cada tema (ver dominio
canciones).

## Escala

Pensado para un volumen moderado (decenas de playlists, cientos de
canciones con página) sin infraestructura de embeddings. Si crece más,
considerar una herramienta de búsqueda local (grep/ripgrep sobre `wiki/`,
o un índice tipo `qmd`) antes de saltar a RAG vectorial.
