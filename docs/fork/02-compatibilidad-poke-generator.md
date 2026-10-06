# 02 — Compatibilidad con poke-generator

Objetivo: que Himitsu Kichi (`Plathinnum/poke-generator`) pueda pasar de pokeapi.co al fork sin cambios bruscos ni
fallos. Este documento describe qué consume hoy la app, qué supuestos no pueden romperse y en qué orden conviene hacer
la transición. El análisis no modificó poke-generator.

## Qué consume hoy la app

Todo el acceso remoto vive en `src/services/pokemonService.ts` (constante `API = 'https://pokeapi.co/api/v2'`) y en los
scripts de `scripts/`, que generan datos versionados en `src/data/`.

### En runtime (navegador)

| Petición                                  | Búsqueda por | Campos que se leen                                                                                                   |
| ----------------------------------------- | ------------ | -------------------------------------------------------------------------------------------------------------------- |
| `pokemon/{id}`                            | ID           | `id`, `name`, `types`, `abilities[]` (`ability.name`, `is_hidden`, `slot`), `moves[]` (`move`, `version_group_details[].version_group.name`), `sprites` (raíz, `other.home`, `other.official-artwork`, `other.showdown`, `other.dream_world`, variantes `female` y `shiny`) |
| `pokemon-form/{id}`                       | ID           | `id`, `name`, `sprites`, `types`                                                                                     |
| `pokemon-species/{id}`                    | ID           | `shape`, `egg_groups`, `is_legendary`, `is_mythical`, `genera`, `evolution_chain.url`, `flavor_text_entries`          |
| `evolution-chain/{id}/`                   | URL absoluta | `chain` recursivo (`species.url`, `evolves_to`); la URL se toma tal cual de `pokemon-species`                       |
| `type/{nombre}`                           | Nombre       | `pokemon[].pokemon.url` (de donde se extrae el ID)                                                                   |
| `item-pocket/berries`                     | Nombre       | `categories[].url`, que luego se sigue tal cual                                                                      |
| `item-category/{slug}` (12 categorías)    | Nombre       | `items[]` (`name`, `url`)                                                                                            |
| `item/{id}`                               | ID           | `id`, `name`, `names`, `sprites.default`                                                                             |
| `move/{id o slug}`                        | Ambos        | `id`, `name`, `names`, `type`, `damage_class`, `power`, `accuracy`, `pp`, `priority`                                 |
| `ability/{slug}`                          | Nombre       | `id`, `name`, `names`                                                                                                |
| `nature/{id}`, `nature/{nombre}`          | Ambos        | `id`, `name`, `names`, `increased_stat`, `decreased_stat`                                                            |
| `nature?limit=30`                         | Listado      | `results[].name`                                                                                                     |

Además descarga en runtime cuatro CSV **directamente de la rama `master` de `PokeAPI/pokeapi`** en GitHub:
`item_names.csv`, `move_names.csv`, `moves.csv` y `move_changelog.csv`. Los parsea por posición de columna y filtra
`local_language_id = 7` (español). También pide imágenes a `raw.githubusercontent.com/PokeAPI/sprites/master/...`
(sprites HOME por forma, objetos en `items/gen9`, `items/gen8` y raíz, y cuatro `official-artwork` fijados a mano).

### En scripts de mantenimiento

| Script                              | Fuente                                                                                         | Salida                                         |
| ----------------------------------- | ---------------------------------------------------------------------------------------------- | ---------------------------------------------- |
| `syncPokemonFormCatalog.mjs`        | `pokemon.csv`, `pokemon_species.csv`, `pokemon_forms.csv` de `master`                          | `src/data/pokemonFormCatalog.json` (~990 KB)   |
| `syncPokemonFormNames.mjs`          | `pokemon?limit=5000`, `pokemon_forms.csv`, `pokemon_form_names.csv`                           | `src/data/pokemonFormNames.generated.ts`       |
| `syncPokemonHomeFormSprites.mjs`    | `pokemon_forms.csv` y la API de GitHub sobre el árbol de `PokeAPI/sprites`                      | `src/data/pokemonHomeFormSprites.generated.ts` |
| `effectDescriptions.mjs --review`   | `pokeapi.co/api/v2` (efectos y textos de sabor)                                                | Caché local para revisión manual               |

### Fuera del código

- **Pruebas E2E.** `e2e/support/network.ts` sólo reproduce los hosts `pokeapi.co` y `raw.githubusercontent.com`, y
  guarda las grabaciones en `e2e/fixtures/network/<host>/...`. Un host nuevo se bloquearía en las pruebas hasta grabarlo.
- **Pruebas unitarias.** `pokemonService.test.ts` compara URLs completas de `pokeapi.co` y `raw.githubusercontent.com`.
- **CSP.** `public/_headers` permite `connect-src https://pokeapi.co https://raw.githubusercontent.com` e
  `img-src https://raw.githubusercontent.com` (hoy en modo informe).

## Supuestos que el fork no puede romper

1. **IDs estables.** Los equipos guardados (localStorage y Supabase) persisten `pokemonId`, `formId`, `moveIds`,
   `abilityId`, `natureId` e IDs de objetos; el catálogo generado también se indexa por `pokemonId` y `formId`
   (por ejemplo, Magearna (#801) Color Vetusto es `10147`). Una renumeración, propia o heredada de PokeAPI original,
   corrompería equipos existentes. **Toda sincronización con PokeAPI original debe comprobar que no cambian IDs.**
2. **Nombres técnicos estables.** La app busca por slug (`ability/{slug}`, `move/{slug}`, `type/{nombre}`,
   `item-category/{slug}`) y su catálogo contiene cientos de slugs literales. Renombrar un identificador equivale a
   romper un contrato.
3. **Búsqueda por nombre e ID, con y sin barra final.** La app pide `pokemon/1` (sin barra) y sigue URLs con barra.
4. **Formato de `url`.** Extrae IDs con `url.split('/').slice(-2, -1)`: el ID debe ser el penúltimo segmento y la
   URL debe terminar en `/`.
5. **URLs absolutas del mismo origen.** `evolution_chain.url` y `categories[].url` se siguen sin reescribir: deben
   apuntar al host del fork, no a pokeapi.co ni a `localhost`.
6. **Idiomas.** `language.name` `es` (España, ID 7), `es-419` (ID 14) y `en`; los CSV se filtran por el ID 7.
7. **Grupos de versión.** La app usa los nombres `scarlet-violet`, `sword-shield`, `ultra-sun-ultra-moon` y los IDs
   25, 20 y 18 para interpretar `move_changelog.csv`.
8. **Columnas de CSV.** `moves.csv`, `move_changelog.csv` y los `*_names.csv` se leen por posición: añadir o reordenar
   columnas rompe los parsers mientras la app siga leyendo CSV crudos.
9. **Paginación.** `nature?limit=30` y, en scripts, `pokemon?limit=5000` deben devolver todo el listado.
10. **CORS y errores.** `GET` sin credenciales desde cualquier origen; un recurso inexistente debe responder `404`
    (la app distingue errores transitorios de respuestas con estado).
11. **Sprites no nulos.** Si el fork se construye sin el submódulo de sprites, todas las imágenes saldrían en `null`
    y las cards quedarían en su fallback visual.

## Opciones de publicación

> **Descartadas (05/10/2026).** El fork no se publicará: la app migrará a la API propia diseñada en `plathinnum-dev`
> (ver la nota al inicio de [04](04-plan-y-decisiones-pendientes.md)). Las opciones se conservan como registro del
> análisis.

| Opción                                   | Descripción                                                                                                                         | Ventajas                                                                                 | Costes y riesgos                                                                                                                   |
| ---------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------- |
| **A. JSON estático en Cloudflare** (recomendada) | CI construye la base, recorre la API con `ditto` usando el dominio final y publica los JSON. Un Worker resuelve nombres, barra final y `limit`/`offset`. | Igual que PokeAPI original; sin servidor ni base en producción; encaja con el despliegue actual de la app en Cloudflare Workers. | ~15.800 archivos JSON: cabe en el límite gratuito de Workers Static Assets (20.000 por versión; 100.000 en el plan de pago). Las peticiones que pasen por el Worker cuentan para el límite gratuito de 100.000 diarias. Hay que diseñar el ruteo. |
| B. Django en vivo                        | Contenedor Docker con Gunicorn, PostgreSQL y Redis (Compose o Kubernetes del repositorio).                                           | Comportamiento idéntico a Django, incluido `?q=`; sin paso de rastreo.                   | Servidor y base que operar, escalar y pagar; latencia mayor que un CDN.                                                           |
| C. Sin API: datos en build de la app     | El fork publica CSV o JSON versionados y poke-generator genera su dataset en build (paso ER-13 de la app).                          | Elimina las peticiones de datos en runtime; builds reproducibles.                        | Cambia la arquitectura de la app; no sustituye a A o B si se quiere una API pública propia.                                        |

A y C son complementarias: A permite cambiar sólo la URL base de la app; C es la evolución que la app ya tenía
planificada (ER-13) y, con el fork, puede fijarse a un commit propio en vez de a `master` de PokeAPI original.

La opción A no depende de Cloudflare: el mismo JSON estático precomprimido puede servirse desde un servidor propio
con nginx o Caddy, separado del que aloja la app. Tamaños y comparación en
[05](05-tamanos-y-hosting.md).

Las imágenes conviene tratarlas aparte: `POKEAPI_SPRITES_PREFIX` permite que el build publique URLs de un servidor o
bucket propio (paso ER-25 de la app) sin tocar código del fork. Basta con servir `pokemon/other/home`,
`pokemon/other/official-artwork` e `items/` (~908 MB en PNG): cubren todo lo que muestra la app, siempre que el build
use ese mismo subconjunto y el servidor envíe cabeceras CORS
(ver [05](05-tamanos-y-hosting.md#servir-sólo-los-sprites-de-la-app)).

## Estrategia de transición sin cambios bruscos

1. **La app no cambia mientras el fork no esté verificado.** pokeapi.co sigue siendo la fuente hasta pasar la prueba
   de paridad.
2. **Construir el fork completo** (submódulos inicializados, `make build-db`) y levantarlo en local.
3. **Prueba de paridad.** Comparar, con las URLs normalizadas al mismo host, las respuestas del fork contra las
   grabaciones de `e2e/fixtures/network/pokeapi.co` de poke-generator, que ya cubren los recorridos críticos. Deben
   coincidir salvo los cambios intencionales del fork, que se documentan.
4. **Comprobación de IDs.** Un script compara IDs y slugs de `pokemon`, `pokemon_forms`, `moves`, `abilities`,
   `items` y `natures` contra los usados por el catálogo y los equipos de la app. Cualquier diferencia bloquea la
   publicación.
5. **Publicar en un dominio propio** (por ejemplo, un subdominio de la app) con la opción elegida.
6. **Cambio en la app, en una rama propia de poke-generator:** URL base configurable (una constante o variable
   `VITE_`), CSP con el host nuevo, `RECORDED_HOSTS` de E2E y grabaciones regeneradas, y pruebas unitarias con la URL
   base parametrizada. Los CSV crudos pasan a leerse del fork en un commit fijado.
7. **Reversión.** Mientras exista la URL base configurable, volver a pokeapi.co es un cambio de configuración.

## Riesgos detectados

- **Sincronización con PokeAPI original.** Sin remoto `upstream`, el fork se desactualiza en silencio. Las
  sincronizaciones deben revisar cambios de IDs, slugs y columnas de CSV antes de publicarse.
- **Datos curados duplicados.** Mientras convivan los datos curados de la app y los del fork, una corrección puede
  quedar en un solo lado. Cada migración debe retirar el dato de la app en el mismo ciclo en que el fork lo publica.
- **Tamaño de los sprites.** El submódulo de sprites ocupa 9,8 GB sin historial y ~9 GB más de historial Git. La CI
  del fork debe usar un clon parcial con checkout disperso de las carpetas servidas, nunca `submodules: recursive`.
- **CORS de imágenes.** La app carga los sprites con `crossorigin="anonymous"`: un servidor de imágenes sin
  `Access-Control-Allow-Origin` deja las cards sin imagen.
- **Variante del español.** La app prefiere `es` (España) y la decisión del 23/09/2026 cura los textos de Leyendas
  Pokémon: Z-A en variante España. En PokeAPI, `es-419` nació como copia de `es`; los datos del fork en `es` no deben
  copiarse automáticamente a `es-419` cuando el texto sea específico de España.
