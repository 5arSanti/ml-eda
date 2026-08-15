# Retail EDA R Markdown Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Produce a Spanish `analysis.Rmd` that loads `data/retail.csv`, reports column types, drops structural duplicates, mean-imputes two numeric columns, exports `data/retail_clean.csv`, and reports descriptives plus histograms, boxplots, pairplot, and Pearson vs Spearman heatmaps with an explicit recommendation (no predictive model).

**Architecture:** One self-contained R Markdown file is the pipeline and the report. Objects stay in memory (`retail_raw` → `retail_dedup` → `retail_clean`). The original CSV is never overwritten. Verification uses `stopifnot()` via `Rscript` and `knitr::purl` + `source` (no `testthat`). Knit to `analysis.html` at the end.

**Tech Stack:** R 4.x, tidyverse (`readr`, `dplyr`, `tidyr`, `ggplot2`, `tibble`), GGally, rmarkdown, knitr.

## Global Constraints

- Informe y títulos de gráficos en español.
- Paquetes permitidos: `tidyverse`, `GGally`, `rmarkdown`/`knitr`. No `corrplot`. No imputación múltiple. No `testthat`. No scripts en `R/`. No `lm` ni machine learning.
- Columnas exactas: `mes`, `ciudad`, `categoria`, `ventas_totales_millones`, `ticket_promedio`, `unidades_vendidas`, `num_transacciones`.
- Duplicados: solo `dplyr::distinct(.keep_all = TRUE)` (todas las columnas). No de-duplicar por `mes` ni por subconjunto.
- Imputación: media global **después** de `distinct`, solo `ticket_promedio` y `unidades_vendidas`.
- Tipos: `glimpse()` y tabla de `class()` **antes** de `distinct()`.
- `mes` en el CSV exportado: `character` `YYYY-MM` (si `readr` lo leyó como `Date`, formatear antes de exportar).
- Heatmaps: `ggplot2::geom_tile`, límites de color −1 a 1; no sustituirlos con GGally.
- Pairplot: `GGally::ggpairs` de las 4 numéricas, color por `categoria`.
- Outliers: visualizar, no eliminar.
- `stop()` si falta el CSV, faltan columnas, imputación deja NA, o `distinct()` deja 0 filas.
- Advertir (no abortar) si los NA no son 4 y 2, o si se eliminaron duplicados.
- Nombres de objetos: `retail_raw`, `retail_dedup`, `retail_clean`, `required_cols`, `num_cols`.

---

## File map

| File | Responsibility |
|---|---|
| `data/retail.csv` | Input. Same 7 columns as the dictionary. If the authentic Falabella/Éxito file is not in the repo, create a fixture with 4 NA in `ticket_promedio`, 2 NA in `unidades_vendidas`, and ≥1 fully duplicated row. |
| `analysis.Rmd` | Entire narrative + pipeline + plots. |
| `data/retail_clean.csv` | Generated export (do not hand-edit). |
| `analysis.html` | Knit output. |
| `README.md` | Install packages and `rmarkdown::render("analysis.Rmd")`. |
| `purl_analysis.R` | Temporary; created by `knitr::purl` during verification; do not commit. |

Shared names later tasks must use:

```r
required_cols <- c(
  "mes", "ciudad", "categoria",
  "ventas_totales_millones", "ticket_promedio",
  "unidades_vendidas", "num_transacciones"
)
num_cols <- c(
  "ventas_totales_millones", "ticket_promedio",
  "unidades_vendidas", "num_transacciones"
)
```

Purl helper used in several verification steps (run from repo root):

```bash
Rscript -e 'knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE); source("purl_analysis.R")'
```

---

### Task 1: Fixture CSV, Rmd skeleton, README dependencies

**Files:**
- Create: `data/retail.csv`
- Create: `analysis.Rmd`
- Modify: `README.md`

**Interfaces:**
- Consumes: nothing
- Produces: `data/retail.csv` with exactly `required_cols`; `analysis.Rmd` YAML + `setup` chunk defining `required_cols` and `num_cols`; README install commands

- [ ] **Step 1: Write the failing check**

Run from repo root:

```bash
Rscript -e 'stopifnot(file.exists("data/retail.csv"))'
```

Expected: FAIL with `file.exists("data/retail.csv") is not TRUE` (or similar).

- [ ] **Step 2: Create the fixture CSV**

If an authentic `retail.csv` with the 7 dictionary columns is already present, keep it (it must have some NA in `ticket_promedio` and `unidades_vendidas` to exercise imputation). Otherwise run:

```r
dir.create("data", showWarnings = FALSE)
set.seed(42)
retail <- tidyr::expand_grid(
  mes = c("2024-01", "2024-02", "2024-03", "2024-04", "2024-05", "2024-06"),
  ciudad = c("Bogotá", "Medellín", "Cali", "Barranquilla"),
  categoria = c("Ropa", "Hogar", "Tecnología", "Alimentos")
)
n <- nrow(retail)
retail$ventas_totales_millones <- round(runif(n, 80, 450), 2)
retail$ticket_promedio <- round(runif(n, 25000, 180000), 0)
retail$unidades_vendidas <- as.numeric(round(runif(n, 400, 8000), 0))
retail$num_transacciones <- as.integer(round(runif(n, 200, 5000), 0))
na_ticket <- sample.int(n, 4)
na_unidades <- sample.int(n, 2)
retail$ticket_promedio[na_ticket] <- NA_real_
retail$unidades_vendidas[na_unidades] <- NA_real_
retail <- dplyr::bind_rows(retail, retail[1, , drop = FALSE])
readr::write_csv(retail, "data/retail.csv")
```

Save as a one-off `Rscript` invocation or run in an R session. Confirm afterwards:

```bash
Rscript -e '
d <- readr::read_csv("data/retail.csv", show_col_types = FALSE)
need <- c("mes","ciudad","categoria","ventas_totales_millones","ticket_promedio","unidades_vendidas","num_transacciones")
stopifnot(all(need %in% names(d)))
stopifnot(sum(is.na(d$ticket_promedio)) == 4)
stopifnot(sum(is.na(d$unidades_vendidas)) == 2)
stopifnot(any(duplicated(d)))
cat("ok fixture\n")
'
```

Expected: `ok fixture`

- [ ] **Step 3: Write `analysis.Rmd` skeleton**

Create `analysis.Rmd` with this exact content:

````rmd
---
title: "Análisis exploratorio: retail Falabella / Éxito"
output:
  html_document:
    toc: true
    toc_float: true
    df_print: paged
lang: es
---

```{r setup, include=FALSE}
knitr::opts_chunk$set(echo = TRUE, message = FALSE, warning = TRUE)
library(tidyverse)
library(GGally)

required_cols <- c(
  "mes",
  "ciudad",
  "categoria",
  "ventas_totales_millones",
  "ticket_promedio",
  "unidades_vendidas",
  "num_transacciones"
)
num_cols <- c(
  "ventas_totales_millones",
  "ticket_promedio",
  "unidades_vendidas",
  "num_transacciones"
)
```

# Objetivo

Preprocesamiento (duplicados estructurales e imputación por media), análisis descriptivo y comparación de correlaciones de Pearson y Spearman sobre ventas mensuales agregadas (Falabella / Éxito). No hay identificador de cliente. No se ajusta un modelo predictivo.
````

- [ ] **Step 4: Update `README.md`**

Replace `README.md` with:

````markdown
# ml-eda

Informe reproducible en R Markdown: preprocesamiento y EDA de `data/retail.csv`.

## Dependencias

R 4.x y:

```r
install.packages(c("tidyverse", "GGally", "rmarkdown"))
```

## Datos

Colocar el CSV en `data/retail.csv` (columnas: `mes`, `ciudad`, `categoria`, `ventas_totales_millones`, `ticket_promedio`, `unidades_vendidas`, `num_transacciones`).

## Tejer el informe

```r
rmarkdown::render("analysis.Rmd")
```

Genera `analysis.html` y `data/retail_clean.csv`.
````

- [ ] **Step 5: Re-run the existence check**

```bash
Rscript -e 'stopifnot(file.exists("data/retail.csv")); stopifnot(file.exists("analysis.Rmd")); cat("ok\n")'
```

Expected: `ok`

- [ ] **Step 6: Commit**

```bash
git add data/retail.csv analysis.Rmd README.md
git commit -m "chore: add retail fixture, Rmd skeleton, and README deps"
```

---

### Task 2: Load CSV, report types, validate columns

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `data/retail.csv`; `required_cols` from setup
- Produces: `retail_raw` (tibble); printed `glimpse()` and class table **before** any `distinct()`

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("retail_raw", envir = env))
'
```

Expected: FAIL (`retail_raw` does not exist).

- [ ] **Step 2: Append load + types section to `analysis.Rmd`**

Append after the Objetivo section:

````rmd
# Carga y tipos de columna

Los tipos se reportan **antes** de eliminar duplicados estructurales.

```{r load-raw}
csv_path <- "data/retail.csv"
if (!file.exists(csv_path)) {
  stop("No existe data/retail.csv. Coloca el insumo en esa ruta.")
}
retail_raw <- readr::read_csv(csv_path, show_col_types = FALSE)
missing <- setdiff(required_cols, names(retail_raw))
if (length(missing) > 0) {
  stop(
    "Faltan columnas del diccionario: ",
    paste(missing, collapse = ", ")
  )
}
retail_raw <- dplyr::select(retail_raw, dplyr::all_of(required_cols))
```

```{r types-before-distinct}
dplyr::glimpse(retail_raw)
class_tbl <- tibble::tibble(
  columna = names(retail_raw),
  clase = vapply(retail_raw, function(x) paste(class(x), collapse = ", "), character(1))
)
class_tbl
```

```{r na-before-clean}
na_before <- colSums(is.na(retail_raw))
na_before
n_ticket_na <- unname(na_before[["ticket_promedio"]])
n_unidades_na <- unname(na_before[["unidades_vendidas"]])
if (n_ticket_na != 4 || n_unidades_na != 2) {
  warning(
    "Se esperaban 4 NA en ticket_promedio y 2 en unidades_vendidas; ",
    "encontrados: ticket_promedio=", n_ticket_na,
    ", unidades_vendidas=", n_unidades_na,
    ". Se continúa."
  )
}
```
````

- [ ] **Step 3: Run purl + source check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("retail_raw", envir = env))
stopifnot(nrow(env$retail_raw) > 0)
stopifnot(identical(names(env$retail_raw), env$required_cols))
cat("ok load\n")
'
```

Expected: `ok load`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: load retail.csv and report column types before cleaning"
```

---

### Task 3: Structural duplicates

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `retail_raw`
- Produces: `retail_dedup`; `n_dropped_dups` (numeric, `nrow(retail_raw) - nrow(retail_dedup)`)

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("retail_dedup", envir = env))
'
```

Expected: FAIL (`retail_dedup` does not exist).

- [ ] **Step 2: Append duplicate-cleaning section**

````rmd
# Duplicados estructurales

Solo se eliminan filas idénticas en **todas** las columnas. No se de-duplica por `mes` ni por ninguna otra llave.

```{r structural-dups}
n_raw <- nrow(retail_raw)
retail_dedup <- dplyr::distinct(retail_raw, .keep_all = TRUE)
n_dedup <- nrow(retail_dedup)
n_dropped_dups <- n_raw - n_dedup
if (n_dedup == 0) {
  stop("distinct() dejó 0 filas.")
}
cat("Filas originales:", n_raw, "\n")
cat("Filas tras distinct():", n_dedup, "\n")
cat("Filas duplicadas eliminadas:", n_dropped_dups, "\n")
if (n_dropped_dups > 0) {
  warning("Se eliminaron ", n_dropped_dups, " fila(s) duplicadas por estructura.")
}
```
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("retail_dedup", envir = env))
stopifnot(exists("n_dropped_dups", envir = env))
stopifnot(env$n_dropped_dups >= 0)
stopifnot(nrow(env$retail_dedup) == nrow(dplyr::distinct(env$retail_raw)))
stopifnot(nrow(env$retail_dedup) > 0)
cat("ok distinct\n")
'
```

Expected: `ok distinct`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: drop structural duplicate rows only"
```

---

### Task 4: Mean imputation and export `retail_clean.csv`

**Files:**
- Modify: `analysis.Rmd`
- Create (generated): `data/retail_clean.csv`

**Interfaces:**
- Consumes: `retail_dedup`, `num_cols`, `required_cols`
- Produces: `retail_clean` (factors allowed in memory); `data/retail_clean.csv` with 7 columns, `mes` as `YYYY-MM` text, no NA in `num_cols`

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e 'stopifnot(file.exists("data/retail_clean.csv"))'
```

Expected: FAIL (`file.exists` is not TRUE).

- [ ] **Step 2: Append imputation + export**

````rmd
# Imputación y exportación

Media global, después de quitar duplicados, solo en `ticket_promedio` y `unidades_vendidas`.

```{r impute-export}
mean_ticket <- mean(retail_dedup$ticket_promedio, na.rm = TRUE)
mean_unidades <- mean(retail_dedup$unidades_vendidas, na.rm = TRUE)
retail_clean <- retail_dedup |>
  dplyr::mutate(
    ticket_promedio = tidyr::replace_na(ticket_promedio, mean_ticket),
    unidades_vendidas = tidyr::replace_na(unidades_vendidas, mean_unidades)
  )

if (any(is.na(retail_clean$ticket_promedio)) ||
    any(is.na(retail_clean$unidades_vendidas))) {
  stop("La imputación por media dejó NA en ticket_promedio o unidades_vendidas.")
}

if (inherits(retail_clean$mes, "Date")) {
  retail_clean$mes <- format(retail_clean$mes, "%Y-%m")
} else {
  retail_clean$mes <- as.character(retail_clean$mes)
}

retail_clean <- retail_clean |>
  dplyr::mutate(
    ciudad = factor(ciudad),
    categoria = factor(
      categoria,
      levels = c("Ropa", "Hogar", "Tecnología", "Alimentos")
    )
  )

cat("Filas post-imputación:", nrow(retail_clean), "\n")
print(colSums(is.na(retail_clean)))

dir.create("data", showWarnings = FALSE)
retail_export <- retail_clean |>
  dplyr::mutate(
    ciudad = as.character(ciudad),
    categoria = as.character(categoria)
  ) |>
  dplyr::select(dplyr::all_of(required_cols))
readr::write_csv(retail_export, "data/retail_clean.csv")

stopifnot(file.exists("data/retail_clean.csv"))
exported <- readr::read_csv("data/retail_clean.csv", show_col_types = FALSE)
stopifnot(identical(names(exported), required_cols))
stopifnot(sum(is.na(dplyr::select(exported, dplyr::all_of(num_cols)))) == 0)
```
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
source("purl_analysis.R")
d <- readr::read_csv("data/retail_clean.csv", show_col_types = FALSE)
need <- c("mes","ciudad","categoria","ventas_totales_millones","ticket_promedio","unidades_vendidas","num_transacciones")
stopifnot(identical(names(d), need))
stopifnot(sum(is.na(d$ticket_promedio)) == 0)
stopifnot(sum(is.na(d$unidades_vendidas)) == 0)
stopifnot(!file.exists("data/retail.csv") || {
  raw <- readr::read_csv("data/retail.csv", show_col_types = FALSE)
  sum(is.na(raw$ticket_promedio)) > 0
})
cat("ok export\n")
'
```

Expected: `ok export`. `data/retail.csv` still has NA; `data/retail_clean.csv` does not.

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd data/retail_clean.csv
git commit -m "feat: mean-impute ticket and units; export retail_clean.csv"
```

---

### Task 5: Descriptive tables

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `retail_clean`, `num_cols`
- Produces: printed tables `desc_num`, `freq_ciudad`, `freq_categoria`, `freq_mes` (tibbles)

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("desc_num", envir = env))
'
```

Expected: FAIL (`desc_num` does not exist).

- [ ] **Step 2: Append descriptives**

````rmd
# Análisis descriptivo

Resúmenes sobre `retail_clean`. Los atípicos no se recortan.

```{r descriptive-numeric}
desc_num <- retail_clean |>
  dplyr::select(dplyr::all_of(num_cols)) |>
  tidyr::pivot_longer(dplyr::everything(), names_to = "variable", values_to = "valor") |>
  dplyr::group_by(variable) |>
  dplyr::summarise(
    n = dplyr::n(),
    media = mean(valor),
    mediana = median(valor),
    sd = sd(valor),
    min = min(valor),
    max = max(valor),
    iqr = IQR(valor),
    .groups = "drop"
  )
desc_num
```

```{r descriptive-cats}
freq_ciudad <- retail_clean |>
  dplyr::count(ciudad, name = "n") |>
  dplyr::arrange(dplyr::desc(n))
freq_categoria <- retail_clean |>
  dplyr::count(categoria, name = "n") |>
  dplyr::arrange(dplyr::desc(n))
freq_mes <- retail_clean |>
  dplyr::count(mes, name = "n") |>
  dplyr::arrange(mes)
freq_ciudad
freq_categoria
freq_mes
```
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(nrow(env$desc_num) == 4)
stopifnot(all(c("n","media","mediana","sd","min","max","iqr") %in% names(env$desc_num)))
stopifnot(nrow(env$freq_ciudad) >= 1)
stopifnot(nrow(env$freq_categoria) >= 1)
stopifnot(nrow(env$freq_mes) >= 1)
cat("ok desc\n")
'
```

Expected: `ok desc`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: add numeric and categorical descriptive tables"
```

---

### Task 6: Histograms and boxplots

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `retail_clean`, `num_cols`
- Produces: ggplot objects `p_hist`, `p_box_global`, `p_box_cat`, `p_box_ciudad`; `ciudades_boxplot` (character vector of city names used)

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(inherits(env$p_hist, "gg"))
'
```

Expected: FAIL (`p_hist` missing or not a ggplot).

- [ ] **Step 2: Append plots**

Use a fixed category palette:

```r
pal_categoria <- c(
  "Ropa" = "#4C78A8",
  "Hogar" = "#F58518",
  "Tecnología" = "#54A24B",
  "Alimentos" = "#E45756"
)
```

Append:

````rmd
# Gráficos univariados

Paleta fija por categoría. Histogramas en un `facet_wrap`. Boxplots globales, por categoría y por ciudad (top 8 si hay más de 8 ciudades).

```{r plot-prep}
pal_categoria <- c(
  "Ropa" = "#4C78A8",
  "Hogar" = "#F58518",
  "Tecnología" = "#54A24B",
  "Alimentos" = "#E45756"
)
long_num <- retail_clean |>
  dplyr::select(ciudad, categoria, dplyr::all_of(num_cols)) |>
  tidyr::pivot_longer(
    dplyr::all_of(num_cols),
    names_to = "variable",
    values_to = "valor"
  )
```

```{r histograma}
p_hist <- ggplot(long_num, aes(x = valor)) +
  geom_histogram(bins = 20, fill = "#4C78A8", color = "white") +
  facet_wrap(~variable, scales = "free") +
  labs(
    title = "Histogramas de variables numéricas",
    x = "Valor",
    y = "Frecuencia"
  ) +
  theme_minimal()
p_hist
```

```{r boxplot-global}
p_box_global <- ggplot(long_num, aes(x = variable, y = valor)) +
  geom_boxplot(fill = "#9ecae1") +
  labs(
    title = "Boxplots globales",
    x = NULL,
    y = "Valor"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 20, hjust = 1))
p_box_global
```

```{r boxplot-categoria}
p_box_cat <- ggplot(long_num, aes(x = categoria, y = valor, fill = categoria)) +
  geom_boxplot() +
  facet_wrap(~variable, scales = "free") +
  scale_fill_manual(values = pal_categoria) +
  labs(
    title = "Boxplots por categoría",
    x = "Categoría",
    y = "Valor",
    fill = "Categoría"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 20, hjust = 1))
p_box_cat
```

```{r boxplot-ciudad}
n_ciudades <- dplyr::n_distinct(retail_clean$ciudad)
if (n_ciudades <= 8) {
  ciudades_boxplot <- as.character(unique(retail_clean$ciudad))
  nota_ciudad <- "Se muestran todas las ciudades."
} else {
  ciudades_boxplot <- retail_clean |>
    dplyr::group_by(ciudad) |>
    dplyr::summarise(ventas = sum(ventas_totales_millones), .groups = "drop") |>
    dplyr::slice_max(ventas, n = 8) |>
    dplyr::pull(ciudad) |>
    as.character()
  nota_ciudad <- paste(
    "Más de 8 ciudades: se muestran las 8 con mayor suma de ventas_totales_millones:",
    paste(ciudades_boxplot, collapse = ", ")
  )
}
cat(nota_ciudad, "\n")
p_box_ciudad <- long_num |>
  dplyr::filter(as.character(ciudad) %in% ciudades_boxplot) |>
  ggplot(aes(x = ciudad, y = valor)) +
  geom_boxplot(fill = "#a1d99b") +
  facet_wrap(~variable, scales = "free") +
  labs(
    title = "Boxplots por ciudad",
    subtitle = nota_ciudad,
    x = "Ciudad",
    y = "Valor"
  ) +
  theme_minimal() +
  theme(axis.text.x = element_text(angle = 30, hjust = 1))
p_box_ciudad
```
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(inherits(env$p_hist, "gg"))
stopifnot(inherits(env$p_box_global, "gg"))
stopifnot(inherits(env$p_box_cat, "gg"))
stopifnot(inherits(env$p_box_ciudad, "gg"))
stopifnot(length(env$ciudades_boxplot) <= 8)
cat("ok plots\n")
'
```

Expected: `ok plots`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: add faceted histograms and boxplots"
```

---

### Task 7: Pairplot

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `retail_clean`, `num_cols`, `pal_categoria`
- Produces: `p_pairs` (`ggmatrix` from GGally)

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(inherits(env$p_pairs, "ggmatrix"))
'
```

Expected: FAIL (`p_pairs` missing).

- [ ] **Step 2: Append pairplot**

````rmd
# Pairplot

`GGally::ggpairs` de las cuatro numéricas, color por `categoria` (no se omite).

```{r pairplot, fig.width=9, fig.height=8}
p_pairs <- GGally::ggpairs(
  retail_clean,
  columns = num_cols,
  mapping = ggplot2::aes(color = categoria),
  upper = list(continuous = GGally::wrap("cor", size = 3)),
  lower = list(continuous = GGally::wrap("points", alpha = 0.4, size = 0.7)),
  diag = list(continuous = GGally::wrap("densityDiag", alpha = 0.6))
) +
  ggplot2::scale_color_manual(values = pal_categoria) +
  ggplot2::scale_fill_manual(values = pal_categoria) +
  ggplot2::labs(title = "Pairplot de variables numéricas por categoría")
p_pairs
```
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(inherits(env$p_pairs, "ggmatrix"))
cat("ok pairs\n")
'
```

Expected: `ok pairs`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: add GGally pairplot colored by category"
```

---

### Task 8: Pearson vs Spearman heatmaps, table, recommendation

**Files:**
- Modify: `analysis.Rmd`

**Interfaces:**
- Consumes: `retail_clean`, `num_cols`, `p_pairs` (visual override via `curvatura_evidente`)
- Produces: `cor_pearson`, `cor_spearman` (matrices); `cor_compare` (tibble with `par`, `r_pearson`, `rho_spearman`, `delta`, `nota`); `k_delta`; `recomendacion`; heatmaps `p_heat_pearson`, `p_heat_spearman`

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(exists("cor_compare", envir = env))
'
```

Expected: FAIL (`cor_compare` does not exist).

- [ ] **Step 2: Append correlation section**

Decision tree (must run in this order, no overlap):

1. If `curvatura_evidente` is `TRUE` (set after looking at the pairplot) → Spearman
2. Else if `k == 0` → Pearson
3. Else if `k >= 3` → Spearman
4. Else → mixto

Default `curvatura_evidente <- FALSE`. After the first successful knit, if the pairplot is clearly curved/monotonic-non-linear, change it to `TRUE` and re-knit.

````rmd
# Correlaciones Pearson y Spearman

Las matrices usan solo las cuatro numéricas (`cor()`, no factores). Los heatmaps son `ggplot2::geom_tile` con la misma escala (−1 a 1). No se ajusta un modelo predictivo: la salida es una recomendación de medida de asociación.

```{r cor-matrices}
mat_num <- retail_clean |>
  dplyr::select(dplyr::all_of(num_cols)) |>
  as.data.frame()
cor_pearson <- cor(mat_num, method = "pearson", use = "pairwise.complete.obs")
cor_spearman <- cor(mat_num, method = "spearman", use = "pairwise.complete.obs")
stopifnot(isTRUE(all.equal(cor_pearson, t(cor_pearson))))
stopifnot(isTRUE(all.equal(cor_spearman, t(cor_spearman))))
stopifnot(all(abs(diag(cor_pearson) - 1) < 1e-10))
stopifnot(all(abs(diag(cor_spearman) - 1) < 1e-10))
cor_pearson
cor_spearman
```

```{r cor-heatmaps}
cor_to_long <- function(mat, metodo) {
  as.data.frame(mat) |>
    tibble::rownames_to_column("var1") |>
    tidyr::pivot_longer(-var1, names_to = "var2", values_to = "r") |>
    dplyr::mutate(metodo = metodo)
}

plot_heatmap <- function(df_long, titulo) {
  ggplot(df_long, aes(x = var1, y = var2, fill = r)) +
    geom_tile(color = "white") +
    geom_text(aes(label = sprintf("%.2f", r)), size = 3) +
    scale_fill_gradient2(
      limits = c(-1, 1),
      midpoint = 0,
      low = "#d73027",
      mid = "white",
      high = "#1a9850"
    ) +
    labs(title = titulo, x = NULL, y = NULL, fill = "coef") +
    theme_minimal() +
    theme(axis.text.x = element_text(angle = 30, hjust = 1))
}

p_heat_pearson <- plot_heatmap(
  cor_to_long(cor_pearson, "pearson"),
  "Heatmap de correlación de Pearson"
)
p_heat_spearman <- plot_heatmap(
  cor_to_long(cor_spearman, "spearman"),
  "Heatmap de correlación de Spearman"
)
p_heat_pearson
p_heat_spearman
```

```{r cor-compare}
idx <- which(upper.tri(cor_pearson), arr.ind = TRUE)
cor_compare <- tibble::tibble(
  par = paste(rownames(cor_pearson)[idx[, 1]],
              colnames(cor_pearson)[idx[, 2]],
              sep = " — "),
  r_pearson = cor_pearson[idx],
  rho_spearman = cor_spearman[idx],
  delta = rho_spearman - r_pearson
) |>
  dplyr::mutate(
    nota = dplyr::case_when(
      abs(delta) < 0.1 ~ "|Δ| < 0.1: Pearson y Spearman coinciden en lo esencial",
      TRUE ~ "|Δ| ≥ 0.1: la asociación lineal y la monótona difieren"
    )
  )
k_delta <- sum(abs(cor_compare$delta) >= 0.1)
cor_compare
cat("Pares con |delta| >= 0.1 (k) =", k_delta, "\n")
```

```{r recomendacion}
# Tras ver el pairplot: TRUE si hay curvatura o relación monótona no lineal evidente.
curvatura_evidente <- FALSE

if (isTRUE(curvatura_evidente)) {
  recomendacion <- "Spearman"
  motivo <- "El pairplot muestra curvatura o monotonía no lineal; Spearman describe mejor la asociación."
} else if (k_delta == 0) {
  recomendacion <- "Pearson"
  motivo <- "k = 0 y sin curvatura evidente: un modelo lineal posterior es razonable."
} else if (k_delta >= 3) {
  recomendacion <- "Spearman"
  motivo <- paste0(
    "k = ", k_delta,
    " pares con |Δ| ≥ 0.1: Spearman es la medida principal; un lineal puede engañar."
  )
} else {
  recomendacion <- "mixto"
  motivo <- paste0(
    "k = ", k_delta,
    " (1 o 2 pares). No hay un único ganador; revisar esos pares en la tabla."
  )
}
cat("Recomendación:", recomendacion, "\n")
cat(motivo, "\n")
```

La recomendación es una **medida de asociación**, no un modelo ajustado. El umbral 0.1 es una guía de lectura, no un test de hipótesis.
````

- [ ] **Step 3: Run check**

```bash
Rscript -e '
knitr::purl("analysis.Rmd", "purl_analysis.R", documentation = 0, quiet = TRUE)
env <- new.env()
source("purl_analysis.R", local = env)
stopifnot(nrow(env$cor_compare) == 6)
stopifnot(all(c("par","r_pearson","rho_spearman","delta","nota") %in% names(env$cor_compare)))
stopifnot(env$recomendacion %in% c("Pearson","Spearman","mixto"))
stopifnot(inherits(env$p_heat_pearson, "gg"))
stopifnot(inherits(env$p_heat_spearman, "gg"))
cat("ok cor\n")
'
```

Expected: `ok cor`

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd
git commit -m "feat: compare Pearson and Spearman with heatmaps and recommendation"
```

---

### Task 9: Knit HTML and finish README

**Files:**
- Modify: `README.md` only if the render command needs a working-directory note
- Create (generated): `analysis.html`

**Interfaces:**
- Consumes: complete `analysis.Rmd`, `data/retail.csv`
- Produces: `analysis.html`; regenerated `data/retail_clean.csv`

- [ ] **Step 1: Write the failing check**

```bash
Rscript -e 'stopifnot(file.exists("analysis.html"))'
```

Expected: FAIL if HTML has not been knitted yet.

- [ ] **Step 2: Render**

```bash
Rscript -e 'rmarkdown::render("analysis.Rmd", output_file = "analysis.html")'
```

Expected: exit 0. Requires pandoc (`rmarkdown::pandoc_available()` must be TRUE). If pandoc is missing, install it before this step (`apt-get install pandoc` or the OS equivalent), then re-run render.

- [ ] **Step 3: Confirm outputs**

```bash
Rscript -e '
stopifnot(file.exists("analysis.html"))
stopifnot(file.exists("data/retail_clean.csv"))
d <- readr::read_csv("data/retail_clean.csv", show_col_types = FALSE)
stopifnot(ncol(d) == 7)
stopifnot(sum(is.na(d)) == 0 || {
  num <- c("ventas_totales_millones","ticket_promedio","unidades_vendidas","num_transacciones")
  sum(is.na(dplyr::select(d, dplyr::all_of(num)))) == 0
})
cat("ok knit\n")
'
rm -f purl_analysis.R
```

Expected: `ok knit`. `purl_analysis.R` removed.

- [ ] **Step 4: Commit**

```bash
git add analysis.Rmd analysis.html data/retail_clean.csv README.md
git commit -m "docs: knit analysis.html and confirm clean CSV export"
```

Do not commit `purl_analysis.R`.

---

## Self-review (spec coverage)

| Spec section | Task |
|---|---|
| Load `data/retail.csv`, `stop` if missing | 2 |
| Types before `distinct` (`glimpse` + class table) | 2 |
| Column name validation | 2 |
| Structural `distinct` only; count dropped | 3 |
| Mean impute `ticket_promedio` and `unidades_vendidas` after distinct | 4 |
| `stop` if NA remain / 0 rows | 3, 4 |
| Warn if NA counts ≠ 4 and 2; warn if dups dropped | 2, 3 |
| Export `data/retail_clean.csv`; `mes` as `YYYY-MM` text; factors as text on disk | 4 |
| Descriptives n/mean/median/sd/min/max/IQR; freq ciudad/categoria/mes | 5 |
| Histograms `facet_wrap`; boxplots global / categoria / ciudad top 8 | 6 |
| `ggpairs` colored by `categoria` | 7 |
| Pearson + Spearman; `geom_tile` −1..1; 6-row compare table; k decision tree | 8 |
| No `lm` / no testthat / Spanish / tidyverse+GGally | all |
| Knit `analysis.html` | 9 |
| README install + render | 1, 9 |

No `testthat` suite: checks are `stopifnot` via `Rscript` as required by the spec.
