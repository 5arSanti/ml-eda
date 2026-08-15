# Diseño: EDA retail Falabella / Éxito (R Markdown)

Fecha: 2026-08-15  
Repo: `ml-eda`  
Enfoque aprobado: informe R Markdown autocontenido (enfoque 1)

## 1. Objetivo

Producir un informe reproducible en R que:

1. Cargue `data/retail.csv` (ventas mensuales por ciudad y categoría; no hay clientes ni `id`).
2. Declare el tipo de cada columna **antes** de cualquier limpieza de duplicados.
3. Elimine solo duplicados **estructurales** (todas las columnas iguales).
4. Impute con **media global** los faltantes de `ticket_promedio` (4) y `unidades_vendidas` (2).
5. Exporte el dataset completo a `data/retail_clean.csv`.
6. Haga análisis descriptivo, histogramas, boxplots, pairplot y heatmaps de correlación **Pearson** y **Spearman**, con una comparación explícita para decidir si un modelo lineal posterior es razonable.

Fuera de alcance: modelos predictivos (`lm`, machine learning, train/test), de-duplicación por llave/`mes`, imputación por grupos, recorte de outliers, Shiny, Quarto, tests `testthat`.

## 2. Entregables

| Artefacto | Rol |
|---|---|
| `analysis.Rmd` | Informe único (narrativa en español + código). |
| `analysis.html` | HTML generado al tejer. |
| `data/retail.csv` | Insumo original. No se modifica. Lo aporta quien implemente / el usuario. |
| `data/retail_clean.csv` | Dataset post-duplicados e imputación. Entregable obligatorio, exportable. |
| `README.md` | Cómo instalar paquetes y tejer el informe. |

No hay scripts en `R/` ni paquete R.

## 3. Estructura del repositorio

```
ml-eda/
  analysis.Rmd
  data/retail.csv
  data/retail_clean.csv    # generado al tejer; no se edita a mano
  README.md
  docs/superpowers/specs/2026-08-15-retail-eda-rmd-design.md
```

## 4. Datos de entrada

Fuente: retail Falabella / Éxito. Granularidad: una fila = periodo × ciudad × categoría (agregado de ventas/transacciones, no de clientes).

| Variable | Descripción | Tipo esperado al cargar |
|---|---|---|
| `mes` | Periodo `YYYY-MM` | `character` |
| `ciudad` | Ciudad de la tienda | `character` |
| `categoria` | Ropa / Hogar / Tecnología / Alimentos | `character` |
| `ventas_totales_millones` | Ventas del mes, millones de $ | `numeric` (double) |
| `ticket_promedio` | Valor promedio por transacción ($) | `numeric` (double); 4 NA |
| `unidades_vendidas` | Unidades vendidas en el mes | `numeric` (double); 2 NA |
| `num_transacciones` | Número de transacciones | `integer` o `double` según el archivo |

`mes` es eje temporal, no identificador de fila. No existe columna `id`.

Los tipos **reales** son los que imprima `dplyr::glimpse()` / `sapply(class)` sobre el CSV cargado. La tabla de arriba es la expectativa. Si `readr` lee `mes` como `Date`, el informe lo documenta y, antes de exportar, lo formatea a `character` `YYYY-MM` para que `retail_clean.csv` coincida con el diccionario.

Faltantes conocidos: solo `ticket_promedio` (4) y `unidades_vendidas` (2). Si el archivo real tiene otros conteos, el informe **advierte** y continúa con media global en esas dos columnas si aún hay NA; no aborta por desajuste de conteo.

## 5. Dependencias

- R 4.x
- `tidyverse` (`readr`, `dplyr`, `tidyr`, `ggplot2`, `tibble`, etc.)
- `GGally` (pairplot: `ggpairs`)
- `rmarkdown` para tejer

No usar `corrplot` ni paquetes de imputación múltiple.

## 6. Flujo del informe (`analysis.Rmd`)

Orden fijo de chunks. Cada paso de limpieza imprime n filas, NA por columna y duplicados estructurales restantes.

### 6.1 Carga

- `readr::read_csv("data/retail.csv")` → `retail_raw`.
- Si el archivo no existe: `stop()` con mensaje que indica la ruta esperada.

### 6.2 Tipos (antes de duplicados)

- `dplyr::glimpse(retail_raw)`.
- Tabla de `columna` + `class()`.
- Validar que existen las 7 columnas del diccionario (nombres exactos). Si falta alguna o el nombre difiere: `stop()`.

### 6.3 Duplicados estructurales

- Criterio único: dos filas son duplicadas si coinciden en **todas** las columnas.
- `dplyr::distinct(retail_raw, .keep_all = TRUE)` → `retail_dedup`.
- Reportar filas eliminadas (`nrow(retail_raw) - nrow(retail_dedup)`).
- No de-duplicar por `mes`, ni por `mes + ciudad + categoria`, ni por ningún subconjunto.

### 6.4 Imputación

- Solo `ticket_promedio` y `unidades_vendidas`.
- Método: media global de la columna **después** de `distinct`, `mean(x, na.rm = TRUE)`.
- Reemplazo: `tidyr::replace_na` o equivalente vectorizado. No agrupar por ciudad ni categoría.
- `ciudad` y `categoria` se pueden convertir a `factor` **después** de imputar, para gráficos.
- Si tras imputar esas dos columnas aún hay NA: `stop()` (media indefinida / columna vacía).
- Resto de columnas: no imputar.

### 6.5 Exportación

- Objeto `retail_clean` (mismos 7 nombres de columna que el crudo).
- `dir.create("data", showWarnings = FALSE)` y `readr::write_csv(retail_clean, "data/retail_clean.csv")`.
- `ciudad` y `categoria` se escriben como texto en el CSV aunque en memoria sean `factor`.
- Comprobar en el informe: archivo existe; mismas columnas; `sum(is.na()) == 0` en las cuatro numéricas.

El CSV original no se sobrescribe.

## 7. Análisis descriptivo

Sobre `retail_clean`:

- Numéricas (`ventas_totales_millones`, `ticket_promedio`, `unidades_vendidas`, `num_transacciones`): n, media, mediana, desviación estándar, min, max, IQR.
- Frecuencias de `ciudad` y `categoria`.
- Conteos de filas por `mes` (cobertura temporal).
- No recortar outliers. Los boxplots los muestran; el texto puede comentar asimetría o atípicos.

## 8. Gráficos

Estilo: títulos y ejes en español; paleta discreta fija para `categoria`; un chunk por gráfico o un `facet_wrap` coherente.

### 8.1 Histogramas

Las cuatro numéricas en formato largo y un solo gráfico `ggplot2` con `geom_histogram` + `facet_wrap(~variable, scales = "free")`. Ejes en unidades originales. No es obligatorio superponer densidad.

### 8.2 Boxplots

- Globales de las cuatro numéricas (asimetría y atípicos).
- Desglose por `categoria`.
- Por `ciudad`: si `n_distinct(ciudad) <= 8`, boxplots de todas; si hay más de 8, solo las 8 ciudades con mayor `sum(ventas_totales_millones)` y el texto indica el filtro.

### 8.3 Pairplot

`GGally::ggpairs` sobre las cuatro numéricas, con color por `categoria` (cuatro niveles conocidos; no se omite el color).

No incluir factores en las matrices `cor()`.

## 9. Correlaciones Pearson vs Spearman

### 9.1 Matrices

Sobre las cuatro numéricas de `retail_clean`:

- `cor(..., method = "pearson")`
- `cor(..., method = "spearman")`

Tras imputar no debe haber NA; usar `use = "pairwise.complete.obs"` igualmente.

### 9.2 Heatmaps

Dos gráficos `ggplot2::geom_tile` (datos en formato largo). Misma paleta divergente y mismos límites de color (−1 a 1). Valor numérico en cada celda.

`GGally` no sustituye estos heatmaps.

### 9.3 Tabla comparativa

Una fila por par único (i < j):

| par | r_pearson | rho_spearman | delta (rho − r) | nota |

### 9.4 Criterio de recomendación (no es un modelo ajustado)

El informe recomienda la **medida de asociación** principal y qué implica para un modelo posterior. No ajusta el modelo.

Hay 6 pares únicos. Sea `k` el número de pares con `|delta| >= 0.1`. Decisión en este orden (sin solaparse):

1. Si el pairplot muestra curvatura o relación monótona no lineal evidente → recomendar **Spearman**.
2. Si no: `k == 0` → recomendar **Pearson** (modelo lineal posterior razonable).
3. Si no: `k >= 3` → recomendar **Spearman** como medida principal; un lineal puede engañar.
4. Si no: `k` es 1 o 2 → caso mixto; no hay un único ganador; el texto explica esos pares y deja la decisión al usuario.

No declarar ganador por un solo par. Umbral `0.1` es guía de lectura, no un test de hipótesis. No se exigen p-valores ni test de normalidad en esta fase.

## 10. Errores vs advertencias

**`stop()` (el knit falla):**

- No existe `data/retail.csv`.
- Faltan columnas del diccionario o nombres distintos.
- Tras imputar, `ticket_promedio` o `unidades_vendidas` siguen con NA.
- Cero filas después de `distinct()`.

**Advertencia en el informe (no abortar):**

- El número de NA no es 4 y 2.
- Había duplicados estructurales (> 0 eliminadas).

## 11. Verificación al tejer

Sin `testthat`. El propio `analysis.Rmd` deja evidencia de:

1. `glimpse()` y clases **antes** de `distinct()`.
2. Conteos: original → post-duplicados → post-imputación; NA por columna en cada paso.
3. `data/retail_clean.csv` escrito, columnas iguales, numéricas sin NA.
4. Matrices de correlación simétricas y diagonal = 1.
5. `rmarkdown::render("analysis.Rmd")` termina sin error y genera `analysis.html` en la raíz (requiere `data/retail.csv` presente).

## 12. Unidades y límites

| Unidad | Hace | Depende de |
|---|---|---|
| Carga y tipos | Lee CSV; muestra clases | `data/retail.csv` |
| Deduplicación | `distinct` de filas idénticas | tibble crudo |
| Imputación | Media global en 2 columnas | tibble sin duplicados estructurales |
| Exportación | Escribe `retail_clean.csv` | `retail_clean` |
| Descriptivos | Tablas resumen y frecuencias | `retail_clean` |
| Histogramas / boxplots | ggplot2 | `retail_clean` |
| Pairplot | `GGally::ggpairs` | `retail_clean` |
| Heatmaps + comparación | `cor` Pearson/Spearman + `geom_tile` + tabla delta | `retail_clean` |

Cada unidad se entiende por su salida (tibble, archivo o gráfico) sin leer el resto del informe.

## 13. Decisiones ya cerradas

- Entregable de análisis: un `.Rmd` (no scripts modulares, no Quarto).
- Paquetes: tidyverse + GGally.
- Duplicados: solo estructurales; no por id/`mes`.
- Imputación: media global; pocos NA.
- CSV limpio: entregable obligatorio.
- Tipos: se informan antes de `distinct()`.
- Idioma del informe: español.
- Outliers: se visualizan, no se eliminan.
- “Mejor modelo”: comparación Pearson vs Spearman como insumo de decisión; no se entrena modelo.
