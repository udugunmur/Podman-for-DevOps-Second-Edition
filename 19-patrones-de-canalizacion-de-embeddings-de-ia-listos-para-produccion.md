## Capítulo 19: Patrones de Canalización de Embeddings de IA Listos para Producción

Hasta ahora, hemos estado en una pequeña aventura decidida: una parte de matemáticas, una parte de significado y una parte de *"Espera, ¿mi base de datos puede hacer eso?"*.

En el [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas*, aprendimos que los vectores son contenedores de significado: huellas digitales matemáticas que permiten a los sistemas comparar la "intención" en lugar de simplemente hacer coincidir palabras clave. Fuimos desde el comienzo más simple posible, creando un *embedding* con cURL, pasando a un script de Python y luego adentrándonos directamente en PostgreSQL con `api.openai_embed`. En el camino, lo hicimos confiable con reintentos acotados, retroceso exponencial y fluctuación (*jitter*), y lo escalamos con una función de *embedding* por lotes que llena de manera segura las tablas de *embeddings* a partir de datos reales de productos. Lo más importante es que cerramos el círculo utilizando esos *embeddings* para la búsqueda semántica y patrones de recomendación, exactamente como se introdujo anteriormente en el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*.

Ahora estamos listos para el siguiente salto: convertir la generación de *embeddings* en un sistema real, no solo en una demostración funcional.

La producción tiene sus propias leyes. En producción, la pregunta no es "¿puedo generar *embeddings*?". La pregunta en producción es una de las siguientes:

- ¿Cuándo los genero, en el momento de la escritura o más tarde?
- ¿Cómo los mantengo actualizados cuando los datos cambian?
- ¿Cómo hago esto de manera eficiente sin disparar la factura de mi LLM por las nubes?
- ¿Dónde ejecuto el trabajo para que mi base de datos no quede como rehén de la latencia de la API?
- ¿Qué sucede cuando alcanzo los límites de tasa o experimento una latencia de red inesperada? ¿Falla un trabajo a mitad de camino?
- ¿Qué sucede cuando cambiamos de LLM, se actualiza la versión del LLM, se aplican cambios de modelo, se alcanzan los límites de tasa o los trabajos fallan a mitad de camino?

En este capítulo, explicaremos lo siguiente:

- Por qué necesitamos una arquitectura de canalización (*pipeline*) para crear *embeddings* en producción
- Cómo construir una canalización robusta
- Los requisitos para una canalización de *embeddings* de IA
- Opciones de arquitectura y principios de diseño
- Cómo hacer una canalización casi en tiempo real para minimizar el retraso del *embedding*
- Por qué una caché de *embeddings* puede ayudar a acelerar el sistema y reducir costos

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Requisitos técnicos

Ilustraremos estos conceptos utilizando la API de OpenAI, que ha sido adoptada por otros proveedores de LLM, incluidos DeepSeek, Meta AI y Mistral. Para seguir los ejemplos de este capítulo, necesitarás una clave de API de OpenAI.

El código de muestra para este capítulo se puede encontrar en `psql_scripts/sample_scripts/chapter_19.sql`. Para acceder al enlace del repositorio, sigue los pasos en la sección "Descarga de archivos de código de ejemplo" en el Prefacio. ¡Te animamos a seguir los ejemplos!

---

### ¿Por qué necesitamos una canalización?

En el [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas*, aprendimos a generar *embeddings*, almacenarlos en PostgreSQL y utilizarlos inmediatamente para la búsqueda semántica y las recomendaciones. Ese fue el "camino feliz": entra un texto, sale un *embedding* y `pgvector` hace que el significado sea buscable. Este camino feliz utilizó un enfoque estrechamente acoplado, donde la base de datos esperaba a que se generara el *embedding* de IA. Sin embargo, ese enfoque tendría un impacto significativo en el rendimiento y la estabilidad de la base de datos en producción.

Los sistemas de producción no viven en el camino feliz. Tienen que manejar complejidades del mundo real y contingencias de red, especialmente al interactuar con un LLM alojado en la nube, en otra zona de disponibilidad o en otro centro de datos. Para casos de uso de producción, recomendamos un enfoque de canalización que desacople el sistema transaccional de la generación de *embeddings* y garantice que el sistema transaccional no se ralentice en caso de latencia, desconexiones temporales de red, exceso de cuotas o fallas en la API del LLM.

Las canalizaciones son un patrón arquitectónico bien establecido que ayuda a crear sistemas débilmente acoplados pero confiables, lo cual es extremadamente importante en la computación distribuida, donde tenemos que lidiar con la latencia y las desconexiones temporales. Las canalizaciones aíslan los procesos de negocio y aseguran que las ralentizaciones de la red u otros problemas no afecten el rendimiento transaccional.

Un ejemplo frecuente es el uso de una canalización para enviar datos transaccionales al almacén de datos (*data warehouse*):

1. El Sistema A (el sistema transaccional) coloca los datos sobre una transacción confirmada en la canalización.
2. El Sistema A marca internamente los datos como "enviados al almacén de datos".
3. El mecanismo de la canalización proporciona una ruta de entrega garantizada y única que asegura que los datos lleguen al almacén de datos, incluso ante fallas o desconexiones intermitentes. La canalización gestiona la latencia, las fallas de API, los reintentos y cualquier limitación de cuota.
4. El almacén de datos recibe los datos y acusa recibo a la canalización.
5. La canalización eliminará los datos de su lista de tareas pendientes.

Por lo tanto, la canalización proporciona un mecanismo de entrega confiable, auditable y desacoplado que separa el sistema transaccional del almacén de datos.

Las canalizaciones de *embeddings* de IA son casos especiales de canalizaciones genéricas, ya que son conscientes de los *embeddings* y están optimizadas para el problema en cuestión.

#### ¿Qué hace especial a la canalización de embeddings de IA?

La canalización de *embeddings* de IA aborda preguntas específicas que van más allá del patrón de canalización estándar:

- ¿Cuándo se debe crear un *embedding*?
- ¿Qué sucede si falla la llamada a la API de *embeddings*?
- ¿Qué pasa si los datos cambian después de haber generado el *embedding*?
- ¿Cómo regeneramos los *embeddings* cuando cambia el LLM?
- ¿Cómo garantizamos que no estamos sirviendo un significado obsoleto?

Si la arquitectura ignora estas preguntas, la capa semántica se vuelve poco confiable. La base de datos aún puede ser correcta en un sentido transaccional, pero la experiencia de IA se desviará: silenciosamente, lentamente y luego, de repente, de golpe. Los resultados de búsqueda se sienten extraños. Las recomendaciones se vuelven repetitivas o irrelevantes. Los usuarios pierden la confianza y el sistema se vuelve difícil de depurar porque nada está obviamente roto; solo el significado es incorrecto.

Este capítulo aborda este problema.

Trataremos la generación de *embeddings* de IA como un flujo de trabajo de ingeniería de datos de primer nivel y mostraremos cómo construirlo con la misma disciplina que aplicamos al procesamiento de transacciones: consistencia, observabilidad, idempotencia y recuperación.

---

### ¿Cuáles son los requisitos de una canalización de IA?

Los requisitos estándar para la comunicación entre sistemas desacoplados y distribuidos son los siguientes:

- Desacoplar los sistemas de origen y destino
- Manejar con elegancia la latencia de red, las desconexiones temporales y las fallas de API
- Proporcionar consistencia, recuperación, seguridad, observabilidad e idempotencia

Los requisitos específicos para *embeddings* de IA añaden necesidades adicionales:

- Decidir cuándo se debe generar el *embedding*
- Manejar la regeneración inteligente de *embeddings* cuando cambian los datos subyacentes, incluyendo evitar la generación innecesaria de *embeddings*
- Gestionar actualizaciones de LLM y cambios de proveedor de LLM

#### Un modelo mental: Dos velocidades de la verdad

Para razonar sobre las canalizaciones de *embeddings*, resulta útil separar el sistema en dos capas de verdad: operativa y semántica.

##### Verdad operativa (rápida, exacta)

Esta es la realidad de la base de datos transaccional, que en nuestro conjunto de datos de muestra está representada por lo siguiente:

- Producto, variante de producto, marca y país de origen
- Precio de la variante de producto
- Inventario
- Categoría

Debe ser correcta de inmediato. Si esta capa miente, el dinero se ve afectado.

##### Verdad semántica (significativa, pero no siempre inmediata)

Esta es la realidad de la IA que representa el significado de los elementos descriptivos como vectores:

- Vectores de producto, categoría y marca
- Vectores de significado de consultas
- Relaciones de similitud

Debe ser correcta eventualmente y, de manera ideal, con rapidez, pero en la mayoría de los casos no debe mantener como rehén al sistema operativo.

Un diseño de producción saludable trata esto como dos velocidades:

- Las actualizaciones operativas son rápidas y deterministas
- Las actualizaciones semánticas siguen de manera rápida, confiable y recuperable

Cuando intentamos forzar que la verdad semántica se comporte como la verdad operativa, pagamos con latencia y fragilidad. Sin embargo, si tratamos la verdad semántica como una idea secundaria casual, entonces pagamos con desvío e irrelevancia. La canalización de *embeddings* equilibra ambas.

#### Tres patrones de canalización

Existen tres patrones prácticos para la generación de *embeddings* en producción: *embedding* sincrónico, *embedding* asincrónico y *embedding* casi en tiempo real. Cubriremos los tres porque diferentes aplicaciones requieren diferentes equilibrios.

##### Embedding sincrónico (al momento de escribir)

El *embedding* se crea inmediatamente cuando se insertan o actualizan los datos:

- **Ideal para**: Bajo volumen de escritura, requisitos estrictos de frescura, idealmente con un LLM alojado localmente
- **Riesgo**: La latencia de la API externa ralentiza las escrituras
- **Modo de falla**: Si la canalización de *embeddings* falla o se ralentiza, la parte operativa se verá afectada

##### Embedding asincrónico (programado en segundo plano)

Las escrituras operativas se mantienen rápidas; el trabajo de *embeddings* ocurre más tarde a través de una cola y un trabajador (*worker*) asincrónico:

- **Ideal para**: Alto volumen de escritura, resiliencia de grado de producción
- **Compromiso**: Consistencia eventual (breves ventanas de "sin *embedding* todavía" o "*embedding* desactualizado")
- **Fortaleza**: Puede reintentar de forma segura sin bloquear las transacciones comerciales

##### Embedding casi en tiempo real (en segundo plano, dirigido por eventos)

Esto es como el modo asincrónico, pero diseñado para una frescura con muy baja latencia:

- **Ideal para**: Búsquedas orientadas al usuario donde el contenido nuevo debe aparecer rápidamente
- **Implementación**: `NOTIFY`/`LISTEN`, transmisión (*streaming*) o sondeo rápido del trabajador
- **Restricción**: Aún debe ser duradero (una tabla de cola sigue siendo la verdad)

Podemos pensar en estos tres patrones como tres marchas o velocidades:

- **Marcha 1**: Sincrónica, acoplamiento estrecho, lógica más simple
- **Marcha 2**: Asincrónica, desacoplada, confiable, escalable
- **Marcha 3**: Tiempo real, asincrónica con disciplina de velocidad

Como se mencionó anteriormente, el primer patrón que utiliza *embedding* sincrónico no es adecuado para sistemas de producción distribuidos y de alto volumen. En la siguiente sección, comenzaremos con la alternativa más adecuada para producción: la generación asincrónica de *embeddings* utilizando un programador en segundo plano. También explicaremos brevemente cómo convertir esa arquitectura en un enfoque casi en tiempo real dirigido por eventos.

---

### Construcción de una canalización asincrónica, de alto rendimiento y confiable

Veamos ahora cómo diseñar una canalización lista para producción que mantenga la generación de *embeddings* separada de las escrituras en la base de datos.

#### Principio de diseño 1: Los disparadores deben encolar, no incrustar

Una tentación común es iniciar la generación del *embedding* directamente dentro de un disparador (*trigger*). Cuando los datos se insertan o actualizan, se activa inmediatamente la llamada al LLM y se almacena el *embedding*.

Esto funciona en una demostración. En producción, se convierte en una interrupción autoinducida, ya que en SQL, el disparador es parte de la transacción de escritura inicial, que no se confirmará (*commit*) hasta que termine el disparador.

Los disparadores SQL deben ser rápidos y deterministas. No deberían hacer lo siguiente:

- Llamar a servicios externos
- Esperar en redes
- Reintentar con retroceso
- Competir con las escrituras OLTP por tiempo y recursos

Por lo tanto, adoptamos un principio estricto: Un disparador debe registrar la intención (esta fila necesita un *embedding*) y un trabajador asincrónico realizará el trabajo costoso.

Los disparadores pueden registrar la necesidad de generar un *embedding* de dos maneras:

- Establecer una bandera `needs_embedding` en la fila (simple)
- Insertar un trabajo en una cola de *embeddings* dedicada (más escalable y observable)

Utilizaremos el segundo enfoque porque facilita la operación en producción: podemos ver el trabajo acumulado (*backlog*), observar fallas de reintento y determinar si necesitamos regular la carga de trabajo o aumentar la capacidad.

#### Principio de diseño 2: La cola de trabajos de embedding

Una cola de *embeddings* es simplemente una tabla que rastrea qué trabajo de *embedding* debe completarse y qué sucedió cuando el sistema intentó generar el *embedding*.

Conceptualmente, cada trabajo necesita capturar lo siguiente:

- ¿Qué entidad cambió (por ejemplo, `product`, `brand`, `category` o `product_variant`)?
- ¿Qué fila cambió?
- ¿Qué tabla de destino de *embeddings* debe poblar?
- ¿Cuál es el estado del *embedding*: `pending`/`running`/`done`/`failed`?
- Contador de reintentos y el último error
- Marcas de tiempo para auditoría y monitoreo

Una tabla de cola es aburrida, y por eso funciona. Proporciona lo siguiente:

- **Durabilidad** (los trabajos no se pierden)
- **Observabilidad** (puedes inspeccionar el trabajo acumulado y las fallas)
- **Control** (puedes regular a los trabajadores y los tamaños de lote)
- **Idempotencia** (puedes reintentar de forma segura)

En la siguiente sección, definiremos un esquema de cola mínimo y mostraremos cómo conectarlo mediante disparadores para la tabla `product.product`, basándonos directamente en las funciones de *embeddings* definidas en el [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas*.

#### Una canalización asincrónica mínima para productos

Ahora construiremos la canalización más pequeña con forma de producción que se basa en el Principio de diseño 1 (los disparadores deben encolar y no incrustar) y el Principio de diseño 2 (cola de trabajos de *embedding*):

1. Un disparador detecta inserciones/actualizaciones en `product.product`.
2. El disparador encola un trabajo en `embeddings.embedding_job`.
3. Una función trabajadora procesa los trabajos en lotes:
   - Construye la cadena de significado del producto
   - Llama a `api.openai_embed(...)`
   - Almacena el *embedding* en `embeddings.product_embedding`
4. El trabajador actualiza el estado del trabajo y registra los errores de manera segura.

Al final de esta sección, tendremos una canalización que hace lo siguiente:

- No ralentiza las escrituras
- Se recupera de errores transitorios de la API
- Admite procesamiento por lotes y regulación (*throttling*)
- Crea *embeddings* de manera confiable para tablas reales

Y una vez que existe esta base, agregar categorías, marcas y variantes de productos se convierte en un patrón repetible en lugar de una nueva invención cada vez.

Para nuestro ejemplo, queremos una arquitectura donde se aplique lo siguiente:

- Las inserciones/actualizaciones en `product.category`, `product.brand`, `product.product` y `product.product_variant` no llaman directamente a OpenAI. En su lugar, encolan un trabajo.
- Un trabajador asincrónico procesa los trabajos y escribe los *embeddings* en:
  - `embeddings.product_category_embedding`
  - `embeddings.product_brand_embedding`
  - `embeddings.product_embedding`
  - `embeddings.product_variant_embedding`

Este diseño mantiene OLTP rápido y traslada el trabajo "costoso, en red y desacoplado" que genera los *embeddings* a un paso controlado en segundo plano.

##### La tabla de cola de trabajos de embedding

Agregamos una pequeña tabla de cola en el esquema `embeddings`. Rastrea qué entidad cambió, qué fila y qué sucedió a continuación:

```sql
CREATE TABLE IF NOT EXISTS embeddings.embedding_job (
  id            bigserial PRIMARY KEY,
  entity_type   text NOT NULL CHECK (entity_type IN ('category','brand','product','variant')),
  entity_id     integer NOT NULL,
  status        text NOT NULL DEFAULT 'pending'
                CHECK (status IN ('pending','running','done','failed')),
  attempts      int  NOT NULL DEFAULT 0,
  last_error    text,
  created_at    timestamptz NOT NULL DEFAULT now(),
  updated_at    timestamptz NOT NULL DEFAULT now()
);

CREATE INDEX IF NOT EXISTS embedding_job_pending_idx
  ON embeddings.embedding_job (status, created_at);

CREATE UNIQUE INDEX IF NOT EXISTS embedding_job_entity_idx
  ON embeddings.embedding_job (entity_type, entity_id);
```

Esto es suficiente para una solución mínima debido a lo siguiente:

- `entity_type` y `entity_id` nos dicen qué incrustar
- `status`/`attempts`/`last_error` brindan visibilidad operativa
- El índice único en `(entity_type, entity_id)` es necesario para la deduplicación; las funciones de encolado/actualización posteriores dependen de esta restricción para garantizar que solo exista un trabajo pendiente por entidad a la vez
- Es duradero, consultable y reintentable

##### Asistente de encolado: Una función para todos los disparadores

```sql
CREATE OR REPLACE FUNCTION embeddings.sf_enqueue_embedding_job(p_entity_type text, p_entity_id int)
RETURNS void
LANGUAGE plpgsql
AS $$
BEGIN
  -- Avoid enqueuing duplicates when multiple updates happen quickly.
  -- If a job is already pending/running for that entity, we do nothing.
  IF EXISTS (
    SELECT 1
    FROM embeddings.embedding_job
    WHERE entity_type = p_entity_type
      AND entity_id   = p_entity_id
      AND status IN ('pending','running')
  ) THEN
    RETURN;
  END IF;

  INSERT INTO embeddings.embedding_job(entity_type, entity_id)
  VALUES (p_entity_type, p_entity_id);
END;
$$;
```

##### Disparadores: Encolar en inserción/actualización

Ahora conectamos disparadores a tus cuatro tablas de origen. Cada disparador llama al asistente, y eso es todo. Ilustremos esto creando el disparador posterior a la actualización o inserción para categorías:

```sql
CREATE OR REPLACE FUNCTION embeddings.trg_enqueue_category_embedding()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  PERFORM embeddings.sf_enqueue_embedding_job('category', NEW.id);
  RETURN NEW;
END;
$$;

DROP TRIGGER IF EXISTS enqueue_category_embedding ON product.category;

CREATE TRIGGER enqueue_category_embedding
AFTER INSERT OR UPDATE OF label, description
ON product.category
FOR EACH ROW
EXECUTE FUNCTION embeddings.trg_enqueue_category_embedding();
```

Se necesitan disparadores similares para `brand`, `product` y `variant`.

> [!IMPORTANT]
> **Punto clave de producción**  
> Estos disparadores son rápidos y deterministas. Solo escriben una sola fila en una tabla de cola. Es muy poco probable que esto afecte el rendimiento de las transacciones.

#### Principio de diseño 3: El trabajador, procesando trabajos pendientes en lotes

Necesitamos un procesador en segundo plano que procese la cola de trabajos, genere *embeddings* y actualice la cola. Mostraremos cómo hacer esto utilizando una función PL/pgSQL que se puede ejecutar desde un programador del sistema operativo (por ejemplo, `cron`), un programador de PostgreSQL (por ejemplo, `pg_cron`), un trabajador de aplicación o un trabajador en segundo plano.

He aquí un ejemplo de una función trabajadora:

```sql
CREATE OR REPLACE FUNCTION embeddings.sf_process_embedding_jobs(
  p_batch_size integer DEFAULT 50
)
RETURNS integer
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = pg_catalog, embeddings, product, api
AS $function$
DECLARE
  v_job        embeddings.embedding_job%ROWTYPE;
  v_processed  integer := 0;
  v_input_text text;
  v_vec        public.vector(1536);
BEGIN
  FOR v_job IN
    SELECT ej.*
    FROM embeddings.embedding_job ej
    WHERE ej.status = 'pending'
    ORDER BY ej.created_at
    LIMIT p_batch_size
    FOR UPDATE SKIP LOCKED
  LOOP
    BEGIN
      -- Mark running
      UPDATE embeddings.embedding_job ej
      SET status     = 'running',
          attempts   = ej.attempts + 1,
          updated_at = now(),
          last_error = NULL
      WHERE ej.id = v_job.id;

      -- Build input text and upsert the embedding into the correct table.
      IF v_job.entity_type = 'category' THEN
        SELECT coalesce(c.label, '') || ' ' || coalesce(c.description, '')
        INTO v_input_text
        FROM product.category c
        WHERE c.id = v_job.entity_id;

        v_vec := api.openai_embed(v_input_text)::public.vector(1536);

        INSERT INTO embeddings.product_category_embedding (product_category_id, embedding)
        VALUES (v_job.entity_id, v_vec)
        ON CONFLICT (product_category_id)
        DO UPDATE
          SET embedding = EXCLUDED.embedding;

      ELSIF v_job.entity_type = 'brand' THEN
        SELECT coalesce(b.label, '') || ' ' || coalesce(b.description, '')
        INTO v_input_text
        FROM product.brand b
        WHERE b.id = v_job.entity_id;

        v_vec := api.openai_embed(v_input_text)::public.vector(1536);

        INSERT INTO embeddings.product_brand_embedding (product_brand_id, embedding)
        VALUES (v_job.entity_id, v_vec)
        ON CONFLICT (product_brand_id)
        DO UPDATE
          SET embedding = EXCLUDED.embedding;

      ELSIF v_job.entity_type = 'product' THEN
        SELECT
          coalesce(p.label, '') || ' ' ||
          coalesce(b.label, '') || ' ' ||
          coalesce(c.label, '') || ' ' ||
          coalesce(p.shortdescription, '') || ' ' ||
          coalesce(p.longdescription, '')
        INTO v_input_text
        FROM product.product p
        JOIN product.brand b
          ON b.id = p.brand_id
        JOIN product.category c
          ON c.id = p.category_id
        WHERE p.id = v_job.entity_id;

        v_vec := api.openai_embed(v_input_text)::public.vector(1536);

        INSERT INTO embeddings.product_embedding (product_id, embedding)
        VALUES (v_job.entity_id, v_vec)
        ON CONFLICT (product_id)
        DO UPDATE
          SET embedding = EXCLUDED.embedding;

      ELSIF v_job.entity_type = 'variant' THEN
        SELECT coalesce(v.attributes::text, '')
        INTO v_input_text
        FROM product.product_variant v
        WHERE v.id = v_job.entity_id;

        v_vec := api.openai_embed(v_input_text)::public.vector(1536);

        INSERT INTO embeddings.product_variant_embedding (product_variant_id, embedding)
        VALUES (v_job.entity_id, v_vec)
        ON CONFLICT (product_variant_id)
        DO UPDATE
          SET embedding = EXCLUDED.embedding;

      ELSE
        RAISE EXCEPTION 'Unknown entity_type: %', v_job.entity_type
          USING ERRCODE = '22023';
      END IF;

      -- Mark done
      UPDATE embeddings.embedding_job ej
      SET status     = 'done',
          updated_at = now()
      WHERE ej.id = v_job.id;

      v_processed := v_processed + 1;

    EXCEPTION WHEN OTHERS THEN
      -- Mark failed but keep the job for retry/inspection
      UPDATE embeddings.embedding_job ej
      SET status     = 'failed',
          last_error = SQLERRM,
          updated_at = now()
      WHERE ej.id = v_job.id;
    END;
  END LOOP;

  RETURN v_processed;
END;
$function$;
```

##### Recorrido paso a paso del trabajador

Sigue estos pasos:

1. **Acepta un tamaño de lote (por defecto 50)**:

```sql
SELECT embeddings.sf_process_embedding_jobs(50);
```

Así, puedes ejecutarlo en fragmentos pequeños (bueno para control y costos) o fragmentos más grandes (bueno para rendimiento).

2. **Selecciona trabajos "pendientes", los más antiguos primero, y los bloquea de forma segura**:

```sql
    SELECT ej.*
    FROM embeddings.embedding_job ej
    WHERE ej.status = 'pending'
    ORDER BY ej.created_at
    LIMIT p_batch_size
    FOR UPDATE SKIP LOCKED
```

Este es un patrón silenciosamente poderoso:

- `FOR UPDATE` bloquea las filas seleccionadas
- `SKIP LOCKED` significa que si otro trabajador ya bloqueó un trabajo, este trabajador lo salta en lugar de esperar

Esto significa que puedes ejecutar múltiples instancias de esta función simultáneamente (trabajos cron, trabajadores en segundo plano, múltiples servidores de aplicaciones) y no procesarán dos veces el mismo trabajo.

3. **Marca cada trabajo como "en ejecución" e incrementa los intentos**:

```sql
      UPDATE embeddings.embedding_job ej
      SET status     = 'running',
          attempts   = ej.attempts + 1,
          updated_at = now(),
          last_error = NULL
      WHERE ej.id = v_job.id;
```

Esto crea visibilidad y rendición de cuentas:

- Puedes ver lo que está en curso (`running`)
- Puedes ver los reintentos mediante `attempts`
- Borra el error anterior al reintentar

4. **Construye el texto a incrustar según el propósito del trabajo**. Cada trabajo tiene lo siguiente:
   - `entity_type` (`category`, `brand`, `product` o `variant`)
   - `entity_id` (el ID en la tabla de origen)

Dependiendo del tipo de entidad, selecciona los campos correctos y los concatena en un solo texto de entrada. He aquí algunos ejemplos:

- `category` usa `product.category.label` + `description`
- `brand` usa `product.brand.label` + `description`
- `product` construye una oración más rica a partir de:
  - Etiqueta del producto
  - Etiqueta de la marca
  - Etiqueta de la categoría
  - Descripción corta
  - Descripción larga (esto es inteligente: los *embeddings* tienden a funcionar mejor cuando proporcionas un contexto más completo)
- `variant` incrusta `attributes::text` (JSONB convertido a texto)

También observa el uso de `coalesce(...)` en todas partes: evita que los valores `NULL` rompan la concatenación.

5. **Llama a la API de embeddings y la convierte a `vector(1536)`**:

```sql
        v_vec := api.openai_embed(v_input_text)::vector(1536);
```

Esto significa lo siguiente:

- `api.openai_embed()` devuelve un *embedding* (probablemente como una matriz o tipo similar a un vector)
- La función lo almacena como un valor `pgvector` con una dimensión de 1536

¿Por qué 1536? Esa es la dimensión esperada para el modelo de *embedding* de OpenAI que elegimos para esta aplicación. El concepto importante es que la longitud del vector debe coincidir con la salida de tu modelo.

6. **Escribe el vector en la tabla de embeddings correspondiente (upsert)**. He aquí un ejemplo para `product`:

```sql
        INSERT INTO embeddings.product_variant_embedding
                    (product_variant_id, embedding)
        VALUES (v_job.entity_id, v_vec)
        ON CONFLICT (product_variant_id)
        DO UPDATE
          SET embedding = EXCLUDED.embedding;
```

Este es un patrón idempotente:

- Si el *embedding* no existe, lo inserta
- Si ya existe, lo actualiza

De modo que puedes volver a ejecutar trabajos de forma segura sin crear duplicados.

Utilizamos `ON CONFLICT DO UPDATE` en lugar de `MERGE` ya que es más fácil, más corto y muy rápido. `MERGE` es útil cuando necesitas múltiples cláusulas `WHEN` o una lógica de coincidencia más compleja.

7. **Marca el trabajo como "hecho" e incrementa el contador de procesados**:

```sql
      -- Mark done
      UPDATE embeddings.embedding_job ej
      SET status     = 'done',
          updated_at = now()
      WHERE ej.id = v_job.id;

      v_processed := v_processed + 1;
```

Esta es una resiliencia pragmática:

- No bloquea todo el lote por una sola fila defectuosa
- Captura el error (`SQLERRM`) para un diagnóstico posterior
- El trabajo permanece visible y puede ser reintentado

Esto es lo que hace que esto sea "mínimo" pero con forma de producción:

- `FOR UPDATE SKIP LOCKED` permite una concurrencia segura (múltiples trabajadores pueden ejecutarse sin duplicar el trabajo)
- `ON CONFLICT DO UPDATE` hace que las escrituras de *embeddings* sean idempotentes
- Los trabajos pueden fallar y ser inspeccionados/reintentados más tarde

---

### Principio de diseño 4: Actualizaciones inteligentes para embeddings

Hasta ahora, hemos construido una canalización asincrónica: los disparadores encolan el trabajo y un trabajador genera *embeddings* más tarde. Eso hace que la generación de *embeddings* sea escalable y adecuada para producción.

Ahora nos enfrentamos a un problema que a menudo rompe silenciosamente los sistemas semánticos: el significado obsoleto.

Los datos operativos cambian instantáneamente. Los *embeddings* no, a menos que los actualicemos deliberadamente. Si no lo hacemos, la búsqueda y las recomendaciones se desvían: nada falla, pero la relevancia empeora lentamente. Los sistemas de producción necesitan dos mejoras: reintentos controlados y detección de cambios.

#### Reintentos controlados (resiliencia a nivel de canalización)

Nuestro `api.openai_embed()` ya reintenta fallas transitorias de la API durante una sola llamada. Pero la canalización necesita su propia disciplina de reintentos porque un trabajo puede fallar por otras razones, como límites de tasa, interrupciones transitorias, tiempos de espera y fluctuaciones de red.

Para manejar eso, agregamos dos columnas a la cola:

- `next_run_at` para programar cuándo se debe reintentar el trabajo
- `max_attempts` para evitar reintentos infinitos

```sql
ALTER TABLE embeddings.embedding_job
ADD COLUMN IF NOT EXISTS next_run_at timestamptz NOT NULL DEFAULT now(),
ADD COLUMN IF NOT EXISTS max_attempts int NOT NULL DEFAULT 10;

CREATE INDEX IF NOT EXISTS embedding_job_runnable_idx
  ON embeddings.embedding_job (status, next_run_at, created_at);
```

Luego, actualizamos el trabajador para que solo seleccione trabajos ejecutables:

- El estado es `pending` o `failed`
- `next_run_at <= now()`
- Los intentos aún están por debajo de `max_attempts`

```sql
  FOR v_job IN
    SELECT ej.*
    FROM embeddings.embedding_job ej
    WHERE ej.status IN ('pending','failed')
      AND ej.next_run_at <= now()
      AND ej.attempts < ej.max_attempts
    ORDER BY ej.created_at
    LIMIT p_batch_size
    FOR UPDATE SKIP LOCKED
```

En caso de fallo, actualiza `next_run_at` usando retroceso exponencial más fluctuación (*jitter*). Esto evita "tormentas de reintentos" donde muchos trabajadores atacan la API a la vez:

```sql
      UPDATE embeddings.embedding_job
      SET status     = 'failed',
          last_error = SQLERRM,
          updated_at = now(),
          next_run_at = now()
            + make_interval(secs =>
                LEAST(
                  300,
                  (2 ^ LEAST(
                         (SELECT attempts FROM embeddings.embedding_job
                             WHERE id = v_job.id),
                         10
                       ))::int
                  + (random() * 1.0)
                )
              )
      WHERE id = v_job.id;
```

Esto asegura que los trabajos no se queden atascados y que calculemos una próxima hora de ejecución para evitar tormentas de actualización:

- El retroceso crece a medida que aumentan los intentos: $2^{\text{intentos}}$
- El *jitter* añade aleatoriedad para que una flota de trabajadores no se abalance a la vez

Aquí hay un ejemplo de regla de retroceso (simple y efectiva):

- `delay_seconds = min(300, (2^attempts) + random(0..1))`
- Con un límite máximo de 5 minutos por cordura

Esto nos da una canalización que hace lo siguiente:

- Reintenta automáticamente
- No reintenta para siempre
- Distribuye los reintentos a lo largo del tiempo en lugar de saturar la API

Esa es la diferencia operativa entre "funciona" y "sigue funcionando".

#### Detección de cambios (frescura sin desperdicio)

Si volvemos a generar el *embedding* en cada actualización de fila, desperdiciamos llamadas en cambios que no afectan el significado. El disparador de actualización se activará en cada actualización de fila, incluso si ninguna de las columnas relevantes para el significado semántico se ha visto afectada. Esto no solo aumentará la latencia, sino que también puede generar un costo significativo en el LLM.

Un mejor patrón es construir la "cadena de significado", generar su resumen criptográfico (*hash*) y generar el *embedding* solo si el hash cambió. La extensión `pgcrypto` nos ayuda a generar el hash:

1. Actualizamos las tablas de *embeddings* para almacenar el hash del contenido:

```sql
ALTER TABLE embeddings.product_category_embedding
    ADD COLUMN IF NOT EXISTS content_hash text;
ALTER TABLE embeddings.product_brand_embedding   
    ADD COLUMN IF NOT EXISTS content_hash text;
ALTER TABLE embeddings.product_embedding         
    ADD COLUMN IF NOT EXISTS content_hash text;
ALTER TABLE embeddings.product_variant_embedding 
    ADD COLUMN IF NOT EXISTS content_hash text;
```

2. Añadimos la extensión `pgcrypto`:

```sql
CREATE EXTENSION IF NOT EXISTS pgcrypto;
```

3. Actualizamos al trabajador para que aproveche el hash y sea inteligente con las actualizaciones requeridas:
   - Construir `input_text`.
   - Calcular `new_hash = sha256(input_text)`.
   - Comparar con el hash almacenado.
   - Si coincide, omitir el *embedding* y marcar el trabajo como hecho.
   - Si difiere, llamar a `api.openai_embed()` y actualizar *embedding* + hash.

Esto mantiene los *embeddings* actualizados mientras controla el costo y el rendimiento.

Aquí está el patrón para una entidad (`product`), mostrado como plantilla (no ejecutable tal cual):

```sql
SELECT encode(digest(v_input_text, 'sha256'), 'hex') INTO v_new_hash;

      IF v_job.entity_type = 'category' THEN
        SELECT e.content_hash
          INTO v_old_hash
        FROM embeddings.product_category_embedding e
        WHERE e.product_category_id = v_job.entity_id;

        IF v_old_hash IS NOT NULL AND v_old_hash = v_new_hash THEN
          UPDATE embeddings.embedding_job
          SET status = 'done', updated_at = now()
          WHERE id = v_job.id;

          v_processed := v_processed + 1;
          CONTINUE;
        END IF;

        v_vec := api.openai_embed(v_input_text)::public.vector(1536);

        INSERT INTO embeddings.product_category_embedding
                            (product_category_id, embedding, content_hash)
        VALUES (v_job.entity_id, v_vec, v_new_hash)
        ON CONFLICT (product_category_id)
        DO UPDATE
          SET embedding     = EXCLUDED.embedding,
              content_hash  = EXCLUDED.content_hash;
```

Este único cambio mejora drásticamente lo siguiente:

- **Control de costos** (menos llamadas a la API)
- **Rendimiento** (menos trabajo por lote)
- **Corrección** (las actualizaciones de significado siempre desencadenan una nueva generación de *embeddings*)

También hace que tu canalización sea más fácil de razonar: el trabajo de *embeddings* ocurre solo cuando el contenido semántico realmente cambia.

#### Cambios indirectos: Cuándo las actualizaciones de categoría/marca deben refrescar los productos

La cadena de significado del *embedding* de tu producto incluye etiquetas de marca y categoría (y a menudo sus descripciones). Eso significa que un cambio en `product.category` o `product.brand` puede hacer que los *embeddings* del producto queden obsoletos incluso si `product.product` no cambió.

La solución mínima de producción es encolar en cascada:

- Cuando cambia una categoría, encola el *embedding* de la categoría y encola todos los productos en esa categoría.
- Cuando cambia una marca, encola el *embedding* de la marca y encola todos los productos de esa marca.

Esto mantiene alineada la "verdad del significado" en todas las tablas relacionadas sin forzar llamadas a *embeddings* dentro de los disparadores.

He aquí un ejemplo de un disparador cuando cambia una categoría:

```sql
CREATE OR REPLACE FUNCTION embeddings.sf_trg_enqueue_category_embedding()
RETURNS trigger
LANGUAGE plpgsql
AS $$
BEGIN
  -- enqueue the category itself
  PERFORM embeddings.sf_enqueue_embedding_job('category', NEW.id);

  -- cascade enqueue products referencing this category
  INSERT INTO embeddings.embedding_job(entity_type, entity_id)
  SELECT 'product', p.id
  FROM product.product p
  WHERE p.category_id = NEW.id
  ON CONFLICT DO NOTHING;  -- relies on a unique constraint (entity_type, entity_id) OR similar

  RETURN NEW;
END;
$$;
```

#### Poniéndolo todo junto

Echemos un vistazo rápido a cómo funcionan juntos todos los componentes para crear una canalización asincrónica para generar *embeddings* de información de productos (ver Figura 19.1):

*Figura 19.1: Los componentes de una canalización de IA asincrónica*

> [!NOTE]
> **Descarga las imágenes en color**  
> Tu compra incluye una copia en PDF en color y sin DRM de este libro, ideal para ver imágenes en color, capturas de pantalla y diagramas. Consulta la sección de beneficios gratuitos con tu libro al final del Prefacio para desbloquear tu copia en PDF.

1. Un disparador posterior a la actualización en la tabla `product` agrega productos a la cola de trabajos.
2. Un programador de trabajos llama al trabajador periódicamente.
3. El trabajador revisa la cola para crear un lote de solicitudes de *embedding* que sean nuevas o estén listas para reintentarse, las marca como `'running'` y las marca con `FOR UPDATE SKIP LOCKED` para asegurarse de que múltiples ocurrencias del trabajador no interfieran entre sí. El trabajador ensambla la solicitud de *embedding* (es decir, etiqueta, descripción corta y descripción larga).
4. El trabajador verifica si ese *embedding* se ha generado previamente. Si es así, utiliza ese *embedding* para actualizar la tabla de *embeddings*. Si no, envía una solicitud al LLM para generar un *embedding*.
5. El trabajador llama al LLM para generar un *embedding*.
6. El trabajador actualiza la cola de trabajos para marcar las solicitudes como `'done'` o, si la solicitud falló, incrementa el contador de reintentos. Si la solicitud tuvo éxito, actualiza la tabla de *embeddings* utilizando inserciones/actualizaciones idempotentes (`ON CONFLICT DO UPDATE`) para garantizar que los reintentos sean seguros.

---

### Hacer que la canalización asincrónica sea casi en tiempo real

Los sistemas en tiempo real normalmente agregan una cosa: activaciones dirigidas por eventos. En lugar de que los trabajadores sondeen cada minuto, el sistema avisa al trabajador inmediatamente cuando llega un trabajo. Hay dos formas comunes de hacer esto en entornos de PostgreSQL: una arquitectura de escuchar y notificar (*listen-and-notify*) y un enfoque de microlotes (*micro-batching*) impulsado por un programador.

#### Opción A: LISTEN/NOTIFY (activaciones rápidas)

Un disparador encola el trabajo y también envía una notificación ligera:

```sql
NOTIFY embedding_jobs, 'product:' || NEW.id;
```

Un proceso trabajador escucha en el canal. Cuando recibe una notificación, ejecuta inmediatamente `process_embedding_jobs()`.

Sin embargo, hay una advertencia importante: `NOTIFY` es una señal, no una fuente de verdad transaccional compatible con ACID.

Las notificaciones pueden perderse si el trabajador está inactivo. Por eso la cola duradera sigue siendo la verdad, y `NOTIFY` es solo un "timbre".

#### Opción B: Microlotes dirigidos por programador (predecible y simple)

En lugar de `LISTEN/NOTIFY`, ejecutas al trabajador con frecuencia:

- Cada pocos segundos (o cada 10 segundos)
- En lotes pequeños

Esto es fácil de operar y no requiere un proceso de escucha de larga duración. Intercambia un poco de carga adicional por simplicidad operativa.

El *embedding* en tiempo real a menudo se implementa mejor como microlotes porque hace lo siguiente:

- Suaviza el uso de la API
- Evita patrones de tráfico con picos
- Reduce el riesgo de alcanzar límites de tasa

#### Ejecución de la canalización

Sigue estos pasos:

1. Crear *embeddings* para nuevas filas:

```sql
SELECT embeddings.sf_process_embedding_jobs(50);
```

Ejecútalo repetidamente hasta que devuelva `0`.

2. Inspeccionar lo que está sucediendo:

```sql
SELECT * FROM embeddings.embedding_job WHERE status = 'pending' ORDER BY created_at;
```

3. Aquí hay algunas fallas (con texto de error):

```sql
SELECT id, entity_type, entity_id, attempts, last_error
FROM embeddings.embedding_job
WHERE status = 'failed'
ORDER BY updated_at DESC;
```

En este punto, tenemos una canalización de *embeddings* asincrónica en funcionamiento que:

- Se mantiene consistente con tu esquema (`product.*` + `embeddings.*`)
- No llama a OpenAI dentro de los disparadores
- Escribe *embeddings* en las tablas de *embeddings* correctas
- Admite el procesamiento por lotes y la concurrencia de forma segura

---

### ¿Cómo elijo el modelo de canalización adecuado?

Hablamos de tres modelos de canalización: *embedding* sincrónico, *embedding* asincrónico y *embedding* casi en tiempo real. En la Tabla 19.1, resumimos la orientación para el desarrollador sobre cuándo usar cada modelo.

| Arquitectura de canalización | Pros | Contras | Cuándo usar |
| :--- | :--- | :--- | :--- |
| **Embedding sincrónico** | Muy fácil de construir.<br>Garantiza que los *embeddings* estén sincronizados con los datos SQL. | Frágil y quebradizo.<br>Expone el sistema transaccional a ralentizaciones de la red y fallas de la API. | Prueba de concepto y demostración.<br>Bajo volumen de escritura y actualización.<br>Cuando el LLM está ubicado en la misma red de muy baja latencia.<br>Cuando la actualidad del *embedding* es más importante que el rendimiento transaccional. |
| **Embedding asincrónico** | Desacoplamiento del sistema transaccional de la generación de *embeddings*.<br>Aislamiento de ralentizaciones de red y de posibles fallas de API.<br>Gran observabilidad, auditoría y manejo de errores. | Los *embeddings* pueden estar desincronizados mientras esperan que se ejecute el trabajador programado (consistencia eventual).<br>El procesamiento de IA, como la búsqueda semántica, puede estar desincronizado con los datos transaccionales. | Solución predeterminada para la mayoría de las generaciones de *embeddings* de alto volumen.<br>Ideal cuando el LLM está en la nube, en una red de mayor latencia o en un horario operativo diferente. |
| **Embedding casi en tiempo real** | Mismas ventajas que las opciones de *embedding* asincrónico, más lo siguiente:<br>Los *embeddings* son casi en tiempo real.<br>La generación de *embeddings* ocurre bajo demanda, no según un cronograma. | Mayor complejidad operativa al utilizar el enfoque `LISTEN/NOTIFY`.<br>El uso de `LISTEN/NOTIFY` es más frágil, especialmente durante reinicios del sistema o al recuperarse de fallas. | Cuando los aspectos de IA y los aspectos transaccionales de SQL deben estar estrechamente sincronizados. |

*Tabla 19.1: Guía de decisión para arquitecturas de canalizaciones de embeddings de IA*

Si bien el *embedding* sincrónico es una forma simple de configurar una demostración o prueba de concepto, el *embedding* asincrónico (utilizando disparadores para encolar, una cola para gestionar el trabajo y un trabajador inteligente) es el enfoque recomendado para producción. La conversión a un modelo de *embedding* casi en tiempo real se puede hacer fácilmente si la latencia de *embedding* de los trabajadores programados se vuelve problemática.

El uso de un enfoque de trabajador inteligente, que crea nuevos *embeddings* solo cuando los datos subyacentes al *embedding* de IA realmente cambian, siempre es recomendable, ya que mejora el rendimiento y reduce los costos.

---

### Embeddings de consulta bajo demanda frente a en caché, y qué sucede cuando los modelos cambian

A estas alturas, hemos construido un sistema que puede generar *embeddings* para categorías, marcas, productos y variantes, y mantenerlos actualizados. Eso cierra el círculo sobre los *embeddings* de datos.

Pero los sistemas semánticos de producción tienen un segundo tipo de *embedding* que a menudo domina el costo y la latencia: los *embeddings* de consultas.

Supongamos que un usuario escribe lo siguiente:

- *"chaqueta de hombre a medida de menos de 500 $"*
- *"atuendo formal"*
- *"polos en diferentes colores"*

Necesitamos un *embedding* para esa consulta para poder compararlo con `product_embedding.embedding`.

Por lo tanto, la pregunta es: ¿Calculamos ese *embedding* de consulta cada vez (bajo demanda) o lo almacenamos en caché?

La respuesta no es binaria. Es un espectro de diseño, y la elección correcta depende de tu presupuesto de latencia, restricciones de costos y patrones de tráfico.

#### Dos tipos de embeddings en un sistema

En producción, normalmente tratas con lo siguiente:

- **Embeddings de datos (almacenados, indexados, actualizados)**:
  - Incluyen categorías, marcas, productos y variantes
  - Cambian con relativa lentitud
  - Vale la pena persistirlos e indexarlos
- **Embeddings de consultas (generados en tiempo de ejecución)**:
  - Las consultas de los usuarios son de alto volumen y alta entropía
  - Muchas son de uso único
  - El almacenamiento en caché puede ayudar, pero solo si se hace deliberadamente

Si tratas los *embeddings* de consultas como *embeddings* de datos y persistes todo, crearás un nuevo problema: una tabla de caché en constante crecimiento llena de consultas basura.

Si nunca almacenas en caché, el costo de la API de *embeddings* puede dispararse y la latencia p95 quedará encadenada al tiempo de respuesta de la API externa.

Por lo tanto, necesitamos un punto medio pragmático.

#### La línea base bajo demanda: La más simple y correcta

El enfoque más simple es lo que ya hicimos en el Capítulo 18 y ejemplos anteriores:

1. Incrustar el texto de la consulta
2. Ejecutar la búsqueda de similitud contra los *embeddings* de productos
3. Opcionalmente, aplicar filtros SQL (precio, categoría, marca, en stock, región)

Este código de muestra ilustra ese enfoque:

```sql
WITH q AS (
  SELECT api.openai_embed('Show me premium men''s clothes for a formal occasion')::vector(1536) AS qvec
)
SELECT
  DISTINCT
  p.id,
  p.label,
  pvp.price,
  1 - (pe.embedding <=> q.qvec) AS similarity
FROM q
JOIN embeddings.product_embedding pe ON TRUE
JOIN product.product p ON p.id = pe.product_id
JOIN product.product_variant pv ON pv.product_id = p.id
JOIN product.product_variant_price pvp
  ON pvp.product_variant_id = pv.id
 AND pvp.current = true
WHERE pvp.price <= 500  -- example hard constraint
ORDER BY pe.embedding <=> q.qvec
LIMIT 5;
```

El resultado es un conjunto de productos que coinciden con el filtro SQL de precio y son similares al significado del texto de la consulta:

```text
 id |                label                 | price |     similarity     
----+--------------------------------------+-------+---------------------
  1 | Dress shirt by The Gap               | 31.72 |  0.5363393106258336
  2 | Dress shirt by Boss                  | 31.93 |  0.5489808484608191
  3 | Dress shirt by Eaton                 | 31.86 |  0.5151862059589092
  4 | Mens Classic Oxford Shirt by The Gap | 31.79 | 0.40913109210524257
  5 | Mens Classic Oxford Shirt from Boss  | 31.69 | 0.43301725786821577
```

Esto es puramente bajo demanda. Es limpio y siempre correcto. Pero crea dos presiones de producción predecibles:

- **Latencia**: La llamada de *embedding* se encuentra en la ruta crítica
- **Costo**: Las consultas repetidas o similares continúan llamando a la API

Por lo tanto, bajo demanda es la línea base. Ahora, optimizamos.

#### Una caché de consultas apta para producción (mínima pero disciplinada)

Una caché es útil cuando se aplica lo siguiente:

- La misma consulta o consultas muy similares se repiten
- Deseas respuestas más rápidas
- Deseas reducir las llamadas a la API de *embeddings*

Pero el almacenamiento en caché debe tener salvaguardas:

- Necesita TTL (*Time-To-Live*)
- Necesita deduplicación
- No debe crecer para siempre

Aquí hay una tabla de caché de *embeddings* de consultas mínima:

```sql
CREATE TABLE IF NOT EXISTS embeddings.query_embedding_cache (
  query_text     text NOT NULL,
  embedding      vector(1536) NOT NULL,
  model_id       text NOT NULL,
  created_at     timestamptz NOT NULL DEFAULT now(),
  last_used_at   timestamptz NOT NULL DEFAULT now(),
  PRIMARY KEY (query_text, model_id)
);

CREATE INDEX IF NOT EXISTS query_embedding_cache_last_used_idx
  ON embeddings.query_embedding_cache (last_used_at);
```

Esta tabla almacena intencionalmente lo siguiente:

- El texto de la consulta original
- Su *embedding*
- El ID del modelo utilizado
- Marcas de tiempo para respaldar el desalojo

Ahora creamos una función auxiliar para obtener el *embedding* de la caché si está presente; de lo contrario, lo calcula y lo almacena:

```sql
CREATE OR REPLACE FUNCTION embeddings.sf_get_query_embedding(
  p_query    text,
  p_model_id text DEFAULT 'text-embedding-3-small'
)
RETURNS vector(1536)
LANGUAGE plpgsql
VOLATILE
SECURITY DEFINER
SET search_path = pg_catalog, embeddings, api, public
AS $function$
DECLARE
  v_embedding vector(1536);
BEGIN
  -- Fast path: cache hit
  SELECT qec.embedding
  INTO v_embedding
  FROM embeddings.query_embedding_cache qec
  WHERE qec.query_text = p_query
    AND qec.model_id    = p_model_id;

  IF v_embedding IS NOT NULL THEN
    UPDATE embeddings.query_embedding_cache qec
    SET last_used_at = now()
    WHERE qec.query_text = p_query
      AND qec.model_id    = p_model_id;

    RETURN v_embedding;
  END IF;

  -- Cache miss: compute embedding
  v_embedding := api.openai_embed(p_query)::vector(1536);

  INSERT INTO embeddings.query_embedding_cache (query_text, embedding, model_id, last_used_at)
  VALUES (p_query, v_embedding, p_model_id, now())
  ON CONFLICT (query_text, model_id)
  DO UPDATE
    SET embedding    = EXCLUDED.embedding,
        model_id     = EXCLUDED.model_id,
        last_used_at = now();

  RETURN v_embedding;
END;
$function$;
```

Ahora tu consulta de búsqueda se convierte en lo siguiente:

```sql
WITH q AS (
  SELECT embeddings.sf_get_query_embedding('Show me premium men''s clothes for a formal occasion')::vector(1536) AS qvec
)
SELECT
  DISTINCT
  p.id,
  p.label,
  pvp.price,
  1 - (pe.embedding <=> q.qvec) AS similarity
FROM q
JOIN embeddings.product_embedding pe ON TRUE
JOIN product.product p ON p.id = pe.product_id
JOIN product.product_variant pv ON pv.product_id = p.id
JOIN product.product_variant_price pvp
  ON pvp.product_variant_id = pv.id
 AND pvp.current = true
WHERE pvp.price <= 500  -- example hard constraint
ORDER BY similarity DESC
LIMIT 5;
```

El resultado sigue siendo el mismo, pero potencialmente evitamos regenerar innecesariamente un *embedding*:

```text
 id |                label                 | price |     similarity     
----+--------------------------------------+-------+---------------------
  1 | Dress shirt by The Gap               | 31.72 |  0.5363510207711434
  2 | Dress shirt by Boss                  | 31.93 |  0.5490152672226669
  3 | Dress shirt by Eaton                 | 31.86 |  0.5152048315046799
  4 | Mens Classic Oxford Shirt by The Gap | 31.79 |  0.4091325876375278
  5 | Mens Classic Oxford Shirt from Boss  | 31.69 | 0.43303663336793985
(5 rows)
```

Con esto, las consultas repetidas se vuelven rápidas. Las llamadas a la API se reducen drásticamente para búsquedas comunes. Ahora puedes controlar el crecimiento de la caché de manera operativa.

#### Desalojo de caché: Mantenerla pequeña y significativa

Una caché sin desalojo no es una caché; es un vertedero. La estrategia más simple es el TTL basado en el "último uso":

```sql
DELETE FROM embeddings.query_embedding_cache
WHERE last_used_at < now() - interval '7 days';
```

Ejecútalo periódicamente mediante un programador (o un trabajador en segundo plano o desde tu aplicación). Esto mantiene la caché relevante y delimitada.

También puedes limitar el tamaño de la caché eliminando las filas más antiguas más allá de un límite, pero el TTL suele ser la mejor primera opción.

#### Cerrar el círculo: Búsqueda y recomendaciones con embeddings de consulta en caché

Una vez que los *embeddings* de consulta están en caché, puedes usar el mismo motor para lo siguiente:

- Búsqueda semántica (consulta a productos)
- Recomendación (producto a productos)
- Vectores de preferencia (promedio `AVG` de *embeddings* de productos comprados a productos)

Y ahora puedes hacerlo con menor latencia y menor costo de API.

En este punto, PostgreSQL no solo almacena tu catálogo. Participa en el ciclo de razonamiento:

1. Crear significado
2. Almacenar significado
3. Indexar significado
4. Recuperar significado
5. Aplicar la verdad empresarial mediante filtros SQL

---

### Una dura verdad: Los cambios de modelo crean desvío semántico (*semantic drift*)

Ahora llegamos a un punto donde la mayoría de los equipos aprenden por las malas. Los *embeddings* no son verdades eternas. Están moldeados por:

- El modelo de *embedding*
- Su tokenización
- Sus datos de entrenamiento y opciones de representación
- La dimensionalidad
- Incluso cambios sutiles de versión

Si cambia el LLM que se utiliza para generar los *embeddings* para los datos o las consultas, suceden dos cosas:

1. La similitud, medida como una distancia en el espacio vectorial, se vuelve insignificante si los *embeddings* de la consulta y del producto provienen de diferentes modelos de *embedding*, ya que es probable que sus dimensiones difieran.
2. Incluso si las dimensiones coinciden, habrá un desvío en el significado y el comportamiento de la clasificación por similitud puede cambiar.

Por eso los sistemas de producción deben tratar el modelo de *embedding* como una versión de API. Es realmente importante nunca comparar *embeddings* generados por diferentes modelos a menos que se sepa explícitamente que son compatibles.

Por lo general, esto no se sabe y a menudo no se puede garantizar, ya que los modelos incorporan nuevos datos que cambian los pesos y pueden afectar las dimensiones.

Por lo tanto, necesitamos una estrategia de actualización limpia y debemos incorporarla desde el primer día.

#### Una estrategia de embedding versionada (segura para producción)

Existen dos enfoques prácticos: cambio radical (*hard cutover*) y ejecución paralela.

##### El cambio radical: Volver a incrustar todo

El cambio radical regenera todos los *embeddings* existentes de una sola vez:

1. Elegir un nuevo modelo.
2. Regenerar todos los *embeddings*.
3. Reconstruir todos los índices en los *embeddings*.
4. Cambiar la aplicación a los nuevos *embeddings*.

Este es el modelo mental más simple. También es el más costoso y requiere tiempo de inactividad para el cambio.

##### Operaciones paralelas: Usar dos conjuntos de embeddings durante la migración

Cuando ejecutamos operaciones paralelas, hacemos lo siguiente:

1. Mantener activos los *embeddings* antiguos para la búsqueda
2. Construir nuevos *embeddings* en paralelo
3. Validar la relevancia
4. Luego cambiar de forma segura

Este modelo requiere estructuras de datos adicionales para rastrear qué modelo se utilizó para un *embedding* específico y tablas de *embeddings* separadas para cada LLM o versión de LLM.

Para la continuidad de este capítulo (y cambios mínimos), la extensión más simple es agregar una columna `model_id` a cada tabla de *embeddings*, como en este ejemplo para productos:

```sql
ALTER TABLE embeddings.product_embedding
ADD COLUMN IF NOT EXISTS model_id text NOT NULL DEFAULT 'text-embedding-3-small';
```

Luego, puedes consultar utilizando una versión de modelo específica:

```sql
WITH q AS (
  SELECT embeddings.sf_get_query_embedding('Show me premium men''s clothes for a formal occasion', 'text-embedding-3-small' )::vector(1536) AS qvec
)
SELECT
  DISTINCT
  p.id,
  p.label,
  pvp.price,
  1 - (pe.embedding <=> q.qvec) AS similarity
FROM q
JOIN embeddings.product_embedding pe ON TRUE
JOIN product.product p ON p.id = pe.product_id
JOIN product.product_variant pv ON pv.product_id = p.id
JOIN product.product_variant_price pvp
  ON pvp.product_variant_id = pv.id
 AND pvp.current = true
WHERE pvp.price <= 500  -- example hard constraint
AND pe.model_id = 'text-embedding-3-small'
ORDER BY similarity DESC
LIMIT 10;
```

Si más adelante introduces un segundo modelo, puedes poblar los *embeddings* para él (en un trabajo en segundo plano) y luego comparar la relevancia antes de cambiar.

Así es como se actualiza sin romper la confianza.

#### ¿Qué desencadena una reconstrucción completa?

Una nueva generación completa de *embeddings* puede desencadenarse por:

- Un cambio del modelo de *embedding* (`text-embedding-3-small` >> otro modelo)
- Un cambio en la composición de la "cadena de significado" (por ejemplo, la decisión de que se debe agregar más información, como añadir la marca o la categoría a los datos del *embedding*)
- Cambios importantes en la distribución de datos (por ejemplo, una nueva categoría de catálogo que utiliza un idioma diferente)
- Cambios en las reglas de normalización (por ejemplo, convertir a minúsculas, eliminar HTML, agregar contexto estructurado)

Lo importante es que cuando cambia la receta de la huella digital semántica, la huella digital debe regenerarse.

Por eso los equipos de producción tratan la canalización de *embeddings* como infraestructura de datos, no como un script de una sola vez.

---

### Resumen

Este capítulo convirtió la generación de *embeddings* en una canalización de producción en lugar de una llamada única a una API. Sobre la base del [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas*, cambiamos la pregunta central de "¿Podemos crear *embeddings*?" a "¿Cómo ponemos en producción los *embeddings* sin romper el OLTP?", con respuestas claras sobre frescura, manejo de fallas y escala.

Introdujimos el modelo de dos velocidades de la verdad: la verdad operativa (`product.*`) debe ser rápida y exacta, mientras que la verdad semántica (`embeddings.*`) debe ser confiable y recuperable, pero nunca debe mantener a las transacciones como rehenes. A partir de esa base, tomamos la decisión de producción clave: los disparadores deben encolar, no incrustar. En lugar de llamar al LLM (OpenAI en nuestro ejemplo) directamente desde los disparadores, implementamos una cola de trabajos de *embedding* duradera (`embeddings.embedding_job`), un asistente de encolado único y disparadores livianos para cambios en categorías, marcas, productos y variantes.

Luego construimos un trabajador mínimo pero "con forma de producción" utilizando procesamiento por lotes y `FOR UPDATE SKIP LOCKED` para una concurrencia segura, e inserciones/actualizaciones idempotentes (`ON CONFLICT DO UPDATE`) para que los reintentos sigan siendo seguros. Para mantener el significado actualizado sin desperdiciar llamadas, agregamos resiliencia a la canalización (reintentos programados con retroceso) y resumen criptográfico de contenido (*content hashing*), incrustando solo cuando la entrada semántica realmente cambia. También manejamos el desvío semántico indirecto mediante el encolado en cascada de trabajos cuando una actualización de marca o categoría debería refrescar los *embeddings* del producto.

Finalmente, comparamos los tres patrones de implementación que vemos en sistemas reales (canalizaciones de *embeddings* sincrónicas, asincrónicas y en tiempo real) y extendimos el ciclo de vida para incluir *embeddings* de consultas, mostrando cuándo la generación bajo demanda es suficiente, cuándo ayuda el almacenamiento en caché y por qué los cambios de modelo requieren disciplina de versiones (`model_id`) para evitar comparar *embeddings* de diferentes espacios.

En el [Capítulo 20](https://subscription.packtpub.com/book/data/9781806028474/20), *PostgreSQL y MCP: El Modelo para un Asistente de IA Robusto*, pasaremos de las canalizaciones a las experiencias. Utilizando el ciclo de vida de *embeddings* estable construido en este capítulo, implementaremos un chatbot SQL básico y luego un chatbot SQL dinámico con salvaguardas, combinando la recuperación de `pgvector` con filtros relacionales (precio, categoría, atributos de variantes y precios actuales) para garantizar que las respuestas sean relevantes, correctas y operativamente seguras.

También presentaremos el Protocolo de Contexto de Modelo (*Model Context Protocol* o MCP) como un puente controlado entre un LLM y "herramientas reales". MCP proporciona un patrón práctico para conectar el asistente a las funciones de PostgreSQL (búsqueda semántica, recuperación y ejecución segura de consultas) mientras mantiene límites claros: el modelo puede solicitar acciones, pero el sistema decide qué está permitido, qué se registra y qué se devuelve. En otras palabras, el Capítulo 20 mostrará cómo construir una interfaz de IA que pueda usar tu base de datos, sin convertir tu base de datos en una apuesta arriesgada.
