```
 ███╗   ███╗██╗   ██╗███████╗██╗ ██████╗    ██╗  ██╗██████╗
 ████╗ ████║██║   ██║██╔════╝██║██╔════╝    ██║ ██╔╝██╔══██╗
 ██╔████╔██║██║   ██║███████╗██║██║         █████╔╝ ██████╔╝
 ██║╚██╔╝██║██║   ██║╚════██║██║██║         ██╔═██╗ ██╔══██╗
 ██║ ╚═╝ ██║╚██████╔╝███████║██║╚██████╗    ██║  ██╗██████╔╝
 ╚═╝     ╚═╝ ╚═════╝ ╚══════╝╚═╝ ╚═════╝    ╚═╝  ╚═╝╚═════╝
```

> Una wiki de música mantenida por un LLM. No es un Notion. No es un RAG. Es algo más raro.

---

## Qué es esto

Este repo implementa el patrón descrito por Andrej Karpathy en su [gist sobre LLM wikis](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f): en vez de hacer RAG sobre fuentes crudas en el momento de la consulta, el LLM construye y mantiene **una wiki persistente e interconectada** que vive entre el usuario y sus fuentes.

La idea es simple: cuando escucho algo que me interesa, le hablo a Kit (el agente de Spotify Studio). Kit verifica los datos, escribe las páginas, actualiza los índices y hace push. La wiki crece a medida que escucho.

Es mi memoria musical externa. Todo lo que sé sobre lo que escucho, de dónde viene, cómo se conecta.

---

## Estructura

```
music-kb/
├── raw/                        # Fuentes crudas — el LLM las lee, nunca las toca
│   ├── playlists/              # Exports de playlists en CSV + JSON de URIs
│   └── listening-stats/        # Snapshots de escuchas por artista (Spotify)
│
└── wiki/                       # Síntesis — generada y mantenida por el LLM
    ├── index.md                # Catálogo de todas las páginas
    ├── log.md                  # Registro cronológico de ingests
    ├── entities/               # Artistas y bandas
    ├── songs/                  # Canciones que merecen su propia página
    ├── albums/                 # Álbumes y compilaciones
    ├── playlists/              # Playlists propias (síntesis de raw/)
    ├── concepts/               # Géneros, épocas, hilos de escucha
    ├── summaries/              # Resúmenes de fuentes extensas
    └── comparisons/            # Análisis que vale la pena conservar
```

---

## Qué hay adentro (hoy)

**Artistas:**
T. Rex · Nick Cave · David Bowie · Morrissey · Los Espíritus · Tótem Uruguay · El Kinto · Eduardo Mateo · Eduardo Darnauchans · Jaime Roos · Francis Andreu · Estela Magnone · Angine de Poitrine

**Álbumes:**
Charco: Canciones del Río de la Plata (2017)

**Canciones:**
Cosmic Dancer · No Dejes Que

**Playlists:**
Rada · Tango Nuevo · Math Rock: Historia y Exponentes

**Conceptos:**
Integración musical rioplatense · Math Rock

---

## Cómo se actualiza

No hay scripts. No hay pipeline. El flujo es:

1. Escucho algo
2. Le digo a Kit: *"crea la entidad para X"*, *"ingestá esta playlist"*, *"dame info de esta canción"*
3. Kit verifica en fuentes abiertas, escribe las páginas y hace push
4. El repo crece

Kit también exporta los datos crudos de Spotify (playlists con URIs, stats de escucha) a `raw/` antes de sintetizarlos en `wiki/`.

---

## Convenciones

- Todo en `wiki/` está en español
- Las páginas usan frontmatter mínimo: `title`, `type`, `sources`, `updated`
- Los links entre páginas son rutas relativas de Markdown
- `raw/` es inmutable: el LLM escribe ahí solo al ingestar una fuente nueva
- Las stats de escucha son snapshots con fecha — no se sobreescriben, se acumulan
- El `CLAUDE.md` en el root es el schema completo que el LLM lee al arrancar

---

## Inspiración

- [Andrej Karpathy — LLM Wiki pattern](https://gist.github.com/karpathy/442a6bf555914893e9891c11519de94f)
- Hermano de `personal-kb` (finanzas, viajes) y `health-kb`

---

*Mantenido por Kit · Spotify Studio · `gfolga/music-kb`*
