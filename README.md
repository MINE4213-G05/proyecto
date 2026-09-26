# OceanWatch Analytics

**Proyecto final - MINE 4213 Soluciones Intensivas en Datos (2026-20)**  
**Entrega 1 · Grupo 05**

## Integrantes

- Daniel Gómez
- Raúl Higuera
- Mauricio Martínez

## Resumen

OceanWatch Analytics analiza mensajes del **Automatic Identification System (AIS)** para entender el tráfico marítimo en aguas de Estados Unidos. El proyecto procesa los archivos publicados por NOAA para los días **1 al 7 de junio de 2023** mediante PySpark en Databricks Free Edition.

El corpus contiene **60.533.559 posiciones AIS**, **31.871 MMSI distintos** en los datos crudos y **31.780 MMSI con el formato válido de nueve dígitos**. El trabajo cubre la ingesta reproducible, el perfilamiento de calidad, cinco preguntas de negocio, el diseño de almacenamiento y la gobernanza de la tabla final en Unity Catalog.

## Estructura del repositorio

- [`01_ingesta_perfilamiento.ipynb`](./01_ingesta_perfilamiento.ipynb): descarga, validación, lectura y perfilamiento de los siete días.
- [`02_preguntas_almacenamiento.ipynb`](./02_preguntas_almacenamiento.ipynb): preguntas de negocio, pruebas de almacenamiento y gobernanza.

Los notebooks deben ejecutarse en ese orden. El primero deja los CSV en:

```text
/Volumes/mine4213/proyecto/data/csv
```

El segundo materializa la tabla final:

```text
mine4213.proyecto.ais_posiciones
```

## Ingesta y perfilamiento

Los siete archivos ZIP se descargan desde [Marine Cadastre - Vessel Traffic](https://hub.marinecadastre.gov/pages/vesseltraffic). La descarga incluye reintentos y verificación de existencia, tamaño, estructura ZIP, CRC y presencia del CSV esperado. Después se descomprimen en un Volume de Unity Catalog y se leen con un esquema explícito de 17 columnas.

El perfilamiento cubre posiciones y buques por día, completitud, distribución por tipo y tamaño, valores fuera de rango, sentinels AIS, identificadores anómalos y duplicados. Entre los principales hallazgos se encuentran:

- No hay coordenadas nulas o fuera de rango ni timestamps fuera del periodo.
- 49.897 posiciones tienen un MMSI que no cumple el formato de nueve dígitos.
- Hay 1.672 claves `MMSI + BaseDateTime` repetidas; 272 presentan posiciones conflictivas.
- Se encontraron 1.388 filas excedentes por duplicado completo.
- 626 posiciones superan 60 nudos y se marcaron como sospechosas.
- Los sentinels `Heading = 511`, `COG = 360` y `SOG = 102.3` aparecen en 55,40 %, 17,00 % y 0,264 % de las posiciones, respectivamente.
- Los mayores faltantes están en `Draft` (64,43 %), `IMO` (42,77 %) y `Status` (32,96 %).

Estos problemas se documentan, pero la tabla final conserva los datos crudos para que las reglas de limpieza de la siguiente entrega sean auditables.

## Preguntas de negocio

### 1. Buques distintos por día

Se comparó `countDistinct` con `approx_count_distinct` sobre MMSI válidos. El método aproximado tuvo un error medio de **4,63 %**, un máximo de **9,21 %** y no mejoró el tiempo de ejecución, por lo que se eligió el conteo exacto para este volumen.

### 2. Tipos de buque con mayor tráfico

Los códigos se enriquecieron con el [catálogo oficial de tipos AIS](https://coast.noaa.gov/data/marinecadastre/ais/VesselTypeCodes2018.pdf). Los tres tipos con más posiciones fueron:

| Código | Tipo | Posiciones | Velocidad media |
|---:|---|---:|---:|
| 31 | Towing | 16.558.120 | 1,677 nudos |
| 37 | Pleasure Craft | 14.476.696 | 1,348 nudos |
| 60 | Passenger | 4.795.209 | 4,109 nudos |

El cálculo de velocidad excluye `SOG = 102.3`, porque representa información no disponible.

### 3. Buques con mayor distancia recorrida

Las posiciones se deduplicaron por MMSI y timestamp, se ordenaron temporalmente y se calculó la distancia Haversine entre observaciones consecutivas. Se descartaron tramos con intervalos no positivos o velocidades implícitas superiores a 60 nudos. El Top 10 se agregó por MMSI y el nombre del buque se añadió después como atributo descriptivo.

### 4. Concentración espacial

El tráfico se agrupó en celdas **H3 de resolución 8**. Las diez celdas más activas se asociaron con los 3.807 puertos del **World Port Index** mediante celdas vecinas (`k-ring`). Los principales focos corresponden a Seattle, San Diego, Bellingham, El Segundo, Port Neches, Ventura/Port Hueneme y Port Everglades.

### 5. Presencia durante la semana

De los **31.780 buques con MMSI válido**, **12.656 transmitieron los siete días (39,82 %)** y **5.964 aparecieron un solo día (18,77 %)**. Los visitantes de un solo día se concentran principalmente en Chesapeake Bay/Annapolis, Port Everglades, San Diego, Seattle y Newport.

## Almacenamiento

El propósito seleccionado fue optimizar una consulta diaria del operador que filtra por fecha y zona marítima y calcula posiciones y buques distintos por hora.

Se compararon CSV y Parquet con los codecs Snappy, Gzip, Zstandard y LZ4. **Parquet con Zstandard** fue el formato más compacto:

| Formato | Tamaño |
|---|---:|
| CSV original | 6,037 GB |
| Parquet Zstandard | 1,310 GB |

Para la tabla se eligió **Delta Lake**, que añade transacciones ACID, estadísticas por archivo, historial, time travel y mantenimiento con `OPTIMIZE`. También se compararon tres layouts:

| Layout Delta | Archivos candidatos | Datos candidatos |
|---|---:|---:|
| Sin layout | 7 de 52 | 177,9 MB |
| Particionado por fecha | 6 de 48 | 177,4 MB |
| `CLUSTER BY (fecha, LAT, LON)` | **2 de 36** | **73,5 MB** |

La tabla final utiliza **Delta sobre Parquet Zstandard con liquid clustering por `fecha`, `LAT` y `LON`**. La escritura final produjo 45 archivos y aproximadamente 1,21 GB. `OPTIMIZE` no reescribió la tabla clusterizada porque los datos ya habían quedado agrupados durante la escritura.

## Gobernanza

El catálogo, el esquema, el Volume, la tabla y sus 18 columnas se documentaron en Unity Catalog. La tabla incluye comentarios, propiedades y tags sobre la fuente, el periodo, el propósito analítico y su estado de calidad. Los comentarios de columna registran unidades, rangos, sentinels y porcentajes relevantes de faltantes.

## Tecnologías y fuentes

- Databricks Free Edition, PySpark, Delta Lake y Unity Catalog.
- H3 para agregación espacial.
- [NOAA Marine Cadastre AIS](https://hub.marinecadastre.gov/pages/vesseltraffic).
- [World Port Index - NGA Pub 150](https://msi.nga.mil/Publications/WPI).
