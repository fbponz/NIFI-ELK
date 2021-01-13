# Entrega 2-NIFI_ELK

## Enunciado entrega 2:

Usando nifi+ELK debéis presentar una solución que muestre sobre un mapa la disposición de delitos presentes en esta api:
https://data.cityofnewyork.us/Social-Services/311-Service-Requests-from-2010-to-Present/erm2-nwe9

### 1)Analisis datos de entrada:

Para obtener el formato de los datos he utilizado el programa [Postman](https://www.postman.com) haciendo un GET sobre la siguiente dirección https://data.cityofnewyork.us/resource/erm2-nwe9.json

    [
        {
            "unique_key": "10693408",
            "created_date": "2015-10-01T00:00:00.000",
            "closed_date": "2015-10-06T00:00:00.000",
            "agency": "DOHMH",
            "agency_name": "Department of Health and Mental Hygiene",
            "complaint_type": "Rodent",
            "descriptor": "Mouse Sighting",
            "location_type": "Catch Basin/Sewer",
            "incident_zip": "10038",
            "incident_address": "59 MAIDEN LANE",
            "street_name": "MAIDEN LANE",
            "cross_street_1": "NASSAU STREET",
            "cross_street_2": "WILLIAM STREET",
            "address_type": "ADDRESS",
            "city": "NEW YORK",
            "facility_type": "N/A",
            "status": "Closed",
            "due_date": "2008-04-10T22:32:27.000",
            "resolution_description": "The Department of Health and Mental Hygiene will review your complaint to determine appropriate action. Complaints of this type usually result in an inspection. Please call 311 in 30 days from the date of your complaint for status",
            "resolution_action_updated_date": "2015-10-06T00:00:00.000",
            "community_board": "01 MANHATTAN",
            "bbl": "1000670001",
            "borough": "MANHATTAN",
            "x_coordinate_state_plane": "981968",
            "y_coordinate_state_plane": "197317",
            "open_data_channel_type": "UNKNOWN",
            "park_facility_name": "Unspecified",
            "park_borough": "MANHATTAN",
            "latitude": "40.708266",
            "longitude": "-74.0082309",
            "location": {
                "latitude": "40.708266",
                "longitude": "-74.0082309",
                "human_address": "{\"address\": \"\", \"city\": \"\", \"state\": \"\", \"zip\": \"\"}"
            }
        }
    ]

De los datos recibidos vamos a prestar especial atención a los siguientes:

1. Unique_key -> Identificador unico de los datos de la llamada a 311.
2. complaint_type -> Clasificación de la llamada al 311.
3. status -> Estado actual de la incidencia.
4. agency_name -> Agencia que se encarga de gestionar la incidencia.
5. Location -> Geo-localización de la llamada al 311.

Una vez tenemos sabemos el formato los datos que vamos a recibir. Debemos Transformar los datos para que puedan ser mostrados en kibana, para ello disponemos de la siguiente documentación [Tipo datos Geo-Point](https://www.elastic.co/guide/en/elasticsearch/reference/current/geo-point.html) Vamos a utilizar la forma 1 Geo-point expresado como un objeto con las claves Lat y Lon.

Si nos focalizamos en la localización, recibimos el siguiente formato:

            "location": {
                "latitude": "40.708266",
                "longitude": "-74.0082309",
                "human_address": "{\"address\": \"\", \"city\": \"\", \"state\": \"\", \"zip\": \"\"}"
            }

Para disponer de los datos y guardarlos en elasticsearch debemos hacer las siguientes transformaciones:
1. Borrar el atributo "human_address".
2. Renombrar el atributo "longitude" a "lon".
3. Renombrar el atributo "latitude" a "lat".

Ahora que tenemos claro el concepto de lo que tenemos que realizar pasamos a detallar la propuesta tecnica.

### 2)Propuesta Tecnica:

Para realizar el ejercició se ha implementado un docker-compose que contiene los siguientes modulos ElasticSearch + Kibana + Nifi. Donde se ha mapeado en los siguientes puertos.

1. ElasticSearch -> http://localhost:9200
2. Kibana -> http://localhost:5601
3. Nifi -> http://localhost:8090

la configuración de la red está habilitada con el driver "bridge".
Para ejecutar el sistema que vamos a utilizar

    docker-compose up -d

#### 2.1)NIFI:

La parte de nifi la vamos a dividir en tres partes primero Ingestión desde la API(InvokeHTTP), la segunda parte realizaremos el transformación de los datos explicada en el apartado 1) y por ultimo explicaremos los pasos para introducirlo en ElasticSearch(PutElasticsearchHTTP).

![Overview](./imagenes/Nifi_Overview.png)

##### 2.1.1)Configuración InvokeHTTP.

Para obtener los datos dese la API se ha utilizado la siguiente configuración en el modulo invokeHTTP.
![Invoke_1_2](./imagenes/Nifi_InvokeHTTP_1_2.png)
![Invoke_2_2](./imagenes/Nifi_InvokeHTTP_2_2.png)

##### 2.1.2)Transformación de datos.

La transformación necesaria para ingestar los datos dentro de elastic Search la he planteado con los siguientes modulos. Primero troceamos los datos recibidos, despues eliminamos la etiqueta "Human_Address" y despues reemplazamos las propiedades Longitude_To_Lon y Latitude_To_Lat.
![Overview_Transform](./imagenes/Nifi_Overview_Data_Transform.png)

Primero Troceamos el fichero JSON, para ello hemos gastado el modulo SplitJSON con la siguiente configuración:
![Split_JSON](./imagenes/Nifi_Data_Split_Json.png)

Despues quitamos la etiqueta "Human_Address" para ello vamos a utilizar el procesador JoltTransformJSON con la siguiente configuración:
![Delete_Human_Address_JSON](./imagenes/Nifi_Data_JoltTransformJSON.png)

Reemplazamos la etiqueta "Longitude" por "Lon" mediante la utilización del procesador ReplaceText con la siguiente configuración:
![Replace_Lon](./imagenes/Nifi_Data_ReplaceLon.png)

Reemplazamos la etiqueta "Latitude" por "Lat" mediante la utilización del procesador ReplaceText con la siguiente configuración:
![Replace_Lat](./imagenes/Nifi_Data_ReplaceLat.png)

##### 2.1.3)Introducirlo en elasticSearch.

Una vez tenemos los datos tratados vamos introducirlos en elastic Search, mediante la utilización del procesador PutElasticSearch con la siguiente configuración:
![PutElasticSearch_1_2](./imagenes/Nifi_PutElasticSearch_1_2.png)
![PutElasticSearch_2_2](./imagenes/Nifi_PutElasticSearch_2_2.png)

#### 2.2) ElasticSearch:

Antes de encender los modelos de NIFI para poder empezar a ingestar los datos sobre elasticSearch debemos de lanzar desde la terminal el siguiente comando

    curl --silent --show-error -XPUT -H 'Content-Type: application/json' \
    http://localhost:9200/311calls \
    -d'{
        "mappings": {
            "properties": {
                "location": {"type": "geo_point"}
            }
        }
        }'

Una vez ejecutado el anterior comando debemos obtener una respuesta como la siguiente, esto indica que se ha creado correctamente el indice.

    {"acknowledged":true,"shards_acknowledged":true,"index":"311calls"}

Esto nos indicara que nuestro
#### 2.2) KIBANA:

