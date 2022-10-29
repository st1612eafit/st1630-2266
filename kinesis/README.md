# Universidad EAFIT
# Curso ST1630 Sistemas Intensivos en Datos, 2022-2
# Profesor: Edwin Montoya M. – emontoya@eafit.edu.co

# Amazon Kinesis Lab 5

# LAB 5 - part 1 - Logs Agent -> Kinesis Firehose -> S3 -> Glue -> Athena:

1. crear un servicio kinesis firehose por la consola web:

        Create: Delivery streaming data with Kinesis Firehose
        Name: purchaseLogs
        Source: Direct PUT or other sources
        Use: Kinesis Agent
        Destination:
        Amazon S3 – select or create a bucket (st1612orderlogs)
        S3 buffer conditions: 
        Buffer interval: 60 segs
        IAM role:
        Create new: ‘firehose_delivery_role’ with defaults

2. crear una instancia EC2 AMI2 linux

Crear el IAM Role = 'MyEC2Kinesis' y Adicionarlo a esta instancia EC2 con los permisos: AmazonKinesisFullAccess y AmazonKinesisFirehoseFullAccess

3. instalar el agente kinesis

        $ sudo yum install –y aws-kinesis-agent

4. descargar los logs (OnlineRetail.csv) ejemplo y LogsGenerator.py (ya estan en el github)

## Nota: antes de Generar Logs, descomprima el archivo: OnlineRetail.csv.gz
        $ gunzip OnlineRetail.csv.gz

5. cambiar permisos, crear directorios, etc:

        $ chmod a+x LogGenerator.py
        $ sudo mkdir /var/log/acmeco

### copie el archivo del repo github: agent.json-with-firehose hacia /etc/aws-kinesis/agent.json
        $ sudo vim /etc/aws-kinesis/agent.json

6. iniciar el agente:

        $ sudo systemctl start aws-kinesis-agent

7. ejecutar un envio de logs:

        $ sudo ./LogGenerator.py 1000

8. chequee en unos minutos el 'bucket' st1612orderlogs

9. ejecute aws glue y consulte con aws athena los datos de S3 st1612orderlogs

# LAB 5 - Logs Agent -> Kinesis Data Streams -> Lambda -> DynamoDB:

1. Crear un Kinesis Data Stream en AWS:

        Create Kinesis Stream:
        name: acmecoOrders
        Nro shards: 1
        Boton: Create Kinesis Stream

2. crear la tabla DynamoDB

        Table Name: acmecoOrders
        Partition Key: CustomerID / Number
        Sort key - optional: OrderID / String
        Use defaults setting
        boton: create

3. Configurar el kinesis-agent para enviar los logs al Kinesis Data Stream

### copie el archivo del repo github: agent.json-with-firehose-and-datastreams hacia /etc/aws-kinesis/agent.json

        $ sudo vim /etc/aws-kinesis/agent.json

        se adiciona al archivo original de firehose: 

           "flows": [
                {
                "filePattern": "/var/log/acmeco/*.log",
                "kinesisStream": "acmecoOrders",
                "partitionKeyOption": "RANDOM",
                "dataProcessingOptions": [
                        {
                        "optionName": "CSVTOJSON",
                        "customFieldNames": ["InvoiceNo", "StockCode", "Description", "Quantity", "InvoiceDate", "UnitPrice", "Customer", "Country"]
                        }
                ]
                },

4. reiniciar el servicio:

        $ sudo systemctl restart aws-kinesis-agent

5. generar logs de prueba:

        $ sudo ./LogGenerator.py 1000

6. ir a la instancia EC2 donde tenemos en kinesis-agent para consumirlos MANUALMENTE y almacenarlos en la base de datos DynamoDB

        $ sudo yum install -y python3-pip
        $ sudo pip3 install boto3

        actualizar las credenciales AWS en el linux

        $ mkdir .aws
        $ aws configure

Nota: Copy las credenciales de AWS EDUCATE en el archivo generado. tener en cuenta la region: us-east-1 y el formato: json

7. Configurar un Consumer del kinesis data stream, mediente un cliente standalone (Consumer.py)

Nota: tenga en cuenta que esta versión es python2, hay que adaptarlo a versión python3

        configurar 'Consumer.py'
        $ chmod a+x Consumer.py
        $ python3 Consumer.py

        en otra terminal, generar nuevos registros para que el consumer los adquiera de kinesis y los inserte en DynamoDB

8. Crear una funcion aws lambda para consumir de kinesis data streams e insertar en una tabla DynamoDB:

        Crear un IAM Role Lambda llamado 'acmecoOrders' con los permisos: 'AmazonKinesisReadOnlyAccess' y 'AmazonDynamoDBFullAccess'

        Crear la function lambda 'Author from scratch':
        Function name: ProcessOrders
        Runtime: python 3.9
        Use an existing role: acmecoOrders
        crearla.

        +Add Trigger: Kinesis Data Stream

9. chequear en la base de datos DynomoDB la inserción de los registros.