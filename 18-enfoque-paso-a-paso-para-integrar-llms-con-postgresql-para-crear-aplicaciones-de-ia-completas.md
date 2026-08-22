## Capítulo 18: Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas

Hasta ahora, hemos estado en una pequeña aventura. Hemos aprendido que los vectores son como contenedores de significado: huellas digitales matemáticas que nos ayudan a entender la "onda" o intención de los datos. Hemos visto cómo los vectores pueden agregar una capa de magia semántica a registros de bases de datos que de otro modo serían ordinarios, y nos hemos familiarizado con el almacenamiento de esos *embeddings* como datos vectoriales en PostgreSQL.

No todos los vectores en PostgreSQL son *embeddings* de IA utilizados para representar significado y respaldar operaciones de IA. En el [Capítulo 14](https://subscription.packtpub.com/book/data/9781806028474/14), *Análisis de Documentos de Texto en PostgreSQL*, vimos que PostgreSQL también admite la búsqueda de texto para búsquedas potentes de texto completo. En este capítulo, te mostraremos cómo conectar PostgreSQL a un LLM para generar vectores de IA tanto para datos almacenados en PostgreSQL como para consultas.

Este capítulo cubre los siguientes temas:

- Cómo usar cURL para enviar texto a un LLM y devolver el vector que representa el *embedding*, o significado, del texto
- Cómo usar Python para hacer lo mismo
- Cómo mover esa operación dentro de la base de datos, utilizando PL/Python
- Formas de generar *embeddings* para colecciones más grandes de datos
- Cómo generar el *embedding* para un término de consulta
- Cómo combinar vectores de IA y datos SQL para crear un motor de recomendaciones al estilo Spotify

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Requisitos técnicos

Ilustraremos estos conceptos utilizando la API de OpenAI, que ha sido adoptada por otros proveedores de LLM, incluidos DeepSeek, Meta AI y Mistral. Para seguir los ejemplos de este capítulo, necesitarás una clave de API (*API key*) de OpenAI.

El código de muestra para este capítulo se puede encontrar en `psql_scripts/sample_scripts/chapter_18.sql`. Para acceder al enlace del repositorio, sigue los pasos en la sección "Descarga de archivos de código de ejemplo" en el Prefacio. ¡Te animamos a seguir los ejemplos!

---

### Creación de un embedding con cURL

Comenzaremos con la herramienta más básica de la caja de herramientas: cURL, pronunciado *curl*. cURL es una herramienta de línea de comandos gratuita y de código abierto para cargar y descargar archivos individuales desde un servidor web a través de HTTP. Está incluida en las distribuciones de macOS, Linux y Windows.

Si aún no has trabajado con cURL, piensa en ella como la pequeña y amigable herramienta de línea de comandos que te permite hablar con servicios web. El siguiente ejemplo es como llamar cortésmente a la puerta digital de OpenAI y decir: "Oye, ¿podrías convertir esta oración en un *embedding* para mí?".

Así es como lo hacemos de la forma más sencilla:

1. Primero, obtén tu clave de API desde el sitio web de OpenAI. Esta clave es como tu saludo secreto, que le permite a OpenAI reconocerte.
2. Ahora, usemos el comando `curl`: Abre tu terminal, reemplaza `YOUR_API_KEY` con la clave de API que recibiste de OpenAI y ejecuta el siguiente comando:

```bash
curl https://api.openai.com/v1/embeddings \ -H "Authorization: Bearer YOUR_API_KEY" \ -H "Content-Type: application/json" \ -d '{ "model": "text-embedding-3-small", "input": "Here is the text I want to turn into an embedding." }'
```

Este comando le dice a OpenAI: "Aquí está mi clave, aquí está el texto y me gustaría un *embedding* generado por el modelo llamado `text-embedding-3-small`, por favor". ¡Y *voilà*! OpenAI responderá con un vector que puedes almacenar y usar de la siguiente manera:

```json
"object": "list", "data": [ { "object": "embedding", "index": 0, "embedding": [ -0.008902248, 0.021756759, … … … ] } ], "model": "text-embedding-3-small", "usage": { "prompt_tokens": 12, "total_tokens": 12 } }
```

Además de devolver el vector de *embeddings* en formato JSON, también nos indica cuántos tokens consumimos.

---

### Pasar de cURL a Python: Un script de embedding simple

Ahora moveremos esa misma llamada a la API de OpenAI a Python. Estos serán nuestros primeros pasos hacia un mundo más automatizado, así que lo mantendremos accesible y práctico.

1. Configurar el entorno de Python.
2. Crear un script simple en Python para solicitar el *embedding*.
3. Ejecutar el script e inspeccionar los resultados.

#### Configuración del entorno de Python

Instala Python (versión 3.12 o posterior) y asegúrate de que `pip` esté disponible con las bibliotecas básicas de solicitudes web instaladas. Recomendamos usar un entorno virtual (`venv`) para mantener las dependencias aisladas. Son como la navaja suiza para las solicitudes web en Python.

Simplemente ejecuta:

```bash
python3 -m venv .venv && source .venv/bin/activate.
```

#### Creación del script de Python

Ahora, escribiremos un script simple de Python que envíe un fragmento de texto a OpenAI y reciba un *embedding*. Lo mantendremos ligero y directo. Reemplaza `YOUR_API_KEY` con la misma clave de API de OpenAI que utilizaste para el ejemplo de `curl`:

```python
import json import ssl import time import random import urllib.request import urllib.error # Your OpenAI API key api_key = 'YOUR_API_KEY' # The text you want to embed input_text = "Here is the text I want to turn into an embedding." # The endpoint and headers url = "https://api.openai.com/v1/embeddings" headers = { "Authorization": f"Bearer {api_key}", "Content-Type": "application/json" } # The data payload data = { "model": "text-embedding-3-small", "input": input_text } # Make the request request = urllib.request.Request( url, data=json.dumps(data).encode('utf-8'), headers=headers, method='POST') try: with urllib.request.urlopen(request) as resp: resp_data = json.load(resp) embedding = resp_data['data'][0]['embedding'] print("Embedding:", embedding) except urllib.error.HTTPError as e: print("Error:", e.code, e.read().decode())
```

El programa de Python transforma texto sin formato en un *embedding*. Piénsalo como una conversación de una sola pregunta con un servicio de IA usando Python en lugar de cURL: le entregamos una oración y nos devuelve un vector (una lista ordenada de números que captura el significado de esa oración en una forma que las computadoras pueden comparar y buscar).

El script comienza importando algunos módulos estándar de Python. `json` está ahí porque la API de OpenAI utiliza la notación de objetos de JavaScript (JSON): enviamos JSON en el cuerpo de la solicitud y recibimos JSON en la respuesta.

El módulo `urllib.request` es el motor de trabajo que realiza la llamada HTTP, y `urllib.error` nos ayuda a manejar fallas de manera limpia si el servidor devuelve un error. También notarás que se importan `ssl`, `time` y `random`. En esta versión mínima, no son estrictamente obligatorios, pero se usan comúnmente cuando luego transformas este script en algo más resistente, por ejemplo, cuando agregas manejo seguro de TLS o implementas lógica de reintentos y retroceso (*retry-and-backoff*) para problemas temporales de red o límites de tasa. En otras palabras, son las "herramientas preparadas para el futuro" que están en el banquillo, listas para el próximo capítulo de la historia.

A continuación, el programa define dos entradas esenciales: la clave de API y el texto que queremos incrustar. La clave de API sirve como tus credenciales y te identifica ante OpenAI. El texto es la materia prima que queremos transformar en un *embedding*. En este ejemplo, el texto es una oración simple, pero podría ser fácilmente la descripción de un producto, una queja de un cliente, un ticket de soporte o un párrafo de un documento de políticas.

Con esas piezas en su lugar, el programa prepara la solicitud a la API. Apunta al punto de conexión de *embeddings* (`/v1/embeddings`) y construye encabezados HTTP que describen quiénes somos y qué estamos enviando. El encabezado `Authorization` lleva la clave de API como un token de portador (*bearer token*), que es la forma estándar en que las APIs modernas autentican las solicitudes. El encabezado `Content-Type` se establece en `application/json`, lo que indica que el cuerpo de la solicitud contiene JSON en lugar de texto plano u otro formato.

El corazón de la solicitud es la carga útil (*payload*) en sí. El programa construye un pequeño objeto JSON que especifica dos cosas: el modelo de *embedding* a utilizar (`text-embedding-3-small`) y el texto de entrada que queremos transformar en un *embedding* de IA. Puedes pensar en esta carga útil como un sobre cuidadosamente rotulado: le dice a OpenAI qué "motor" queremos usar y proporciona el "mensaje" que queremos procesado.

Luego, el script convierte esa carga útil en bytes (porque las solicitudes HTTP finalmente envían bytes a través del cable), crea una solicitud POST y la envía a OpenAI usando `urllib.request.urlopen`. Si la llamada tiene éxito, OpenAI responde con un documento JSON. Dentro de esa respuesta está el vector de *embedding*, accesible en `data[0].embedding`. El programa extrae esa lista de números y la imprime. Esa lista impresa es el *embedding*, o la "huella digital de significado" de tu texto.

Finalmente, el programa incluye un manejo básico de errores. Si OpenAI rechaza la solicitud (por ejemplo, porque falta la clave de API, no es válida o la solicitud está mal formada), `urllib` genera un `HTTPError`. El script captura ese error e imprime tanto el código de estado HTTP como el cuerpo de la respuesta. Esta es una disciplina pequeña pero importante: cuando comienzas a construir canalizaciones de *embeddings* a escala, un informe de errores limpio es lo que te salva de depurar a oscuras.

En resumen, este script hace un trabajo maravillosamente: demuestra el concepto de extremo a extremo. Entra una oración. Sale un vector.

#### Ejecución del script

Simplemente guarda esto como un archivo `.py` y ejecútalo. Tu programa de Python llamará a la API de OpenAI tal como lo hizo cURL, pero ahora tienes una pequeña herramienta reutilizable. Puedes pasar cualquier texto y con gusto obtendrá el *embedding* por ti:

```bash
python3 ./my_embedding.py
```

Esto devuelve los mismos resultados que la llamada `curl`:

```text
Embedding: [-0.008902248, 0.021756759, 0.01281589, … …, -0.009088919, -0.015487208, -0.0034501844, -0.010414923]
```

¡Y ahí lo tenemos! Hemos dado ese primer salto desde un simple comando `curl` a un modesto script de Python. Es como pasar de una sola pincelada al comienzo de una hermosa pintura.

Ahora que hemos creado nuestro programa de Python para generar un *embedding*, moveremos ese código dentro de la base de datos.

---

### Creación de embeddings en PostgreSQL con PL/PythonU

Una vez que has visto *embeddings* creados a partir de un simple script de Python, la siguiente pregunta natural es: ¿puede la base de datos crear *embeddings* directamente? A continuación, haremos exactamente eso, enseñándole a PostgreSQL cómo llamar a la API de *embeddings* de OpenAI desde dentro de la base de datos usando PL/PythonU.

Un breve resumen de lo que discutimos en el [Capítulo 6](https://subscription.packtpub.com/book/data/9781806028474/6), *Funciones y Procedimientos Almacenados*: La `U` en PL/PythonU significa *untrusted* (no confiable). Se considera no confiable porque otorga acceso completo al sistema operativo subyacente, la red y los archivos. Por eso generalmente requiere privilegios elevados y es mejor utilizarlo en una demostración controlada o en una capacidad cuidadosamente gobernada. Para el aprendizaje y la claridad, es perfecto. Para la producción, a menudo moverás esta llamada a una capa de servicios. Pero por ahora, queremos algo simple, directo y ejecutable.

La API de *embeddings* espera tres cosas:

- Un nombre de modelo (usaremos `text-embedding-3-small`)
- El texto de entrada
- Una clave de API para la autorización

La respuesta regresa como JSON, y dentro de ella está el único campo que más nos importa: `data[0].embedding`, el vector de IA representado como una lista de números.

Ese es todo el arco de la historia: texto de entrada, JSON de salida hacia el LLM, *embedding* extraído del JSON y devuelto a SQL.

Convirtamos nuestro script básico de Python a PL/PythonU. La siguiente es una función PL/PythonU que refleja nuestro ejemplo básico de Python pero se ejecuta dentro de PostgreSQL. Utiliza `urllib`, construye una solicitud POST al punto de conexión de *embeddings* de OpenAI, analiza la respuesta JSON y devuelve el *embedding* como una matriz `float4[]`.

La clave de API debe almacenarse en un parámetro de sesión (usando el comando `set_config`) o como una configuración del servidor (`ALTER SYSTEM`).

En la opción 1, tenemos un parámetro a nivel de sesión:

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false);
```

En la opción 2, tenemos un parámetro a nivel de servidor:

```sql
ALTER SYSTEM SET api.openai_api_key = 'your_openai_api_key_here'; -- Reload the configuration to apply the change SELECT pg_reload_conf();
```

`ALTER SYSTEM` escribe el valor en texto plano en `postgresql.auto.conf`, así que trátalo como confidencial y restringe el acceso al archivo (o prefiere variables de entorno/administradores de secretos cuando sea posible).

Esta función PL/Python es intencionalmente mínima y legible. No intenta ser ingeniosa. Intenta ser comprensible:

```sql
CREATE OR REPLACE FUNCTION api.openai_embed(input_text text) RETURNS float4[] LANGUAGE plpython3u AS $$ import json import ssl import urllib.request import urllib.error # Read API key from a session-scoped setting (GUC) rv = plpy.execute( "SELECT current_setting('api.openai_api_key', true) AS k") api_key = rv[0]["k"] if rv and rv[0]["k"] is not None else None if not api_key: raise Exception("OpenAI API key not set. Use: SELECT set_config('api.openai_api_key','sk-...','f');") # Input text we want to embed text = input_text or "" # OpenAI endpoint for embeddings url = "https://api.openai.com/v1/embeddings" # HTTP headers (auth + JSON) headers = { "Authorization": f"Bearer {api_key}", "Content-Type": "application/json" } # JSON payload (model + input text) payload = { "model": "text-embedding-3-small", "input": text } # Create and send POST request req = urllib.request.Request( url, data=json.dumps(payload).encode("utf-8"), headers=headers, method="POST" ) ctx = ssl.create_default_context() try: with urllib.request.urlopen(req, context=ctx, timeout=30) as resp: data = json.loads(resp.read().decode("utf-8")) emb = data["data"][0]["embedding"] return [float(x) for x in emb] except urllib.error.HTTPError as e: body = e.read().decode("utf-8", errors="ignore") raise Exception(f"OpenAI HTTP {e.code}. Body: {body[:400]} ...") $$;
```

La primera versión del script de Python fue diseñada para ejecutarse fuera de la base de datos, como un mensajero en el borde del sistema. Python realizó una solicitud HTTP a OpenAI e imprimió el *embedding* en la pantalla. Fue perfecto para aprender la llamada a la API, pero no estaba conectado a la base de datos.

La función PL/PythonU difiere porque incrusta esa misma capacidad dentro de PostgreSQL. La base de datos deja de ser solo la capa de almacenamiento y se convierte en un participante activo en el flujo de trabajo de *embeddings*. En lugar de "Python llama a OpenAI", la historia se convierte en "SQL llama a una función de base de datos, y esa función llama a OpenAI". El resultado se puede almacenar, indexar, unir, filtrar y gobernar inmediatamente utilizando el mismo motor de base de datos que ya gestiona tus productos y precios.

Otra diferencia clave es cómo se maneja la autenticación. En el script de Python, la clave de API generalmente está codificada en el archivo o se lee desde una variable de entorno. En la versión PL/PythonU, la clave se extrae de una configuración de base de datos (un GUC) usando `current_setting('api.openai_api_key', true)`. Por eso tu flujo de trabajo comienza con una línea SQL como `set_config(...)`. Es una forma limpia y adecuada para demostraciones de mantener los secretos fuera de las tablas y del cuerpo de la función, al tiempo que permite que la base de datos autentique las llamadas a la API.

La mecánica de la solicitud HTTP también está determinada por el entorno de la base de datos. En un script de Python normal, puedes usar cualquier biblioteca que desees (`requests`, `httpx`, etc.). En PL/PythonU, normalmente prefieres bibliotecas estándar como `urllib.request` porque siempre están disponibles y se comportan de manera consistente en todos los entornos, especialmente cuando se llaman dentro de un proceso de servidor de base de datos. La función también crea explícitamente un contexto SSL (`ssl.create_default_context()`), porque los servidores de bases de datos a menudo se ejecutan en entornos bloqueados donde ser explícito acerca de SSL y TLS es una mejor práctica.

El manejo de errores también cambia, porque los errores ahora ocurren "dentro de SQL". En el script de Python, imprimimos mensajes de error en la consola. En PL/PythonU, si algo sale mal, la función genera una excepción, y esa excepción surge como un error SQL, que se puede manejar a nivel de PostgreSQL. Eso es importante ya que permite que las fallas de *embedding* participen en transacciones compatibles con ACID. Si generamos *embeddings* como parte de una canalización de inserción/actualización, PostgreSQL puede tratar las fallas como fallas de transacción: revertirlas, registrarlas y exponerlas a la aplicación que realiza la llamada de manera predecible.

Finalmente, el mayor cambio conceptual es lo que sucede después de que recuperamos el vector de *embedding* del LLM. En el script de Python, el *embedding* era solo una salida. En PostgreSQL, los *embeddings* se tratan como datos y se almacenan en la tabla `embeddings.product_embedding` y se indexan para `pgvector` usando HNSW o IVFFlat, como se describe en el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*. Una vez almacenados e indexados, los *embeddings* se pueden consultar con `<=>` (distancia del coseno) o `<->` (distancia euclidiana). Los *embeddings* dejaron de ser una lista impresa y se convirtieron en parte de la memoria semántica buscable de la base de datos, listos para unirse con tablas de precios, filtrarse por `current=true`, restringirse por región y auditarse como cualquier otra consulta crítica para el negocio.

La diferencia no es la llamada a la API en sí: la llamada es esencialmente la misma. La diferencia es dónde reside la llamada y qué es lo que eso habilita: una sola base de datos, un solo límite de transacción y un solo lugar donde el "significado" y la "realidad empresarial" pueden encontrarse sin fricción.

Aquí está la forma más sencilla de usar `api.openai_embed()` con texto sin formato: sin tablas todavía, solo texto entrante → *embedding* saliente.

Primero, establece tu clave de API de OpenAI para la sesión actual (así es como la lee tu función):

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false);
```

Ahora, llama a la función con cualquier texto:

```sql
-- Tip: If the output is wide, enable expanded display in psql: -- \x -- (or disable the pager: \pset pager off) SELECT api.openai_embed('waterproof trail running shoes');
```

Si ves un error HTTP 401, verifica que tu clave de API sea válida y tenga los permisos/alcances requeridos para solicitudes de *embedding* (y que esté asociada con el proyecto/rol correcto de OpenAI). Si estás utilizando una clave de API restringida, asegúrate de que incluya el alcance necesario para el modelo/solicitud.

Eso devolverá una matriz `float4[]`: el *embedding* (una larga lista de números).

Si deseas trabajar inmediatamente con él como un tipo `pgvector` (para poder usar los operadores de distancia e índices de `pgvector`), conviértelo a `vector(1536)`:

```sql
SELECT api.openai_embed('waterproof trail running shoes')::vector(1536) AS embedding_vec;
```

Y si quieres una "comprobación rápida de cordura" sin imprimir todo el vector, puedes simplemente verificar la dimensión convirtiéndolo a vector y llamando a la función `vector_dims()`:

```sql
SELECT vector_dims(api.openai_embed('waterproof trail running shoes')::vector(1536)) AS dims;
```

Esto debería devolver `1536`, confirmando que el tamaño del *embedding* coincide con la salida del modelo.

En el siguiente paso, haremos que nuestra función de OpenAI sea más robusta asegurándonos de que pueda reintentar la generación del *embedding* y manejar los errores que la API pueda devolver.

---

### Hacer que api.openai_embed sea más robusta: Adición de reintentos y retroceso (*backoff*)

En los primeros ejemplos, nuestra llamada de *embedding* fue intencionalmente simple: enviar texto a OpenAI, recibir un vector y continuar. Eso es perfecto para aprender. Pero una vez que comenzamos a incrustar tablas de tamaño real (cientos o miles de categorías, marcas, productos y variantes), la simplicidad por sí sola se vuelve frágil y tenemos que anticipar ralentizaciones y errores de la vida real.

A esa escala, debemos manejar condiciones que son normales en sistemas complejos y distribuidos:

- Fallas temporales de red
- Problemas de servicio de corta duración
- Breves límites de tasa durante ráfagas de solicitudes
- Errores transitorios 5xx de la infraestructura ascendente

El objetivo del diseño actualizado no es agregar más complejidad; es tener menos ejecuciones de *embedding* fallidas, menos repeticiones y menos tablas de *embeddings* pobladas a medias. En una canalización de base de datos, la solidez no es opcional: es higiene operativa.

La siguiente es la versión modificada y más robusta de la función `api.openai_embed`:

```sql
CREATE OR REPLACE FUNCTION api.openai_embed(input_text text) RETURNS float4[] LANGUAGE plpython3u AS $$ import json, ssl, time, random, urllib.request, urllib.error def call_openai(payload, api_key, org): req = urllib.request.Request( "https://api.openai.com/v1/embeddings", data=json.dumps(payload).encode("utf-8"), headers={ "Content-Type": "application/json", "Authorization": f"Bearer {api_key}", "User-Agent": "aidb-postgres-plpython/1.0", **({"OpenAI-Organization": org} if org else {}) } ) ctx = ssl.create_default_context() with urllib.request.urlopen(req, context=ctx, timeout=30) as resp: raw = resp.read().decode("utf-8") return json.loads(raw) rv = plpy.execute(""" SELECT current_setting('api.openai_api_key', true) AS k, current_setting('api.openai_organization', true) AS o """) api_key = rv[0]["k"] if rv and rv[0]["k"] is not None else None org = rv[0]["o"] if rv and rv[0]["o"] is not None else None if not api_key: raise Exception("OpenAI API key not set. Use: SELECT set_config('api.openai_api_key','sk-...','f');") payload = {"model":"text-embedding-3-small","input": input_text or ""} attempts = 6 for i in range(attempts): try: data = call_openai(payload, api_key, org) # Validate response structure for clearer errors if "data" not in data or not data["data"] or "embedding" not in data["data"][0]: raise Exception(f"Unexpected OpenAI response shape: keys={list(data.keys())}") emb = data["data"][0]["embedding"] if not isinstance(emb, list): raise Exception("Embedding is not a list in OpenAI response") return [float(x) for x in emb] except urllib.error.HTTPError as e: body = e.read().decode("utf-8", errors="ignore") # Retry only on transient HTTP errors if e.code not in (429, 500, 502, 503, 504) or i == attempts - 1: raise Exception(f"OpenAI HTTP {e.code} (attempt {i+1}/{attempts}). Body: {body[:400]} ...") # If rate-limited, honor Retry-After when present retry_after = None try: retry_after = e.headers.get("Retry-After") except Exception: retry_after = None if retry_after: try: sleep_s = float(retry_after) except Exception: sleep_s = (2 ** i) * 0.5 + random.uniform(0, 0.3) else: sleep_s = (2 ** i) * 0.5 + random.uniform(0, 0.3) time.sleep(sleep_s) except urllib.error.URLError as e: # Common transient network error if i == attempts - 1: raise time.sleep((2 ** i) * 0.5 + random.uniform(0, 0.2)) except Exception as e: # Fail fast on unexpected errors on last attempt; otherwise short backoff if i == attempts - 1: raise time.sleep((2 ** i) * 0.5 + random.uniform(0, 0.2)) $$;
```

En el diseño actualizado, agregamos una función auxiliar dedicada `call_openai`, manejamos las claves secretas dentro de la función, agregamos reintentos acotados, condiciones de reintento selectivas, retroceso exponencial y errores claros. Revisemos los cambios y analicemos por qué son importantes.

#### Un asistente dedicado `call_openai(...)`

Definir una función auxiliar separada para la solicitud HTTP separa las responsabilidades:

```python
def call_openai(payload, api_key, org): ...
```

Esto hace que el código sea más fácil de leer y mantener, y proporciona un bucle de reintento limpio: el bucle de reintento no necesita conocer los detalles de la construcción de solicitudes; solo necesita saber si la llamada tuvo éxito o falló.

#### Secretos con ámbito de sesión (clave de API y organización opcional)

En lugar de codificar las credenciales de forma rígida, la función las lee desde la configuración de la sesión:

```sql
current_setting('api.openai_api_key', true) current_setting('api.openai_organization', true)
```

Veamos por qué esto importa:

- Sin secretos en las tablas
- Sin secretos dentro del cuerpo de la función
- Cada sesión puede usar su propia clave (útil en demostraciones, laboratorios o configuraciones multiusuario)
- Puedes revocar el acceso simplemente no estableciendo la clave

Si la clave no está configurada, la función falla rápidamente con un mensaje claro. Eso es intencional: evita fallas misteriosas más adelante.

#### Un bucle de reintentos acotado

Reintentar es el núcleo de la solidez. En lugar de fallar cuando hay un problema en cualquier lugar, reintenta la operación, pero no lo hagas indefinidamente:

```python
attempts = 6 for i in range(attempts): ...
```

La elección de diseño clave aquí son los reintentos acotados. No reintentamos para siempre. Reintentamos una pequeña cantidad de veces y luego fallamos ruidosamente. Eso evita que las sesiones de base de datos se queden colgadas indefinidamente y hace que los modos de falla sean predecibles.

#### Reintentos inteligentes solo para errores transitorios

La solicitud HTTP proporciona mensajes de error detallados, que utilizamos para determinar qué fallas deben reintentarse. Reintentamos solo para los errores que comúnmente son temporales:

- `429` (límite de tasa alcanzado)
- `500`, `502`, `503`, `504` (errores del lado del servidor/puerta de enlace)

Así es como se hace:

```python
if e.code not in (429, 500, 502, 503, 504) or i == attempts - 1: raise ...
```

Esta es una disciplina importante. Si el error es permanente (clave incorrecta, carga útil no válida o permisos insuficientes), reintentar desperdicia tiempo y aumenta la carga.

#### Retroceso exponencial con fluctuación (*jitter*) (el reintento educado)

Si OpenAI (o la red) está bajo estrés, insistir sin pausa solo empeorará el problema, a medida que se acumulen más solicitudes. El reintento ciego es lo peor que puedes hacer en esas circunstancias.

Es por eso que la función utiliza retroceso exponencial:

```python
time.sleep((2 ** i) * 0.5 + random.uniform(0, 0.3))
```

Esto significa lo siguiente:

- El primer reintento espera ~0.5s
- Luego ~1.0s
- Luego ~2.0s
- Luego ~4.0s
- Y así sucesivamente

La fluctuación aleatoria (*jitter*) evita un efecto de avalancha (*thundering herd*) donde múltiples trabajadores reintentan exactamente al mismo tiempo. Este es exactamente el tipo de comportamiento que queremos cuando incrustamos en masa: paciente, educado y resistente.

#### Mejores mensajes de error con fragmento del cuerpo de respuesta

Cuando ocurre un error HTTP, la función captura parte del cuerpo:

```python
body = e.read().decode("utf-8", errors="ignore") raise Exception(f"OpenAI HTTP {e.code}. Body: {body[:400]} ...")
```

Esto facilita significativamente la depuración. En lugar de un `HTTP 401` sin contexto, obtenemos la explicación (clave no válida, modelo no encontrado, etc.). Truncamos el cuerpo para evitar inundar los registros con cargas útiles enormes.

#### Por qué la robustez importa específicamente dentro de PostgreSQL

Cuando ejecutamos *embeddings* desde la capa de la aplicación, las fallas son molestas. Cuando ejecutamos *embeddings* dentro de la base de datos, las fallas pueden ser más costosas porque pueden interrumpir:

- Scripts de carga de datos
- Trabajos de *embeddings* por lotes
- Procedimientos almacenados que incrustan vectores faltantes
- Canalizaciones de demostración que deben "simplemente funcionar" en vivo

Un solo error transitorio no debería desencadenar el reinicio de una ejecución de *embeddings* de 10.000 filas. Una función robusta con reintentos inteligentes incorporados reduce drásticamente esas interrupciones.

Esa es la diferencia entre *"Nuestra canalización a veces falla aleatoriamente"* y *"Nuestra canalización termina de manera confiable, incluso cuando el entorno es ruidoso"*.

Esta función `api.openai_embed` modificada está construida alrededor de una filosofía: Trata las llamadas a la API como el clima, no como las matemáticas. A veces llueve. Tu sistema debería llevar un paraguas.

Con reintentos acotados, condiciones de reintento selectivas, retroceso exponencial y errores claros, hemos hecho que la generación de *embeddings* dentro de PostgreSQL sea mucho más confiable, especialmente cuando pasamos de "un texto" a "catálogos completos de productos".

---

### Creación de embeddings para tablas reales y almacenamiento en tablas de embeddings

Ahora que tenemos una función que se ejecuta dentro de la base de datos y puede crear un *embedding* a partir de un solo fragmento de texto, podemos hacer algo mucho más útil: generar *embeddings* a partir de filas reales en nuestro catálogo de productos y almacenarlos en las tablas de *embeddings* bajo el esquema `embeddings` (ver Figura 18.1).

*Figura 18.1: Catálogo de productos y tablas de embeddings correspondientes*

> [!NOTE]
> **Descarga las imágenes en color**  
> Tu compra incluye una copia en PDF en color y sin DRM de este libro, ideal para ver imágenes en color, capturas de pantalla y diagramas. Consulta la sección de beneficios gratuitos con tu libro al final del Prefacio para desbloquear tu copia en PDF.

El patrón es simple y repetible:

1. Seleccionar la fila a incrustar (categoría, marca, producto o variante).
2. Componer el texto que mejor capture su significado (generalmente una combinación de etiqueta y descripción, o atributos JSON).
3. Llamar a `api.openai_embed(...)` para recuperar un *embedding*.
4. Convertirlo a `vector(1536)`, para que `pgvector` pueda indexarlo y buscarlo de manera eficiente.
5. Insertarlo en la tabla de *embeddings* correspondiente, vinculada por la misma clave principal que la fila original.

Así es como creamos una capa semántica para datos estructurados: mantenemos intacto el modelo relacional y agregamos el significado como una representación paralela.

Antes de ejecutar los ejemplos, no olvides configurar la clave de API para tu sesión:

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false);
```

O configura la clave a nivel de servidor:

```sql
ALTER SYSTEM SET api.openai_api_key = 'your_openai_api_key_here'; -- Reload the configuration to apply the change SELECT pg_reload_conf();
```

#### Incrustar una sola fila de la tabla category

Las categorías son pequeñas pero importantes: representan los "pasillos" del catálogo de productos. El esquema sugiere que el *embedding* de la categoría debe generarse a partir de las columnas llamadas `label` y `description`.

La siguiente consulta selecciona una categoría (`id = 1`), se conecta a OpenAI y crea el *embedding*:

```sql
SELECT c.id, c.label, api.openai_embed(coalesce(c.label,'') || ' ' || coalesce(c.description,''))::vector(1536) AS embedding FROM product.category c WHERE c.id = 1;
```

Esta es la salida abreviada de la consulta:

```text
-[ RECORD 1 ]------------------------------------------------------------------------- id | 1 label | Pants embedding | [0.010736421,0.045328368,-0.0058921133,-0.03895541,-0.010234049,-0.021515902,-0.0010217901,0.004686419,-0.0053179734,0.0049124868,-0.034190048,-0.023740696,0.031060982,0.009250834,-0.0042127534,0.039070237,0.03820903,0.026841052,-0.049634416,0.08256136,-0.012143064,0.021472842,0.012552139,-0.020310208,0.003234921,0.03312789,0.028735716,0.028218988,0.041883525,-0.00029222836,0.031060982,-0.02658269,0.0001299665,0.0048335427,0.014841523,0.03014236,0.007148045,0.007363348,0.018946625,-0.01396596,-0.019032747,-0.053021844,-0.015903683,-0.03232409,0.041682575,0.0024024178,0.004313228,0.05382564,0.028218988,0.032065727,0.043634653,-0.026266912,0.0074135847,0.09846504,0.0…]
```

Ahora, insertaremos ese *embedding* en `embeddings.product_category_embedding`:

```sql
INSERT INTO embeddings.product_category_embedding (product_category_id, embedding) SELECT c.id, api.openai_embed(coalesce(c.label,'') || ' ' || coalesce(c.description,''))::vector(1536) FROM product.category c WHERE c.id = 1 ON CONFLICT (product_category_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

La cláusula `ON CONFLICT` hace que sea seguro volver a ejecutar el proceso cuando cambia la categoría. Si hay datos y se regenera el vector, simplemente se reemplaza.

#### Incrustar una sola fila de la tabla brand

Las marcas se comportan de la misma manera que las categorías: su significado proviene de `label` y `description`:

```sql
INSERT INTO embeddings.product_brand_embedding (product_brand_id, embedding) SELECT b.id, api.openai_embed(coalesce(b.label,'') || ' ' || coalesce(b.description,''))::vector(1536) FROM product.brand b WHERE b.id = 1 ON CONFLICT (product_brand_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

#### Incrustar una sola fila de la tabla product

Los productos conllevan un significado más rico porque tienen múltiples campos descriptivos. Para calcular el *embedding*, combinamos la etiqueta, la descripción corta y la descripción larga. En la práctica, a menudo es aún mejor incluir la etiqueta de la marca y la etiqueta de la categoría (porque influyen en el significado y la relevancia de la búsqueda), pero comenzaremos primero con la definición del libro de texto y luego mostraremos la versión mejorada.

Aquí está el *embedding* de producto de libro de texto (solo etiqueta y descripciones):

```sql
INSERT INTO embeddings.product_embedding (product_id, embedding) SELECT p.id, api.openai_embed( coalesce(p.label,'') || ' ' || coalesce(p.shortdescription,'') || ' ' || coalesce(p.longdescription,'') )::vector(1536) FROM product.product p WHERE p.id = 1 ON CONFLICT (product_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

Aquí está el *embedding* de producto mejorado (también incluye las etiquetas de categoría y marca):

```sql
INSERT INTO embeddings.product_embedding (product_id, embedding) SELECT p.id, api.openai_embed( coalesce(p.label,'') || ' ' || coalesce(pb.label,'') || ' ' || coalesce(pc.label,'') || ' ' || coalesce(p.shortdescription,'') || ' ' || coalesce(p.longdescription,'') )::vector(1536) FROM product.product p JOIN product.brand pb ON pb.id = p.brand_id JOIN product.category pc ON pc.id = p.category_id WHERE p.id = 1 ON CONFLICT (product_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

Esta versión tiende a producir una mejor recuperación porque el *embedding* ve la "identidad semántica completa" del producto, no solo la descripción sin procesar.

#### Incrustar una sola fila de variante de producto

Las variantes de productos son donde el comercio del mundo real se vuelve interesante, con tallas, colores, materiales y ajustes, a menudo almacenados en JSONB, al igual que en nuestro modelo de datos:

```sql
SELECT p.label, pv.attributes FROM product p JOIN product_variant pv ON pv.product_id = p.id WHERE p.id = 1;
```

Por ejemplo, la camisa de vestir de The Gap viene en seis variantes:

```text
label| attributes --------------+-------------------------------------------- ----------- Dress shirt ..| {"fit": "slim", "size": "S", "color": "white", "collar": "spread"} Dress shirt ..| {"fit": "slim", "size": "M", "color": "white", "collar": "spread"} Dress shirt ..| {"fit": "slim", "size": "L", "color": "white", "collar": "spread"} Dress shirt...| {"fit": "classic", "size": "S", "color": "blue", "collar": "spread"} Dress shirt...| {"fit": "classic", "size": "M", "color": "blue", "collar": "spread"} Dress shirt ..| {"fit": "classic", "size": "L", "color": "blue", "collar": "spread"}
```

Dado que la API de *embeddings* requiere texto, convertimos el JSON a texto. PostgreSQL nos ofrece una opción fácil y estable: `pv.attributes::text` (donde `::` es el operador de conversión explícita de tipos de PostgreSQL). El siguiente código inserta un *embedding* de variante:

```sql
INSERT INTO embeddings.product_variant_embedding (product_variant_id, embedding) SELECT pv.id, api.openai_embed(pv.attributes::text)::vector(1536) FROM product.product_variant pv WHERE pv.id = 1 ON CONFLICT (product_variant_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

Para un *embedding* de calidad aún mayor, se puede formatear el JSON en un formato más legible para humanos (por ejemplo, `Color: Navy, Size: L, Material: Cotton`), pero el enfoque mínimo es perfectamente válido para nuestro primer sistema en funcionamiento.

#### Incrustar múltiples filas: El mismo patrón, ahora en masa

Una vez que funcionan las inserciones de una sola fila, escalar se reduce a repetir el mismo patrón sin el filtro `WHERE id = ...`.

##### Categorías (en masa)

```sql
INSERT INTO embeddings.product_category_embedding (product_category_id, embedding) SELECT c.id, api.openai_embed(coalesce(c.label,'') || ' ' || coalesce(c.description,''))::vector(1536) FROM product.category c ON CONFLICT (product_category_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

##### Marcas (en masa)

```sql
INSERT INTO embeddings.product_brand_embedding (product_brand_id, embedding) SELECT b.id, api.openai_embed(coalesce(b.label,'') || ' ' || coalesce(b.description,''))::vector(1536) FROM product.brand b ON CONFLICT (product_brand_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

##### Productos (en masa)

```sql
INSERT INTO embeddings.product_embedding (product_id, embedding) SELECT p.id, api.openai_embed( coalesce(p.label,'') || ' ' || coalesce(pb.label,'') || ' ' || coalesce(pc.label,'') || ' ' || coalesce(p.shortdescription,'') || ' ' || coalesce(p.longdescription,'') )::vector(1536) FROM product.product p JOIN product.brand pb ON pb.id = p.brand_id JOIN product.category pc ON pc.id = p.category_id ON CONFLICT (product_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

##### Variantes (en masa)

```sql
INSERT INTO embeddings.product_variant_embedding (product_variant_id, embedding) SELECT pv.id, api.openai_embed(pv.attributes::text)::vector(1536) FROM product.product_variant pv ON CONFLICT (product_variant_id) DO UPDATE SET embedding = EXCLUDED.embedding;
```

Notarás que no creamos *embeddings* para `product.product_variant_price` ni para `product.country_of_origin`. Eso es intencional. Estas tablas son en su mayoría metadatos estructurados y factuales. Los precios cambian con el tiempo y se filtran mejor con SQL. El país de origen es un atributo estricto, no un significado difuso.

En otras palabras, los *embeddings* son mejores para campos ricos en significado (descripciones, atributos y nombres). SQL es mejor para restricciones de verdad (precio, validez, actualidad, región y cumplimiento).

---

### Incrustar en lotes con una función PL/pgSQL

La incrustación de una sola fila nos muestra la mecánica. La incrustación masiva es la forma en que construimos un sistema que realmente se ejecuta. En el momento en que tenemos cientos o miles de filas, necesitamos una forma sólida y repetible de hacer lo siguiente:

- Encontrar filas que aún no tienen *embeddings*
- Generar *embeddings* para ellas
- Almacenarlas de forma segura
- Hacerlo en lotes pequeños y controlados, para no sobrecargar la API ni la base de datos

Para ilustrar esto, envolveremos `api.openai_embed` en una función que calcula *embeddings* por lotes para productos. Puedes implementar conceptos similares para otras tablas.

En un nivel alto, esta función de *embedding* por lotes se comporta como una línea de producción en una fábrica:

1. Elige los siguientes N productos sin un *embedding*.
2. Para cada producto, construye la "cadena de significado" (etiqueta + marca + categoría + descripciones).
3. Llama a `api.openai_embed(...)` para generar el *embedding*.
4. Inserta el *embedding* en `embeddings.product_embedding`.
5. Continúa hasta que el lote esté listo.
6. Devuelve cuántos productos fueron incrustados.

Esto nos brinda una herramienta muy práctica que podemos ejecutar repetidamente hasta que la tabla de *embeddings* esté completamente poblada.

Aquí está la función `api.embed_product(batch_size)`:

```sql
CREATE OR REPLACE FUNCTION api.embed_product(batch_size int DEFAULT 200) RETURNS integer LANGUAGE plpgsql AS $$ DECLARE r RECORD; done_count int := 0; BEGIN FOR r IN SELECT p.id, p.label, p.shortdescription, p.longdescription, pc.label AS category_label, pb.label AS brand_label FROM product.product p JOIN product.category pc ON pc.id = p.category_id JOIN product.brand pb ON pb.id = p.brand_id LEFT JOIN embeddings.product_embedding pe ON pe.product_id = p.id WHERE pe.product_id IS NULL ORDER BY p.id LIMIT batch_size LOOP BEGIN MERGE INTO embeddings.product_embedding AS target USING (SELECT r.id AS product_id) AS source ON (target.product_id = source.product_id) WHEN NOT MATCHED THEN INSERT (product_id, embedding) VALUES (r.id, api.openai_embed( coalesce(r.label, '') || ' ' || coalesce(r.brand_label, '') || ' ' || coalesce(r.category_label, '') || ' ' || coalesce(r.shortdescription, '') || ' ' || coalesce(r.longdescription, ''))::vector(1536)); done_count := done_count + 1; EXCEPTION WHEN OTHERS THEN RAISE NOTICE 'Failed to embed product id %: %', r.id, SQLERRM; END; END LOOP; RETURN done_count; END; $$;
```

Veamos cómo funciona la función `api.embed_product(batch_size)`:

#### Solo selecciona filas que aún necesitan embeddings

Esta unión izquierda (*left join*) es crucial:

```sql
LEFT JOIN embeddings.product_embedding pe ON pe.product_id = p.id WHERE pe.product_id IS NULL
```

Significa: "Tráeme solo los productos que aún no tienen una fila en la tabla de embeddings". Eso hace que la función sea idempotente en la práctica. Puedes ejecutarla repetidamente y continuará seleccionando solo el trabajo pendiente.

#### Procesa en lotes pequeños para mantenerse estable

La variable `batch_size` proporciona una válvula de seguridad. En lugar de incrustar todo a la vez (lo que podría desencadenar límites de tasa o transacciones largas), el código calcula *embeddings* para un fragmento manejable: 200 productos a la vez por defecto.

Así es como mantienes las canalizaciones estables:

- Ráfagas más pequeñas
- Menos fallas
- Recuperación más fácil
- Carga predecible

Debido a que esta lógica se ejecuta dentro de una función, el trabajo aún se ejecuta dentro de una sola transacción. Para trabajos de *embeddings* de larga duración o de producción (donde deseas que persista el progreso parcial y evitar volver a pagar por los *embeddings*), prefiere un procedimiento o un ejecutor de trabajos que confirme el progreso por lote.

#### Incrusta una cadena de significado, no solo una columna

Dentro del bucle, construye una sola entrada de texto concatenando lo siguiente:

- Etiqueta del producto
- Etiqueta de la marca
- Etiqueta de la categoría
- Descripción corta
- Descripción larga

Esta es una buena práctica porque el modelo de *embeddings* ve una identidad semántica más rica. Es más difícil malinterpretar un producto cuando proporcionas más contexto.

#### Utiliza el comando MERGE para evitar condiciones de carrera

Este es un elemento de diseño clave en la función y merece especial atención:

```sql
MERGE INTO embeddings.product_embedding ... WHEN NOT MATCHED THEN INSERT ...
```

Esto es importante cuando dos trabajos de *embedding* se ejecutan al mismo tiempo (o activas accidentalmente la función dos veces). Sin `MERGE`, ambas sesiones podrían intentar insertar registros con el mismo valor de `product_id`, y una fallaría debido a una violación de restricción única.

El uso de `MERGE` hace que la operación sea segura:

- Si la fila ya está allí, no hace nada
- Si falta, la operación la inserta

Esto hace que la función sea tolerante a la concurrencia, lo cual es un concepto clave en aplicaciones de producción distribuidas de alto volumen. Para patrones simples de inserción si falta bajo alta concurrencia, `INSERT ... ON CONFLICT DO NOTHING` a menudo es una alternativa igualmente válida y a veces preferible.

#### Continúa incluso si falla una fila

Este es el bloque de manejo de errores:

```sql
EXCEPTION WHEN OTHERS THEN RAISE NOTICE ...
```

Significa: "Si la generación de embedding falla para un producto, regístralo y continúa".

Eso es exactamente lo que queremos en trabajos masivos. Un registro defectuoso no debería bloquear la incrustación de miles de registros correctos.

#### Devuelve un recuento para que puedas monitorear el progreso

La función devuelve el recuento de *embeddings* que se generaron:

```sql
RETURN done_count;
```

Esto hace que la función sea fácil de automatizar. Podemos ejecutarla en un bucle hasta que devuelva `0`.

#### Cómo ejecutar la incrustación masiva

Sigue estos pasos:

1. Establece la clave de API para la sesión:

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false);
```

2. Ejecuta un lote:

```sql
SELECT api.embed_product(200);
```

3. Llama a la consulta repetidamente hasta que devuelva `0`, lo que indica que no quedan productos sin un *embedding* asociado:

```sql
SELECT api.embed_product(200); SELECT api.embed_product(200); SELECT api.embed_product(200);
```

#### Por qué este enfoque es simple y confiable (y por qué eso es bueno)

Quizás notes que el comentario dice "API por lotes", pero el código llama a `api.openai_embed` por fila. No es un error; es una compensación deliberada por estabilidad.

La incrustación masiva se puede hacer de dos maneras:

- **Procesamiento por lotes real de la API** (enviar una matriz de entradas a OpenAI en una sola solicitud):
  - Menos llamadas HTTP
  - Manejo de errores más complejo (una entrada incorrecta puede contaminar todo el lote)
  - Mapeo más complicado entre entradas y filas
- **Incrustación por fila dentro de un bucle por lotes** (esta función):
  - Extremadamente simple
  - Las fallas se aíslan a una sola fila
  - Fácil de reanudar
  - Más fácil de explicar en un libro de texto
  - Más llamadas HTTP en general

#### Extender el mismo patrón a otras tablas

Una vez que comprendes `api.embed_product`, es fácil crear funciones masivas similares:

- `api.embed_categories(batch_size)`
- `api.embed_brands(batch_size)`
- `api.embed_variants(batch_size)`

Misma receta, diferentes tablas.

Y con eso, hemos superado un hito importante: la base de datos ya no solo almacena datos de productos, sino que está construyendo una capa semántica sobre ellos, un lote cuidadoso a la vez.

---

### Cerrar el círculo: Uso de los embeddings generados para impulsar recomendaciones

En el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, introdujimos una idea poderosa: una vez que tus datos tienen *embeddings*, puedes dejar de tratar las recomendaciones como un "sistema de IA" separado y comenzar a tratarlas como parte de una solución de base de datos integrada. `pgvector` le da a PostgreSQL la capacidad de comparar significados usando operadores de distancia vectorial, y SQL te da la capacidad de convertir la "similitud" en algo operativo: en stock, precio actual, región correcta, seguro y permitido.

En este capítulo, nos hemos centrado en integrar LLMs (OpenAI) con PostgreSQL y en cómo crear *embeddings*, desde bloques de texto individuales hasta la población masiva. Ahora, cerramos el círculo: una vez que existen los *embeddings*, ¿qué hacemos con ellos?

La respuesta es simple: los consultamos.

El [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, ya demostró los patrones básicos de recomendación, y los reutilizaremos aquí como la "recompensa" para el *embedding* que acabamos de construir.

#### Recomendaciones de consulta a producto utilizando api.openai_embed

En lugar de comenzar desde `product_id` (de producto a producto), comenzamos desde una solicitud de usuario, como *"Muéstrame ropa de hombre premium para una ocasión formal"*.

El primer paso es convertir esa pregunta en un *embedding* (un vector de significado). Hacemos eso una vez, dentro de un CTE (ver [Capítulo 13](https://subscription.packtpub.com/book/data/9781806028474/13)), y lo reutilizamos.

Aquí está la versión básica: búsqueda semántica sobre productos:

```sql
WITH q AS ( SELECT api.openai_embed('Show me premium men''s clothes for a formal occasion')::vector(1536) AS qvec ) SELECT p.id, p.label, 1 - (pe.embedding <=> q.qvec) AS similarity FROM q JOIN embeddings.product_embedding pe ON TRUE JOIN product.product p ON p.id = pe.product_id ORDER BY pe.embedding <=> q.qvec LIMIT 10;
```

Esta es la salida:

```text
id | label | similarity ----+-----------------------------------------+--------------------- 2 | Dress shirt by Boss | 0.5490152672226669 1 | Dress shirt by The Gap | 0.5363510207711434 14 | Suit coat - business perfect, by Brioni | 0.5311552845054526 3 | Dress shirt by Eaton | 0.5152048315046799 13 | Sports coat - business casual, by Boss | 0.4872733071716657 12 | Chinos - casual, by Boss | 0.4739027932940256 18 | Calvin Klein Cotton Shirt | 0.4679474551709031 17 | Polo shirt, Boss | 0.46449703189748415 11 | Chinos - casual, by The Gap | 0.456206622349822 21 | Ralph Lauren Chinos | 0.45597515034971825 (10 rows)
```

Esto hace lo siguiente:

- `api.openai_embed(...)` convierte la oración del usuario en un vector de 1.536 dimensiones.
- `<=>` calcula la distancia del coseno entre el vector de consulta y cada *embedding* de producto.
- `ORDER BY ... LIMIT` devuelve las coincidencias más cercanas.

Este es el patrón más simple para la búsqueda semántica en SQL.

Ahora, veamos una versión avanzada: agregando restricciones SQL del mundo real.

En el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, mostramos que la similitud por sí sola no es suficiente en aplicaciones del mundo real. En producción, normalmente también queremos lo siguiente:

- Solo artículos con un precio actual
- Opcionalmente, solo artículos dentro de un presupuesto
- Opcionalmente, solo artículos que estén en stock
- Opcionalmente, filtrado por categoría/marca/región

Aquí hay un ejemplo simple que combina coincidencia semántica con restricciones SQL estrictas:

```sql
WITH q AS ( SELECT api.openai_embed('Show me premium men''s clothes for a formal occasion')::vector(1536) AS qvec ) SELECT DISTINCT p.id, p.label, pvp.price, 1 - (pe.embedding <=> q.qvec) AS similarity FROM q JOIN embeddings.product_embedding pe ON TRUE JOIN product.product p ON p.id = pe.product_id JOIN product.product_variant pv ON pv.product_id = p.id JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true WHERE pvp.price <= 500 -- example hard constraint ORDER BY pe.embedding <=> q.qvec LIMIT 10;
```

La salida es una lista que coincide con la semántica de la solicitud y está restringida por el precio:

```text
id | label | price | similarity ----+--------------------------------------+-------+--------------------- 1 | Dress shirt by The Gap | 31.72 | 0.5363591589713608 2 | Dress shirt by Boss | 31.93 | 0.5489709540894915 3 | Dress shirt by Eaton | 31.86 | 0.5152130280498575 4 | Mens Classic Oxford Shirt by The Gap | 31.79 | 0.409137946639596 5 | Mens Classic Oxford Shirt from Boss | 31.69 | 0.4330028633473605 6 | T-Shirt by The Gap | 31.69 | 0.37526652216911316 7 | T-Shirt by Diesel | 31.74 | 0.3588232398033142 8 | 501 Original Fit Jeans | 98.58 | 0.2530273199081421 9 | Casual Leather Jacket by Boss | 82.16 | 0.45369914174079895 10 | Casual Leather Jacket by Aeropostale | 81.96 | 0.43804830681989826 (10 rows)
```

Ahora obtenemos "cosas que coinciden con el significado" y "cosas que están dentro de nuestro presupuesto". Ese es el verdadero beneficio de procesar y administrar vectores y SQL juntos en una base de datos multimodal.

*Figura 18.2: Enfoque paso a paso para procesar una consulta híbrida*

La Figura 18.2 nos muestra un análisis paso a paso de la ejecución de esta consulta híbrida, incluyendo la coincidencia de vectores basada en IA y el filtrado basado en SQL:

1. En el paso 1, llamamos a `api.embed_product` para crear los *embeddings* para los registros de productos.
2. En los pasos 2 y 3, `api.embed_product` utiliza la función auxiliar `call_openai` para enviar la información combinada de etiqueta y descripción al LLM y recuperar los *embeddings*.
3. En el paso 4, la función `api.embed_product` fusiona los *embeddings* en la tabla `embeddings.product_embedding`.
4. En los pasos 5 y 6, la primera cláusula del CTE utiliza `api.openai_embed` para enviar el texto de la consulta *"Show me premium men's clothes for a formal occasion"* al LLM y almacena el *embedding* en `q.qvec`.
5. En el paso 7, PostgreSQL crea un plan de consulta que considera las estadísticas y los índices para el operador de distancia del coseno de IA (`<=>`) y los operadores de igualdad en las tablas `product`, `product_variant` y `product_variant_price`.
6. Finalmente, en el paso 8, PostgreSQL devuelve el resultado de la consulta híbrida.

#### Un hábito de rendimiento pequeño pero importante

Observa que siempre calculamos el *embedding* una vez en un CTE:

```sql
WITH q AS (SELECT api.openai_embed(...)::vector(1536) AS qvec)
```

Eso evita llamadas repetidas a la API y garantiza que la consulta utilice consistentemente el mismo vector.

---

### Resumen

Este capítulo mostró cómo convertir los *embeddings* de un concepto abstracto de IA en una capacidad operativa y orientada a la producción dentro de PostgreSQL.

Comenzamos reforzando una distinción clave: los *embeddings* son vectores especializados, diseñados específicamente para capturar el significado semántico. No todos los vectores son un *embedding*, y no todos los *embeddings* son útiles a menos que puedan generarse de manera confiable y utilizarse repetidamente. Para hacer concreta la idea, comenzamos con el flujo de trabajo más simple posible: crear un *embedding* con una llamada `curl` y luego reproducir la misma solicitud en un script mínimo de Python. Ese paso inicial importa porque desmitifica el proceso: crear un *embedding* no es magia. Es una llamada a la API estándar a un servicio web que devuelve una lista estructurada de números que representan significado.

Luego, trasladamos el flujo de trabajo de *embeddings* a la propia base de datos. Al implementar `api.openai_embed` en PL/PythonU, PostgreSQL dejó de ser un motor de almacenamiento pasivo y se convirtió en un participante activo en el ciclo de vida de la IA. Este cambio es más importante de lo que parece: acerca la creación de *embeddings* a los datos, mantiene la gobernanza y la auditabilidad en el mismo lugar, y permite orquestar la creación de *embeddings* junto con operaciones SQL en lugar de incorporarla como una ocurrencia tardía.

La realidad de la producción entró luego en escena, ruidosamente: límites de tasa de API, fluctuaciones de red e interrupciones de servicio transitorias que ocurren en los peores momentos posibles. Mejoramos `api.openai_embed` con un patrón de confiabilidad disciplinado: reintentos acotados, retroceso exponencial y fluctuación (*jitter*). Esta es la diferencia entre una función de tutorial y una operativa. No solo hace que la función sea más robusta; hace que el sistema general esté más calmado bajo presión y sea mucho más probable que complete con éxito ejecuciones grandes.

Con una primitiva de *embedding* individual confiable en la mano, escalamos a datos reales. La función `api.embed_product(batch_size)` introdujo un patrón práctico para el *embedding* masivo: procesar el trabajo en lotes pequeños y controlados, construir una "cadena de significado" más rica a partir de múltiples atributos (etiqueta, marca, categoría y descripciones) y almacenar de forma segura los *embeddings* utilizando lógica idempotente. El uso de `MERGE` hace que el flujo de trabajo sea tolerante a la concurrencia, evitando condiciones de carrera cuando se ejecutan múltiples trabajos de *embedding* al mismo tiempo. Al incrustar solo las filas que carecen de vectores, el proceso se vuelve repetible: puedes ejecutarlo una y otra vez hasta que el conjunto de datos esté completo.

Utilizando los patrones introducidos en el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, demostramos cómo los *embeddings* generados desbloquean inmediatamente funciones reales de la aplicación: búsqueda semántica de consulta a producto, similitud de producto a producto y coincidencia de gustos al estilo Spotify al incrustar la pregunta del usuario (o declaración de preferencia) con `api.openai_embed` y clasificar los resultados con los operadores de distancia de `pgvector`. El sistema ahora se comporta como un motor semántico sin dejar de adherirse a la disciplina de SQL.

En el [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*, tomaremos esta base y la operacionalizaremos por completo. Diseñaremos canalizaciones de *embeddings* para producción (sincrónicas, asincrónicas y en tiempo real) y mostraremos cómo mantener actualizados los *embeddings* cuando cambian los datos. También exploraremos cuándo los *embeddings* de consulta deben generarse bajo demanda frente a cuándo deben almacenarse en caché, y construiremos dos asistentes prácticos: un chatbot SQL básico y un chatbot más avanzado que puede generar SQL dinámico de forma segura y predecible.

Ahora has aprendido a crear *embeddings*, almacenarlos y usarlos dentro de PostgreSQL. A continuación, convertiremos esa capacidad en un diseño de sistema: uno que se mantenga confiable a escala, permanezca correcto a medida que evolucionan los datos y entregue capacidades de IA sin fragmentar tu arquitectura en múltiples bases de datos y código pegamento personalizado.
