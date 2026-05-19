# Paso 1. Creá la herramienta MCP y el agente de IA en watsonx Orchestrate

En este paso, vas a crear la herramienta MCP y el agente de IA en **watsonx Orchestrate**. Las configuraciones de la herramienta MCP y del agente de IA fueron creadas y validadas con la ayuda de **IBM Bob**.

Para más detalles sobre cómo usar Bob para crear herramientas MCP y agentes, revisá este tutorial: [Usando IBM Bob para construir agentes de watsonx Orchestrate y herramientas MCP](https://developer.ibm.com/tutorials/build-agents-mcp-tools-watsonx-orchestrate-using-bob/).

## Actualizá el archivo .env

Actualizá el archivo `.env` con los detalles de tu clúster **ksqlDB**, ya que tu herramienta MCP va a comunicarse con el recién creado a través de ksqlDB.

## Importá la herramienta MCP en watsonx Orchestrate

Reemplazá `/Users/ahmedazraq/Documents/git/oic-i-agentic-ai-tutorials/confluent-agents` con la **ruta absoluta** de tu proyecto.

```bash
orchestrate toolkits add --kind mcp --name "sku-availability-checker" --description "Verificador de disponibilidad de inventario en tiempo real usando Confluent Kafka y ksqlDB" --language python --package-root "/Users/ahmedazraq/Documents/git/oic-i-agentic-ai-tutorials/confluent-agents" --command "python3 get_sku_availability.py" --tools "*"
```


## Importá el agente

Importá el agente a través del archivo de definición YAML.

```bash
orchestrate agents import -f sku-availability-agents.yaml
```

## Probá el agente

1. Abrí la interfaz de **watsonx Orchestrate**
2. Andá a **Administrar agentes**
3. Hacé clic en **SKU_Availability_Agent**
4. Observá que la herramienta MCP ya está importada
5. Revisá el comportamiento del agente

**Pregunta de prueba:**

> ¿Cuáles son los SKUs disponibles en Mall of Egypt?


---

# Paso 2. Creá el agente RAG agéntico en watsonx Orchestrate

En este paso, vas a crear el **Agente Buscador de Sustitutos**, que es responsable de sugerir alternativas de productos adecuadas cuando un SKU solicitado no está disponible en una sucursal específica. A diferencia del **Agente de Disponibilidad de SKU**, que depende del estado de Kafka en tiempo real, este agente razona sobre documentos de productos empresariales usando **RAG agéntico**.

## Propósito

El propósito de este paso es demostrar cómo un agente puede combinar comprensión semántica de especificaciones de productos con razonamiento estructurado, en lugar de depender de reglas estáticas o mapeos codificados.

## Acciones del Agente Buscador de Sustitutos

El **Agente Buscador de Sustitutos** realiza las siguientes acciones:

- Lee especificaciones y descripciones de productos de documentos empresariales
- Entiende las características de un SKU solicitado (categoría, nivel, factor de forma, características clave)
- Encuentra productos similares usando búsqueda de similitud semántica
- Devuelve 2–3 SKUs sustitutos con una breve explicación de por qué son buenas alternativas

> **Nota:** Este agente no interactúa directamente con Kafka. La disponibilidad de inventario es manejada por el Agente de Disponibilidad de SKU en el paso anterior.

## Creá el Agente Buscador de Sustitutos

El **Agente Buscador de Sustitutos** está definido en un archivo YAML proporcionado en el repositorio Git.

1. Ubicá el archivo de definición del agente: `Substitute_Finder_Agent.yaml`, en tu clon local del repositorio

2. Importá el agente en watsonx Orchestrate usando el **Kit de Desarrollo de Agentes (ADK)**:

```bash
cd confluent-agents
orchestrate agents import -f Substitute_Finder_Agent.yaml
```

3. Después de que se complete la importación, **desplegá el agente** para que se active
4. Hacé clic en el botón **Desplegar**
5. Desplegá nuevamente en la ventana de **Resumen pre-despliegue**


> En este punto, el agente está creado y desplegado, pero todavía no tiene acceso a documentos empresariales. En el siguiente paso, vas a adjuntar el catálogo de productos como su fuente de conocimiento.

## Subí el catálogo de productos a watsonx Orchestrate

Este tutorial usa un único documento de Word que representa un **catálogo de productos interno**. El documento contiene múltiples entradas de productos en un formato consistente, lo que lo hace adecuado para búsqueda semántica y coincidencia de similitud.

### Ubicá el archivo del catálogo

En tu clon local del repositorio, ubicá el archivo: **`product-catalog.docx`**

Está incluido en la carpeta de recursos del tutorial.

### Productos de muestra

El catálogo incluye productos de muestra para este paso como:
- `LAPTOP-DELL-XPS-15`
- `LAPTOP-HP-SPECTRE-X360`
- Y más...

Estos productos intencionalmente comparten varios atributos, como:
- Categoría
- Clase de procesador
- Memoria
- Almacenamiento
- Nivel de uso

Esta superposición permite que el agente los identifique como sustitutos adecuados a través de similitud semántica.

### Pasos para subir el catálogo

1. Abrí la interfaz de **watsonx Orchestrate**
2. Navegá a la sección para administrar documentos empresariales o fuentes de conocimiento
3. Agregá un **Nuevo Conocimiento**
4. Hacé clic en **Subir Archivos**
5. Seleccioná el archivo `product-catalog`
6. Hacé clic en **Siguiente**
7. Establecé el nombre como `enterprise_documents` y agregá una descripción
8. Hacé clic en **Guardar**
9. Esperá hasta que se complete la indexación y confirmá que el documento está disponible para búsqueda semántica

## Probá el agente en la interfaz de watsonx Orchestrate

Probá el agente de forma aislada antes de integrarlo con el agente supervisor en el siguiente paso.

### Prompt 1 – Prueba de fundamentación

```
Del catálogo de productos empresariales, recuperá la entrada para el SKU LAPTOP-DELL-XPS-15 y listá sus atributos clave.
```

**Resultado esperado:** El agente recupera la entrada del catálogo y lista los atributos definidos en el documento, sin hacer preguntas de seguimiento o confirmación.

### Prompt 2 – Prueba de similitud

```
LAPTOP-DELL-XPS-15 no está disponible. Sugerí una laptop similar usando el catálogo de productos.
```

**Resultado esperado:** El agente recomienda `HP-SPECTRE-X360` y explica la recomendación usando atributos compartidos del catálogo.

---

# Paso 3. Creá el agente supervisor en watsonx Orchestrate

En este paso, vas a crear un **Agente Asociado de Tienda** que actúa como agente supervisor. Su rol es coordinar los agentes creados anteriormente y proporcionar un único punto de interacción orientado al cliente para los asociados de tienda.

> **Importante:** Este agente no interactúa directamente con Kafka o documentos empresariales. En cambio, delega tareas a agentes especializados según la solicitud del usuario y combina sus respuestas en una respuesta clara y amigable para el cliente.

## Responsabilidades del Agente Asociado de Tienda

El **Agente Asociado de Tienda** es responsable de:

- Entender la pregunta del asociado de tienda
- Delegar verificaciones de inventario al **Agente de Disponibilidad de SKU**
- Delegar recomendaciones alternativas al **Agente Buscador de Sustitutos** cuando sea necesario
- Presentar una respuesta final y concisa adecuada para la interacción con el cliente

Este patrón demuestra cómo funciona la **orquestación de agentes** en watsonx Orchestrate, donde un agente supervisor coordina múltiples agentes específicos de dominio.

## Lógica del Agente Asociado de Tienda

El **Agente Asociado de Tienda** sigue esta lógica:

1. **Recibir** una pregunta del usuario sobre disponibilidad de producto en una sucursal específica

2. **Delegar** la solicitud al Agente de Disponibilidad de SKU

3. **Si el SKU solicitado está disponible:**
   - Devolver disponibilidad y cantidad

4. **Si el SKU solicitado no está disponible:**
   - Delegar al Agente Buscador de Sustitutos
   - Devolver alternativas recomendadas con explicaciones breves

> **Nota:** La búsqueda de sucursal a sucursal está intencionalmente excluida de este tutorial y puede agregarse más adelante como una extensión.

## Creá el Agente Asociado de Tienda

El **Agente Asociado de Tienda** está definido usando un archivo de configuración YAML proporcionado en el repositorio.

1. Ubicá el archivo de definición del agente: `Store_Associate_Agent.yaml`, en la misma carpeta `confluent-agents` en el repositorio que clonaste anteriormente

2. Importá el agente usando el **Kit de Desarrollo de Agentes**:

```bash
cd confluent-agents
orchestrate agents import -f Store_Associate_Agent.yaml
```

3. Después de que se complete la importación, **desplegá el agente** para que se active
4. Hacé clic en el botón **Desplegar**
5. En la ventana de **Resumen pre-despliegue**, hacé clic en **Desplegar** nuevamente

## Probá el agente

### Prueba A (Sin stock + sustitutos)

```
¿Tenés LAPTOP-DELL-XPS-15 en MallOfEgypt?
```

### Prueba B (Ejemplo en stock)

```
¿Tenés LAPTOP-MACBOOK-PRO-16 en MallOfEgypt?
```

---

# Resumen y próximos pasos

En este tutorial, aprendiste cómo construir un **sistema de IA agéntica impulsado por eventos** usando **Confluent Cloud** y **watsonx Orchestrate**. Al consumir eventos de Kafka y correlacionarlos con contexto de documentos, el agente puede razonar sobre señales operativas en vivo y explicar su significado. Este enfoque permite sistemas de IA más receptivos y conscientes del contexto mientras mantiene el razonamiento transparente y controlado.

## El rol de IBM Bob

Donde se usó **IBM Bob**, jugó un papel clave en agilizar la experiencia de desarrollo a lo largo de este tutorial. Al convertir instrucciones en lenguaje natural en código completamente funcional, configuraciones de herramientas y comportamientos de agentes validados, Bob aceleró cada etapa del flujo de trabajo, desde la creación de tópicos de Kafka y clústeres ksqlDB hasta la generación de definiciones de herramientas MCP y archivos YAML de agentes. 

Esto permitió que el equipo de desarrollo se enfocara en:
- Arquitectura
- Patrones de razonamiento
- Diseño impulsado por eventos

En lugar de tareas de configuración de bajo nivel, demostrando cómo la **ingeniería de software asistida por IA** puede mejorar dramáticamente la productividad y consistencia.