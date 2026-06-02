# Prerrequisitos

Antes de comenzar con el tutorial, completá los siguientes pasos :

## Prerrequisito 1: Cloná el repositorio

Cloná el repositorio oficial del tutorial y posicionate en la carpeta del proyecto:

```bash
git clone https://github.com/ignacio-ibm00/Repo-TechSummit-Lab2.git
```

> **Nota:** Todos los archivos mencionados en los pasos siguientes (`get_sku_availability.py`, `sku-availability-agents.yaml`, `Substitute_Finder_Agent.yaml`, `Store_Associate_Agent.yaml`, `product-catalog.docx`) se encuentran dentro de esta carpeta `confluent_agents`.

---

## Prerrequisito 2: Creá tu IBM Cloud API Key

Necesitás crear una **API Key de IBM Cloud** para autenticarte y acceder a los servicios necesarios para este laboratorio.

### Pasos para crear la API Key

1. Iniciá sesión en tu cuenta de IBM Cloud: [https://cloud.ibm.com/login](https://cloud.ibm.com/login)

2. En el menú **Gestionar**, seleccioná **Acceso (IAM)**.

   ![Paso 2: Access IAM](assets/cloud_inicio.png)

3. En el menú **Claves de API**, hacé clic en el botón **Crear**.

   ![Paso 3: Create API Key](assets/crear_api_key.png)

4. Ingresá un nombre para tu API Key.

   ![Paso 4: Guardar API Key](assets/guardar_api_key.png)

5. Hacé clic en **Crear** para generar tu API Key.

6. **Importante:** Copiá y guardá tu API Key en un lugar seguro. No vas a poder verla nuevamente después de cerrar esta ventana.

> **Nota de seguridad:** Tratá tu API Key como una contraseña. No la compartas públicamente ni la subas a repositorios de código.

---

## Prerrequisito 3: Instalá y configurá el ADK de watsonx Orchestrate

Instalá y configurá el **Agent Development Kit (ADK)** de watsonx Orchestrate para poder importar herramientas y agentes desde la línea de comandos.

### 3.1. Instalá el ADK

Ejecutá el siguiente comando para instalar o actualizar el ADK:

```bash
pip install --upgrade ibm-watsonx-orchestrate
```

### 3.2. Creá y activá tu ambiente

El ADK usa el concepto de "ambientes" para gestionar diferentes instancias de watsonx Orchestrate. Seguí estos pasos:

**a) Creá un nuevo ambiente:**

```bash
orchestrate env add -n <nombre_del_ambiente> -u <url-instancia-de-servicio>
```

Reemplazá `<nombre_del_ambiente>` con un nombre descriptivo, por ejemplo: `labtech` o `mi-ambiente-wxo` y `<url-instancia-de-servicio>` por la URL de tu instancia.

> **¿Cómo obtener la URL de tu instancia?** 

1. Accedé desde el dashboard de IBM Cloud: 👉 [https://cloud.ibm.com](https://cloud.ibm.com)

2. Hacé clic en el menú de hamburguesa

   ![Menú hamburguesa](assets/menu_hamburguesa.png)

3. Seleccioná **Lista de recursos**

   ![Lista de recursos](assets/lista_recursos.png)

4. Seleccioná la instancia de **watsonx Orchestrate** dentro del menú desplegable "IA / Aprendizaje automático".

   ![Selección de instancia](assets/seleccion_instancia.png)

5. Copi la URL de la instancia de **watsonx Orchestrate** y reemplazá `<url-instancia-de-servicio>` con ella.

   ![URL](assets/url.png)

**b) Activá el ambiente:**

```bash
orchestrate env activate <nombre_del_ambiente>
```

**c) Ingresá tu API Key:**

Cuando se te solicite, ingresá la **API Key de IBM Cloud** que creaste en el Prerrequisito 2.


> **Importante:** Guardá el nombre de tu ambiente, lo vas a necesitar si el token expira durante el lab (ver sección de Troubleshooting al final del tutorial).

Para más detalles, consultá la [documentación oficial](https://developer.watson-orchestrate.ibm.com/getting_started/installing).

---

## Prerrequisito 4: Configuración de Confluent Cloud (Condicional)

**Este paso es solo necesario si NO participaste del Lab 1 de Confluent.**

En el lab anterior se configuró la capa de eventos en tiempo real utilizando Apache Kafka sobre Confluent Cloud. Se creó un tópico Kafka para recibir eventos de inventario, se configuró el procesamiento necesario para mantener una vista actualizada de disponibilidad y se publicaron mensajes de ejemplo que simulan movimientos de stock.

La idea es que watsonx Orchestrate trabaje con datos operacionales actualizados en tiempo real. Durante este lab, el agente consultará la información generada desde Confluent para analizar el estado actual del inventario y tomar decisiones con mayor contexto.

### Si no participaste del Lab 1

Ejecutá el siguiente comando para crear automáticamente los recursos necesarios:

```bash
cd Repo-TechSummit-Lab2/confluent_agents
python setup_topic_with_samples.py
```

Este comando generará:
- Tópico Kafka `inventory.transactions`
- Configuración de procesamiento
- Eventos de ejemplo (20 transacciones de inventario)

> **Nota:** Asegurate de tener configurado correctamente el archivo `.env` con tus credenciales de Confluent Cloud antes de ejecutar el comando.

---

# Paso 1. Creá la herramienta MCP y el agente de IA en watsonx Orchestrate

En este paso, vas a crear la herramienta MCP y el agente de IA en **watsonx Orchestrate**. Las configuraciones de la herramienta MCP y del agente de IA fueron creadas y validadas con la ayuda de **IBM Bob**.

Para más detalles sobre cómo usar Bob para crear herramientas MCP y agentes, revisá este tutorial: [Usando IBM Bob para construir agentes de watsonx Orchestrate y herramientas MCP](https://developer.ibm.com/tutorials/build-agents-mcp-tools-watsonx-orchestrate-using-bob/).

## Importá la herramienta MCP en watsonx Orchestrate

Una **herramienta MCP** (Model Context Protocol) es la forma en que watsonx Orchestrate expone una función externa —en este caso, un script Python— para que un agente de IA pueda invocarla. Al importarla, le estás diciendo a Orchestrate: "este script existe, tiene estas capacidades, y los agentes pueden usarlo como herramienta".

El comando registra el script `get_sku_availability.py` como una herramienta llamada `sku-availability-checker`. Cuando un agente necesite consultar el stock, llamará a esta herramienta y devolverá los datos desde Kafka vía ksqlDB.


```bash
orchestrate toolkits add --kind mcp --name "sku-availability-checker" --description "Verificador de disponibilidad de inventario en tiempo real usando Confluent Kafka y ksqlDB" --language python --package-root "Repo-TechSummit-Lab2/confluent_agents" --command "python get_sku_availability.py" --tools "*"
```

## Importá el agente

El archivo `sku-availability-agents.yaml` define el comportamiento del agente: su nombre, descripción, qué herramientas puede usar y cómo debe razonar. Importarlo registra ese agente en watsonx Orchestrate para que pueda ser desplegado y probado.
Asegura de estar parado en la carpeta "Repo-TechSummit-Lab2/confluent_agents"

```bash
orchestrate agents import -f sku-availability-agent.yaml
```

> **Nota:** Si después de importar el agente no lo ves en la UI de watsonx Orchestrate, recargá la página (F5 o Ctrl+R).

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

## Accedé a watsonx Orchestrate

Si es la primera vez que accedés a watsonx Orchestrate, seguí estos pasos:

1. Accedé desde el dashboard de IBM Cloud: 👉 [https://cloud.ibm.com](https://cloud.ibm.com)

2. Hacé clic en el menú de hamburguesa

   ![Menú hamburguesa](assets/menu_hamburguesa.png)

3. Seleccioná **Lista de recursos**

   ![Lista de recursos](assets/lista_recursos.png)

4. Seleccioná la instancia de **watsonx Orchestrate** dentro del menú desplegable "IA / Aprendizaje automático".

   ![Selección de instancia](assets/seleccion_instancia.png)

5. Hacé clic en **Iniciar watsonx Orchestrate**

   ![Launch watsonx Orchestrate](assets/launch_wxo.png)

6. Andá a **Crear**

   ![UI de watsonx Orchestrate](assets/wxo_ui.png)

## Desplegá el agente

1. Vas a ver el agente `SKU_Availability_Agent` que acabás de importar — hacé clic en él
   ![Paso 1: Hacer click en agente](assets/desplegar_agente.png)
2. En la esquina superior derecha, hacé clic en **Desplegar**
   ![Paso 2: Descplegar agente](assets/desplegar_agente.png)
3. Confirmá el despliegue en la ventana de **Resumen previo al despliegue**
   ![Paso 3: Confirmar despliegue](assets/deploy_resumen.png)

## Probá el agente

**Pregunta de prueba:**

> ¿Cuáles son los SKUs disponibles en el Dot Shopping?

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

Asegura de estar parado en la carpeta "Repo-TechSummit-Lab2/confluent_agents"

```bash
orchestrate agents import -f Substitute_Finder_Agent.yaml
```

> **Nota:** Si después de importar el agente no lo ves en la UI de watsonx Orchestrate, recargá la página (F5 o Ctrl+R).

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

5. Vas a ver el agente `Substitute_Finder_Agent` que acabás de importar — hacé clic en él
5. En la esquina superior derecha, hacé clic en **Desplegar**
6. Confirmá el despliegue en la ventana de **Resumen previo al despliegue**

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

1. Dentro del agente, dirigite a la sección **Conocimiento**
2. Hacé clic en el botón **Añadir origen**
3. Seleccioná la opción para agregar una nueva base de conocimiento
4. En esta pantalla vas a ver las conexiones disponibles que se pueden integrar como fuente de conocimiento para el agente. Para este laboratorio, vamos a cargar un archivo local
5. Hacé clic en **Cargar archivos**
6. Seleccioná el archivo **product-catalog.docx**
7. Hacé clic en **Next**
8. Ingresá el nombre **enterprise_documents** y agregá una breve descripción como: "Esta knowledge base contiene un catálogo de laptops con sus principales especificaciones técnicas, casos de uso y características destacadas."
9. Hacé clic en **Save**
10. Esperá unos minutos hasta que finalice la indexación
11. Verificá que el documento figure como disponible y listo para ser usado en búsquedas semánticas

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

Asegura de estar parado en la carpeta "Repo-TechSummit-Lab2/confluent_agents"

```bash
orchestrate agents import -f Store_Associate_Agent.yaml
```

> **Nota:** Si después de importar el agente no lo ves en la UI de watsonx Orchestrate, recargá la página (F5 o Ctrl+R).

Una vez completada la importación, desplegá el agente desde la UI de watsonx Orchestrate para que quede activo:

3. Vas a ver el agente `Store_Associate_Agent` que acabás de importar — hacé clic en él
4. En la esquina superior derecha, hacé clic en **Deploy**
5. Confirmá el despliegue en la ventana de **Pre-deployment summary**

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

Asegura de estar parado en la carpeta "Repo-TechSummit-Lab2/confluent_agents"

```bash
orchestrate agents import -f Customer_Shopping_Assistant.yaml
```

> **Nota:** Si después de importar el agente no lo ves en la UI de watsonx Orchestrate, recargá la página (F5 o Ctrl+R).

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

---

# Troubleshooting

## Error: Token expirado o faltante

Si al ejecutar comandos del ADK de Orchestrate te aparece un mensaje como:

```
[ERROR] - The token found for environment 'labtech' is missing or expired.
Use `orchestrate env activate labtech` to fetch a new one
```

**Solución:**

Volvé a activar el ambiente ejecutando el siguiente comando:

```bash
orchestrate env activate [tu_nombre_del_env]
```

Cuando se te solicite, ingresá tu API Key de IBM Cloud.

> **Nota:** Los tokens de autenticación tienen un tiempo de expiración. Si dejás de trabajar por un período prolongado, es normal que necesites reactivar el ambiente.
