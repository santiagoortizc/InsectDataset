# CLAUDE.md

Guía para trabajar en este repositorio.

## Qué es este repositorio

Construye un corpus de mamíferos de bosque del Tolima: selección de especies, descarga desde
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

- [00_especies.ipynb](00_especies.ipynb) — inventario de mamíferos research-grade del Tolima,
  fotos licenciadas estimadas por especie y filtro de bosque. Escribe `species_list.json` con
  el `taxon_id` de cada especie.
- [01_scraping.ipynb](01_scraping.ipynb) — paginación de la API, descarga incremental a
  `Mamiferos/` y `metadata.csv`, e inventario del techo disponible por especie.
- [02_preprocesamiento.ipynb](02_preprocesamiento.ipynb) — cinco etapas que dejan su
  resultado en disco y saltan lo ya hecho, así que pueden correrse en días distintos:
  detección → métricas → auditoría → recortes → reparto y manifiesto.
- [03_eda.ipynb](03_eda.ipynb) — análisis del corpus y de los umbrales de calidad.

### Selección de especies

Las especies salen del Tolima; las fotos salen del mundo entero (restringirlas al
departamento dejaría decenas por clase). iNaturalist no etiqueta hábitat, así que "bosque"
se cura a mano: el notebook 00 escribe `especies_bosque.csv` y se marca con `1` la columna
`bosque`. Hay que excluir domésticas y comensales (perro, gato, ganado, caballo, cabra, rata).
Si el CSV está guardado desde Excel viene en cp1252; el notebook lo lee igual.

### Artefactos

| Ruta                    | Contenido                                                      | En git |
| ----------------------- | -------------------------------------------------------------- | ------ |
| `Mamiferos/`            | imágenes descargadas                                           | no     |
| `cache/`                | inventario del Tolima, detecciones y métricas, para reanudar   | no     |
| `pesos/`                | YOLO-World y el codificador de CLIP                            | no     |
| `procesado/`            | recortes aceptados                                             | no     |
| `especies_bosque.csv`   | curación manual de especies de bosque                          | sí     |
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
- **El manifiesto se escribe al final**: abandonar el notebook a mitad no deja un reparto
  a medias en el registro.

## Problemas conocidos

- **Umbrales sin recalibrar**: `UMBRALES` y `AREA_MIN_RELATIVA` del notebook 02 se fijaron para
  mariposas. En mamíferos de bosque el pelaje oscuro, las cámaras trampa nocturnas o infrarrojas
  y los animales pequeños en el encuadre pueden dispararlos. Recalibrar una vez, a la vista del
  notebook 03, y congelar los nuevos literales.
- **Desbalance entre clases**: las fotos estimadas por especie van de unos 440 a más de 500 mil.
  `POR_ESPECIE` en el notebook 01 limita la descarga.
- **Detección sin validar**: los prompts de YOLO-World (`mammal`, `animal`, `bat`) no se han
  probado contra fotos reales; medir cuántas quedan sin caja antes de confiar en ellos.
- **Fuga entre splits**: el reparto es por foto, no por observación; fotos del mismo individuo
  caen en splits distintos. El notebook 02 la mide al final; corregirlo exige repartir agrupando
  por `observation_id`.
- **Especies escasas**: pocos mamíferos del Tolima superan las 400 fotos licenciadas. Subir
  el tope no las mueve; solo crecerían incluyendo observaciones sin identificación confirmada,
  a cambio de ruido de etiqueta.
