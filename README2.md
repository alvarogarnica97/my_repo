## Versiones utilizadas
Las versiones utilizadas para la ejecución en local son:
Kafka 2.12-2.3.0
Spark 2.4.0
Mongo 4.4.1
Python 3.7.4

Aunque no son las más actuales, se han empleado estas ya que para la predicción de vuelos, se ha empleado Spark-submit en vez de Intellij.

## Descarga de datos
El fichero JSON con las distancias se obtiene con el siguiente comando:

```
resources/download_data.sh
```

Los ficheros generados se instalan en la carpeta /data.

## Instalación de dependencias
Las librerías requeridas se instalan ejecutando el siguiente comando desde la carpeta raíz:

```
pip install -r requirements.txt
```

## Kafka y Zookeeper
Para ejecutar Zookeeper y Kafka, nos ubicamos en la carpeta de Kafka descargada y ejecutamos los siguientes comandos:

```
bin/zookeeper-server-start.sh config/zookeeper.properties

bin/kafka-server-start.sh config/server.properties

bin/kafka-topics.sh \
        --create \
        --zookeeper localhost:2181 \
        --replication-factor 1 \
        --partitions 1 \
        --topic flight_delay_classification_request

```

El primero inicia Zookeeper, el segundo inicia Kafka, y el tercero crea el topic de Kafka. Al tercer comando nos debería responder con:

```
Created topic flight_delay_classification_request
```

## MongoDB
En el comando de este paso se va a emplear el fichero JSON descargado en el primer paso, por lo que es importante que ese haya funcionado correctamente para poder ejecutar este. El comando a ejecutar desde la carpeta raíz para importar los registros en Mongo es:
```
./resources/import_distances.sh
```
A lo cual debería responder algo similar a:
```
2020-11-29T18:52:55.335+0100	connected to: mongodb://localhost/
2020-11-29T18:52:55.643+0100	4696 document(s) imported successfully. 0 document(s) failed to import.
MongoDB shell version v4.4.1
connecting to: mongodb://127.0.0.1:27017/agile_data_science?compressors=disabled&gssapiServiceName=mongodb
Implicit session: session { "id" : UUID("47556e9c-ff49-48f9-9fba-a44555aa2b54") }
MongoDB server version: 4.4.1
{
	"numIndexesBefore" : 2,
	"numIndexesAfter" : 2,
	"note" : "all indexes already exist",
	"ok" : 1
}
```

## Modelo con Spark
En primer lugar es importante tener declaradas las variables de entorno JAVA_HOME y SPARK_HOME. Debería ser algo similar a:
```
export JAVA_HOME=/Library/Java/JavaVirtualMachines/jdk1.8.0_191.jdk/Contents/Home 
export SPARK_HOME=/path_to_spark/spark-2.4.0-bin-hadoop2.7 
```

Ya se puede proceder a entrenar el modelo:
```
python3 resources/train_spark_mllib_model.py .
```

El resultado es que en la carpeta /models se crean todos los modelos.

Posteriormente, en la clase de Scala MakePrediction se cambia el valor base_path por el path de la raíz del proyecto:
```
val base_path= "/path_to_project/practica_big_data_2020" 
```

## Ejecución con Spark-Submit
En primer lugar, dentro de flight_prediction nos aseguramos que en el fichero build.sbt estén las versiones utilizadas. Una vez hecho esto, ejecutamos los dos siguientes comandos:
```
sbt compile
sbt package
```

Como resultado, se genera una carpeta /target, y si buceamos por ella hasta /target/scala-2.11, vemos que se ha generado un fichero .jar, el cual se va a emplear en el siguiente comando de Spark-submit para ejecutar el predictor de vuelos. Este comando se ejecuta desde la carpeta de Spark descargada:
```
bin/spark-submit --class es.upm.dit.ging.predictor.MakePrediction --master local --packages org.mongodb.spark:mongo-spark-connector_2.11:2.3.2,org.apache.spark:spark-sql-kafka-0-10_2.11:2.4.0 /Users/Gar/Desktop/BDFI/practica_big_data_2020/flight_prediction/target/scala-2.11/flight_prediction_2.11-0.1.jar
```

## Servidor Web
Una vez se ha ejecutado el predictor de vuelos, para lanzar el servidor web de Flask, primero hay que añadir una variable de entorno con la ubicación de la carpeta raíz del proyecto:
```
export PROJECT_HOME=/path_to_project/practica_big_data_2020 
```

Ejecutamos el servidor web:
```
python3 predict_flask.py
```
