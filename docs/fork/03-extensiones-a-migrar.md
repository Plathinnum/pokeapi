# 03 — Extensiones a migrar

poke-generator compensa limitaciones de PokeAPI con datos curados y reglas propias. Este documento separa lo que es un
**hecho de la franquicia** (pertenece al fork, como filas de CSV o un ajuste del build) de lo que es una **regla de
producto** de la app (debe quedarse en poke-generator). Mezclar ambas cosas convertiría al fork en una API a medida
de una sola app y complicaría sincronizarlo con PokeAPI original.

Las cifras se midieron el 03/10/2026 contra los CSV del commit `bc92d3b6` y los archivos de `src/data/` de
poke-generator.

## Criterios

- **Va al fork** si es un dato verificable de los juegos que PokeAPI ya modela, o modelaría con su esquema actual:
  traducciones, textos, condiciones de forma, correcciones de valores, sprites existentes no enlazados.
- **Se queda en la app** si decide qué ofrece o cómo se comporta Himitsu Kichi: disponibilidad en el Team Builder,
  agrupación visual, roles de forma, alias de búsqueda, calendario del Pokémon del día, configuración competitiva.
- **Lineamientos de PokeAPI.** Datos como filas de CSV en `data/v2/csv/`, sin migraciones de datos; si hace falta
  esquema nuevo, modelo + migración + `build.py` + serializador + pruebas en `pokemon_v2/tests.py` + `openapi.yml`
  regenerado. Cada cambio pasa `check-csv`, `make test` y el formato de ruff.
- **Fuentes.** Mismas exigencias que `docs/domain/pokemon-research.md` de la app: texto verbatim de fuente oficial,
  contrastado con al menos dos fuentes, sin traducir a mano.

## Resumen

| #  | Extensión en poke-generator                                   | Destino en el fork                                  | Encaje      | Esfuerzo |
| -- | ------------------------------------------------------------- | --------------------------------------------------- | ----------- | -------- |
| 1  | Nombres de forma en español                                   | `pokemon_form_names.csv`                            | Alto        | Medio    |
| 2  | Textos de Pokédex por forma en español                        | `pokemon_form_flavor_text.csv`                      | Alto        | Alto     |
| 3  | Efectos breves de habilidad en español                        | `ability_prose.csv`                                 | Alto        | Bajo     |
| 4  | Efectos breves de movimiento en español                       | `move_effect_prose.csv`                             | Parcial     | Alto     |
| 5  | Categoría por forma                                           | Sin tabla; requiere esquema nuevo                   | Bajo        | Medio    |
| 6  | Condiciones de forma faltantes                                | `pokemon_form_conditions.csv`                       | Alto        | Medio    |
| 7  | Correcciones de datos                                         | `pokemon_species.csv`, `pokemon_forms.csv`, `pokemon_moves.csv` | Alto | Bajo |
| 8  | Sprites HOME por forma y `official-artwork` manuales          | Build actual (#1681) o submódulo de sprites         | Alto        | Bajo     |
| 9  | Sprites de objetos Gen 8 y Gen 9                              | `build.py` (`ItemSprites`)                          | Medio       | Medio    |
| 10 | Valores de movimiento de Pokémon Champions                    | `moves.csv` + `move_changelog.csv`, o se queda      | Decisión    | Alto     |
| 11 | CSV leídos en runtime desde `master`                          | Publicación del fork en commit fijado               | Alto        | Bajo     |
| 12 | Reglas del catálogo de formas, alias, Pokémon del día, etc.   | Se quedan en la app                                 | —           | —        |

## Detalle

### 1. Nombres de forma en español

- **En la app:** `src/data/pokemonNameDefinitions.ts` (191 definiciones con nombre visible y alias) y
  `pokemonFormNames.generated.ts` (generado desde `pokemon_form_names.csv`).
- **Hueco en PokeAPI:** 185 definiciones corresponden a una fila de `pokemon_forms.csv`; **159 de ellas no tienen
  nombre en español** (`local_language_id = 7`). En total, 261 formas no predeterminadas carecen de nombre en español.
- **Destino:** filas nuevas en `pokemon_form_names.csv` (`form_name` y `pokemon_name`) para `es`, y para `es-419`
  sólo cuando el nombre oficial coincida.
- **Se queda en la app:** los alias de búsqueda (por ejemplo, «Nidoran hembra» para Nidoran♀ (#29)) y los nombres
  de especie con grafía especial; son decisiones de búsqueda y presentación.

### 2. Textos de Pokédex por forma

- **En la app:** `src/data/pokemonFormDexEntries.es.json`, 313 formas con uno o más textos.
- **En PokeAPI:** `pokemon_form_flavor_text.csv` ya existe y el serializador de `pokemon-form` lo publica como
  `flavor_text_entries`, pero sólo contiene 249 filas en inglés.
- **Bloqueo:** la tabla exige `version_id` por texto y el JSON de la app **guarda sólo el texto, sin el juego**. Los
  juegos de cada texto están descritos en `docs/domain/pokemon-research.md` de la app y en sus fuentes (WikiDex,
  Bulbapedia), pero hay que reconstruir la asociación texto → versión antes de migrar.
- **Destino:** filas `language_id = 7` en `pokemon_form_flavor_text.csv`, una por forma y versión.

### 3. Efectos breves de habilidad

- **En la app:** `src/data/abilityEffects.es.json`, 174 habilidades; todos los slugs existen en `abilities.csv`.
- **En PokeAPI:** `ability_prose.csv` sólo tiene filas en inglés (314); ninguna en español.
- **Destino:** `short_effect` en español en `ability_prose.csv`. Hay que decidir qué va en la columna `effect` (texto
  largo): dejarla vacía, si el build y el serializador lo aceptan, o redactar un efecto completo.
- **Matiz:** los textos de la app representan Gen 8 como línea base y siguen un estilo condensado propio (límite de
  130 caracteres). Al publicarse en una API general pierden ese contexto; conviene documentarlo en el fork.

### 4. Efectos breves de movimiento

- **En la app:** `src/data/moveEffects.es.json`, 662 movimientos, uno por slug.
- **En PokeAPI:** `move_effect_prose.csv` se indexa por **efecto**, no por movimiento, y usa el marcador
  `$effect_chance` para la probabilidad.
- **Conflictos medidos:** los 662 movimientos usan 370 efectos; en 23 efectos los textos de la app difieren entre
  movimientos (201 movimientos afectados), sobre todo porque la app escribe la probabilidad literal («10 %», «20 %»).
  Además, 93 movimientos de `moves.csv` no tienen `effect_id`.
- **Destino:** posible con plantillas `$effect_chance` y efectos nuevos para los movimientos sin `effect_id`, pero
  exige reescribir textos. Conviene dejarlo para una fase posterior y mantenerlo en la app mientras tanto.

### 5. Categoría por forma

- **En la app:** `src/data/pokemonFormCategories.es.json` (4 formas: Hoopa (#720) Desatado, Calyrex (#898) Jinete
  de Hielo y Jinete Espectral, Basculin (#550) Raya Blanca).
- **En PokeAPI:** la categoría (`genus`) sólo existe por especie en `pokemon_species_names.csv`.
- **Destino:** requeriría una tabla o columna nueva y un campo nuevo en `pokemon-form`. Con 4 casos, se recomienda
  dejarlo en la app.

### 6. Condiciones de forma faltantes

PokeAPI ya modela los disparadores de forma en `pokemon_form_conditions.csv` (234 condiciones: 153 por objeto
equipado, 35 por habilidad, 34 Gigamax, 6 por objeto clave, 4 por objeto consumido y 2 por movimiento). Ya cubre casi
todo lo que la app curó en `scripts/syncPokemonFormCatalog.mjs`: megapiedras, placas, discos, memorias, orbes, Ultra
Necrozma (#800), Zygarde (#718) y Minior (#774).

Condiciones que la app usa o deduce y que PokeAPI no tiene:

| Forma                                      | Condición en la app o en los juegos                        | Estado en el fork                                                     |
| ------------------------------------------ | --------------------------------------------------------- | --------------------------------------------------------------------- |
| Mega-Rayquaza (#384)                       | Movimiento `dragon-ascent`                                | Falta; encaja en el disparador `move` existente.                      |
| Enamorus (#905) Forma Tótem                | Objeto clave `reveal-glass`                               | Falta; Tornadus (#641), Thundurus (#642) y Landorus (#645) sí lo tienen. |
| Kyurem (#646) Blanco y Negro               | Objeto clave `dna-splicers`                               | Falta; el objeto existe en `items.csv`.                               |
| Calyrex (#898) Jinetes                      | Objeto clave `reins-of-unity`                             | Falta; el objeto existe.                                              |
| Formas de Rotom (#479)                     | Objeto clave `rotom-catalog`                              | Falta; el objeto existe.                                              |
| Formas de Deoxys (#386)                    | Objeto clave `meteorite`                                  | Falta; el objeto existe.                                              |
| Necrozma (#800) Melena Crepuscular y Alas del Alba | Objetos clave de fusión                            | Falta; **los objetos no existen en `items.csv`** (a investigar).      |
| Castform (#351) y Cherrim (#421)            | Clima del equipo                                          | Sólo existe la condición de habilidad; el clima requeriría un disparador nuevo. Se queda en la app. |
| Xerneas (#716) Modo Activo                  | Automático en combate                                     | Sin condición; es una regla de presentación de la app.               |

Cada fila se verifica contra fuentes antes de añadirse; la tabla anterior es una lista de candidatos, no datos
confirmados para el fork.

### 7. Correcciones de datos

| Dato                                         | Situación en PokeAPI                                                   | Corrección propuesta                                                    |
| -------------------------------------------- | ---------------------------------------------------------------------- | ----------------------------------------------------------------------- |
| `gender_rate` de Oinkologne (#916)            | `0` (sólo macho) aunque existe `oinkologne-female` como variedad       | `4`; la app la corrige con `genderRateOverrides`.                       |
| `is_battle_only` de Eternatus (#890) Eternamax | `0`                                                                   | `1`, según `pokemon-research.md` de la app.                             |
| Movimientos de Magearna (#801) Color Vetusto  | 259 filas en `pokemon_moves.csv` frente a 265 de la forma base         | Completar las filas faltantes; la app copia el repertorio de `801`.     |
| Nombres en español faltantes                 | Habilidades `eelevate`, `fire-mane`, `aura-guard`; objetos `hopo-berry`, `roseli-berry`, `god-stone`, `fresh-start-mochi`, `roto-stick` | Añadir los nombres oficiales verificados.                               |

Estas correcciones son buenos candidatos para proponerse también a PokeAPI original: reducen la divergencia del fork.

### 8. Sprites de Pokémon

- **En la app:** `pokemonHomeFormSprites.generated.ts` (sprites HOME de 180 formas que `pokemon-form` no enlazaba),
  cuatro `official-artwork` fijados a mano para formas de Koraidon (#1007) y Miraidon (#1008), y el conjunto
  `FORM_IDS_WITHOUT_OWN_SPRITE` (formas que deben heredar el sprite HOME de su base).
- **En PokeAPI:** desde el 24/09/2026 (#1681), `build.py` busca `{pokemon_id}-{form_identifier}` en todas las
  colecciones, incluida `home`. Es probable que el fork ya publique esos sprites.
- **Acción:** construir el fork con el submódulo de sprites y comprobar `pokemon-form/{id}` para las 180 formas y los
  cuatro casos de `official-artwork`. Si los publica, el manifiesto de la app queda redundante y se retira; si falta
  alguno, se corrige en `build.py` o en el repositorio de sprites del fork.
- **Cobertura medida (03/10/2026):** `home` y `official-artwork` cubren las 186 formas no predeterminadas del catálogo
  y todas las variedades que la app ofrece; `showdown`, `dream-world` y los clásicos no rescatan ningún caso
  adicional. Las 16 variedades sin HOME se detallan en
  [05](05-tamanos-y-hosting.md#servir-sólo-los-sprites-de-la-app).
- **Se queda en la app:** la regla de Megaevoluciones sin dimorfismo y la elección de qué sprite mostrar.

### 9. Sprites de objetos

- **En la app:** prioriza `items/gen9/{slug}.png`, después `items/gen8/` y la raíz; para `adamant-crystal`,
  `lustrous-globe` y `griseous-core` usa el orbe equivalente.
- **En PokeAPI:** `ItemSprites` sólo publica `items/{slug}.png` en `sprites.default`.
- **Destino:** añadir claves nuevas a `sprites` del objeto (por ejemplo, `generation-viii` y `generation-ix`) en
  `build.py`. Es un cambio aditivo del contrato JSON: no rompe consumidores, pero diverge de PokeAPI original. La
  equivalencia con los orbes es una decisión visual de la app y se queda allí.

### 10. Valores de Pokémon Champions

- **En la app:** `moveOverrides.champions.es.json` (potencia, precisión, PP, tipo y descripción de 41 movimientos),
  `abilityOverrides.champions.es.json` (2 habilidades) y la fórmula de PP de Champions.
- **En PokeAPI:** existe el grupo de versión `champions` (ID 32) sin datos, y `generation-ix/champions` en la
  configuración de sprites.
- **Conflicto de convención:** PokeAPI guarda en `moves.csv` el valor del juego más reciente y el historial en
  `move_changelog.csv`. Cargar Champions como «actual» cambiaría los valores que hoy la app trata como Gen 9 base, y
  obligaría a la app a reconstruir Escarlata/Púrpura desde el historial. Es una decisión de producto
  (ver [04](04-plan-y-decisiones-pendientes.md)).
- **Se queda en la app:** la fórmula de PP y las descripciones curadas.

### 11. CSV en runtime

- **En la app:** `item_names.csv`, `move_names.csv`, `moves.csv` y `move_changelog.csv` se descargan desde `master` de
  PokeAPI original en cada sesión (~750 KB con caché de 5 minutos), y los scripts leen otros CSV de la misma rama.
- **Destino:** el fork pasa a ser la fuente, fijada a un commit (`raw.githubusercontent.com/Plathinnum/pokeapi/<commit>/...`)
  o, mejor, generada en el build de la app (ER-13). Con el fork, las traducciones de los puntos 1 a 3 llegan a la app
  por el mismo camino.

### 12. Lo que se queda en poke-generator

- Roles, mutabilidad y disponibilidad del catálogo de formas (`unavailableSlugs`, `hiddenTotemSlugs`,
  `presentationFamilies`, `availabilityOverrides`, `logicalFormDefinitions`, `abilityBoundVarieties`, etc.).
- Disparadores de combate simulados (clima del equipo, Canto Arcáico en el propio repertorio) y la resolución de
  apariencias del equipo.
- Cadenas de grupos de versión por versión competitiva, teratipos, reglas de sexo, títulos colectivos, alias de
  búsqueda de objetos, iconos de tipos.
- Pokémon del día, changelog, glosario de efectos y textos de interfaz.

Si alguna de estas reglas resultara útil para otros consumidores, se reevalúa caso por caso; por defecto, el fork no
publica reglas de producto.
