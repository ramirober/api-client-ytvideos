# API Client for Youtube Videos

## Simple API Client in Rust for consuming the Youtube Videos public API

### Author: Ramiro Berbetoros

---

## Descripción

Worker (proceso de línea de comandos) escrito en Rust que consulta el **YouTube Data
API v3**, obtiene la metadata de un video y la persiste en una base SQLite siguiendo
un paradigma de "second brain": cada recurso externo se guarda de forma polimórfica
(JSON en una columna `TEXT`) bajo un mismo esquema de tablas, permitiendo versionar
los cambios de schema del proveedor a lo largo del tiempo.

Uso previsto:

```bash
cargo run -- "https://www.youtube.com/watch?v=dQw4w9WgXcQ"
# también acepta el ID pelado:
cargo run -- dQw4w9WgXcQ
```

---

## Contrato con el API externo (YouTube Data API v3)

Este es el "contrato" que el cliente espera/respeta al hablar con YouTube. La versión
vigente del API es **v3** (no existe una v4 publica todavia). Documentacion oficial:
<https://developers.google.com/youtube/v3/docs/videos>.

### Endpoint

```
GET https://www.googleapis.com/youtube/v3/videos
```

### Autenticacion

Mediante **API key** pasada como query param `key`. La key se genera en Google Cloud
Console habilitando "YouTube Data API v3". Para leer metadata de videos **públicos**
con `videos.list` no es necesario OAuth: la API key es suficiente.

### Request

| Parámetro | Valor                                                   | Obligatorio |
| --------- | ------------------------------------------------------- | ----------- |
| `part`    | `snippet,contentDetails,statistics,status,topicDetails` | Sí          |
| `id`      | `<VIDEO_ID>` (ej. `dQw4w9WgXcQ`)                        | Sí          |
| `key`     | `<API_KEY>`                                             | Sí          |

**Costo de cuota:** 1 unidad por llamada (cuota diaria por defecto: 10.000 unidades).

**Input del cliente.** El programa recibe una URL de YouTube o un ID. Se extrae el
`VIDEO_ID` de cualquiera de estos formatos:

- `https://www.youtube.com/watch?v=<ID>`
- `https://youtu.be/<ID>`
- `https://www.youtube.com/shorts/<ID>`
- `<ID>` (11 caracteres, sin URL)

Ejemplo de request completo:

```
GET https://www.googleapis.com/youtube/v3/videos?part=snippet,contentDetails,statistics,status,topicDetails&id=dQw4w9WgXcQ&key=API_KEY
```

### Response esperada (schema)

La respuesta es un objeto con un array `items`. Como consultamos por un único `id`,
esperamos `items[0]`. Estructura:

| Campo   | Tipo   | Descripción                           |
| ------- | ------ | ------------------------------------- |
| `kind`  | string | Siempre `"youtube#videoListResponse"` |
| `etag`  | string | ETag de la respuesta                  |
| `items` | array  | Lista de recursos `youtube#video`     |

Cada elemento de `items` (recurso `video`) tiene `kind`, `etag`, `id` y los `part`
solicitados. Campos relevantes por part:

#### `snippet`

| Campo                  | Tipo           | Notas                                         |
| ---------------------- | -------------- | --------------------------------------------- |
| `publishedAt`          | datetime (ISO) | Fecha de publicación                          |
| `channelId`            | string         |                                               |
| `title`                | string         | Máx. 100 caracteres                           |
| `description`          | string         | Máx. 5000 bytes                               |
| `thumbnails`           | object         | `default`/`medium`/`high`/`standard`/`maxres` |
| `channelTitle`         | string         |                                               |
| `tags`                 | string[]       | Opcional                                      |
| `categoryId`           | string         |                                               |
| `liveBroadcastContent` | string         | `none` \| `live` \| `upcoming`                |
| `defaultLanguage`      | string         | Opcional                                      |
| `localized`            | object         | `title`, `description`                        |
| `defaultAudioLanguage` | string         | Opcional                                      |

#### `contentDetails`

| Campo               | Tipo    | Notas                                |
| ------------------- | ------- | ------------------------------------ |
| `duration`          | string  | Duración ISO 8601 (ej. `PT4M13S`)    |
| `dimension`         | string  | `2d` \| `3d`                         |
| `definition`        | string  | `hd` \| `sd`                         |
| `caption`           | string  | `"true"` \| `"false"` (string)       |
| `licensedContent`   | boolean |                                      |
| `regionRestriction` | object  | `allowed[]` / `blocked[]` (opcional) |
| `contentRating`     | object  | Ratings por país (opcional)          |
| `projection`        | string  | `rectangular` \| `360`               |

#### `statistics`

| Campo           | Tipo            | Notas                                 |
| --------------- | --------------- | ------------------------------------- |
| `viewCount`     | string (uint64) | Numérico como string                  |
| `likeCount`     | string (uint64) | Opcional                              |
| `dislikeCount`  | string (uint64) | **Retirado** (privado desde dic-2021) |
| `favoriteCount` | string (uint64) | **Deprecado**, siempre `0`            |
| `commentCount`  | string (uint64) | Opcional (puede estar deshabilitado)  |

#### `status`

| Campo                     | Tipo    | Notas                               |
| ------------------------- | ------- | ----------------------------------- |
| `uploadStatus`            | string  | `processed` \| `uploaded` \| ...    |
| `privacyStatus`           | string  | `public` \| `private` \| `unlisted` |
| `license`                 | string  | `youtube` \| `creativeCommon`       |
| `embeddable`              | boolean |                                     |
| `publicStatsViewable`     | boolean |                                     |
| `madeForKids`             | boolean |                                     |
| `selfDeclaredMadeForKids` | boolean | Opcional                            |

#### `topicDetails`

| Campo              | Tipo     | Notas                                        |
| ------------------ | -------- | -------------------------------------------- |
| `topicCategories`  | string[] | URLs de Wikipedia que describen el contenido |
| `relevantTopicIds` | string[] | **Deprecado** / puede no venir               |

### Schema mutable / polimorfismo

El schema del proveedor **no es estable**: hay campos opcionales que pueden aparecer
o desaparecer según el video y la evolución del API. Por eso se persiste como JSON
crudo (`meta_payload`) en lugar de columnas rígidas. Casos a tener en cuenta:

- `liveStreamingDetails` solo aparece en transmisiones en vivo.
- `dislikeCount` fue retirado; `favoriteCount` está deprecado (siempre `0`).
- `topicDetails.relevantTopicIds` / `topicIds` están deprecados.
- `regionRestriction`, `contentRating`, `tags`, `defaultLanguage` son opcionales.

---

## Mapeo a la base de datos

La respuesta del API se proyecta sobre la tabla `assets`:

| Columna        | Origen                                                      |
| -------------- | ----------------------------------------------------------- |
| `asset_uri`    | `https://www.youtube.com/watch?v=<id>` (canónica, `UNIQUE`) |
| `title`        | `snippet.title`                                             |
| `entity`       | `"video"` (constante)                                       |
| `provider`     | `"youtube"` (constante)                                     |
| `meta_payload` | JSON completo de los `part` capturados (`items[0]`)         |
| `created_at`   | Gestionado por la DB                                        |
| `updated_at`   | Gestionado por la DB en cada actualización                  |

### Versionado de schema

- `assets.meta_payload` mantiene **siempre el JSON vigente** del recurso.
- Cuando el schema cambia (o se actualiza el asset), el snapshot previo de la fila se
  guarda en `assets_history.history_payload` antes de mutar, preservando el historial
  de versiones del recurso.

Schema de las tablas (ver sección _Estado / Base de datos_):

```sql
CREATE TABLE IF NOT EXISTS assets (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    asset_uri TEXT UNIQUE NOT NULL,
    title TEXT,
    entity TEXT NOT NULL,
    provider TEXT NOT NULL,
    created_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    updated_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    meta_payload TEXT -- JSON polimórfico con la metadata del proveedor
);

CREATE TABLE IF NOT EXISTS assets_history (
    id INTEGER PRIMARY KEY AUTOINCREMENT,
    asset_id INTEGER NOT NULL,
    changed_at DATETIME DEFAULT CURRENT_TIMESTAMP,
    history_payload TEXT NOT NULL, -- Snapshot JSON completo de la fila antes de mutar
    FOREIGN KEY (asset_id) REFERENCES assets (id) ON DELETE CASCADE
);
```

---

## Estado / Base de datos

La base SQLite `redes2.db` **ya está creada y testeada** (tablas `assets` y
`assets_history`). El código del worker la tratará como un placeholder existente: no
se recrea en esta fase. La capa de persistencia y el cliente HTTP en Rust se
implementarán en una etapa posterior.
