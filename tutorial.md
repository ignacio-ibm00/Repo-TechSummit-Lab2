# Antes de empezar: Cloná el repositorio

Antes de ejecutar cualquier comando, necesitás clonar el repositorio oficial del tutorial y posicionarte en la carpeta del proyecto:

```bash
git clone https://github.com/IBM/oic-i-agentic-ai-tutorials
cd oic-i-agentic-ai-tutorials/confluent-agents
```

> **Nota:** Todos los archivos mencionados en los pasos siguientes (`get_sku_availability.py`, `sku-availability-agents.yaml`, `Substitute_Finder_Agent.yaml`, `Store_Associate_Agent.yaml`, `product-catalog.docx`) se encuentran dentro de esta carpeta `confluent-agents`.

---

# Paso 1. Creá la herramienta MCP y el agente de IA en watsonx Orchestrate

En este paso, vas a crear la herramienta MCP y el agente de IA en **watsonx Orchestrate**. Las configuraciones de la herramienta MCP y del agente de IA fueron creadas y validadas con la ayuda de **IBM Bob**.

Para más detalles sobre cómo usar Bob para crear herramientas MCP y agentes, revisá este tutorial: [Usando IBM Bob para construir agentes de watsonx Orchestrate y herramientas MCP](https://developer.ibm.com/tutorials/build-agents-mcp-tools-watsonx-orchestrate-using-bob/).

## Copiá el archivo .env del Lab 1

Este lab necesita un archivo `.env` con las variables de conexión al clúster de Confluent (Kafka + ksqlDB). Las dos variables clave son:


| Variable            | Descripción                                                                       |
| ------------------- | ---------------------------------------------------------------------------------- |
| `BOOTSTRAP_SERVERS` | Dirección del broker de Kafka (ej.`localhost:9092`)                               |
| `KSQLDB_ENDPOINT`   | URL del servidor ksqlDB para consultas en tiempo real (ej.`http://localhost:8088`) |

No se necesitan credenciales adicionales porque el clúster corre en un entorno local aislado.

**Si hiciste el Lab 1**, ese archivo ya existe — copialo a la carpeta `confluent-agents`:

```bash
cp /ruta/al/Lab1/inventory-pipeline/.env confluent-agents/.env
```

> Reemplazá `/ruta/al/Lab1` con la ruta absoluta donde tenés el Lab 1 en tu máquina.

**Si no hiciste el Lab 1**, creá manualmente el archivo `confluent-agents/.env` con este contenido:

```env
BOOTSTRAP_SERVERS= *****
KSQLDB_ENDPOINT=****
```

> Ajustá los valores si tu clúster de Confluent usa puertos o direcciones distintas.

## Importá la herramienta MCP en watsonx Orchestrate

Una **herramienta MCP** (Model Context Protocol) es la forma en que watsonx Orchestrate expone una función externa —en este caso, un script Python— para que un agente de IA pueda invocarla. Al importarla, le estás diciendo a Orchestrate: "este script existe, tiene estas capacidades, y los agentes pueden usarlo como herramienta".

El comando registra el script `get_sku_availability.py` como una herramienta llamada `sku-availability-checker`. Cuando un agente necesite consultar el stock, llamará a esta herramienta y devolverá los datos desde Kafka vía ksqlDB.

Reemplazá `/Users/ahmedazraq/Documents/git/oic-i-agentic-ai-tutorials/confluent-agents` con la **ruta absoluta** de tu proyecto.

```bash
orchestrate toolkits add --kind mcp --name "sku-availability-checker" --description "Verificador de disponibilidad de inventario en tiempo real usando Confluent Kafka y ksqlDB" --language python --package-root "/Users/ahmedazraq/Documents/git/oic-i-agentic-ai-tutorials/confluent-agents" --command "python3 get_sku_availability.py" --tools "*"
```

## Importá el agente

El archivo `sku-availability-agents.yaml` define el comportamiento del agente: su nombre, descripción, qué herramientas puede usar y cómo debe razonar. Importarlo registra ese agente en watsonx Orchestrate para que pueda ser desplegado y probado.

```bash
orchestrate agents import -f sku-availability-agents.yaml
```

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

1. En el menú lateral izquierdo, hacé clic en **Build**
2. Vas a ver el agente `SKU_Availability_Agent` que acabás de importar — hacé clic en él
3. En la esquina superior derecha, hacé clic en **Deploy**
4. Confirmá el despliegue en la ventana de **Pre-deployment summary**

## Probá el agente

1. Abrí la interfaz de **watsonx Orchestrate**
2. Andá a **Administrar agentes**
3. Hacé clic en **SKU_Availability_Agent**
4. Observá que la herramienta MCP ya está importada
5. Revisá el comportamiento del agente

**Pregunta de prueba:**

> ¿Cuáles son los SKUs disponibles en Mall of Egypt?

**Respuesta esperada:** El agente devuelve una tabla con la disponibilidad en tiempo real de todos los SKUs en la sucursal indicada, incluyendo un resumen de cuáles están sin stock.

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

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

3. En el menú lateral izquierdo, hacé clic en **Build**
4. Vas a ver el agente `Substitute_Finder_Agent` que acabás de importar — hacé clic en él
5. En la esquina superior derecha, hacé clic en **Deploy**
6. Confirmá el despliegue en la ventana de **Pre-deployment summary**

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

1. Dentro del agente, dirigite a la sección [`Knowledge`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/knowledge).
2. Hacé clic en el botón [`Add Source`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/index.html?id=8924ea03-22e4-4b6f-94ab-6d25c7feb9e1&parentId=2&origin=6e582a5d-694b-4ec7-9859-ecdc12368314&swVersion=4&extensionId=IBM.bob-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView).
3. Seleccioná la opción para agregar una nueva base de conocimiento.
4. En esta pantalla vas a ver las conexiones disponibles que se pueden integrar como fuente de conocimiento para el agente. Para este laboratorio, vamos a cargar un archivo local.
5. Hacé clic en [`Upload Files`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/index.html?id=8924ea03-22e4-4b6f-94ab-6d25c7feb9e1&parentId=2&origin=6e582a5d-694b-4ec7-9859-ecdc12368314&swVersion=4&extensionId=IBM.bob-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView).
6. Seleccioná el archivo [`product-catalog`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/product-catalog).
7. Hacé clic en [`Next`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/index.html?id=8924ea03-22e4-4b6f-94ab-6d25c7feb9e1&parentId=2&origin=6e582a5d-694b-4ec7-9859-ecdc12368314&swVersion=4&extensionId=IBM.bob-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView).
8. Ingresá el nombre [`enterprise_documents`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/enterprise_documents) y agregá una breve descripción.
9. Hacé clic en [`Save`](vscode-webview://0hjanbl2bnk8ul0eelg5qih29rdeb2eelms4argv756c0ag7719r/index.html?id=8924ea03-22e4-4b6f-94ab-6d25c7feb9e1&parentId=2&origin=6e582a5d-694b-4ec7-9859-ecdc12368314&swVersion=4&extensionId=IBM.bob-code&platform=electron&vscode-resource-base-authority=vscode-resource.vscode-cdn.net&parentOrigin=vscode-file%3A%2F%2Fvscode-app&purpose=webviewView).
10. Esperá unos minutos hasta que finalice la indexación.
11. Verificá que el documento figure como disponible y listo para ser usado en búsquedas semánticas.

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

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

3. En el menú lateral izquierdo, hacé clic en **Build**
4. Vas a ver el agente `Store_Associate_Agent` que acabás de importar — hacé clic en él
5. En la esquina superior derecha, hacé clic en **Deploy**
6. Confirmá el despliegue en la ventana de **Pre-deployment summary**

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

## ¿Qué lograste hasta acá?

Con los tres pasos anteriores construiste un **sistema multi-agente** completo para uso interno:

- El **SKU Availability Agent** sabe consultar el stock en tiempo real desde Kafka vía ksqlDB.
- El **Substitute Finder Agent** sabe razonar sobre el catálogo de productos para encontrar alternativas similares.
- El **Store Associate Agent** actúa como supervisor: recibe la pregunta, decide qué agente especialista invocar según el resultado, y devuelve una respuesta unificada.

Este patrón de orquestación —un agente coordinador que delega en agentes con objetivos específicos— permite escalar la complejidad sin que ningún agente individual tenga que saber todo. Cada uno hace una sola cosa bien, y el supervisor combina los resultados.

---

# Paso 4. Creá el Asistente de Compra para el cliente final

En los pasos anteriores, construiste agentes orientados a **procesos internos**: verificar stock, encontrar sustitutos y coordinar la respuesta para un asociado de tienda. En este paso, vas a crear un agente pensado para el **cliente final** que visita el sitio web de la tienda.

La diferencia clave es el punto de partida: el cliente no sabe qué SKU quiere, describe su necesidad en lenguaje natural ("busco una laptop para editar videos") y espera una respuesta amigable, sin términos técnicos. El agente se encarga de traducir esa necesidad en una recomendación concreta con disponibilidad en tiempo real.

## ¿Qué hace este agente?

El **Asistente de Compra** realiza las siguientes acciones:

1. Entiende la necesidad del cliente (uso, presupuesto) a partir de una conversación natural
2. Busca en el catálogo de productos los artículos que mejor se adaptan a esa necesidad
3. Verifica en tiempo real si esos productos están disponibles en la sucursal que el cliente va a visitar
4. Responde con una recomendación clara, en tono de asesor de ventas, sin exponer ningún detalle técnico interno

> **Nota:** Este agente reutiliza la misma base de conocimiento (`enterprise_documents`) y el mismo agente de disponibilidad (`SKU_Availability_Agent`) que ya creaste en los pasos anteriores. No necesitás configurar nada nuevo.

## Creá el Asistente de Compra

El agente está definido en el archivo `Customer_Shopping_Assistant.yaml` dentro de la carpeta `confluent-agents`.

1. Importá el agente usando el **Kit de Desarrollo de Agentes (ADK)**:

```bash
cd confluent-agents
orchestrate agents import -f Customer_Shopping_Assistant.yaml
```

2. Después de que se complete la importación, abrí la interfaz de **watsonx Orchestrate**, andá a **Administrar agentes** y hacé clic en **Asistente de Compra**

## Vinculá la base de conocimiento

1. En el menú lateral del agente, hacé clic en **Knowledge**
2. Hacé clic en **Add source**
3. Seleccioná la opción para agregar una fuente **existente**
4. Elegí `enterprise_documents` de la lista (es el catálogo de productos que ya cargaste en el Paso 2)
5. Hacé clic en **Save**
6. Esperá a que el documento aparezca como disponible en la sección de Knowledge

> **¿Por qué es necesario?** Sin la base de conocimiento, el agente respondería usando el conocimiento general del modelo de lenguaje en lugar del catálogo de productos real de la empresa. Esto puede generar especificaciones o precios incorrectos.

## Vinculá el agente de disponibilidad

El Asistente de Compra necesita delegar la consulta de stock en tiempo real al `SKU_Availability_Agent`:

1. En el menú lateral del agente, hacé clic en **Toolset**
2. En la sección **Agents**, hacé clic en **Add agent**
3. Seleccioná "Local Instance" y toca en `SKU_Availability_Agent` de la lista
4. Hacé clic en **Save**

> **¿Por qué es necesario?** Sin este vínculo, el agente no puede consultar el stock real de Kafka y no sabrá si el producto está disponible en la sucursal del cliente.

## Desplegá el agente

Una vez completada la configuración, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

1. En el menú lateral izquierdo, hacé clic en **Build**
2. Vas a ver el agente `Customer_Shopping_Assistant` que acabás de importar — hacé clic en él
3. En la esquina superior derecha, hacé clic en **Deploy**
4. Confirmá el despliegue en la ventana de **Pre-deployment summary**

## Probá el agente

Abrí la interfaz de **watsonx Orchestrate**, andá a **Administrar agentes** y hacé clic en **Asistente de Compra**.

### Prueba A – El agente guía al cliente con preguntas

```
Busco una laptop para trabajo creativo.
```

**Resultado esperado:** El agente no asume nada. Pregunta amigablemente por la sucursal que el cliente va a visitar antes de continuar.

### Prueba B – Recomendación con disponibilidad en tiempo real

```
Busco una laptop para diseño gráfico, voy a ir al Dubai Mall.
```

**Resultado esperado:** El agente consulta el catálogo para identificar los modelos más adecuados para diseño gráfico, verifica el stock en DubaiMall en tiempo real y responde con hasta 2 opciones disponibles usando el nombre comercial del producto, sin mencionar términos técnicos.

### Prueba C – Producto sin stock con alternativa

```
Quiero el iPhone más nuevo, voy al Mall of Egypt.
```

**Resultado esperado:** El agente detecta que el iPhone 17 Pro Max está sin stock en Mall of Egypt, lo informa de forma amigable y sugiere alternativas disponibles como el Samsung S24 Ultra o el Google Pixel 8 Pro, con una breve explicación de por qué son buenas opciones.

## Obtené el snippet para embeber el agente en la web

Este paso es necesario para el **Lab 3**, donde el agente se va a integrar en una página web. watsonx Orchestrate genera automáticamente un snippet de JavaScript listo para usar.

1. En el menú lateral del agente, hacé clic en **Channels**
2. Seleccioná la pestaña  **Embedded agent** -  **Live**
3. En la sección **Embed on your website**, copiá el snippet de código que aparece
4. Guardalo — lo vas a necesitar en el Lab 3

El snippet tiene esta estructura:

```html
<script>
  window.wxOConfiguration = {
    orchestrationID: "...",
    hostURL: "https://ca-tor.watson-orchestrate.cloud.ibm.com",
    rootElementID: "root",
    deploymentPlatform: "ibmcloud",
    crn: "...",
    chatOptions: {
      agentId: "...",
      agentEnvironmentId: "...",
    }
  };
  setTimeout(function () { ... });
</script>
```

> **Nota:** Los valores de `orchestrationID`, `crn`, `agentId` y `agentEnvironmentId` son únicos para tu instancia. No los compartas públicamente.

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