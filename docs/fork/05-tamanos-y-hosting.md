# 05 — Tamaños y alojamiento en un servidor propio

Mediciones del 03/10/2026 para alojar el fork en un servidor propio, separado del que hoy sirve poke-generator. Las
cifras salen del árbol actual de cada repositorio de PokeAPI (API de árboles de Git), de las releases publicadas y
de una muestra de imágenes convertida con `sharp`. «MB» son megabytes decimales.

## Resumen

| Componente                                  | Archivos | Tamaño                                   | Observaciones                                                     |
| ------------------------------------------- | -------- | ---------------------------------------- | ----------------------------------------------------------------- |
| CSV fuente (`data/v2/csv`)                  | 187      | 43 MB                                    | Fuente de verdad; no es un formato para servir como API.          |
| Base SQLite construida (`db.sqlite3`)       | 1        | 128 MB                                   | Release `master-branch` de PokeAPI.                               |
| Dump PostgreSQL (`pokeapi.pgdump`)          | 1        | 12 MB comprimido                         | La base restaurada ocupa bastante más (a medir en la fase 0).      |
| Imagen Docker `pokeapi/pokeapi:master`      | —        | 43 MB                                    | Sin base ni sprites.                                              |
| JSON estático (`PokeAPI/api-data`)          | 15.537   | 506 MB con sangría · 300 MB minificado · **23 MB con gzip por archivo** | Lo que sirve pokeapi.co hoy.                                      |
| Sprites, repositorio completo               | 93.139   | 9,8 GB (más ~9 GB de historial Git si se clona) | 60 % son WebP y GIF animados de juegos que la app no usa.         |
| Sprites que usa poke-generator              | ~22.800  | ~1,57 GB                                 | Detalle abajo.                                                    |
| Sprites esenciales optimizados (WebP)       | ~6.700   | ~150 MB en 512 px · ~85 MB en 256 px     | Estimación desde una muestra de 40 imágenes.                     |
| Cries (`PokeAPI/cries`)                     | 2.000    | 28 MB                                    | OGG; la app no los usa.                                           |

## API de datos

### JSON estático (como pokeapi.co)

- 15.537 respuestas JSON con sangría de 2 espacios. La mayor parte del peso está en `pokemon/` (316 MB, por la lista
  de movimientos por juego) y `location-area/` (69 MB).
- Comprimen mucho: `pokemon/25/` pasa de 435 KB a 7,6 KB con gzip; la mediana de `pokemon/{id}/` es 145 KB en bruto.
  Guardando cada archivo ya comprimido, el conjunto ocupa 23 MB.
- `api-data` guarda **URLs relativas** (`/api/v2/move/268/`) y listados completos en cada `index.json`
  (`pokemon/index.json` trae las 1.351 entradas). pokeapi.co reescribe las URLs a absolutas
  (`https://pokeapi.co/api/v2/...`) y resuelve `limit`/`offset` y la búsqueda por nombre en su capa de despliegue.
  Un servidor propio debe hacer lo mismo: la app sigue `evolution_chain.url` tal cual, y una URL relativa apuntaría
  al dominio de la app.

### Django en vivo

- Imagen de 43 MB, PostgreSQL (dump de 12 MB; base restaurada de cientos de MB), Redis y Gunicorn con
  `2 × núcleos` procesos e hilos.
- Cada respuesta se calcula con consultas y serializadores pesados (`pokemon/{id}` agrupa cientos de movimientos);
  depende de la caché de Redis y cachalot para rendir.
- Sólo aporta algo frente al JSON estático si se necesita `?q=`, GraphQL (Hasura) o consultas dinámicas.

### Los 8 CSV que usa la app

| Uso                    | Archivos                                                               | Bruto  | gzip   |
| ---------------------- | ---------------------------------------------------------------------- | ------ | ------ |
| Runtime (cada sesión)  | `item_names`, `move_names`, `moves`, `move_changelog`                   | 763 KB | 263 KB |
| Scripts de la app      | `pokemon`, `pokemon_species`, `pokemon_forms`, `pokemon_form_names`     | 310 KB | 100 KB |

No sustituyen a la API: la complementan. Además son ineficientes: de `item_names.csv` (510 KB, todos los idiomas)
la app sólo usa las filas en español, que ocupan 41 KB (17 KB con gzip); de `move_names.csv`, 17 KB.

## Sprites

| Colección                     | Uso en la app                          | Archivos | Tamaño   |
| ----------------------------- | -------------------------------------- | -------- | -------- |
| `pokemon/other/home`          | Imagen principal de las cards          | 3.260    | 408 MB   |
| `pokemon/other/official-artwork` | Primer respaldo                     | 3.471    | 498 MB   |
| `pokemon/other/showdown`      | Respaldo (GIF animados)                | 6.300    | 610 MB   |
| `pokemon/other/dream-world`   | Respaldo (SVG)                         | 1.146    | 43 MB    |
| `pokemon/` raíz               | Último respaldo (sprites clásicos)     | 6.596    | 8 MB     |
| `items/`                      | Objetos                                | 2.035    | 2 MB     |
| `pokemon/versions/*`          | No se usa                              | 69.730   | 8,3 GB   |

Muestra convertida (20 HOME de 512 × 512 y 20 official-artwork de 475 × 475):

| Formato              | HOME      | official-artwork |
| -------------------- | --------- | ---------------- |
| PNG original         | 127 KB    | 129 KB           |
| WebP 512 px, q80     | 18 KB     | 24 KB            |
| WebP 256 px, q80     | 12 KB     | 13 KB            |
| AVIF 512 px, q55     | 13 KB     | 13 KB            |
| AVIF 256 px, q55     | 9 KB      | 8 KB             |

`build.py` necesita saber qué archivos existen para publicar URLs no nulas; para eso basta el listado de archivos
del repositorio de sprites, no hace falta servir ni clonar los 9,8 GB en el servidor de producción.

### Servir sólo los sprites de la app

Cobertura medida cruzando `pokemon.csv`, `pokemon_forms.csv` y el catálogo de poke-generator con los listados de
cada colección:

- Las 186 formas no predeterminadas del catálogo tienen sprite en `home` u `official-artwork`.
- De las 1.351 variedades, 16 no tienen HOME. Todas son `special-unavailable` en el catálogo de la app (no se
  ofrecen en el Team Builder):

| ID    | Variedad                   | Especie         | `official-artwork` | Shiny | Otros respaldos          |
| ----- | -------------------------- | --------------- | ------------------ | ----- | ------------------------ |
| 10080 | `pikachu-rock-star`        | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10081 | `pikachu-belle`            | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10082 | `pikachu-pop-star`         | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10083 | `pikachu-phd`              | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10084 | `pikachu-libre`            | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10085 | `pikachu-cosplay`          | Pikachu (#25)   | Sí                 | Sí    | showdown, clásico        |
| 10158 | `pikachu-starter`          | Pikachu (#25)   | Sí                 | No    | showdown, clásico        |
| 10159 | `eevee-starter`            | Eevee (#133)    | Sí                 | No    | showdown, clásico        |
| 10264 | `koraidon-limited-build`   | Koraidon (#1007) | No                | No    | Ninguno                  |
| 10265 | `koraidon-sprinting-build` | Koraidon (#1007) | Sí                | Sí    | Ninguno                  |
| 10266 | `koraidon-swimming-build`  | Koraidon (#1007) | No                | No    | Ninguno                  |
| 10267 | `koraidon-gliding-build`   | Koraidon (#1007) | Sí                | Sí    | Ninguno                  |
| 10268 | `miraidon-low-power-mode`  | Miraidon (#1008) | No                | No    | Ninguno                  |
| 10269 | `miraidon-drive-mode`      | Miraidon (#1008) | Sí                | Sí    | Ninguno                  |
| 10270 | `miraidon-aquatic-mode`    | Miraidon (#1008) | No                | No    | Ninguno                  |
| 10271 | `miraidon-glide-mode`      | Miraidon (#1008) | Sí                | Sí    | Ninguno                  |

  Las cuatro fisonomías con `official-artwork` son las que la app fija a mano (`MANUAL_OFFICIAL_ARTWORK_OVERRIDES`) y
  muestra en el carrusel del Pokémon del día; las otras cuatro de Koraidon y Miraidon no tienen ningún sprite en
  ninguna colección. Los Pikachu compañero y Eevee compañero no tienen `official-artwork` shiny: su variante shiny
  caería al sprite normal.
- `showdown`, `dream-world` y los sprites clásicos de la raíz **no rescatan ningún caso** que no cubran las dos
  colecciones anteriores.

Conjunto suficiente: `pokemon/other/home` (408 MB), `pokemon/other/official-artwork` (498 MB) e `items/` (2 MB):
~908 MB en PNG, o ~150 MB en WebP. Para servir sólo ese subconjunto hacen falta estos ajustes:

1. **Construir con el mismo subconjunto.** `build.py` publica una URL sólo si el archivo existe en el submódulo. Si
   se construye con el repositorio completo pero se sirve un subconjunto, la API publicaría URLs que dan `404`. Con
   un checkout disperso (`sparse-checkout` de esas tres carpetas, y clon parcial para no bajar los ~9 GB de
   historial), el build deja en `null` las colecciones no servidas sin tocar código. El workflow de publicación del
   fork debe usar ese checkout, no `submodules: recursive`.
2. **Misma estructura de rutas.** Conservar `sprites/pokemon/other/home/...` y `sprites/items/...` en el servidor y
   cambiar sólo `POKEAPI_SPRITES_PREFIX`.
3. **CORS en las imágenes.** La app carga los sprites con `crossorigin="anonymous"` (`PokemonCardMedia.vue`,
   `PokemonPreview.vue`, `spriteBounds.ts`). Sin `Access-Control-Allow-Origin` en la respuesta, el navegador no
   muestra la imagen.
4. **URLs que arma la propia app.** El manifiesto HOME por forma, los candidatos de objetos (`items/gen9`,
   `items/gen8`, raíz) y los cuatro `official-artwork` fijados a mano no salen de la API: usan
   `raw.githubusercontent.com` escrito en `pokemonService.ts`. La app necesita una base de sprites configurable,
   además del host en la CSP (`img-src`).
5. **Verificación.** Un chequeo en CI que compruebe que toda URL de sprite publicada en los JSON, y en los
   manifiestos de la app, existe en el servidor.
6. **Si se convierte a WebP más adelante:** servir la variante WebP en la misma URL `.png` según la cabecera
   `Accept` evita cambiar la configuración de `build.py` y la app; cambiar las extensiones obligaría a tocar ambas.

Los campos `showdown`, `dream_world` y los clásicos llegarían en `null`; la app ya filtra los candidatos vacíos, así
que no requiere cambios por eso. Si el fork quiere ser una API pública con todos los sprites, en cambio, haría falta
un prefijo por colección en `build.py` (propias en el servidor, el resto en GitHub).

## Comparación para un servidor aparte

| Criterio               | JSON estático precomprimido                         | Django en vivo                                 |
| ---------------------- | --------------------------------------------------- | ---------------------------------------------- |
| Disco (datos)          | ~325 MB (minificado + `.gz`)                        | Imagen + PostgreSQL + Redis (varios cientos de MB) |
| Memoria                | La del servidor web (decenas de MB)                 | PostgreSQL + Redis + Gunicorn (1 a 2 GB)       |
| CPU por petición       | Ninguna: lectura de archivo                         | Consultas y serialización                      |
| Operación              | Servidor web y reglas de ruteo                      | Base, migraciones, caché y procesos            |
| Paridad con pokeapi.co | Igual mecanismo                                     | Mismo JSON, distinto comportamiento de URLs y paginación |
| Actualización          | CI construye la base, genera JSON y sube archivos   | Reconstruir la base en el servidor             |

## Conclusión

- **El JSON estático es como sirve pokeapi.co hoy** y es la opción más barata y simple para un servidor propio:
  ~325 MB de disco, sin base de datos en producción y con Django sólo en CI para generarlo. Requiere configurar el
  servidor web (por ejemplo, nginx o Caddy con archivos `.gz` precomprimidos) con tres reglas: nombre → ID, barra
  final opcional y `limit`/`offset`, más la reescritura de URLs a absolutas al publicar.
- **Los 8 CSV no son una alternativa a la API**, sino una dependencia extra de la app que conviene retirar: o se
  sustituyen por respuestas del fork, o por un dataset filtrado en el build de la app (ER-13), que reduce los
  763 KB por sesión a unas decenas de KB.
- **Los sprites dominan el disco y el tráfico.** Servir sólo lo que usa la app en WebP baja de ~1,6 GB a ~150 MB y,
  por card, de ~127 KB a ~18 KB. Servir el repositorio completo (9,8 GB) sólo tiene sentido si el fork quiere ser
  un espejo público de todos los sprites.
- **Cries:** 28 MB; se pueden alojar sin coste relevante, aunque la app no los usa hoy.
- **Total orientativo** del servidor propio: menos de 1 GB con datos, sprites optimizados y cries; ~2 GB si se
  conservan los PNG originales de las colecciones usadas. El coste real lo marcará el tráfico de imágenes, no el
  almacenamiento.
