---
title: "Flujo de trabajo VariationNaturalEcosystemsArea – Indicador de variación del área de ecosistemas naturales continentales en Colombia"
author: 
  - name: "Rincon-Parra VJ"
    email: "rincon-v@javeriana.edu.co"
    affiliation: "Gerencia de información cientifica. Instituto de Investigación de Recursos Biológicos Alexander von Humboldt - IAvH"
output: 
  github_document:
    md_extension: +gfm_auto_identifiers
    preserve_yaml: true
    toc: true
    toc_depth: 6
---

Flujo de trabajo VariationNaturalEcosystemsArea – Indicador de variación
del área de ecosistemas naturales continentales en Colombia
================
true

- [Organizar directorio de trabajo](#organizar-directorio-de-trabajo)
- [Establecer parámetros de sesión](#establecer-parámetros-de-sesión)
  - [Cargar librerias/paquetes necesarios para el
    análisis](#cargar-libreriaspaquetes-necesarios-para-el-análisis)
- [Establecer entorno de trabajo](#establecer-entorno-de-trabajo)
  - [Definir inputs y direccion
    output](#definir-inputs-y-direccion-output)
- [Cargar insumos](#cargar-insumos)
- [Estimar area por periodo](#estimar-area-por-periodo)
- [Plot de cambio y tendencia](#plot-de-cambio-y-tendencia)

Esta rutina estima el Indicador de Variación del Área de Ecosistemas
Estratégicos con Potencial de Captura de Carbono en Colombia. Los
ecosistemas estratégicos en Colombia incluyen los bosques andinos, los
bosques secos, los humedales, los páramos y los manglares, que
desempeñan un papel fundamental en la captura y almacenamiento de
carbono por su capacidad de absorber CO2 de la atmósfera y almacenarlo
en la biomasa y el suelo, contribuyendo así a la regulación del ciclo
del carbono y la mitigación del cambio climático.

Bajo esta idea, cuantificar las variaciones de extensión de estos
ecosistemas brinda información sobre su capacidad para seguir capturando
carbono. Este índice mide los cambios en la extensión de coberturas
naturales en las zonas delimitadas como ecosistemas estratégicos en
Colombia. De manera que se describen tendencias de pérdida o ganancia de
áreas naturales de estos ecosistemas, lo cual es fundamental para
entender y mantener su potencial de captura de carbono.

## Organizar directorio de trabajo

<a id="ID_seccion1"></a>

Las entradas de ejemplo de este ejercicio están almacenadas en
[IAvH/Unidades
compartidas/MBI/VariationStrategicEcosystemsCarbonCapture](https://drive.google.com/drive/folders/1DGOnUX2tLY5agzcAnOmLWUPN1qKCcOJ7?usp=drive_link).
Están organizadas de esta manera que facilita la ejecución del código:

    script
    │- Script_VariationNaturalEcosystemsArea
    │    
    └-input
    │ │
    │ └- studyArea
    │ │   │
    │ │   │- studyArea.gpkg
    │ │
    │ │
    │ └- strategicEcosystems
    │ │   │
    │ │   │- StratEcoSystem_1.gpkg
    │ │   │- ...
    │ │   │- StratEcoSystem_2.gpkg
    │ │
    │ │
    │ └- covs
    │     │
    │     │- NatCovs_tiempo_1.gpkg
    │     │- ...
    │     │- NatCovs_tiempo_n.gpkg
    │     
    └-output

## Establecer parámetros de sesión

### Cargar librerias/paquetes necesarios para el análisis

``` r
## Establecer parámetros de sesión ####
### Cargar librerias/paquetes necesarios para el análisis ####

#### Verificar e instalar las librerías necesarias ####
packagesPrev <- installed.packages()[,"Package"]  
packagesNeed <- librerias <- c("this.path", "magrittr", "dplyr", "plyr", "pbapply", "data.table", "raster", "terra", "sf", "ggplot2", 
                               "tidyr", "reshape2","openxlsx")  # Define los paquetes necesarios para ejecutar el codigo
new.packages <- packagesNeed[!(packagesNeed %in% packagesPrev)]  # Identifica los paquetes que no están instalados
if(length(new.packages)) {install.packages(new.packages, binary = TRUE)}  # Instala los paquetes necesarios que no están previamente instalados

#### Cargar librerías ####
lapply(packagesNeed, library, character.only = TRUE)  # Carga las librerías necesarias
```

## Establecer entorno de trabajo

El flujo de trabajo está diseñado para establecer el entorno de trabajo
automáticamente a partir de la ubicación del código. Esto significa que
tomará como `dir_work` la carpeta raiz donde se almacena el código
“~/scripts. De esta manera, se garantiza que la ejecución se realice
bajo la misma organización descrita en el paso de [Organizar directorio
de trabajo](#ID_seccion1).

``` r
## Establecer entorno de trabajo ####
dir_work <- this.path::this.path() %>% dirname()  # Establece el directorio de trabajo
```

### Definir inputs y direccion output

Dado que el código está configurado para definir las entradas desde la
carpeta input, en esta parte se debe definir una lista llamada input en
la que se especifica el nombre de cada una de las entradas necesarias
para su ejecución. Para este ejemplo, basta con usar file.path con
referencia a `input_folder` y el nombre de los archivos para definir su
ruta y facilitar su carga posterior. No obstante, se podría definir
cualquier ruta de la máquina local como carpeta input donde se almacenen
las entradas, o hacer referencia a cada archivo directamente.

Asimismo, el código genera una carpeta output donde se almacenarán los
resultados del análisis. La ruta de esa carpeta se almacena por defecto
en el mismo directorio donde se encuentra el código.

Este código ejecuta una serie de análisis espaciales a partir de
polígonos. Es fundamental que los insumos sean polígonos y utilicen un
sistema de coordenadas coherente. Todas las entradas espaciales deben
tener el mismo sistema de coordenadas. En este flujo de trabajo, toda la
información cartográfica se maneja en el sistema de coordenadas WGS84
(EPSG:4326). El código puede ejecutarse en cualquier sistema de
coordenadas, siempre y cuando todos los insumos espaciales tengan la
misma proyección. Se pueden usar distintos tipos de vectores espaciales
(por ejemplo, .gpkg, .geoJson, .shp). En este ejemplo, se utiliza .gpkg
por ser el más eficiente para análisis espaciales.

``` r
### Definir entradas necesarias para la ejecución del análisis ####

# Definir la carpeta de entrada-insumos
input_folder<- file.path(dir_work, "input"); # "~/input"

# Crear carpeta output
output<- file.path(dir_work, "output"); dir.create(output)

#### Definir entradas necesarias para la ejecución del análisis ####
input <- list(
  studyArea= file.path(input_folder, "studyArea", "ColombiaDeptos.gpkg"),  # Ruta del archivo espacial que define el área de estudio
  timeNatCoverList= list( # Lista de rutas de archivos espaciales que representan coberturas naturales en diferentes años.  Cada elemento en la lista se nombra con el año correspondiente al que representa el archivo de cobertura natural. Esto permitira ordenarlos posteriormente
    "2002"= file.path(input_folder, "covs", "CLC_natural_2002.gpkg"), # Cobertura natural del año 2002 IDEAM
    "2008"= file.path(input_folder, "covs", "CLC_natural_2008.gpkg"), # Cobertura natural del año 2008 IDEAM
    "2009"= file.path(input_folder, "covs", "CLC_natural_2009.gpkg"),  # Cobertura natural del año 2009 IDEAM
    "2018"= file.path(input_folder, "covs", "CLC_natural_2018.gpkg") # Cobertura natural del año 2018 IDEAM
  )
)
```

La lista de entradas incluye studyArea como la ruta al archivo espacial
del área de estudio.

La lista StratEcoSystemList compila las rutas de los archivos espaciales
de la delimitación de los ecosistemas estratégicos. Cada elemento de la
lista corresponde a la zona de delimitación de cada ecosistema
estratégico. Esta tarea es compleja debido a que muchos de los mapas
disponibles sobre ecosistemas estratégicos reportan delimitaciones
generales o remanentes actuales sin información detallada del área
potencial base. Esto implica que las delimitaciones pueden no reflejar
completamente el rango histórico o potencial de los ecosistemas, lo cual
puede afectar la precisión de los análisis espaciales. Para este
ejercicio, se utilizaron como límites los biomas asociados a los
ecosistemas estratégicos de bosques andinos, bosques secos, humedales,
páramos y manglares. Estos biomas fueron seleccionados debido a su
importancia ecológica y su capacidad significativa para capturar y
almacenar carbono. Sin embargo, es importante señalar que el código es
flexible y puede ejecutarse con las delimitaciones de ecosistemas
estratégicos que el investigador considere más apropiadas para su
estudio. Cada elemento de la lista debe nombrarse con el nombre del
ecosistema estratégico al que hace referencia para facilitar la
exploración y el análisis de los resultados.

Por último, la entrada de la lista timeNatCoverList compila las rutas de
archivos espaciales de coberturas naturales en diferentes momentos. En
este caso, se utilizaron los polígonos de coberturas naturales según la
clasificación Corine Land Cover para Colombia, reportados por el IDEAM a
escala 1:100k para los años 2002, 2009, 2008 y 2017. No se realiza
procesamiento posterior a estos mapas, ya que el código asume que los
polígonos de dichas entradas corresponden solo a coberturas naturales en
esos periodos. Es importante que los nombres de cada elemento a cargar
se especifiquen con años numéricos, ya que esto será útil para organizar
el análisis de cambio y tendencias posterior.

## Cargar insumos

``` r
## Cargar insumos ####

# Este codigo maneja toda la informacion cartografica en el sistema de coordenadas WGS84 4326 https://epsg.io/4326
sf::sf_use_s2(F) # desactivar el uso de la biblioteca s2 para las operaciones geométricas esféricas. Esto optimiza algunos analisis de sf.

### Cargar area de estudio ####
studyArea<- terra::vect(input$studyArea) %>% terra::buffer(0) %>% terra::aggregate() %>% sf::st_as_sf() # se carga y se disuleve para optimizar el analisis

### Cargar coberturas ####
list_covs<- pblapply(input$timeNatCoverList, function(x) st_read(x))
list_covs<- list_covs[sort(names(list_covs))] # ordenar por año

#### Corte de coberturas por area de estudio ####
list_covs_studyArea<- pblapply(list_covs, function(NatCovs) {
  test_crop_studyArea<- NatCovs  %>%  st_crop( studyArea )
  test_intersects_studyArea<- sf::st_intersects(studyArea, test_crop_studyArea) %>% as.data.frame()
  NatCovs_studyArea<- st_intersection(studyArea[unique(test_intersects_studyArea$row.id)], test_crop_studyArea[test_intersects_studyArea$col.id,])
})
```

Una vez cargados los insumos del área de estudio y las coberturas
naturales en diferentes períodos de tiempo, se realiza el corte de esos
mapas de coberturas naturales por el área de estudio. El objeto
`list_covs_studyArea` corresponde a la representación espacial de esa
intersección.

## Estimar area por periodo

Con los insumos ajustados al área de estudio, para estimar la variación
de la extensión es necesario calcular el área de esas coberturas
naturales en el área de estudio. Se calcula el área en km² para cada
periodo y se almacena en tablas de área por periodo.

``` r
## Estimar area por periodo ####
area_cobsNat<- pblapply(names(list_covs_studyArea), function(i_testArea) {
  area_pol<-  list_covs_studyArea[[i_testArea]] %>% dplyr::mutate(period= i_testArea, area_km2= st_area(.) %>%  units::set_units("km2")) %>% 
    st_drop_geometry() %>% dplyr::group_by(period) %>% dplyr::summarise(area_km2= as.numeric(sum(area_km2, na.rm=T)))
  area_pol
  }) %>% plyr::rbind.fill()
print(area_cobsNat)
```

Esto permite realizar un seguimiento de las variaciones en la extensión
de los ecosistemas naturales continentales a lo largo del tiempo.
Nuestro objetivo es estandarizar esa variación en un índice. Para ello,
procedemos a estimar el cambio respecto a un periodo definido. En este
caso, estimamos el cambio y el porcentaje de cambio de extensión
respecto al último periodo anterior de medición, utilizando la fórmula
de cambio: periodo anterior - periodo actual, y el porcentaje como el
cambio sobre el periodo anterior.

![](README_figures/ecuaciones_tendencia.png)

Adicionalmente, calculamos la tendencia acumulativa de cambio, que
refleja la tendencia general de los cambios a lo largo del tiempo. La
tendencia acumulativa se obtiene sumando el porcentaje de cambio actual
y la media de los porcentajes de cambio de los periodos anteriores,
proporcionando una visión más integral de las tendencias a largo plazo
en la extensión de los ecosistemas naturales.

``` r
## Estimar cambio respecto al periodo anterior y tendencia ####
changeArea_cobsNat<- area_cobsNat %>% dplyr::mutate(changeArea= NA, perc_changeArea= NA, trend=NA)

for(i in seq(nrow(changeArea_cobsNat)) ){
if(i>1){
  changeArea_cobsNat[i,"changeArea"]<- changeArea_cobsNat[i,"area_km2"]  - changeArea_cobsNat[i-1,"area_km2"] # estimar cambio en extension
  changeArea_cobsNat[i,"perc_changeArea"]<-  changeArea_cobsNat[i,"changeArea"] / changeArea_cobsNat[i-1,"area_km2"] # estimar cambio porcentual
  changeArea_cobsNat[i,"trend"]<-  changeArea_cobsNat[i,"perc_changeArea"] + ifelse(is.na(changeArea_cobsNat[i-1,"perc_changeArea"]), 0, mean(changeArea_cobsNat[2:(i-1),"perc_changeArea"], na.rm=T)) # estimar tendencia de cambio
  }
}
print(changeArea_cobsNat)
```

## Plot de cambio y tendencia

Por ultimo se ajustaron los resultados para visualización que permita
entender las tendencias obtenidas. Esto permite observar los cambios en
la extensión de los ecosistemas naturales y su tendencia acumulativa, lo
que facilita la interpretación de los datos y apoya la toma de
decisiones en la gestión de los recursos naturales.

``` r
## Plot de cambio y tendencia ####
changeArea_cobsNat_data<- changeArea_cobsNat %>% dplyr::mutate(period= as.numeric(period))

changeArea_cobsNat_plotdata<- tidyr::pivot_longer(changeArea_cobsNat_data, cols = -period, names_to = "variable", values_to = "value")

changeArea_plot<- ggplot(changeArea_cobsNat_plotdata, aes(x = period, y = value, color = variable)) +
  geom_line(group = 1) +
  geom_point() +
  facet_wrap(~ variable, scales = "free_y") +
  theme_minimal()

print(changeArea_plot)
```

![](README_figures/results_trend.png)

Las áreas estimadas por periodos se presentan como valores absolutos, el
cambio en área como el delta entre los periodos comparados, mientras que
el porcentaje de cambio de área y la tendencia se representan en una
escala entre -1 y 1. En esta escala, -1 representa una pérdida de
extensión de coberturas naturales en ecosistemas continentales, y 1
indica que se mantuvo o se superó la extensión de referencia. Esto
permite una interpretación clara y directa de los cambios y tendencias
en la extensión de los ecosistemas naturales a lo largo del tiempo.
