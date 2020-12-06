# BigData
Repositorio con los códigos utilizados para la realización del proyecto de Big Data en la asignatura Tópicos Especiales en Telemática.


## 1. Ingesta, almacenamiento de datos
Como este proceso se realiza automáticamente, los comandos encontrados en este archivo no son ejecutados directamente. Por el contrario, son guardado en un archivo "jobs" como secuencias para su posterior ejecución.

Este proceso se realiza en una instancia EC2 con permisos de modificación en S3 y EMR.

Ingesta y almacenamiento de datos
Para la ingesta, se comenzó por descargar el set de datos de Covid-19 desde el punto de acceso brindado por en INS.
```
curl -o covid.data https://www.datos.gov.co/resource/gt2j-8ykr.csv
```

Posterior a la descarga, se procede a subir el dataset al bucket s3, donde será almacenado para su procesamiento.
```
aws s3 cp /home/ec2-user/data/covid.data s3://buckettelematica/covid_dataset/
```

## 2. Procesamiento de datos en HIVE
Para realizar el procesamiento de datos, se optó por utilizar la herramienta HIVE. Los queries utilizados para el procesamiento son los siguientes:

### Creación de la tabla
```
CREATE EXTERNAL TABLE Covid (fecha_reporte_web STRING,ID_de_caso INT,Fecha_de_notificacion STRING,
                             Codigo_DIVIPOLA_departamento INT,Nombre_departamento STRING,
                             Codigo_DIVIPOLA_municipio STRING,Nombre_municipio STRING,
                             Edad INT,Unidad_de_medida_de_edad STRING,
                             Sexo STRING,Tipo_de_contagio STRING,
                             Ubicacion_del_caso STRING,Estado STRING,
                             Codigo_ISO_del_pais STRING,Nombre_del_pais STRING,
                             Recuperado STRING,Fecha_de_inicio_de_sintomas STRING,
                             Fecha_de_muerte STRING,Fecha_de_diagnostico STRING,
                             Fecha_de_recuperacion STRING,Tipo_de_recuperacion STRING,
                             Pertenencia_etnica STRING,Nombre_del_grupo_etnico STRING)
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE 
LOCATION 's3://buckettelematica/covid_dataset/';
```

### Query 1: Número de casos por edad
```
INSERT OVERWRITE DIRECTORY 's3://buckettelematica/outputs/casos_por_edad/'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE 
SELECT
edad,
count(*)
FROM covid
GROUP BY edad
ORDER BY edad asc;
```

### Query 2: Número de fallecidos por edad
```
INSERT OVERWRITE DIRECTORY 's3://buckettelematica/outputs/muertos_por_edad/'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE 
SELECT
edad,
count(*)
FROM covid
WHERE estado = 'Fallecido'
GROUP BY edad
ORDER BY count(*) desc;
```

### Query 3: Número de casos por municipio
```
INSERT OVERWRITE DIRECTORY 's3://buckettelematica/outputs/casos_por_municipio/'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE 
SELECT
nombre_municipio,
count(*)
FROM covid
GROUP BY nombre_municipio
ORDER BY count(*) desc;
```

### Query 4: Número de casos por municipio
```
INSERT OVERWRITE DIRECTORY 's3://buckettelematica/outputs/dias_de_recuperacion/'
ROW FORMAT DELIMITED FIELDS TERMINATED BY ','
STORED AS TEXTFILE 
-- Dias para recuperacion promedio
SELECT
edad,
-AVG(datediff(
    to_date(from_unixtime(UNIX_TIMESTAMP(fecha_de_inicio_de_sintomas, 'dd/MM/yyyy'))),
    to_date(from_unixtime(UNIX_TIMESTAMP(fecha_de_recuperacion, 'dd/MM/yyyy')))
))
FROM covid
GROUP BY edad
ORDER BY edad;
```

## 3. Automatización del proceso
Para lograr que esto se realice automáticamente y de manera periódica, se crea un archivo para ser usado como Step en la creación del cluster EMR y se prorgaman tareas con crontab para la desgarca de datos y creación del cluster.

### Step
En el proceso de creación de un cluster EMR, se encuentra la opción "agregar Steps". Se agregó como step un archivo, almacenado en S3, cuyo contenido son los queries HIVE llamados en el punto 2 y se activó la opción de terminar el cluster al acabar todos los steps, con el fin de no tener un gasto de créditos muy alto.
Igualmente, al crear el cluster, se exportó el *comando para su clonación automática desde AWS CLI*.

### Crontab
Desde la instancia EC2, se creó el archivo "data/jobs" cuyo contenido son *los comandos descritos en el punto 1*, junto con el *comando para la clonación del cluster EMR* con sus respectivos Steps ya preconfigurados.

Para configurar _crontab_ se realizó el siguiente proceso:
#### Abrir el archivo _crontab_ ya existente en la máquina EC2:
```
$ crontab -e
```
#### Configurar el tiempo de repetición y el comando a ejecutar:
```
0 1 * * * data/jobs
```
De esta manera se ejecutará la cadena de comandos, encontrada en el archivo data/jobs, todos los días a la 1:00am.
