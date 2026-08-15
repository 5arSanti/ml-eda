# ml-eda

Informe reproducible en R Markdown: preprocesamiento y EDA de `data/retail.csv`.

## Dependencias

R 4.x y:

```r
install.packages(c("tidyverse", "GGally", "rmarkdown"))
```

Además, [pandoc](https://pandoc.org/) debe estar disponible en el sistema para tejer el HTML (`rmarkdown::pandoc_available()`).

## Datos

El repositorio incluye `data/retail.csv` como **muestra sintética reproducible** (`set.seed(42)`), no ventas reales de Falabella o Éxito. Sirve para ejercitar el flujo de limpieza e imputación; puede sustituirse por el archivo auténtico con las mismas 7 columnas: `mes`, `ciudad`, `categoria`, `ventas_totales_millones`, `ticket_promedio`, `unidades_vendidas`, `num_transacciones`.

## Tejer el informe

```r
rmarkdown::render("analysis.Rmd")
```

Genera `analysis.html` y `data/retail_clean.csv`.
