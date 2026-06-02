# Prerrequisitos para Instructores

## 1. Preparar el archivo .env de cada cuenta

Cada participante necesita un archivo `.env` configurado con sus credenciales de Confluent Cloud. Este archivo debe estar en la carpeta `confluent_agents/`.

**Contenido del .env:**

```bash
# ksqlDB Configuration
KSQLDB_ENDPOINT=https://<host>/ksqldb
KSQLDB_API_KEY=admin
KSQLDB_API_SECRET=<ksqldb-password>

# Kafka Configuration
BOOTSTRAP_SERVERS=<host>:9094,<host>:9095,<host>:9096
KAFKA_API_KEY=kafka-admin
KAFKA_API_SECRET=<kafka-password>

# Topic Configuration
TOPIC_NAME=inventory.transactions
```

**Pasos:**
1. Obtener las credenciales de cada cuenta de Confluent Cloud (host, passwords de ksqlDB y Kafka)
2. Crear un archivo `.env` por participante reemplazando los valores `<host>`, `<ksqldb-password>` y `<kafka-password>`
3. Distribuir cada archivo al participante correspondiente antes del lab

> **Nota:** Los archivos .env contienen credenciales sensibles. Distribuirlos de forma segura.

## 2. Recordar de hacer una intro a watsonx Orchestrate

Cuando se importa el primer agente, hacer una pequeaña introducción de dónde está cada cosa en orchestrate (muy a alto nivel)