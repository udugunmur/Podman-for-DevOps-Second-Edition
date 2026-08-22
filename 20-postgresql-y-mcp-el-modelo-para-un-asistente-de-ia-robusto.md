## Capítulo 20: PostgreSQL y MCP: El Modelo para un Asistente de IA Robusto

Si el [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*, fue la sala de máquinas (canalizaciones, colas, reintentos y frescura), entonces este capítulo es la cabina de mando. Hasta ahora, le hemos enseñado a PostgreSQL a almacenar significado (`pgvector`), mantener significado (canalizaciones de *embeddings*) y recuperar significado (búsqueda semántica y recomendaciones). Esa es una base poderosa, pero aún deja una brecha entre la capacidad y la experiencia. Los usuarios no quieren "ejecutar una consulta vectorial". Quieren preguntar cosas como las siguientes:

- *Encuentra algo parecido a una chaqueta de hombre a medida de menos de 500 $.*
- *Muéstrame polos en diferentes colores, pero solo los que tengan precio actual.*
- *¿Cuáles son las alternativas más cercanas a este producto?*

Un buen asistente hace que esas preguntas se sientan naturales y que las respuestas se sientan confiables.

Este capítulo trata sobre la construcción de ese asistente sin sacrificar la disciplina de un sistema de base de datos. Diseñaremos dos capas de chatbot:

- **Un asistente de consultas curadas o chat SQL básico**: Piensa en él como un asistente "basado en menú": selecciona a partir de plantillas de consulta preaprobadas, por lo que el comportamiento se mantiene seguro por construcción, predecible en rendimiento y fácil de auditar.
- **Un asistente de consultas gobernadas o SQL dinámico**: Piensa en él como un asistente "supervisado": puede generar SQL para preguntas más complejas, pero solo dentro de límites estrictos (`SELECT`-only, `LIMIT` forzados, listas blancas de esquemas y ejecución registrada).

Hasta aquí, esto todavía suena a "chat".

Pero en el momento en que dejamos que un LLM elija acciones como ejecutar una consulta, llamar a una función o recuperar filas, hemos cruzado una línea: ya no estamos construyendo una conversación. Estamos otorgando capacidad. Y la capacidad cambia las reglas del juego.

> [!CAUTION]
> **¡Un prompt de IA puede sugerir un comportamiento, pero no puede imponerlo!**  
> Puedes decirle al LLM: *"Solo ejecuta SELECT"*, pero la base de datos no leerá tu prompt; ejecutará lo que el LLM le diga.  
> Puedes indicar en el prompt: *"Siempre usa LIMIT 10"*, pero la base de datos no aplicará tus buenas intenciones si el LLM le indica lo contrario. Puedes escribir *"Nunca toques tablas sensibles"*, pero "nunca" no es un límite de seguridad.

Esta es la conclusión en producción: Una vez que el modelo puede operar, la seguridad debe pasar del lenguaje al diseño del sistema. Ahí es donde entra el Protocolo de Contexto de Modelo (*Model Context Protocol* o MCP), no como una palabra de moda, sino como la capa que faltaba entre la "intención del LLM" y la "ejecución del sistema".

Este capítulo cubre los siguientes temas:

- Construcción de un chatbot de IA en PostgreSQL
- El chatbot de SQL dinámico
- Introducción a MCP: El contrato entre modelos y herramientas

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### ¿Qué es el MCP? Una explicación sencilla

El MCP es el contrato que convierte *"el modelo quiere hacer algo"* en *"el sistema permite una llamada a una herramienta específica, bajo una política, y devuelve evidencia"*. Sin MCP, el modelo tiende a actuar como un pasante ingenioso: escribe sugerencias en texto libre y espera que alguien más las ejecute de manera segura.

Con MCP, el modelo actúa como un cliente de una API: debe elegir entre las herramientas que has definido, enviar entradas tipadas y aceptar los resultados devueltos por el sistema.

Por lo tanto, en lugar de que el modelo improvise: *"Aquí tienes algo de SQL que deberías ejecutar..."*, el MCP hace que la interacción sea explícita y exigible:

- **La elección de la herramienta es explícita** (el modelo debe elegir una capacidad con nombre, no inventar una)
- **Las entradas son tipadas** (`p_question text`, `p_k`, `p_row_limit`, filtros opcionales)
- **La ejecución está sujeta a políticas** (rol de solo lectura, lista de permitidos de esquema, tiempos de espera, límites de filas)
- **Las salidas son estructuradas** (filas como evidencia, más metadatos opcionales)
- **Cada acción es auditable** (qué herramienta se ejecutó, qué SQL se ejecutó, qué se devolvió)

Ahora nuestro asistente deja de ser "una función de chat" y se convierte en algo más valioso: una interfaz gobernada para la base de datos.

Eso significa que podemos exponer una superficie de herramientas pequeña y segura construida a partir de las funciones y vistas de PostgreSQL que ya creamos, herramientas tales como:

- `api.sf_similar_items(p_query_text, p_k)` para recuperación semántica
- Vistas curadas sobre `product.*` y `embeddings.*`
- Rutas de ejecución controladas que devuelven filas (y nada más)

Nos mantendremos basados en el mismo esquema de comercio electrónico utilizado a lo largo de este libro: `product.category`, `product.brand`, `product.product`, `product.product_variant`, `product.product_variant_price` y las tablas de *embeddings* en `embeddings.*`.

El objetivo no es solo "un chatbot". El objetivo es un asistente de nivel de producción que combine significado semántico con verdad relacional: reglas de precios, atributos de variantes y restricciones comerciales, sin dejar de ser seguro, observable y predecible.

Y eso nos lleva de forma natural a la siguiente sección: implementar el enrutamiento de herramientas MCP para nuestro asistente de Postgres. Porque una vez que tienes un contrato, la siguiente pregunta es simple y práctica: ¿Qué herramienta debe manejar qué pregunta y cómo enrutamos de forma segura en cada ocasión?

---

### Construcción de un chatbot de IA en PostgreSQL

En los capítulos anteriores, le enseñamos a PostgreSQL a almacenar, indexar y recuperar significado. Ahora vamos a enseñarle algo más visible: cómo comunicarse y hablar con el usuario en lenguaje natural.

No la clase de conversación de una "demostración superficial", sino la clase fundamentada, donde las respuestas provienen de filas y no de la imaginación.

Construiremos nuestro chatbot utilizando tres funciones. Piensa en ellas como tres roles en una pequeña obra de teatro:

- **Recuperador (*Retriever*)**: Encuentra los productos más relevantes utilizando *embeddings* (`pgvector` + uniones SQL)
- **Narrador (*Narrator*)**: Convierte los resultados de la consulta en una respuesta humana sin inventar hechos
- **Orquestador (*Orchestrator*)**: Coordina a ambos, devolviendo la respuesta y la evidencia

Utilizaremos estas funciones exactamente como se detallan a continuación.

#### El recuperador: api.sf_similar_items

El recuperador es el motor semántico del asistente. Toma una pregunta del usuario, la convierte en un vector de *embedding*, la compara con los *embeddings* de productos y devuelve las coincidencias más cercanas, ya unidas con la categoría y el precio actual, para que los resultados sean utilizables de inmediato. La siguiente función implementa el componente recuperador:

```sql
CREATE OR REPLACE FUNCTION api.sf_similar_items( p_query_text text, p_k integer DEFAULT 10 ) RETURNS TABLE ( product_id integer, name text, category text, shortdescription text, longdescription text, price numeric, distance double precision ) LANGUAGE plpgsql VOLATILE SECURITY DEFINER SET search_path = pg_catalog, public, api, product, embeddings AS $$ DECLARE v_qvec vector(1536); BEGIN v_qvec := api.openai_embed(p_query_text)::vector(1536); RETURN QUERY WITH res AS MATERIALIZED ( SELECT p.id AS product_id, p.label::text AS name, c.label::text AS category, p.shortdescription::text AS shortdescription, p.longdescription::text AS longdescription, pvp.price AS price, (v_qvec <=> pe.embedding) AS distance FROM product.product p JOIN embeddings.product_embedding pe ON pe.product_id = p.id JOIN product.category c ON c.id = p.category_id JOIN product.product_variant pv ON pv.product_id = p.id JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true ) SELECT res.product_id, res.name, res.category, res.shortdescription, res.longdescription, res.price, res.distance FROM res ORDER BY res.distance LIMIT p_k; END; $$;
```

Cuando un usuario dice *"busco ropa a medida"*, no queremos una coincidencia de palabras clave como *"la palabra a medida debe aparecer en la descripción"*.

Queremos una coincidencia basada en el significado: *"a medida"* debería encontrar artículos como blazers, camisas formales y sacos deportivos, incluso si la palabra *"a medida"* no está presente.

Eso es exactamente lo que hace `api.sf_similar_items`:

- **Paso A**: Convertir la pregunta en un vector de *embedding* usando `api.openai_embed(query_text)`
- **Paso B**: Comparar ese vector con los vectores de productos almacenados en `embeddings.product_embedding`
- **Paso C**: Unir nuevamente con la verdad relacional:
  - Nombre del producto + descripciones
  - Categoría
  - Solo precios actuales mediante `product_variant_price.current = true`
- **Paso D**: Devolver los `p_k` resultados principales ordenados por distancia (menor distancia = significado más cercano)

¡Las uniones importan! Observa que esta consulta no es "solo vectorial". Es una combinación de vectores y disciplina SQL:

- Los vectores encuentran lo que se siente relevante
- SQL asegura que los resultados sean válidos (categoría, precio actual, uniones a filas reales)

Por lo tanto, la base de datos se mantiene honesta.

Desde el principio, le damos a esto una forma apta para producción. Observa lo que **no** hicimos:

- No volcamos toda la tabla de productos en el modelo
- No dejamos que el modelo inventara uniones
- No permitimos que un sistema externo se convirtiera en la fuente de la verdad

En su lugar, `api.sf_similar_items` hace que la recuperación de significado se sienta como una función SQL regular: predecible, delimitada y auditable:

- `LIMIT p_k` delimita la carga de trabajo
- `MATERIALIZED` mantiene estable el plan de recuperación (y evita sorpresas de reevaluación)
- Las uniones imponen la verdad relacional: categoría + precio actual

Esta es la primera regla de la IA nativa de bases de datos: usa vectores para encontrar candidatos y luego usa SQL para imponer la realidad.

La consulta básica produce la siguiente salida:

```sql
SELECT product_id, name, category, price FROM api.sf_similar_items('looking for tailored clothing', 1); 
```

```text
product_id | name | category | price ------------+-----------------------------------------+----------+------- 14 | Suit coat - business perfect, by Brioni | Pants | 98.43 (1 row)
```

#### El narrador: api.sf_answer_with_openai

Las filas son verídicas, pero no siempre son amigables. Esta función es la voz: toma la pregunta y las filas (como JSON) y produce una respuesta humana clara.

La elección de diseño más importante es que le dice explícitamente al modelo que no invente nada más allá de las filas. El modelo se convierte en un narrador, no en un oráculo. La siguiente función implementa el componente narrador:

```sql
CREATE OR REPLACE FUNCTION api.sf_answer_with_openai( p_question text, p_rows jsonb ) RETURNS text LANGUAGE plpython3u VOLATILE SECURITY DEFINER SET search_path = pg_catalog, public, api, product, embeddings AS $$ import json import random import ssl import time import urllib.error import urllib.request def _get_guc(p_name: str): v_rv = plpy.execute( "SELECT current_setting(%s, true) AS v", [p_name], ) if not v_rv or v_rv[0]["v"] is None: return None return v_rv[0]["v"] def _call_openai_chat_completions(p_payload: dict, p_api_key: str, p_org): v_headers = { "Content-Type": "application/json", "Authorization": f"Bearer {p_api_key}", "User-Agent": "pg18book-plpython-openai-chat/1.0", } if p_org: v_headers["OpenAI-Organization"] = p_org v_req = urllib.request.Request( url="https://api.openai.com/v1/chat/completions", data=json.dumps(p_payload).encode("utf-8"), headers=v_headers, method="POST", ) v_ctx = ssl.create_default_context() with urllib.request.urlopen(v_req, context=v_ctx, timeout=30) as v_resp: v_raw = v_resp.read().decode("utf-8", errors="replace") return json.loads(v_raw) def _extract_content(p_data: dict) -> str: if "choices" not in p_data or not p_data["choices"]: raise Exception("Unexpected OpenAI response: missing choices") v_msg = p_data["choices"][0].get("message", {}) v_content = v_msg.get("content", None) if v_content is None: raise Exception("Unexpected OpenAI response: missing message content") return str(v_content).strip() v_api_key = _get_guc("api.openai_api_key") v_org = _get_guc("api.openai_organization") if not v_api_key: raise Exception( "OpenAI API key not set. Use: " "SELECT set_config('api.openai_api_key','sk-...','f');" ) v_rows = json.dumps(p_rows) v_system = ( "You are a helpful assistant.\n" "Take the user question and the SQL rows returned, and write a clear, " "human reply.\n" "- Mention the question.\n" "- Summarize how many results were found.\n" "- List items with their label, price if present, and category.\n" "- Do not invent anything beyond rows JSON.\n" "Schema tables/columns:\n" "- product.product(id, category_id, brand_id, label, shortdescription, " "longdescription)\n" "- embeddings.product_embedding(product_id, embedding)\n" "- product.category(id, label, description)\n" "- product.brand(id, label, description)\n" "- product.product_variant(id, product_id, attributes)\n" "- product.product_variant_price(id, product_variant_id, price, validity, " "current)\n" "- api.category_complements(category_name, complements)\n" "\n" "Rules:\n" "- Categories are broad: product.category.label values like 'Jeans', " "'Shirt', 'Skirt'.\n" "- Gender words (women, men, kids) appear in product.label, NOT in " "category.\n" "- Price: product_variant_price.price (alias pvp).\n" "- Color/size attributes are in product_variant.attributes (JSONB).\n" "- Category: product.category.label (alias pc). Join pc ON pc.id = " "p.category_id.\n" "- To use price/color: JOIN product_variant pv ON pv.product_id = p.id " "AND JOIN product_variant_price pvp ON pvp.product_variant_id = pv.id.\n" "- Use LOWER() for case-insensitive filters, e.g. LOWER(p.label) LIKE " "'%women%'.\n" "- Only SELECT. No semicolons.\n" "- Prefer clear aliases (product_name, price).\n" "- If no rows are returned just respond.\n" ) v_user = f"Question: {p_question}\n\nRows: {v_rows}" v_payload = { "model": "gpt-4o-mini", "messages": [ {"role": "system", "content": v_system}, {"role": "user", "content": v_user}, ], "temperature": 0.2, "max_tokens": 500, } v_attempts = 6 for v_i in range(v_attempts): try: v_data = _call_openai_chat_completions(v_payload, v_api_key, v_org) return _extract_content(v_data) except urllib.error.HTTPError as v_e: v_body = v_e.read().decode("utf-8", errors="ignore") if hasattr(v_e, "read") else "" v_code = getattr(v_e, "code", None) v_transient = v_code in (429, 500, 502, 503, 504) v_last = (v_i == v_attempts - 1) if (not v_transient) or v_last: raise Exception( f"OpenAI HTTP {v_code} (attempt {v_i + 1}/{v_attempts}). " f"Body: {v_body[:400]} ..." ) v_retry_after = None try: v_retry_after = v_e.headers.get("Retry-After") except Exception: v_retry_after = None if v_retry_after: try: v_sleep_s = float(v_retry_after) except Exception: v_sleep_s = (2 ** v_i) * 0.5 + random.uniform(0, 0.3) else: v_sleep_s = (2 ** v_i) * 0.5 + random.uniform(0, 0.3) time.sleep(v_sleep_s) except urllib.error.URLError: v_last = (v_i == v_attempts - 1) if v_last: raise time.sleep((2 ** v_i) * 0.5 + random.uniform(0, 0.2)) except Exception: v_last = (v_i == v_attempts - 1) if v_last: raise time.sleep((2 ** v_i) * 0.5 + random.uniform(0, 0.2)) $$;
```

Repasemos lo que hace la función:

La recuperación nos da filas, pero las filas no son intuitivas para el usuario. Un chatbot necesita comunicarse en lenguaje natural: necesita hablar. Eso es lo que hace `api.sf_answer_with_openai(p_question, p_rows)`:

- Recibe:
  - La pregunta original del usuario
  - Los resultados SQL en formato JSON (`p_rows`)
- Le pide al modelo que produzca una respuesta clara
- Pero con una regla estricta: **no inventar nada más allá de las filas**

Este diseño es sutil pero crucial. El LLM no explora tu base de datos libremente. No consulta tablas por su cuenta. Solo ve los registros que tú le pasas. Eso mantiene al modelo en el rol en el que destaca: resumir y comunicar, no actuar como base de datos.

Esta función no es "IA haciendo SQL". Es IA haciendo lo que debe: convertir evidencia estructurada en lenguaje humano.

#### El orquestador: api.sf_chat

Ahora combinaremos la recuperación y la narración en una sola llamada a `api.sf_chat`. Este es nuestro primer chatbot.

```sql
CREATE OR REPLACE FUNCTION api.sf_chat( p_question text, p_k integer DEFAULT 10 ) RETURNS TABLE ( assistant_text text, rows jsonb ) LANGUAGE plpgsql VOLATILE SECURITY DEFINER SET search_path = pg_catalog, public, api, product, embeddings AS $$ DECLARE v_data jsonb; BEGIN SELECT jsonb_agg(t) INTO v_data FROM ( SELECT s.product_id, s.name, s.category, s.shortdescription, s.longdescription, s.price, s.distance FROM api.sf_similar_items(p_question, p_k) s ) t; assistant_text := api.sf_answer_with_openai(p_question, v_data); rows := coalesce(v_data, '[]'::jsonb); RETURN NEXT; END; $$;
```

Lo que hace es combinar los dos pasos en una interfaz simple:

1. Obtener artículos semánticamente similares usando `api.sf_similar_items`
2. Agregar esas filas en formato JSON
3. Pasarlas a `api.sf_answer_with_openai`
4. Devolver:
   - `assistant_text`
   - `rows` (para que podamos inspeccionar lo que utilizó)

Esto nos brinda un asistente con las siguientes cualidades:

- **Basado en datos reales**
- **Reproducible** (las mismas filas se asignan a la misma respuesta)
- **Depurable** (puedes inspeccionar las filas)

Lo que obtienes de vuelta consta intencionalmente de dos partes:

- `assistant_text`: la respuesta de cara al usuario
- `rows`: la evidencia en la que se basó la respuesta

Esa segunda parte es tu ancla de verdad. Cuando alguien pregunte *"¿Por qué el bot recomendó esto?"*, no discutes. Señalas las filas subyacentes a la respuesta.

*Figura 20.1: Coordinador, recuperador, narrador y sus respectivas interfaces de LLM*

> [!NOTE]
> **Descarga las imágenes en color**  
> Tu compra incluye una copia en PDF en color y sin DRM de este libro, ideal para ver imágenes en color, capturas de pantalla y diagramas. Consulta la sección de beneficios gratuitos con tu libro al final del Prefacio para desbloquear tu copia en PDF.

La Figura 20.1 muestra cómo el coordinador, el recuperador y el narrador trabajan juntos para generar la respuesta. El recuperador utiliza la API de generación de *embeddings* del LLM para identificar filas significativas, y el narrador utiliza la API de completado de chat para convertir las filas en una respuesta en lenguaje natural fácilmente comprensible.

#### La primera conversación funcional

Antes de chatear, necesitamos *embeddings* y una clave de API:

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false); SELECT api.embed_products(25);
```

Ahora puedes ejecutar consultas como esta:

```sql
-- Tip (psql): results can be wide. Use expanded output or disable the pager: -- \x -- \pset pager off SELECT * FROM api.sf_chat('looking for tailored clothing', 1); SELECT * FROM api.sf_chat('Tell me about polo shirts in different colors', 2); SELECT * FROM api.sf_chat('What are the best pants for men?', 1);
```

Tomemos este ejemplo:

```sql
SELECT * FROM api.sf_chat('looking for tailored clothing', 1);
```

Produce el siguiente resultado:

```text
-[ RECORD 1 ]--+----------------------------------------------------------------- assistant_text | You asked for tailored clothing, and I found 1 | result that matches your request. | Here are the details: | - **Product Name**: Suit coat - business perfect, by | Brioni | - **Price**: $98.43 | - **Category**: Pants | | This suit coat is crafted from premium wool or | wool-blend fabrics, offering a sharp yet comfortable | silhouette. It is designed for versatility, suitable | for both formal occasions and smart-casual wear. rows | [{"name": "Suit coat - business perfect, by Brioni", | "price": 98.43, "category": "Pants", "distance": |0.5368103661054147, "product_id": 14, "..."}]
```

En esta etapa, tenemos un asistente en funcionamiento. Pero aún no lo hemos hecho inteligente respecto a las restricciones. Un usuario real rara vez pide únicamente artículos "similares". Piden lo siguiente:

- *"menos de 100 $"*
- *"solo precio actual"*
- *"color azul"*
- *"casual de negocios"*
- *"Muéstrame camisas, no chaquetas."*

Ahí es donde necesitamos un comportamiento dinámico.

#### Haciéndolo más robusto: Uso de embeddings.sf_get_query_embedding

En este momento, `api.sf_similar_items` genera el *embedding* del texto de la consulta en cada ocasión. Eso es correcto, pero en producción introduce latencia y aumenta los costos porque interactúa con el LLM en cada solicitud.

Aquí es donde encaja nuestro asistente previo que utiliza la caché de consultas. Podemos mantener el diseño de la función idéntico y simplemente cambiar:

```sql
SELECT api.openai_embed(query_text)::vector(1536) INTO qvec;
```

por:

```sql
SELECT embeddings.sf_get_query_embedding(query_text)::vector(1536) INTO qvec;
```

Ese único cambio hace que las consultas repetidas sean más rápidas y económicas, y proporciona un punto limpio para la disciplina de versiones de modelos (`model_id`) y el desalojo de caché, como se discutió en el [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19).

---

### El chatbot de SQL dinámico

Ahora pasamos de un chatbot "fijo" a uno dinámico: el asistente puede interpretar la solicitud de un usuario, traducirla a una consulta SQL y ejecutarla.

Aquí es donde muchos sistemas se descarrilan, porque un chatbot de SQL dinámico debe obedecer dos leyes:

1. El modelo puede proponer SQL, pero el sistema debe hacer cumplir la política.
2. Cada consulta ejecutada debe ser delimitada y auditable.

Por lo tanto, no le damos al modelo una ejecución SQL sin restricciones. Le damos un carril de ejecución seguro.

#### Qué significa "dinámico" en nuestro esquema

Un usuario podría preguntar: *"Muéstrame polos de menos de 80 $ que tengan precio actual y menciona los colores disponibles"*.

Nuestro esquema nos dice exactamente qué significa "colores":

- El color reside en `product.product_variant.attributes` (JSONB)
- El precio reside en `product.product_variant_price`
- El precio actual requiere `pvp.current = true`
- La etiqueta del producto contiene palabras como *"polo"*

Por lo tanto, una consulta segura podría verse así:

```sql
SELECT p.id, p.label, c.label AS category, pvp.price, pv.attributes->>'color' AS color FROM product.product p JOIN product.category c ON c.id = p.category_id JOIN product.product_variant pv ON pv.product_id = p.id JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true WHERE LOWER(p.label) LIKE '%polo%' AND pvp.price <= 80 LIMIT 3;
```

Produciría la siguiente salida:

```text
id | label | category | price | color ----+---------------------------+----------+-------+------- 20 | Tommy Hilfiger Polo Shirt | Shirts | 31.72 | navy 20 | Tommy Hilfiger Polo Shirt | Shirts | 31.72 | white 20 | Tommy Hilfiger Polo Shirt | Shirts | 31.72 | red (3 rows)
```

El trabajo del chatbot es generar consultas como esta, pero únicamente dentro de un entorno aislado (*sandbox*).

#### Un ejecutor de SQL dinámico seguro (SELECT-only + LIMIT + Lista de permitidos)

Hasta ahora, nuestro chatbot ha sido disciplinado. Toma una pregunta, utiliza *embeddings* para recuperar productos relevantes y confía en el modelo solo para explicar las filas que se le proporcionaron. Eso es seguro, fundamentado y adecuado para producción.

Pero los usuarios reales no siempre preguntan "¿qué es similar?". Piden restricciones:

- *"Muéstrame polos de menos de 80 $."*
- *"Solo artículos con precio actual."*
- *"Solo ropa de hombre."*
- *"Muestra diferentes colores y tallas."*
- *"Dame 10 opciones, no 100."*

Aquí es donde la siguiente evolución resulta tentadora: dejar que el modelo escriba SQL.

Esa tentación también es donde muchos desarrolladores construyen accidentalmente un riesgo de seguridad crítico. Así que haremos algo mejor.

Construiremos un chatbot de SQL dinámico, pero lo pondremos sobre raíles para mantenerlo en el camino correcto:

- El modelo puede proponer SQL
- PostgreSQL vigilará el SQL
- La respuesta estará basada en filas

Esta sección introduce un patrón de producción que reutilizarás en todas partes: **Los modelos sugieren. Los sistemas imponen. Las bases de datos verifican.**

#### Qué significa "seguro" en la práctica

Un ejecutor de SQL dinámico seguro debe exigir algunos aspectos innegociables:

- **Solo SELECT (`SELECT-only`)**: Sin inserciones. Sin actualizaciones. Sin eliminaciones. Sin DDL. Sin "ups, eliminé la tabla".
- **Un límite estricto de filas**: Cada consulta debe tener un `LIMIT`, y debemos limitarlo nosotros mismos.
- **Una lista de permitidos (*allowlist*) de esquemas/tablas**: El modelo solo obtiene acceso al conjunto de tablas que permitimos explícitamente:
  - `product.*`
  - `embeddings.*` (opcionalmente)
  - Vistas/funciones `api.*` (opcionalmente)
- **Sin ejecución de múltiples declaraciones**: Solo una consulta. Sin puntos y coma. Sin sentencias apiladas.
- **Comportamiento auditable**: Deberíamos poder registrar lo que se preguntó y lo que se ejecutó.

Ahora implementaremos la versión segura más simple usando una función auxiliar: `api.sf_safe_select(...)`.

#### Un ejecutor de consultas "seguro" mínimo: api.sf_safe_select

Esta función acepta una cadena SQL (generada por el modelo), la valida, impone un `LIMIT` máximo y devuelve filas en JSONB que podemos alimentar directamente a `api.sf_answer_with_openai(...)`.

Esta es la definición de la función para un ejecutor SQL seguro:

```sql
CREATE OR REPLACE FUNCTION api.sf_safe_select( p_sql text, p_max_rows integer DEFAULT 50 ) RETURNS jsonb LANGUAGE plpgsql STABLE SECURITY DEFINER SET search_path = pg_catalog, api, product, embeddings AS $$ DECLARE v_sql text; v_rows jsonb; BEGIN --- Normalize whitespace v_sql := regexp_replace(coalesce(p_sql, ''), '\s+', ' ', 'g'); --- Single statement only (reject semicolons) IF position(';' IN v_sql) > 0 THEN RAISE EXCEPTION 'Only a single SELECT statement is allowed (no semicolons).' USING ERRCODE = '42601'; END IF; --- Must start with SELECT or WITH (CTE) IF NOT (v_sql ~* '^\s*(select|with)\y') THEN RAISE EXCEPTION 'Only SELECT queries are allowed.' USING ERRCODE = '42501'; END IF; --- Reject dangerous keywords (defense-in-depth) IF v_sql ~* '\y(insert|update|delete|merge|drop|alter|create|truncate|grant|revoke|comment|vacuum|analyze|copy|call|do)\y' THEN RAISE EXCEPTION 'Non-SELECT keywords detected. Query rejected.' USING ERRCODE = '42501'; END IF; --- Disallow system schemas IF v_sql ~* '\y(pg_catalog|information_schema)\y' THEN RAISE EXCEPTION 'System schemas are not allowed.' USING ERRCODE = '42501'; END IF; --- Require allowed schemas (intentionally strict) IF v_sql !~* '\y(product|api|embeddings)\.' THEN RAISE EXCEPTION 'Query must reference allowed schemas (product., embeddings., api.).' USING ERRCODE = '42501'; END IF; --- Add LIMIT only if missing IF v_sql !~* '\ylimit\y' THEN --- If query ends with FOR UPDATE/SHARE variants, insert LIMIT before it IF v_sql ~* '\yfor\s+(update|share|no\s+key\s+update|key\s+share)\y\s*$' THEN v_sql := regexp_replace( v_sql, '(\yfor\s+(update|share|no\s+key\s+update|key\s+share)\y\s*)$', format(' LIMIT %s \1', p_max_rows), 1, 1, 'i' ); ELSE v_sql := v_sql || format(' LIMIT %s', p_max_rows); END IF; END IF; --- Wrap to return JSON v_sql := format( 'SELECT coalesce(jsonb_agg(t), ''[]''::jsonb) FROM (%s) t', v_sql ); EXECUTE v_sql INTO v_rows; RETURN coalesce(v_rows, '[]'::jsonb); END; $$;
```

A continuación se muestran ejemplos de cómo el ejecutor seguro detiene SQL potencialmente malicioso:

```sql
SELECT api.sf_safe_select('INSERT INTO TEST VALUES(1,2)', 3);
```

```text
ERROR: Only SELECT queries are allowed. CONTEXT: PL/pgSQL function safe_select(text,integer) line 16 at RAISE
```

```sql
SELECT api.sf_safe_select('SELECT * FROM pg_catalog.pg_class', 4);
```

```text
ERROR: System schemas are not allowed. CONTEXT: PL/pgSQL function safe_select(text,integer) line 26 at RAISE
```

```sql
SELECT api.sf_safe_select('SELECT * FROM public.test', 4);
```

```text
ERROR: Query must reference allowed schemas (product., embeddings., api.). CONTEXT: PL/pgSQL function safe_select(text,integer) line 31 at RAISE
```

Esto es intencionalmente mínimo pero real:

- Bloquea trucos de múltiples declaraciones
- Bloquea la intención obvia de no usar `SELECT`
- Bloquea el acceso al esquema del sistema
- Requiere que las consultas toquen únicamente `product`, `embeddings` o `api`
- Limita estrictamente la salida mediante un límite obligatorio

¿Es perfecto? No. Pero es una base sólida y demuestra el patrón: el modelo no tiene rienda suelta, la base de datos sigue al mando.

#### Convertir una pregunta en SQL (el modelo propone, la base de datos impone)

Ahora necesitamos una función que le pida al modelo una consulta SQL.

El prompt debe ser estricto:

- Debe utilizar nuestro esquema
- Solo debe usar `SELECT`
- Siempre debe incluir `LIMIT`
- Debe preferir las uniones que ya sabemos que son correctas (especialmente el precio con `current = true`)
- Debe devolver solo SQL (sin comentarios)

Aquí está la definición de la función para el generador de SQL:

```sql
CREATE OR REPLACE FUNCTION api.sf_sql_from_question( p_question text, p_row_limit integer DEFAULT 20 ) RETURNS text LANGUAGE plpython3u VOLATILE AS $$ import json import ssl import urllib.request rv = plpy.execute("SELECT current_setting('api.openai_api_key', true) AS k") v_api_key = rv[0]["k"] if rv and rv[0]["k"] is not None else None if not v_api_key: raise Exception("OpenAI API key not set.") v_system = f""" You are a PostgreSQL SQL generator for an e-commerce schema. Return ONLY SQL (no markdown, no explanation, no semicolons). Hard rules: - SELECT or WITH only. - Must include LIMIT {p_row_limit}. - Use only these schemas: product, embeddings, api. - Prefer current price only: JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true - product table: product.product p (id, category_id, brand_id, label, shortdescription, longdescription, image_filename) - category table: product.category c (id, label, description) - brand table: product.brand b (id, label, description) - variants: product.product_variant pv (id, product_id, attributes JSONB) - price: product.product_variant_price pvp (product_variant_id, price, validity, current) - For color/size use pv.attributes->>'color', pv.attributes->>'size' - For case-insensitive text: use LOWER(p.label) LIKE '%...%' Output columns should be useful: id, label, category, price, and attributes when relevant. """ v_user = f"Question: {p_question}" v_payload = { "model": "gpt-4o-mini", "messages": [ {"role": "system", "content": v_system}, {"role": "user", "content": v_user} ], "temperature": 0.0, "max_tokens": 250 } v_headers = { "Content-Type": "application/json", "Authorization": f"Bearer {v_api_key}" } v_ctx = ssl.create_default_context() v_req = urllib.request.Request( "https://api.openai.com/v1/chat/completions", data=json.dumps(v_payload).encode("utf-8"), headers=v_headers ) with urllib.request.urlopen(v_req, context=v_ctx) as resp: v_data = json.loads(resp.read().decode("utf-8")) v_sql = v_data["choices"][0]["message"]["content"].strip() # Defensive cleanup: remove trailing semicolons if model slips v_sql = v_sql.replace(";", "") return v_sql $$;
```

Esta función es intencionalmente acotada. No intenta ser ingeniosa, intenta ser segura.

##### Cómo funciona, paso a paso

1. **Lee la clave de API de OpenAI desde la configuración de Postgres**:

```python
rv = plpy.execute("SELECT current_setting('api.openai_api_key', true) AS k") v_api_key = rv[0]["k"] if rv and rv[0]["k"] is not None else None if not v_api_key: raise Exception("OpenAI API key not set.")
```

Si la clave no está configurada, la función genera el error "OpenAI API key not set". Así es como la base de datos almacena las credenciales de forma segura (mediante un parámetro GUC).

2. **Construye las instrucciones para el modelo de IA (el prompt `SYSTEM`)**:
   - Eres un generador de SQL para este esquema de comercio electrónico
   - Devuelve solo SQL: sin explicaciones, sin markdown
   - Reglas de seguridad/forma:
     - Solo `SELECT` o `WITH`
     - Debe incluir `LIMIT {p_row_limit}`
     - Solo usar esquemas: `product`, `embeddings` y `api`
     - Preferir filas de precio actual usando una condición de unión específica
   - Proporciona el mapa del esquema (tablas + columnas importantes)
   - Da convenciones para cosas tales como:
     - `pv.attributes->>'color'` para color
     - `LOWER(p.label) LIKE '%...%'` para búsqueda sin distinción de mayúsculas/minúsculas
   - Sugiere qué columnas generar (`id`, `label`, `category`, `price`, etc.)

3. **Envía la pregunta del usuario al modelo (prompt `USER`)**:

```python
v_user = f"Question: {p_question}"
```

Esta es la pregunta real que ingresaste.

4. **Llama a la API de Chat Completions de OpenAI**:
   - `model`: `gpt-4o-mini`
   - `messages`: `[system, user]`
   - `temperature`: `0.0` (hace que la salida sea determinista/menos "creativa")
   - `max_tokens`: `250` (limita el tamaño de la respuesta)

5. **Extrae el SQL de la respuesta**:

```python
v_sql = v_data["choices"][0]["message"]["content"].strip()
```

6. **Devuelve la cadena SQL**:
   Esa cadena se puede pasar a tu función `api.sf_safe_select()` para ejecutarse de forma segura.

#### El chatbot dinámico: api.sf_dynamic_chat

Ahora ensamblaremos la canalización:

1. El modelo propone SQL (`api.sf_sql_from_question`)
2. La base de datos valida y ejecuta de forma segura (`api.sf_safe_select`)
3. El modelo narra los resultados (`api.sf_answer_with_openai`)
4. El chatbot devuelve tanto la respuesta como la evidencia (filas y SQL)

Aquí está la definición de función para nuestro chatbot dinámico:

```sql
CREATE OR REPLACE FUNCTION api.sf_dynamic_chat( p_question text, p_row_limit integer DEFAULT 20 ) RETURNS TABLE ( assistant_text text, sql_used text, rows jsonb ) LANGUAGE plpgsql VOLATILE AS $$ DECLARE v_sql text; v_rows jsonb; BEGIN --- Ask model for SQL v_sql := api.sf_sql_from_question(p_question, p_row_limit); --- Execute safely (SELECT-only + limit + allowlist) v_rows := api.sf_safe_select(v_sql, p_row_limit); --- Narrate based only on returned rows assistant_text := api.sf_answer_with_openai(p_question, v_rows); sql_used := v_sql; rows := v_rows; RETURN NEXT; END; $$;
```

Este es el momento en que el chatbot se vuelve "dinámico", pero permanece disciplinado.

El modelo no es tu DBA; es tu analista junior con un cuaderno: útil, rápido y ocasionalmente confiado de más. Por lo tanto, PostgreSQL sigue siendo el adulto en la sala. He aquí un ejemplo:

Primero, asegúrate de que la clave de API de OpenAI esté configurada:

```sql
SELECT set_config('api.openai_api_key','sk-...YOUR_KEY...', false);
```

Luego ejecuta la consulta:

```sql
SELECT * FROM api.sf_dynamic_chat('Show me polo shirts under $80 with current price', 1);
```

Esto crea la siguiente salida:

```text
-[ RECORD 1 ]--+----------------------------------------------------------------------------------------------------------------- assistant_text | You asked for polo shirts under $80 with the current price. I found 3 results for you. | | 1. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size M, Color navy, Style | classic polo | 2. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size L, Color white, Style | classic polo | | 3. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size XL, Color red, Style | classic polo sql_used | SELECT p.id, p.label, c.label AS category, | pvp.price, pv.attributes | FROM product.product p | JOIN product.category c ON p.category_id = c.id | JOIN product.product_variant pv ON p.id = | pv.product_id | JOIN product.product_variant_price pvp ON | pvp.product_variant_id = pv.id AND pvp.current = | true | WHERE LOWER(p.label) LIKE '%polo%' AND pvp.price < | 80 | LIMIT 10 rows | [{"id": 20, "label": "Tommy Hilfiger Polo Shirt", "price": 31.72, | "category": "Shirts", "attributes": {"logo": "flag", "size":..}]
```

Aquí hay otro ejemplo:

```sql
SELECT * FROM api.sf_dynamic_chat('Show me men''s shirts in blue, include the color attribute', 1);
```

El modelo debería utilizar:

- `pv.attributes->>'color'`
- `LOWER(p.label) LIKE '%men%'` o una señal de texto similar
- `pvp.current = true`

> [!NOTE]
> El SQL dinámico es excelente para filtros, pero no es una recuperación semántica por defecto. Si deseas una recuperación basada en el significado, aún querrás `api.sf_similar_items`.

Por ahora, lo mantendremos simple:

- **SQL dinámico** maneja restricciones estructuradas
- **Búsqueda semántica** maneja la recuperación de significado

#### ¿Por qué no agregamos embeddings.sf_get_query_embedding en este punto?

Podemos usar `embeddings.sf_get_query_embedding` aquí para mayor solidez, pero no dentro de `api.sf_sql_from_question` porque esa función genera SQL, no incrusta texto. En su lugar, pertenece a la ruta de herramientas semánticas:

- `api.sf_similar_items` actualmente llama a `api.openai_embed(query_text)` en cada solicitud
- En producción, podemos reemplazarlo con *embeddings* en caché:
  - Menos llamadas a la API
  - Menor latencia para consultas repetidas
  - El control de versiones del modelo se vuelve manejable

Así que establecemos el siguiente refinamiento:

- **SQL dinámico** = chatbot de filtrado estructurado
- **Recuperación semántica** = chatbot de significado
- **Caché de embeddings de consulta** = capa de eficiencia de producción

En producción, normalmente agregas:

- Un rol de base de datos dedicado con acceso de solo lectura a los esquemas permitidos
- Registro de consultas (pregunta, SQL, duración, recuento de filas, usuario)
- Tiempos de espera (`statement_timeout`)
- Análisis más estricto (validación basada en AST en lugar de expresiones regulares)
- Un protocolo de herramientas (MCP) para que el modelo elija entre herramientas seguras en lugar de escribir SQL arbitrario

#### Enseñar al asistente cuándo usar búsqueda semántica vs. SQL dinámico

En este punto, hemos construido dos "cerebros" dentro de PostgreSQL:

- **Cerebro semántico (el significado primero)**: `api.sf_chat()`, que usa `api.sf_similar_items()`, *embeddings* y `pgvector`.
- **Cerebro estructurado (las reglas primero)**: `api.sf_dynamic_chat()`, que genera SQL, usa `api.sf_safe_select()`, devuelve filas y narra.

Ambos son útiles. Ambos son correctos. Pero ambos pueden decepcionar si se usan en el momento equivocado.

Un motor semántico es brillante para *"No sé las palabras exactas, pero sé lo que quiero decir"*. Un motor SQL estructurado es brillante para *"Sé exactamente lo que quiero; por favor, obedece las restricciones"*. Tu trabajo en producción no es elegir uno, sino enrutar la solicitud a la herramienta adecuada de manera consistente, predecible y segura.

Construiremos una pequeña función "agente de tráfico": un enrutador (*router*).

#### El modelo mental de dos carriles

Piensa en las preguntas de los usuarios como si cayeran en dos carriles:

##### Carril A: Semántico (significado/intención)

Ejemplos:

- *"busco ropa a medida"*
- *"algo parecido a una chaqueta de cuero pero más ligera"*
- *"atuendos para una ocasión formal"*
- *"similar a Boss pero más informal"*

Estas preguntas son poco especificadas a propósito. El usuario le pide al sistema que interprete la intención; por lo tanto, usa `api.sf_chat(p_question, p_k)`.

##### Carril B: Estructurado (filtros/restricciones)

Ejemplos:

- *"polos de menos de 80 $"*
- *"muestra camisas azules e incluye el color"*
- *"solo artículos con precio actual"*
- *"limita a 10 y ordena por precio"*

Estas preguntas piden obediencia a reglas más que significado; por lo tanto, el sistema debe usar `api.sf_dynamic_chat(p_question, p_row_limit)`.

#### Un enrutador simple: api.sf_route_chat

Vamos a hacer algo deliberadamente sencillo:

1. Clasificar la pregunta mediante heurísticas ligeras
2. Elegir una de las dos herramientas
3. Devolver la respuesta y la evidencia

```sql
CREATE OR REPLACE FUNCTION api.sf_route_chat( p_question text, p_k integer DEFAULT 10, p_row_limit integer DEFAULT 20 ) RETURNS TABLE ( tool_used text, assistant_text text, rows jsonb ) LANGUAGE plpgsql VOLATILE AS $$ DECLARE v_question_lc text := lower(coalesce(p_question, '')); BEGIN /* Routing heuristic: - If question contains explicit constraints (price, color, size, limit, current, category), prefer dynamic SQL chatbot. - Otherwise, prefer semantic chatbot. This is intentionally conservative: structured constraints should be enforced by SQL. */ IF v_question_lc ~ ( '\yunder\y|\yless than\y|\yprice\y|\$|\ycolor\y|\ysize\y|\ylimit\y|' || '\ycurrent\y|\yin[- ]stock\y|\ycategory\y|\ybrand\y' ) THEN tool_used := 'dynamic_sql'; SELECT dc.assistant_text, dc.rows INTO assistant_text, rows FROM api.sf_dynamic_chat(p_question, p_row_limit) dc; RETURN NEXT; ELSE tool_used := 'semantic'; SELECT c.assistant_text, c.rows INTO assistant_text, rows FROM api.sf_chat(p_question, p_k) c; RETURN NEXT; END IF; END; $$;
```

Esta lógica de enrutamiento es intencionalmente firme:

- Si el usuario expresa restricciones, deja que SQL las imponga.
- Si el usuario expresa una intención, deja que los *embeddings* la interpreten.

Esa regla te ahorrará una categoría completa de problemas donde "el chatbot suena bien pero viola la realidad".

He aquí un par de ejemplos:

```sql
SELECT * FROM api.sf_route_chat('Show me one polo shirts under $80 with current price', 1,1);
```

```text
-[ RECORD 1 ] tool_used | dynamic_sql assistant_text | You asked for polo shirts under $80 with the current price. I found | 3 results for you. | | 1. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size M, Color navy, Style classic polo | | 2. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size L, Color white, Style classic polo | | 3. **Label**: Tommy Hilfiger Polo Shirt | - **Price**: $31.72 | - **Category**: Shirts | - **Attributes**: Size XL, Color red, Style classic polo rows | [{"id": 20, "label": "Tommy Hilfiger Polo Shirt", "price": 31.72, | "category": "Shirts", "attributes": {"logo": "flag", "size": "M", | "color": "navy", "style": "classic polo"}}, {"id": 20, "label": | "Tommy Hilfiger Polo Shirt", "price": 31.72, "category": "...]
```

Otro ejemplo:

```sql
SELECT * FROM api.sf_route_chat('I am looking for tailored clothing', 1,1);
```

```text
-[ RECORD 1 ] tool_used | semantic assistant_text | You asked for tailored clothing, and I found 2 results that match | your request. | | 1. **Suit coat - business perfect, by Brioni** | - **Price:** $98.43 | - **Category:** Pants | - **Description:** Classic tailored suit coat - premium wool, | sharp silhouette, versatile elegance. | | 2. **Suit coat - business perfect, by Brioni** | - **Price:** $98.43 | - **Category:** Pants | - **Description:** Classic tailored suit coat - premium wool, | sharp silhouette, versatile elegance. | | Both items are the same suit coat, offering a refined and versatile | option for tailored clothing. rows | [{"name": "Suit coat - business perfect, by Brioni", "price": | 98.43, "category": "Pants", "distance": 0.5368103661054147, | "product_id": 14, "longdescription": "Refined Versatility:...}]
```

#### Manejo de preguntas híbridas (el mundo real)

Al mundo real le encantan los híbridos: *"Muéstrame algo parecido a una chaqueta de cuero, pero de menos de 150 $"*.

Esto es tanto semántico como estructurado. Nuestro enrutador simple probablemente elegirá SQL dinámico (porque vio "menos de 150 $"), pero el SQL dinámico puede no capturar bien "parecido a una chaqueta de cuero".

Por lo tanto, el patrón adecuado para producción es:

1. Recuperación semántica para obtener candidatos.
2. Filtrado SQL para imponer restricciones.
3. Narración con evidencia.

#### Un chat híbrido: Candidatos semánticos y filtros SQL

Híbrido aquí significa dos tipos diferentes de lógica cooperando:

- **Primero, la recuperación semántica (búsqueda vectorial)** responde: "¿Qué artículos se sienten como esta pregunta?". Nos da una lista de candidatos clasificados por significado.
- **Segundo, el filtrado relacional (restricciones SQL)** responde: "¿Cuáles de estos candidatos están realmente permitidos?". Precio, categorías, atributos de talla/color en JSONB; las reglas de inventario son restricciones estrictas; SQL es el juez.

Utilizamos un enfoque por etapas:

1. **Significado primero**: recuperar los N mejores candidatos por similitud de *embeddings* (alto *recall*)
2. **Reglas segundo**: aplicar filtros SQL estrictos a esos candidatos (alta precisión)
3. **Explicar con evidencia**: narrar solo lo que sobrevivió, basado en las filas devueltas

Piensa en SQL como un guardia de seguridad: los vectores arman la lista de invitados, SQL verifica la identificación.

La siguiente función implementa el chat híbrido:

```sql
CREATE OR REPLACE FUNCTION api.sf_hybrid_chat( p_question text, p_max_price numeric DEFAULT NULL, p_k integer DEFAULT 20 ) RETURNS TABLE ( assistant_text text, rows jsonb ) LANGUAGE plpgsql VOLATILE AS $function$ DECLARE v_data jsonb; BEGIN SELECT jsonb_agg(t) INTO v_data FROM ( SELECT s.product_id, s.name, s.category, s.shortdescription, s.longdescription, s.price, s.distance FROM api.sf_similar_items(p_question, p_k) AS s WHERE (p_max_price IS NULL OR s.price <= p_max_price) ORDER BY s.distance LIMIT 10 ) t; assistant_text := api.sf_answer_with_openai(p_question, coalesce(v_data, '[]'::jsonb)); rows := coalesce(v_data, '[]'::jsonb); RETURN NEXT; END; $function$;
```

Ejemplo de uso híbrido:

```sql
SELECT * FROM api.sf_hybrid_chat('something like a leather jacket but lighter', 150, 20);
```

---

### Introducción a MCP: El contrato entre modelos y herramientas

Una vez que permites que un modelo realice acciones como ejecutar consultas, obtener datos y llamar a funciones, has pasado de "asistente" a "agente". Los agentes necesitan lo que los humanos siempre han necesitado: un contrato.

No intuiciones, no esperanzas, no "le dijimos que se portara bien": un contrato real. Y eso es lo que proporciona el MCP.

#### ¿Por qué es importante el MCP para los asistentes de bases de datos?

Una base de datos no es una ventana de chat. Es una fuente de verdad. Los riesgos de usar IA con una base de datos no son solo las alucinaciones:

- El modelo consulta lo incorrecto
- Devuelve datos confidenciales que no pretendías exponer
- Ejecuta consultas costosas que saturan tu clúster
- Omite reglas comerciales por accidente
- Mezcla la "intención semántica" con la "verdad transaccional" de forma incorrecta

El MCP introduce tres disciplinas que los prompts de IA por sí solos no pueden garantizar:

1. **La elección de la herramienta se vuelve explícita**: En lugar de que el modelo decida silenciosamente, debe elegir de un conjunto de herramientas definido:
   - `api.sf_chat(p_question, p_k)` (recuperación semántica y narración)
   - `api.sf_dynamic_chat(p_question, p_row_limit)` (generación de SQL protegida y ejecución segura)
   - `api.sf_hybrid_chat(p_question, p_max_price, p_k)` (semántica primero, restricciones SQL segundo)
2. **Las entradas se tipan y validan**: Una herramienta es una llamada a una función con argumentos tipados (`p_question: text`, `p_k: int`, `p_max_price: numeric`, `p_row_limit: int`).
3. **Las políticas se vuelven exigibles**: Rol de base de datos de solo lectura, esquemas en lista de permitidos, límites de filas forzados, tiempos de espera de declaraciones, verificaciones de permisos específicas de herramientas y registros auditables.

> [!IMPORTANT]
> **Un prompt es un consejo. Un protocolo es gobernanza.**

*Figura 20.2: Yuxtaposición de RAG y MCP*

La Figura 20.2 compara RAG y MCP. Mientras que RAG tradicional aporta contexto no estructurado al prompt del modelo, el asistente de PostgreSQL aporta evidencia estructurada en filas con verdad relacional garantizada por la base de datos, y MCP proporciona la capa contractual para que el modelo actúe únicamente mediante herramientas gobernadas.

---

### Diseño de herramientas listas para MCP para PostgreSQL

Para que una función de PostgreSQL se comporte como una herramienta MCP adecuada, debe satisfacer cuatro reglas:

- **Regla 1: Entradas estables y tipadas** (`p_question text`, `p_k int`, `p_row_limit int`, `p_max_price numeric`).
- **Regla 2: Salidas delimitadas** (límite estricto de cuántas filas se devuelven y qué columnas).
- **Regla 3: Respuestas basadas en evidencia primero** (filas estructuradas como evidencia antes de narrar).
- **Regla 4: Auditabilidad** (registrar qué pregunta se hizo, qué herramienta se usó, qué SQL se ejecutó, duración y filas).

#### Hacer que api.sf_similar_items esté lista para MCP

Separamos el cálculo del *embedding* de la ejecución de la búsqueda y aplicamos límites explícitos de filas (`p_k` acotado entre 1 y 50):

```sql
CREATE OR REPLACE FUNCTION api.sf_similar_items_v2( p_query_text text DEFAULT NULL, p_qvec_in vector(1536) DEFAULT NULL, p_k int DEFAULT 10 ) RETURNS TABLE( product_id int, name text, category text, shortdescription text, longdescription text, price numeric, distance double precision ) LANGUAGE plpgsql AS $$ DECLARE v_qvec vector(1536); p_k_safe int; BEGIN p_k_safe := GREATEST(1, LEAST(p_k, 50)); -- hard cap IF p_qvec_in IS NOT NULL THEN v_qvec := p_qvec_in; ELSE IF p_query_text IS NULL OR btrim(p_query_text) = '' THEN RAISE EXCEPTION 'p_query_text or p_qvec_in must be provided'; END IF; SELECT embeddings.sf_get_query_embedding(p_query_text)::vector(1536) INTO v_qvec; END IF; RETURN QUERY WITH res AS MATERIALIZED ( SELECT p.id AS product_id, p.label::text AS name, c.label::text AS category, p.shortdescription::text AS shortdescription, p.longdescription::text AS longdescription, pvp.price AS price, (v_qvec <=> pe.embedding) AS distance FROM product.product p JOIN embeddings.product_embedding pe ON p.id = pe.product_id JOIN product.category c ON c.id = p.category_id JOIN product.product_variant pv ON pv.product_id = p.id JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true ORDER BY pe.embedding <=> v_qvec ) SELECT * FROM res LIMIT p_k_safe; END; $$;
```

Uso con la caché de consultas:

```sql
WITH q AS ( SELECT embeddings.sf_get_query_embedding('looking for tailored clothing') AS qvec ) SELECT * FROM q, api.sf_similar_items_v2(NULL, q.qvec, 5);
```

#### Adición de un registro de auditoría

Creamos una tabla de auditoría para registrar cada interacción:

```sql
CREATE TABLE IF NOT EXISTS api.chat_audit_log ( id bigserial PRIMARY KEY, created_at timestamptz NOT NULL DEFAULT now(), tool_used text NOT NULL, question text NOT NULL, sql_used text, row_count int, success boolean NOT NULL DEFAULT true, error text );
```

Registro de llamada:

```sql
INSERT INTO api.chat_audit_log(tool_used, question, sql_used, row_count, success) VALUES ('dynamic_sql', p_question, v_sql, jsonb_array_length(v_rows), true);
```

---

### Implementación del enrutamiento de herramientas MCP para nuestro asistente de IA de PostgreSQL

A continuación se muestra la función de enrutador lista para MCP que clasifica la consulta, aplica límites de seguridad y registra la auditoría:

```sql
DROP FUNCTION IF EXISTS api.sf_route_chat( p_question text, p_k integer, p_row_limit integer ); CREATE OR REPLACE FUNCTION api.sf_route_chat( p_question text, p_k integer DEFAULT 10, p_row_limit integer DEFAULT 20 ) RETURNS TABLE ( tool_used text, assistant_text text, rows jsonb, sql_used text ) LANGUAGE plpgsql VOLATILE AS $function$ DECLARE v_q text := lower(coalesce(p_question, '')); v_has_constraints boolean; v_has_intent boolean; BEGIN v_has_constraints := v_q ~ ( '\yunder\y|\yprice\y|\$|\ycheapest\y|\ymost expensive\y|\ycount\y|\yavg\y|' || '\ygroup\y|\ybrand\y|\ycategory\y|\ycolor\y|\ysize\y' ); v_has_intent := v_q ~ ( '\ylike\y|\ysimilar\y|\yrecommend\y|\ytailored\y|\yformal\y|\ycasual\y|\yvibe\y' ); --- Hybrid (constraints + intent) IF v_has_constraints AND v_has_intent THEN tool_used := 'hybrid'; sql_used := NULL; SELECT h.assistant_text, h.rows INTO assistant_text, rows FROM api.sf_hybrid_chat(p_question, NULL, greatest(1, least(p_k, 30))) h; --- One audit row per successful call INSERT INTO api.chat_audit_log (tool_used, question, sql_used, row_count, success) VALUES ('hybrid', p_question, NULL, jsonb_array_length(coalesce(rows, '[]'::jsonb)), true); RETURN NEXT; RETURN; END IF; --- Dynamic SQL (constraints) IF v_has_constraints THEN tool_used := 'dynamic_sql'; SELECT d.assistant_text, d.rows, d.sql_used INTO assistant_text, rows, sql_used FROM api.sf_dynamic_chat(p_question, greatest(1, least(p_row_limit, 50))) d; --- One audit row per successful call INSERT INTO api.chat_audit_log (tool_used, question, sql_used, row_count, success) VALUES ( 'dynamic_sql', p_question, sql_used, jsonb_array_length(coalesce(rows, '[]'::jsonb)), true ); RETURN NEXT; RETURN; END IF; --- Semantic default tool_used := 'semantic'; sql_used := NULL; SELECT c.assistant_text, c.rows INTO assistant_text, rows FROM api.sf_chat(p_question, greatest(1, least(p_k, 30))) c; --- One audit row per successful call INSERT INTO api.chat_audit_log (tool_used, question, sql_used, row_count, success) VALUES ('semantic', p_question, NULL, jsonb_array_length(coalesce(rows, '[]'::jsonb)), true); RETURN NEXT; END; $function$;
```

Ejecución semántica/híbrida:

```sql
SELECT tool_used, assistant_text FROM api.sf_route_chat('looking for tailored clothing', 2, 2);
```

```text
-[ RECORD 1 ]--+------------------------------------------------------------------------------------------------------- tool_used | hybrid assistant_text | You asked for tailored clothing, and I found 2 results for you. | | 1. **Suit coat - business perfect, by Brioni** | - **Price:** $98.43 | - **Category:** Pants | - **Description:** Classic tailored suit coat - premium wool, | sharp silhouette, versatile elegance... | Both items are the same, featuring a refined and versatile design | suitable for various occasions.
```

Ejecución de SQL dinámico:

```sql
SELECT tool_used, sql_used, assistant_text FROM api.sf_route_chat('Show me polo shirts under $80', 2, 2);
```

```text
-[ RECORD 1 ]--+---------------------------------------------------------------------------------------------------- tool_used | dynamic_sql sql_used | WITH polo_shirts AS ( | SELECT p.id, p.label, c.label AS category, pvp.price, | pv.attributes + | FROM product.product p | JOIN product.category c ON p.category_id = c.id | JOIN product.product_variant pv ON p.id = pv.product_id | JOIN product.product_variant_price pvp ON | pvp.product_variant_id = pv.id AND pvp.current = true+ | WHERE LOWER(p.label) LIKE '%polo%' AND pvp.price < 80 | ) | SELECT id, label, category, price, attributes | FROM polo_shirts | LIMIT 2 assistant_text | You asked to see polo shirts under $80. I found 2 results for you. | | 1. **Label**: Tommy Hilfiger Polo Shirt | **Price**: $31.72 | **Category**: Shirts | **Attributes**: Size - M, Color - Navy, Style - Classic Polo | | 2. **Label**: Tommy Hilfiger Polo Shirt | **Price**: $31.72 | **Category**: Shirts | **Attributes**: Size - L, Color - White, Style - Classic Polo
```

---

### Adición de controles de producción para herramientas MCP

#### Barrera de seguridad 1: Usar un rol de solo lectura para la ejecución de herramientas

```sql
-- A role used by the MCP server / assistant runtime CREATE ROLE mcp_assistant NOINHERIT; -- Allow it to connect GRANT CONNECT ON DATABASE aidb TO mcp_assistant; -- Allow usage on schemas it needs GRANT USAGE ON SCHEMA api, product, embeddings TO mcp_assistant; -- Allow SELECT from required tables GRANT SELECT ON product.category, product.brand, product.product, product.product_variant, product.product_variant_price, embeddings.product_embedding TO mcp_assistant; -- Allow execute on the specific functions we expose as tools GRANT EXECUTE ON FUNCTION api.sf_chat(text,int), api.sf_similar_items(text,int), api.sf_route_chat(text,int,int), api.sf_dynamic_chat(text,int) TO mcp_assistant;
```

#### Barrera de seguridad 2: Forzar tiempos de espera

A nivel de rol:

```sql
ALTER ROLE mcp_assistant SET statement_timeout = '2s';
```

A nivel de función:

```sql
PERFORM set_config('statement_timeout', '2000ms', true);
```

#### Barrera de seguridad 3: Límites estrictos de filas

Aplicar `LEAST(p_row_limit, 50)` en todas las funciones.

#### Barrera de seguridad 4: Restringir SQL dinámico con un validador

```sql
CREATE OR REPLACE FUNCTION api.sf_is_safe_select( p_sql text ) RETURNS boolean LANGUAGE plpgsql STABLE AS $function$ DECLARE v_sql_lc text := lower(coalesce(p_sql, '')); BEGIN --- Block empty IF btrim(v_sql_lc) = '' THEN RETURN false; END IF; --- Only one statement IF position(';' IN v_sql_lc) > 0 THEN RETURN false; END IF; --- Must start with SELECT or WITH IF v_sql_lc !~ '^\s*(select|with)\y' THEN RETURN false; END IF; --- Block dangerous keywords IF v_sql_lc ~ '\y(insert|update|delete|drop|alter|create|truncate|grant|revoke|copy|call|do)\y' THEN RETURN false; END IF; --- Restrict schemas (adjust as needed) IF v_sql_lc ~ '\y(pg_catalog|information_schema)\y' THEN RETURN false; END IF; RETURN true; END; $function$;
```

#### Barrera de seguridad 5: Controles de costos para llamadas a embeddings

```sql
IF length(btrim(p_question)) < 3 THEN RAISE EXCEPTION 'Query too short to embed safely'; END IF;
```

```sql
SELECT embeddings.sf_get_query_embedding(p_question) INTO qvec;
```

#### Barrera de seguridad 6: Devolver siempre filas de evidencia mediante vistas seguras

Creamos una vista curada en el esquema `api` para que las herramientas no consulten tablas sin procesar:

```sql
CREATE OR REPLACE VIEW api.product_search_vw AS SELECT p.id AS product_id, p.label AS product_name, c.label AS category, b.label AS brand, p.shortdescription AS shortdescription, p.longdescription AS longdescription, pv.id AS product_variant_id, pv.attributes AS attributes, pvp.price AS price FROM product.product p JOIN product.category c ON c.id = p.category_id JOIN product.brand b ON b.id = p.brand_id JOIN product.product_variant pv ON pv.product_id = p.id JOIN product.product_variant_price pvp ON pvp.product_variant_id = pv.id AND pvp.current = true;
```

---

### Listas de verificación operativas y ruta de extensión

#### Lista de verificación operativa: Ejecución exitosa del chatbot de IA

- **Corrección**:
  - El asistente solo debe responder utilizando filas de evidencia devueltas por SQL.
  - Si las filas están vacías, debe decir que no encontró nada, sin improvisar.
  - Mantener formas de respuesta estables (`assistant_text`, `rows`, `sql_used`).
  - Estandarizar nombres y orden de columnas en vistas.
- **Rendimiento**:
  - Imponer límites de filas en cada ruta de herramienta.
  - Usar tiempos de espera (`statement_timeout`) para los roles de ejecución.
  - Indexar las tablas utilizadas (`pgvector` HNSW/IVFFlat en `embeddings.product_embedding`).
- **Seguridad**:
  - Ejecutar herramientas bajo un rol dedicado de solo lectura (`mcp_assistant`).
  - Validar SQL dinámico (`SELECT`-only, sin puntos y coma, sin palabras clave peligrosas).
  - Consultar vistas `api.*_vw` en lugar de tablas sin procesar.
- **Observabilidad**:
  - Devolver siempre `tool_used` y `sql_used`.
  - Registrar llamadas en `api.chat_audit_log` (marca de tiempo, herramienta, pregunta, recuento de filas, latencia, error).
- **Higiene de seguridad**:
  - Rotar la clave de API periódicamente y evitar codificarla de forma fija.
  - Considerar controles de salida de red si la base de datos realiza llamadas a APIs externas.

#### Ruta de extensión

En una implementación típica, la pregunta del usuario fluye desde la interfaz de usuario de la aplicación a un servidor MCP en la capa de aplicación. Ese servidor MCP selecciona la herramienta adecuada del registro de herramientas, impone barreras de seguridad (tiempos de espera y límites de filas) y luego llama a PostgreSQL utilizando un rol restringido. PostgreSQL devuelve resultados estructurados (filas de evidencia y narrativa del asistente) y el servidor MCP registra la llamada a la herramienta antes de devolver la respuesta final al usuario.

---

### Resumen

En este capítulo, llevamos a nuestro asistente de base de datos desde "funciona" hasta "se mantiene sólido en producción".

Construimos un flujo de trabajo de chatbot práctico sobre la misma base de PostgreSQL y `pgvector` desarrollada a lo largo del libro: *embeddings* almacenados junto con la verdad relacional, búsqueda de similitud realizada con SQL estándar y respuestas fundamentadas en filas de evidencia en lugar de conjeturas libres. Con `api.sf_similar_items`, convertimos el lenguaje natural en un *embedding* de consulta con `api.openai_embed`, recuperamos los productos más cercanos de `embeddings.product_embedding` y devolvimos resultados estructurados. Luego agregamos una capa de interpretación con `api.sf_answer_with_openai`, que convierte las filas en una respuesta legible y amigable, manteniéndose explícitamente restringida a lo que la base de datos realmente produjo.

A partir de ahí, nos centramos en la forma de producción del sistema: tratamos las funciones de la base de datos como herramientas con contratos (entradas estables, salidas estables, límites predecibles y un registro de auditoría claro). Introdujimos la idea de una superficie de API curada utilizando vistas `api.*_vw` para que el asistente consulte solo lo que expones intencionalmente. Reforzamos las disciplinas esenciales de producción: corrección, rendimiento, seguridad, observabilidad e higiene de seguridad.

Finalmente, enmarcamos cómo estas ideas se alinean de forma natural con el pensamiento de MCP: las herramientas no son "llamadas ingeniosas", son interfaces gobernadas. El resultado es una arquitectura de chatbot que preserva las fortalezas de PostgreSQL (determinismo, gobernanza y componibilidad) al tiempo que agrega una interfaz conversacional consciente del significado que permanece fundamentada en datos y es operativamente confiable.

En el próximo capítulo, nos alejaremos para conectar los puntos a lo largo de todo el viaje. Revisarás las ideas centrales de las cargas de trabajo transaccionales, analíticas y de IA, y verás cómo PostgreSQL las une en una sola plataforma unificada.
