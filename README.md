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
