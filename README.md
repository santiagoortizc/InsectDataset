# Insectos plaga del Tolima

Construcción de un corpus de insectos plaga agrícola del Tolima, Colombia: selección de
especies, descarga desde iNaturalist, preprocesamiento y partición reproducible.

Este repositorio **produce el corpus; no entrena nada**. El repositorio de modelado
consume sus artefactos. La dependencia va en un solo sentido.

Es una adaptación del corpus de mariposas del Mariposario del Tolima.

## Artefactos

| Ruta                  | Qué es                                                        | Versionado |
| --------------------- | ------------------------------------------------------------- | ---------- |
| `Insectos/<especie>/` | imágenes descargadas                                          | no         |
| `procesado/<especie>/` | recortes cuadrados 448×448, ya filtrados                     | no         |
| `especies_plaga.csv`  | curación manual de las especies plaga agrícola                | sí         |
| `species_list.json`   | especies seleccionadas con su `taxon_id`                      | sí         |
| `metadata.csv`        | una fila por foto: licencia, atribución, observación, sha256  | sí         |
| `auditoria.csv`       | una fila por foto: recorte, métricas, veredicto y motivo      | sí         |
| `manifiesto.json`     | parámetros, conteos y reparto train/test/validate             | sí         |

Las imágenes no van a git: son obras de terceros y pesan GB. Lo que se versiona es
la receta, que permite reconstruirlas, verificar los bytes y explicar por qué se
descartó cada foto.

## Flujo

1. **`00_especies.ipynb`** lista los insectos research-grade que iNaturalist registra en
   el Tolima, estima sus fotos con licencia abierta y escribe `especies_plaga.csv`. iNaturalist
   no etiqueta plagas, así que se marca a mano con `1` la columna `plaga` de cada plaga
   agrícola, y se vuelve a correr para generar `species_list.json`. Las especies
   salen del Tolima, pero las fotos salen del mundo entero.
2. **`01_scraping.ipynb`** descarga por `taxon_id` a `Insectos/` y registra la procedencia.
   Es incremental.
3. **`02_preprocesamiento.ipynb`** detecta, mide calidad, audita, recorta y reparte.
4. **`03_eda.ipynb`** analiza el corpus y los umbrales.

## Preprocesamiento

1. **Recorte.** YOLO-World localiza el insecto y la caja se convierte en un cuadrado con
   margen. Ante un borde el cuadrado se desplaza en vez de rellenarse: un borde negro sería
   una textura sintética que la red aprendería como atajo.
2. **Calidad.** Varianza del Laplaciano para el desenfoque, medida sobre el recorte ya
   normalizado a 384×384; mediana del canal V y fracción de píxeles quemados o negros para la
   exposición; lado mínimo del recorte para no ampliar detalle inexistente.
3. **Insecto visible.** En hormigas (Atta y Solenopsis), CLIP descarta los recortes que ve como
   hormiguero, suelo u hojarasca en vez de hormiga.
4. **Duplicados.** pHash y dHash sobre el recorte, comparados solo dentro de cada especie.
   De cada grupo se conserva la foto más nítida.

Cada foto sale con un veredicto y un motivo excluyente.

## Solo licencias abiertas

Se aceptan `cc0`, `cc-by`, `cc-by-nc`, `cc-by-sa` y `cc-by-nc-sa`. La API de iNaturalist
filtra del lado del servidor.

Quedan fuera dos casos:

- **Sin licencia declarada** (todos los derechos reservados).
- **`cc-by-nc-nd`**: el ND prohíbe obras derivadas, y recortar y aumentar imágenes para
  entrenar es exactamente eso.

## Uso

```bash
jupyter lab 00_especies.ipynb           # inventario, curación y species_list.json
jupyter lab 01_scraping.ipynb           # descarga
jupyter lab 02_preprocesamiento.ipynb   # recorte, calidad, deduplicación y reparto
jupyter lab 03_eda.ipynb                # análisis
```

La detección necesita `ultralytics`, que el resto del repositorio no usa.

## Pendiente

- Calibrar los umbrales de calidad y duplicados para insectos plaga: follaje denso, camuflaje,
  luz dura de campo e individuos diminutos (pulgones, trips, mosca blanca) pueden dispararlos.
- Validar los prompts de YOLO-World contra fotos reales, sobre todo larvas e insectos pequeños.
- Corregir la fuga entre splits repartiendo por `observation_id`.
