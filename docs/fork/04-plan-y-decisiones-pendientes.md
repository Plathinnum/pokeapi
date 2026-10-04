# 04 — Plan y decisiones pendientes

Plan propuesto para adoptar el fork sin interrumpir Himitsu Kichi. Cada fase termina en un estado estable y
reversible; ninguna cambia poke-generator hasta la fase 4.

## Fase 0 — Preparación del fork

- Añadir el remoto `upstream` (`https://github.com/PokeAPI/pokeapi.git`) y definir la cadencia de sincronización.
- Decidir la rama de trabajo del fork (`master` sincronizada con PokeAPI original más una rama propia, o `master`
  propia con merges periódicos).
- Inicializar el submódulo de sprites con clon parcial y checkout disperso de `pokemon/other/home`,
  `pokemon/other/official-artwork` e `items/` (~908 MB), en vez de los 9,8 GB completos; cries sólo si se quieren
  publicar (28 MB).
- Instalar `uv` y ejecutar `make install`, `make setup`, `make build-db` y `make test` para fijar una línea base.
- Completar el `AGENTS.md` del fork con las reglas de este directorio (idioma, lineamientos de PokeAPI, registro de
  divergencias). `AGENTS.md` y `CLAUDE.md` son locales: están en `.gitignore` y no se versionan, así que un clon
  nuevo o una sesión en la nube no los tendrá.
- No conectar CircleCI al fork y no ejecutar `Resources/scripts/updater.sh`: publican en repositorios de PokeAPI.

**Criterio de aceptación:** el fork sin cambios construye la base, pasa sus pruebas y sirve `pokemon/1/` en local con
sprites no nulos.

## Fase 1 — Paridad sin cambios de datos

- Script de paridad: recorre las URLs grabadas en `e2e/fixtures/network/pokeapi.co` de poke-generator, pide lo mismo
  al fork local y compara el JSON con los hosts normalizados.
- Script de estabilidad: compara IDs y slugs de las tablas que la app persiste o referencia (ver
  [02](02-compatibilidad-poke-generator.md#supuestos-que-el-fork-no-puede-romper)).
- Registrar cada diferencia como esperada (cambio reciente de PokeAPI original aún no publicado en pokeapi.co) o como
  defecto.

**Criterio de aceptación:** cero diferencias no explicadas en los recursos que consume la app.

## Fase 2 — Publicación propia

- Elegir la opción de publicación (recomendada: JSON estático en Cloudflare, ver
  [02](02-compatibilidad-poke-generator.md#opciones-de-publicación)).
- Workflow de GitHub Actions propio: build de la base, `ditto` con el dominio final, verificación de paridad y
  despliegue. Se dispara en `master` y en una rama de staging con su propia URL de previsualización.
- Worker de ruteo: nombre → ID, barra final opcional, `limit`/`offset` sobre el índice completo y `404` en recursos
  inexistentes.
- Imágenes: mantener `raw.githubusercontent.com` al principio o servir desde el inicio el subconjunto de la app
  (`home`, `official-artwork`, `items/`) en un servidor propio con CORS, cambiando `POKEAPI_SPRITES_PREFIX`. La
  conversión a WebP (~150 MB en total) puede llegar después sin cambiar URLs, entregando WebP en la misma ruta `.png`
  según la cabecera `Accept`.
- Chequeo de CI: toda URL de sprite de los JSON publicados y de los manifiestos de la app existe en el servidor.

**Criterio de aceptación:** la prueba de paridad pasa contra la URL publicada, no sólo contra el entorno local.

## Fase 3 — Migración de extensiones (en el fork)

Orden sugerido, de menor a mayor riesgo (detalle en [03](03-extensiones-a-migrar.md)):

1. Correcciones de datos (punto 7) y nombres en español faltantes.
2. Condiciones de forma verificadas (punto 6).
3. Comprobación de sprites HOME y `official-artwork` tras #1681 (punto 8).
4. Nombres de forma en español (punto 1).
5. Efectos breves de habilidad (punto 3).
6. Textos de Pokédex por forma, tras reconstruir el juego de cada texto (punto 2).
7. Sprites de objetos por generación (punto 9).
8. Efectos breves de movimiento y Champions, si se decide migrarlos (puntos 4 y 10).

Cada paso es una rama corta del fork con pruebas en `pokemon_v2/tests.py` (o el validador de CSV correspondiente) y
una entrada en un registro de divergencias de esta carpeta.

## Fase 4 — Adopción en poke-generator

En una rama de poke-generator, después de publicar el fork:

- URL base configurable y apuntada al fork; CSP, hosts de E2E, grabaciones y pruebas unitarias actualizados.
- CSV leídos del fork en un commit fijado (o dataset en build, ER-13).
- Retiro de los datos curados que el fork ya publica, en el mismo cambio en que se empiezan a leer del fork, para no
  mantener dos fuentes.
- Actualización de `docs/project/decisions.md`, `docs/domain/pokemon-research.md` y del README de la app (atribución
  de datos).

## Decisiones pendientes del propietario

1. **Alcance del fork.** ¿Sólo fuente de datos de Himitsu Kichi o una API pública propia? Define si basta la opción C
   (datos en build) o se necesita la A o la B.
2. **Publicación.** JSON estático precomprimido (recomendado; ~325 MB de disco) en un servidor propio separado de la
   app o en Cloudflare, Django en vivo, o ambos. Comparación en [05](05-tamanos-y-hosting.md).
3. **Plan de Cloudflare (si se elige).** Con ~15.800 archivos, el plan gratuito de Workers Static Assets alcanza (límite 20.000),
   pero deja poco margen si se añaden cries u otros recursos; el plan de pago sube el límite a 100.000.
4. **Sprites.** Servir sólo el subconjunto de la app (sin cambios de código en el fork) o todas las colecciones
   (requiere un prefijo por colección en `build.py`); usar `PokeAPI/sprites` directamente o un fork propio; y
   cuándo pasar a WebP (ER-25). Tamaños en [05](05-tamanos-y-hosting.md).
5. **Relación con PokeAPI original.** Proponer aguas arriba las correcciones (puntos 6 y 7) para reducir la
   divergencia, o mantenerlas sólo en el fork. Si se proponen, aplica la política de IA de `CONTRIBUTING.md`.
6. **Champions.** Modelarlo en `moves.csv`/`move_changelog.csv` (cambia los valores «actuales» para todos los
   consumidores) o mantener los overrides en la app.
7. **Variante del español.** Confirmar que los datos nuevos del fork se cargan en `es` (España) y cuándo se replican
   en `es-419`.
8. **Inclusión documental.** `AGENTS.md` de poke-generator define «los documentos» como `docs/`, `AGENTS.md` y
   `BACKLOG.md` de la app. Hay que decidir si esta carpeta del fork entra en esa definición para las revisiones
   habituales.
