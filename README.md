# Lab 2 — Agentic AI con Confluent y watsonx Orchestrate

Este laboratorio muestra cómo construir un sistema multi-agente de IA impulsado por eventos usando **Confluent Platform** (Kafka + ksqlDB) y **IBM watsonx Orchestrate**.

## Estructura del repositorio

```
Repo-TechSummit-Lab2/
├── prerrequisitos-instructores.md
├── README.md
├── tutorial.md
├── assets/
└── confluent_agents/
    ├── .env.example
    ├── Customer_Shopping_Assistant.yaml
    ├── Store_Associate_Agent.yaml
    ├── Substitute_Finder_Agent.yaml
    ├── get_sku_availability.py
    ├── produce_messages.py
    ├── product-catalog.docx
    ├── requirements.txt
    ├── sample-transactions.json
    ├── setup_topic_with_samples.py
    └── sku-availability-agent.yaml
```

## Configuración inicial

1. Cloná este repositorio y posicionate en la carpeta del lab:

```bash
git clone <url-del-repo>
cd Repo-TechSummit-Lab2/confluent_agents
```

2. Copiá la plantilla de variables de entorno y completá con tus credenciales:

```bash
cp .env.example .env
```

Editá `.env` con el endpoint y credenciales de tu entorno Confluent. Consultá `.env.example` para el detalle de las variables.

3. Instalá las dependencias Python:

```bash
pip install -r requirements.txt
```

## Siguientes pasos

Seguí el tutorial paso a paso en `tutorial.md`.
