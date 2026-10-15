# CLAUDE.md

Guía para trabajar en este repositorio.

## Qué es este repositorio

Construye un corpus de insectos plaga del Tolima: selección de especies, descarga desde
iNaturalist y preprocesamiento. **No entrena modelos**; el repositorio de modelado consume
los artefactos y la dependencia va en un solo sentido.

Código y comentarios en español, `camelCase` para funciones, mayúsculas para constantes.
Comentarios escasos y de una línea.

## Comandos

```bash
jupyter lab 00_especies.ipynb           # inventario del Tolima y lista de especies
jupyter lab 01_scraping.ipynb           # descarga desde iNaturalist
jupyter lab 02_preprocesamiento.ipynb   # detección, calidad, auditoría, recortes y reparto
jupyter lab 03_eda.ipynb                # análisis del corpus
```

Dependencias implícitas: `requests`, `pandas`, `numpy`, `opencv-python`, `jupyter`, y
`ultralytics` solo para la detección del notebook 02.

## Arquitectura

**El código vive en los notebooks.** `corpus/__init__.py` tiene solo las rutas y la
lectura/escritura de CSV; todo lo demás está inline donde se ejecuta, para no tener que
saltar a un módulo al depurar.

- [00_especies.ipynb](00_especies.ipynb) — inventario de insectos research-grade del Tolima,
  fotos licenciadas estimadas por especie y filtro de plaga agricola. Escribe `species_list.json` con
  el `taxon_id` de cada especie.
- [01_scraping.ipynb](01_scraping.ipynb) — paginación de la API, descarga incremental a
  `Insectos/` y `metadata.csv`, e inventario del techo disponible por especie.
- [02_preprocesamiento.ipynb](02_preprocesamiento.ipynb) — seis etapas que dejan su
  resultado en disco y saltan lo ya hecho, así que pueden correrse en días distintos:
  detección → métricas → insecto visible (CLIP) → auditoría → recortes → reparto y manifiesto.
- [03_eda.ipynb](03_eda.ipynb) — análisis del corpus y de los umbrales de calidad.

### Selección de especies

Las especies salen del Tolima; las fotos salen del mundo entero (restringirlas al
departamento dejaría decenas por clase). iNaturalist no etiqueta plaga agricola, así que 
se cura a mano: el notebook 00 escribe `especies_plaga.csv` y se marca con `1` la columna
`plaga`.
Si el CSV está guardado desde Excel viene en cp1252; el notebook lo lee igual.

### Artefactos

| Ruta                    | Contenido                                                      | En git |
| ----------------------- | -------------------------------------------------------------- | ------ |
| `Insectos/`             | imágenes descargadas                                           | no     |
| `cache/`                | inventario del Tolima, detecciones, métricas y puntajes CLIP   | no     |
| `pesos/`                | YOLO-World y el codificador de CLIP                            | no     |
| `procesado/`            | recortes aceptados                                             | no     |
| `especies_plaga.csv`    | curación manual de especies plaga agricola                     | sí     |
| `species_list.json`     | especies seleccionadas con su `taxon_id`                       | sí     |
| `metadata.csv`          | una fila por foto: licencia, atribución, observación, sha256   | sí     |
| `auditoria.csv`         | una fila por foto: recorte, métricas, veredicto y motivo       | sí     |
| `manifiesto.json`       | parámetros, conteos y reparto train/test/validate              | sí     |

Las imágenes no van a git: son obras de terceros y pesan GB. Se versiona la receta, que
permite reconstruirlas y verificar los bytes.

## Invariantes

- **Solo licencias abiertas**: `cc0, cc-by, cc-by-nc, cc-by-sa, cc-by-nc-sa`. Quedan fuera
  las fotos sin licencia declarada y `cc-by-nc-nd`, cuyo ND prohíbe obras derivadas.
- **`PAUSA = 0.25`** entre llamadas a la API: cortesía con iNaturalist, no bajarla.
- **Escritura de `metadata.csv`**: se carga entero, se acumula y se reescribe **completo tras
  cada especie**. Reescribir entero (nunca anexar) evita el corte que ya borró la procedencia
  una vez; hacerlo por especie evita que interrumpir el notebook deje imágenes en disco sin
  registro. Se deduplica por `photo_id`, así que reejecutar es seguro e incremental.
- **Clase vacía en los prompts de YOLO-World**: sin ella el detector deja sin caja 10 de
  cada 40 fotos; con ella, 1 de 40. Nunca gana la predicción. No quitarla.
- **Umbrales congelados como literales**: recalcularlos por percentil cambiaría el dataset
  cada vez que el corpus crece.
- **Cascada de motivos excluyente**: el primero que aplica gana, así el resumen suma el
  corpus entero sin doble contabilidad. La deduplicación va al final para que el
  representante conservado ya haya pasado los filtros de calidad.
- **Semilla combinada con la especie**: agregar una especie no altera el reparto de las que
  ya estaban. No cambiar a una sola semilla global.
- **Reparto por observación**: cada `observation_id` cae entero en un solo split, así fotos del
  mismo individuo no pasan de train a evaluación (antes se fugaba el 42 % del test). El 02 mide
  la fuga al final y debe dar 0.
- **`species_list.json` decide qué entra al 02**: las carpetas de `Insectos/` y las filas en
  caché de especies retiradas se ignoran. Para retirar una especie basta desmarcar `plaga` y
  regenerar la lista con el 00; no hace falta borrar sus imágenes ni el caché.
- **Filtro CLIP solo en `FILTRO_CLIP`**: descarta como `sin_insecto_visible` los recortes de
  hormigas que CLIP ve como hormiguero, suelo u hojarasca (`UMBRAL_CLIP = 0.30`). Los textos
  hablan de hormigas; aplicarlo a otras clases las vaciaría. Cambiar `TEXTOS_CLIP` obliga a
  borrar `cache/clip.csv`.
- **Clase ≠ especie solo vía `FUSIONES`**: el 02 agrupa especies que no se distinguen por foto
  (hoy *Atta cephalotes* y *A. colombica* → `atta_spp`). La auditoría guarda la especie real;
  `procesado/` y el manifiesto usan la clase.
- **El manifiesto se escribe al final**: abandonar el notebook a mitad no deja un reparto
  a medias en el registro.

## Problemas conocidos

- **Umbrales a medio recalibrar**: `fracSaturados` subió de 0.06 a 0.12 porque descartaba alas
  blancas y élitros brillantes (ascia, dione, strategus), no fotos quemadas. El resto de `UMBRALES`
  y `AREA_MIN_RELATIVA` siguen siendo los de mariposas; revisar a la vista del notebook 03.
- **Desbalance entre clases**: las fotos estimadas por especie van de unas decenas a cientos de
  miles, porque las plagas comunes están sobrerrepresentadas. `POR_ESPECIE` en el notebook 01
  limita la descarga.
- **Detección débil en hormigas, orugas y agallas**: con `insect`, `caterpillar`, `beetle` queda
  sin caja el 14 % del corpus, ~36 % en atta y ~30 % en erinnyis e iatrophobia. Agregar `ant` y
  `larva` se probó: en hormigas recupera sobre todo suelo y senderos sin hormiga visible. En iatrophobia la
  caja encuadra la agalla de la yuca: se conserva como clase de síntoma. Allomerus auropunctatus
  se retiró porque las hormigas apenas se veían en los recortes.
- **Variación por estadio**: huevo, larva, pupa y adulto de una misma plaga se ven como
  especies distintas, y las clases pueden quedar mezcladas o desbalanceadas por estadio. Decidir
  si se etiqueta por estadio o solo por especie.
