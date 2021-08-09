# Introducción a Seurat

Instructor: [Kevin E. Meza-Landeros](https://twitter.com/KevsGenomic)

## Diapositivas

Presentación: [aquí](https://docs.google.com/presentation/d/18ZCddwDD9lY8j4gmt1fO-Xqa8qJwCh_Zu_kXSjluW1Q/edit?usp=sharing) 

## Una perspectiva diferente 

Seurat es un paquete R diseñado para control de calidad, análisis y exploración de datos de secuencia de ARN de una sola célula. Seurat tiene como objetivo permitir a los usuarios identificar e interpretar fuentes de heterogeneidad a partir de mediciones transcriptómicas unicelulares e integrar diversos tipos de datos unicelulares. 

Seurat es desarrollado y mantenido por el laboratorio de [Satija](https://satijalab.org/seurat/authors.html) y se publica bajo la Licencia Pública GNU (GPL 3.0).

En este tutorial se ve como procesar los datos de scRNAseq con un nuevo paquete. Los pasos a realizar son en esencia los mismos que ya revisamos con el tutorial de la OSCA de RStudio.  
No olvides nunca que el paquete mas adecuado y que deberás utilizar dependerá mayoritariamente de tus datos y el procesamiento que se adecúe a estos.  
Además... siempre es bueno diversos puntos de vista sobre las cosas, no es así?

Aprende mas sobre Seurat: [aquí](https://satijalab.org/seurat/)

## Kick-start

En este tutorial partimos a partir de que ya se tienen los archivos FASTQ resultados de secuenciación.  

- ¿Con qué datos estoy trabajando?  
Peripheral Blood Mononuclear Cells (PBMC) disponibles gratuitamente de 10X Genomics. Son en total 2,700 céluas únicas secuenciadas con Illumina NextSeq 500.
Puedes descargar los datos de [aqui](https://cf.10xgenomics.com/samples/cell/pbmc3k/pbmc3k_filtered_gene_bc_matrices.tar.gz) (7.3MB).  
Descarga el archivo comprimido y procede a descomprimirlo. Se creara el siguiente directorio *filtered_gene_bc_matrices/hg19/*, aquí estarán los archivos que necesitaremos.

Este tutorial solo es la punta del **iceberg** de lo que se puede hacer con la paquetera de Seurat. Para comenzar a sumergirte en este mundo no dudes en visitar la página oficial mantenida por Satija Lab [Vignettes](https://satijalab.org/seurat/articles/get_started.html)

A continuación estableceremos nuestros directorio de trabajo y leeremos los datos anteriores.  
La función Read10X () lee en la salida de cellranger de 10X (de donde se obtuvieron los FASTQs), devolviendo una matriz de recuento única identificada molecularmente (UMI). Los valores en esta matriz representan el número de moléculas para cada característica (es decir, gen; fila) que se detectan en cada celda (columna).


```r
#library(dplyr)
#library(Seurat)
#library(patchwork)
# Load the PBMC dataset
#proydir <- "/mnt/BioAdHoc/Groups/vd-vijay/kmlanderos/CDSB/clustering/"
#pbmc.data <- Read10X(data.dir = paste0(proydir, "data/filtered_gene_bc_matrices/hg19/"))
# Initialize the Seurat object with the raw (non-normalized data).
#pbmc <- CreateSeuratObject(counts = pbmc.data, project = "pbmc3k", min.cells = 3, min.features = 200)
#pbmc
```
Veamos la estructura del Objeto de Seurat

```r
#str(pbmc)
```

- ¿Cómo se ven los datos en una matriz de recuento?  
Examinemos algunos genes en las primeras treinta células. El . los valores en la matriz representan ceros (no se detectan moléculas). Dado que la mayoría de los valores en una matriz scRNA-seq son 0, Seurat utiliza una representación de matriz dispersa (*sparse matrix*) siempre que sea posible. Esto da como resultado un ahorro significativo de memoria y velocidad.  
EN ESTE CASO UNA MATRIZ NO DISPERSA OCUPA 27 VECES MAS ESPACIO!


```r
#pbmc.data[c("CD3D", "TCL1A", "MS4A1"), 1:30]
#dense.size <- object.size(as.matrix(pbmc.data))
#dense.size
#sparse.size <- object.size(pbmc.data)
#sparse.size
#dense.size / sparse.size
```
## Quality Control

Algunas métricas de control de calidad comúnmente utilizadas por la comunidad incluyen:

- El número de genes únicos detectados en cada célula.
    - Las células de baja calidad o las gotitas vacías suelen tener muy pocos genes.
    - Los dobletes o multipletes celulares pueden exhibir un recuento de genes aberrantemente alto
- De manera similar, el número total de moléculas detectadas dentro de una célula (se correlaciona fuertemente con genes únicos)
- El porcentaje de lecturas que se asignan al genoma mitocondrial
    - Las células de baja calidad / moribundas a menudo exhiben una extensa contaminación mitocondrial
    - Calculamos métricas de control de calidad mitocondrial con la función PercentageFeatureSet (), que calcula el porcentaje de recuentos que se originan a partir de un conjunto de características.

El operador [[puede agregar columnas a los metadatos del objeto. Este es un gran lugar para almacenar estadísticas de control de calidad. Entonces calculamos y añadimos la cantidad de lecturas que corresponden al genoma mitocondrial.


```r
#pbmc[["percent.mt"]] <- PercentageFeatureSet(pbmc, pattern = "^MT-")
```
Visualizamos las métricas de control de calidad mencionadas anteriormente como un diagrama de violín. Ademas vemos la correlacion entre el numero de moleculaas de RNA detectadas en cada clula con el número de genes únicos y con el porcentaje de lecturas que corresponden a mtADN.


```r
#pdf(paste0(proydir,"plots/QC_metrics.pdf"))
#plot <- VlnPlot(pbmc, features = c("nFeature_RNA", "nCount_RNA", "percent.mt"), ncol = 3)
#print(plot)

#plot1 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "percent.mt")
#plot2 <- FeatureScatter(pbmc, feature1 = "nCount_RNA", feature2 = "nFeature_RNA")
#print(plot1 + plot2)
#dev.off()
```
Finalmente filtramos aquellas células que se salen de los estndares de cada uno de los parámetros.


```r
#Filter
#pbmc <- subset(pbmc, subset = nFeature_RNA > 200 & nFeature_RNA < 2500 & percent.mt < 5)
```
- Dónde se almacenan la métricas de QC en Seurat?
Estan almacenadas en la seccion de **meta-data** del objeto Seurat


```r
#head(pbmc@meta.data, 5)
```

## Normalización

De forma predeterminada, se emplea un método de normalización de escala global "LogNormalize" que normaliza las medidas de expresión de características para cada celda por la expresión total, multiplica esto por un factor de escala (10.000 por defecto) y transforma el resultado en logaritmos. Los valores normalizados se almacenan en pbmc [["RNA"]] @ data 


```r
#pbmc <- NormalizeData(pbmc, normalization.method = "LogNormalize", scale.factor = 10000)
```

## Detección de Genes (caractersticas) Altamente Variables

A continuación, calculamos un subconjunto de características que exhiben una alta variación de celda a celda en el conjunto de datos (es decir, están altamente expresadas en algunas células y poco expresadas en otras). El equipo de Seurat y otros equipos han descubierto que centrarse en estos genes en el análisis posterior ayuda a resaltar la señal biológica en conjuntos de datos unicelulares.

Nuestro procedimiento en Seurat se describe en detalle aquí y mejora las versiones anteriores al modelar directamente la relación de varianza media inherente a los datos de una sola celda, y se implementa en la función *FindVariableFeatures()*. De forma predeterminada, devolvemos 2000 características por conjunto de datos. Estos se utilizarán en análisis posteriores, como PCA. 


```r
#pbmc <- FindVariableFeatures(pbmc, selection.method = "vst", nfeatures = 2000)

# Identify the 10 most highly variable genes
#top10 <- head(VariableFeatures(pbmc), 10)
#top10

# plot variable features with and without labels
#pdf(paste0(proydir,"plots/VariableFeatures.pdf"))
#plot1 <- VariableFeaturePlot(pbmc)
#plot2 <- LabelPoints(plot = plot1, points = top10, repel = TRUE)
#print(plot1 + plot2)
#dev.off()
```

## Detalles de la sesión de R


```r
## Información de la sesión de R
Sys.time()
```

```
## [1] "2021-08-09 00:06:42 UTC"
```

```r
proc.time()
```

```
##    user  system elapsed 
##   0.507   0.135   0.520
```

```r
options(width = 120)
sessioninfo::session_info()
```

```
## ─ Session info ───────────────────────────────────────────────────────────────────────────────────────────────────────
##  setting  value                       
##  version  R version 4.1.0 (2021-05-18)
##  os       Ubuntu 20.04.2 LTS          
##  system   x86_64, linux-gnu           
##  ui       X11                         
##  language (EN)                        
##  collate  en_US.UTF-8                 
##  ctype    en_US.UTF-8                 
##  tz       UTC                         
##  date     2021-08-09                  
## 
## ─ Packages ───────────────────────────────────────────────────────────────────────────────────────────────────────────
##  package     * version date       lib source        
##  bookdown      0.22    2021-04-22 [1] RSPM (R 4.1.0)
##  bslib         0.2.5.1 2021-05-18 [1] RSPM (R 4.1.0)
##  cli           3.0.1   2021-07-17 [2] RSPM (R 4.1.0)
##  digest        0.6.27  2020-10-24 [2] RSPM (R 4.1.0)
##  evaluate      0.14    2019-05-28 [2] RSPM (R 4.1.0)
##  htmltools     0.5.1.1 2021-01-22 [1] RSPM (R 4.1.0)
##  jquerylib     0.1.4   2021-04-26 [1] RSPM (R 4.1.0)
##  jsonlite      1.7.2   2020-12-09 [2] RSPM (R 4.1.0)
##  knitr         1.33    2021-04-24 [2] RSPM (R 4.1.0)
##  magrittr      2.0.1   2020-11-17 [2] RSPM (R 4.1.0)
##  R6            2.5.0   2020-10-28 [2] RSPM (R 4.1.0)
##  rlang         0.4.11  2021-04-30 [2] RSPM (R 4.1.0)
##  rmarkdown     2.9     2021-06-15 [1] RSPM (R 4.1.0)
##  sass          0.4.0   2021-05-12 [1] RSPM (R 4.1.0)
##  sessioninfo   1.1.1   2018-11-05 [2] RSPM (R 4.1.0)
##  stringi       1.7.3   2021-07-16 [2] RSPM (R 4.1.0)
##  stringr       1.4.0   2019-02-10 [2] RSPM (R 4.1.0)
##  withr         2.4.2   2021-04-18 [2] RSPM (R 4.1.0)
##  xfun          0.24    2021-06-15 [2] RSPM (R 4.1.0)
## 
## [1] /__w/_temp/Library
## [2] /usr/local/lib/R/site-library
## [3] /usr/local/lib/R/library
```

## Patrocinadores {-}

Agradecemos a nuestros patrocinadores:

<a href="https://comunidadbioinfo.github.io/es/post/cs_and_s_event_fund_award/#.YJH-wbVKj8A"><img src="https://comunidadbioinfo.github.io/post/2021-01-27-cs_and_s_event_fund_award/spanish_cs_and_s_award.jpeg" width="400px" align="center"/></a>

<a href="https://www.r-consortium.org/"><img src="https://www.r-consortium.org/wp-content/uploads/sites/13/2016/09/RConsortium_Horizontal_Pantone.png" width="400px" align="center"/></a>