# Lab 2 — Agentic AI con Confluent y watsonx Orchestrate

Este laboratorio muestra cómo construir un sistema multi-agente de IA impulsado por eventos usando **Confluent Platform** (Kafka + ksqlDB) y **IBM watsonx Orchestrate**.

## Estructura del repositorio

```
Lab2/
├── tutorial.md                          # Guía paso a paso del laboratorio
└── confluent_agents/
    ├── .env.example                     # Plantilla de variables de entorno (sin credenciales)
    ├── get_sku_availability.py          # MCP tool: consulta inventario en tiempo real vía ksqlDB
    ├── sku-availability-agent.yaml      # Definición del agente SKU Availability
    ├── Substitute_Finder_Agent.yaml     # Definición del agente Substitute Finder (RAG)
    ├── Store_Associate_Agent.yaml       # Definición del agente supervisor Store Associate
    ├── produce_messages.py              # Script para producir mensajes al topic Kafka
    ├── setup_topic_with_samples.py      # Script para crear el topic y cargar datos de muestra
    ├── sample-transactions.json         # Datos de inventario de muestra
    ├── product-catalog.docx             # Catálogo de productos (base de conocimiento para RAG)
    └── requirements.txt                 # Dependencias Python
```

## Configuración inicial

1. Cloná este repositorio y posicionarte en la carpeta del lab:

```bash
git clone <url-del-repo>
cd Lab2/confluent_agents
```

2. Copiá el archivo de variables de entorno y completá con tus credenciales:

```bash
cp .env.example .env
```

Editá `.env` con el endpoint y credenciales de tu entorno Confluent. Ver `.env.example` para referencia de cada variable.

3. Instalá las dependencias Python:

```bash
pip install -r requirements.txt
```

## Siguientes pasos

Seguí el tutorial paso a paso en `tutorial.md`.
