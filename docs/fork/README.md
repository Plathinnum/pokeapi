# Documentación del fork

Esta carpeta pertenece al fork `Plathinnum/pokeapi` y no forma parte de PokeAPI original. Documenta por qué existe el
fork, cómo se mantiene sincronizado con el proyecto original y qué necesita la aplicación Himitsu Kichi
(`Plathinnum/poke-generator`) para consumirlo sin cambios bruscos.

Los documentos se escriben en español neutro. Los identificadores técnicos (slugs, columnas, rutas) se citan tal cual
entre comillas invertidas; los Pokémon se nombran con su número de Pokédex nacional entre paréntesis.

## Índice

| Documento                                                            | Contenido                                                                                       |
| -------------------------------------------------------------------- | ----------------------------------------------------------------------------------------------- |
| [01 — Estructura de PokeAPI](01-estructura-pokeapi.md)               | Arquitectura, flujo de datos, dependencias externas, CI y cómo publica PokeAPI original.        |
| [02 — Compatibilidad con poke-generator](02-compatibilidad-poke-generator.md) | Contrato que consume la app, supuestos que no deben romperse y estrategia de transición.        |
| [03 — Extensiones a migrar](03-extensiones-a-migrar.md)              | Inventario de lo que la app añadió sobre PokeAPI y dónde encaja cada parte en el fork.          |
| [04 — Plan y decisiones pendientes](04-plan-y-decisiones-pendientes.md) | Fases propuestas, criterios de aceptación y decisiones que debe tomar el propietario.            |
| [05 — Tamaños y alojamiento](05-tamanos-y-hosting.md)                | Peso de la API, sprites y cries, y comparación de formas de servirlos desde un servidor propio. |

## Convenciones del fork

- **Sin divergencias silenciosas.** Todo cambio que se aparte de PokeAPI original se registra en esta carpeta con su
  motivo, para poder resolver conflictos al sincronizar y decidir si se propone aguas arriba.
- **Ubicación.** La carpeta `docs/fork/` evita choques de rutas si PokeAPI original añade algún día su propia carpeta
  `docs/`. Los cambios de esta carpeta no disparan ningún workflow de CI (ver [01](01-estructura-pokeapi.md#ci-y-flujo-de-contribución)).
- **Lineamientos de PokeAPI.** Los cambios de datos o código del fork siguen `CONTRIBUTING.md`: pruebas para cada
  cambio, `make format`, `make lint-check`, `make typecheck`, el hook `check-csv` y la regeneración de `openapi.yml`
  cuando cambian `pokemon_v2/` o `config/`.

Fecha del análisis inicial: 03/10/2026, sobre el commit `bc92d3b6` (30/09/2026) de `master`.
