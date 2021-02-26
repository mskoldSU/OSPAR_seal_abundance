Preparing files for OSPAR seals reporting
================

``` r
library(tidyverse)
```

    ## -- Attaching packages --------------------------------------- tidyverse 1.3.0 --

    ## v ggplot2 3.3.3     v purrr   0.3.4
    ## v tibble  3.0.6     v dplyr   1.0.4
    ## v tidyr   1.1.2     v stringr 1.4.0
    ## v readr   1.4.0     v forcats 0.5.1

    ## -- Conflicts ------------------------------------------ tidyverse_conflicts() --
    ## x dplyr::filter() masks stats::filter()
    ## x dplyr::lag()    masks stats::lag()

``` r
library(sf)
```

    ## Linking to GEOS 3.8.0, GDAL 3.0.4, PROJ 6.3.1

## Read shapefiles with region borders

``` r
BalticAreas <- st_read("polygons/BA.shp") %>% 
  rename(BalticAreas_id = id) %>% 
  filter(!st_is_empty(.))
```

    ## Reading layer `BA' from data source `C:\Users\skold\Dropbox\Projekt\OSPAR_seal_abundance\polygons\BA.shp' using driver `ESRI Shapefile'
    ## replacing null geometries with empty geometries
    ## Simple feature collection with 20 features and 1 field (with 3 geometries empty)
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: 8.718394 ymin: 53.56644 xmax: 30.65053 ymax: 66.04611
    ## geographic CRS: WGS 84

``` r
SwePolygons <- st_read("polygons/SwP.shp") %>% 
  rename(SwePolygons_id = id)
```

    ## Reading layer `SwP' from data source `C:\Users\skold\Dropbox\Projekt\OSPAR_seal_abundance\polygons\SwP.shp' using driver `ESRI Shapefile'
    ## replacing null geometries with empty geometries
    ## Simple feature collection with 390 features and 1 field (with 7 geometries empty)
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: 10.75038 ymin: 55.31165 xmax: 24.15853 ymax: 65.78757
    ## geographic CRS: WGS 84

``` r
SweRegions <- st_read("polygons/SwR.shp") %>% 
  rename(SweRegions_id = id) %>% 
  select(-Regioner)
```

    ## Reading layer `SwR' from data source `C:\Users\skold\Dropbox\Projekt\OSPAR_seal_abundance\polygons\SwR.shp' using driver `ESRI Shapefile'
    ## replacing null geometries with empty geometries
    ## Simple feature collection with 32 features and 2 fields (with 2 geometries empty)
    ## geometry type:  POLYGON
    ## dimension:      XY
    ## bbox:           xmin: 10.63806 ymin: 55.13273 xmax: 24.17829 ymax: 66.02663
    ## geographic CRS: WGS 84

## Read seals data and join with shapefiles to add region id:s

``` r
data <- readxl::read_excel("data/HSvktoMartin.xlsx") %>% 
  select(Date = Datum, LatDD, LonDD, N_HS) %>% 
  st_as_sf(coords = c("LonDD", "LatDD"), remove = FALSE) %>% 
  st_set_crs("+proj=longlat +datum=WGS84") %>% 
  st_join(BalticAreas) %>% 
  st_join(SweRegions) %>% 
  st_join(SwePolygons)
```

    ## although coordinates are longitude/latitude, st_intersects assumes that they are planar
    ## although coordinates are longitude/latitude, st_intersects assumes that they are planar
    ## although coordinates are longitude/latitude, st_intersects assumes that they are planar
    ## although coordinates are longitude/latitude, st_intersects assumes that they are planar

## Compute daily sum over SwePolygons and save

``` r
daily_SwePolygons <- data %>% 
  as_tibble() %>% 
  select(-geometry) %>% 
  group_by(Date, BalticAreas_id, SweRegions_id, SwePolygons_id) %>% 
  summarise(N_HS = sum(N_HS), .groups = "drop")
head(daily_SwePolygons)
```

    ## # A tibble: 6 x 5
    ##   Date                BalticAreas_id SweRegions_id SwePolygons_id  N_HS
    ##   <dttm>                       <dbl>         <dbl>          <dbl> <dbl>
    ## 1 1999-08-23 00:00:00             11             5             60     0
    ## 2 1999-08-23 00:00:00             11             5             62   290
    ## 3 1999-08-23 00:00:00             11             5             63     0
    ## 4 1999-08-23 00:00:00             11             5             64    62
    ## 5 1999-08-23 00:00:00             11             5             66    33
    ## 6 1999-08-23 00:00:00             11             5             69     9

``` r
openxlsx::write.xlsx(daily_SwePolygons, "daily_SwePolygons.xlsx")
```

## Compute daily sum over BalticAreas and save

``` r
daily_BalticAreas <- data %>% 
  as_tibble() %>% 
  select(-geometry) %>% 
  group_by(Date, BalticAreas_id) %>% 
  summarise(N_HS = sum(N_HS), .groups = "drop")
head(daily_BalticAreas)
```

    ## # A tibble: 6 x 3
    ##   Date                BalticAreas_id  N_HS
    ##   <dttm>                       <dbl> <dbl>
    ## 1 1999-08-23 00:00:00             11  3052
    ## 2 1999-08-23 00:00:00             12  2276
    ## 3 1999-08-24 00:00:00             11  3130
    ## 4 1999-08-24 00:00:00             12  2166
    ## 5 1999-08-25 00:00:00             11  3131
    ## 6 1999-08-25 00:00:00             12  2734

``` r
openxlsx::write.xlsx(daily_BalticAreas, "daily_BalticAreas.xlsx")
```

## Convert SwePolygons to geojson and save

``` r
SwePolygons_json <- tibble(id = SwePolygons$SwePolygons_id, 
                           geojson = geojsonsf::sf_geojson(SwePolygons, atomise = TRUE))
openxlsx::write.xlsx(SwePolygons_json, "SwePol_geojson.xlsx")
```
