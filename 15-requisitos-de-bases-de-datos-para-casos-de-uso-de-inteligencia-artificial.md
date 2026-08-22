## Capítulo 15: Requisitos de Bases de Datos para Casos de Uso de Inteligencia Artificial

Las aplicaciones de IA, especialmente los modelos de lenguaje grande (*Large Language Models* o LLMs) y la IA generativa (*Generative AI* o GenAI), están cambiando la forma en que accedemos a la información e interactuamos con la tecnología. Hoy en día, los usuarios esperan que los sistemas interpreten la intención, no solo que devuelvan filas. Este cambio está obligando a las bases de datos a evolucionar más allá de su función tradicional. Por ejemplo, incluso capacidades simples impulsadas por IA como la búsqueda semántica, la generación aumentada por recuperación (*Retrieval-Augmented Generation* o RAG) o la respuesta a preguntas en lenguaje natural requieren que las bases de datos manejen patrones, significados y relaciones que no pueden capturarse únicamente mediante columnas estructuradas.

Los sistemas de bases de datos subyacentes ahora deben cumplir con nuevos requisitos, que difieren de su desarrollo tradicional. Muchos modelos de bases de datos convencionales han sido efectivos para datos estructurados y para garantizar la integridad transaccional, pero no siempre son ideales para cargas de trabajo de IA en escenarios del mundo real. Estas cargas de trabajo dependen frecuentemente de la comprensión de contenido no estructurado, la realización de búsquedas de similitud en vectores de alta dimensionalidad y el trabajo estrecho con *embeddings* generados por LLMs.

Este capítulo examina en detalle las principales motivaciones detrás de estos desafíos y servirá como referencia de cómo PostgreSQL los está abordando a través de su extensibilidad inherente. También analiza la necesidad de procesar contenido no estructurado (como texto e imágenes), las matemáticas detrás de la búsqueda de similitud semántica (búsqueda de vecinos más cercanos o *nearest neighbor search*) y la importancia de los *embeddings* y su conexión con los LLMs y los modelos fundacionales.

Este capítulo cubre los siguientes temas:

- Cómo los requisitos de las bases de datos para IA difieren de las aplicaciones tradicionales
- Trabajo con datos no estructurados como texto, imágenes y audio
- Comprensión de *embeddings* y datos vectoriales
- Búsqueda de vecinos más cercanos (*nearest neighbor search*) y similitud vectorial
- Integración de PostgreSQL con LLMs y servicios de IA
- Gestión, indexación y escalado de datos vectoriales para cargas de trabajo de IA

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Trabajo con lenguaje natural y datos no estructurados

Uno de los cambios más significativos introducidos por las aplicaciones de IA es la importancia primordial de los datos no estructurados. A diferencia de las filas y columnas perfectamente organizadas de las bases de datos relacionales tradicionales, gran parte de la información fundamental para la IA (especialmente para LLMs y GenAI) existe en formatos como texto en lenguaje natural, imágenes, audio y video. Las bases de datos que respaldan las aplicaciones de IA deben ser expertas no solo en almacenar, sino también en procesar, consultar e indexar eficientemente este tipo de datos.

Al trabajar con lenguaje natural y datos no estructurados, entran en juego varias consideraciones que influyen directamente en los requisitos de la base de datos para aplicaciones de IA. A diferencia de las aplicaciones convencionales que normalmente operan con esquemas rígidos y datos estructurados, las aplicaciones de IA, especialmente aquellas que utilizan LLMs, deben lidiar con la variabilidad, la ambigüedad y la inmensidad inherentes del lenguaje humano y diversos medios.

#### Consideraciones de datos para el lenguaje natural

A medida que los sistemas de IA dependen cada vez más de la comprensión del lenguaje natural, las características de los datos en sí se convierten en una parte definitoria del desafío de los datos. El lenguaje natural tiene requisitos diferentes a los registros estructurados tradicionales: crece rápidamente, viene en muchos formatos, conlleva ambigüedad y requiere que se extraiga su significado antes de que sea útil.

Antes de sumergirse en las capacidades de PostgreSQL, es importante comprender las consideraciones básicas de datos que ayudan a las aplicaciones de IA a trabajar con texto y otras entradas no estructuradas:

- **Volumen y velocidad**: Los datos en lenguaje natural (texto, audio y video) a menudo se generan a un alto volumen y velocidad, desde fuentes de redes sociales hasta interacciones de servicio al cliente. Las bases de datos deben escalar horizontalmente (*scale out* agregando nodos/servidores, en lugar de escalar verticalmente un solo servidor) y ser capaces de manejar altas tasas de ingesta. PostgreSQL, con un escalado adecuado, puede manejar tasas de ingesta rápidas.
- **Variedad**: La diversidad de tipos de datos no estructurados (por ejemplo, texto sin formato, PDFs, imágenes con texto incrustado y transcripciones de audio) significa que una sola estructura de datos es insuficiente. La base de datos necesita dar cabida a estructuras de datos polimórficas, que PostgreSQL puede manejar.
- **Veracidad y validez**: Los datos no estructurados pueden ser ruidosos, inconsistentes y contener errores. El sistema de base de datos necesita mecanismos para almacenar y potencialmente limpiar y estandarizar estos datos para garantizar su calidad, lo que a menudo implica la integración con herramientas externas de limpieza de datos. PostgreSQL puede almacenar y potencialmente preprocesar estos datos para garantizar su calidad, integrándose frecuentemente con herramientas externas de limpieza de datos.
- **Significado semántico**: El desafío principal no es solo almacenar los datos sino extraer su significado. Por ejemplo, la reseña de un cliente que dice: *This product is amazing!* conlleva un sentimiento positivo que debe capturarse, no solo el texto sin procesar.

Estas características dan forma a lo que los sistemas de IA necesitan de una base de datos. Influyen en cómo se deben almacenar, procesar y consultar los datos. Por lo tanto, las consideraciones anteriores se traducen en requisitos específicos de la base de datos que admiten cargas de trabajo de IA.

#### Cómo se convierte en requisitos de base de datos

Estas consideraciones se traducen directamente en requisitos de base de datos específicos que diferencian las aplicaciones de IA de las tradicionales:

- **Esquema flexible y soporte para diversos tipos de datos**: Las bases de datos relacionales tradicionales destacan con esquemas tabulares fijos. Las aplicaciones de IA exigen bases de datos que puedan almacenar y consultar tipos de datos no estructurados de forma nativa, como `TEXT`, `JSONB`, `BYTEA` (para imágenes/audio) y, fundamentalmente, `VECTOR` para *embeddings*. Esto permite la integración perfecta de diferentes formatos de datos sin transformaciones complejas. PostgreSQL admite `TEXT`, `JSONB` y `BYTEA` (para imágenes/audio) de forma nativa y, a través de su arquitectura de extensiones, agrega soporte para `VECTOR` a través de la extensión `PGVECTOR` para *embeddings* de IA. La extensibilidad de PostgreSQL proporciona la base para el polimorfismo.
- **Capacidades avanzadas de indexación y búsqueda**: Más allá de las consultas simples de coincidencia exacta, las aplicaciones de IA requieren funcionalidades de búsqueda sofisticadas. Esto incluye lo siguiente:
  - **Búsqueda de texto completo (*Full-text search*)**: Para la recuperación basada en palabras clave dentro de documentos de texto extensos (por ejemplo, encontrar todas las descripciones de productos que contengan `eco-friendly`).
  - **Búsqueda de similitud vectorial (búsqueda de vecinos más cercanos o *nearest neighbor search*)**: Esencial para encontrar elementos semánticamente similares en función de sus representaciones de *embeddings* (por ejemplo, encontrar productos similares a la compra anterior de un usuario, incluso si las palabras clave no coinciden).
  - **Búsqueda híbrida (*Hybrid search*)**: Combinación de búsqueda por palabras clave y búsqueda semántica para obtener resultados más relevantes.
- **Escalabilidad para el almacenamiento y procesamiento de datos no estructurados**: El almacenamiento de petabytes de datos no estructurados, junto con sus formas procesadas (como *embeddings*), requiere una solución de almacenamiento altamente escalable. Además, la base de datos debe admitir el procesamiento distribuido de estos datos para tareas como la extracción de características o la reindexación. PostgreSQL se puede configurar para admitir el procesamiento distribuido de este tipo de datos.
- **Integración con frameworks de IA/ML**: La base de datos no debe ser un silo. Requiere conectores y APIs eficientes para interactuar con frameworks populares de IA/ML (por ejemplo, TensorFlow y PyTorch) para el entrenamiento de modelos, la generación de *embeddings* y el despliegue de inferencias. Esto minimiza el movimiento de datos y mejora la eficiencia general de la canalización.

Antes de explorar estas capacidades con más detalle, ayuda ver cómo aparecen estos requisitos en una aplicación del mundo real. Comparar un sistema tradicional con uno habilitado para IA aclara las diferencias.

#### Aplicaciones de IA vs. Aplicaciones convencionales: un ejemplo de comercio electrónico

Para ilustrar la diferencia, usemos la plataforma de comercio electrónico que presentamos en el [Capítulo 3](https://subscription.packtpub.com/book/data/9781806028474/3), *Una Aplicación de Comercio Electrónico de Muestra*. Los sistemas de comercio electrónico generan naturalmente una mezcla de información estructurada (productos y pedidos) y contenido no estructurado (descripciones e imágenes), lo que los convierte en un ejemplo ideal para demostrar cómo las cargas de trabajo de IA interactúan con una base de datos, los *embeddings* y la búsqueda vectorial.

Veamos una aplicación de comercio electrónico convencional:

- **Objetivo**: Administrar el catálogo de productos, procesar pedidos y realizar un seguimiento del inventario.
- **Datos**: Principalmente datos estructurados en tablas relacionales (por ejemplo, una tabla `products` con columnas como `id`, `label`, `price` y `qty`, y una tabla `sales_transaction` con `id`, `customer_id` y `transaction_date`).
- **Interacciones con la base de datos**: Consultas SQL para operaciones de creación, lectura, actualización y eliminación (CRUD). *Find all products under $50* es una consulta típica.
- **Requisitos de la base de datos**: Cumplimiento de ACID, sólida integridad transaccional y uniones (*joins*) eficientes.

Este modelo tradicional de comercio electrónico funciona bien para las necesidades operativas, pero destaca la brecha cuando se avanza hacia experiencias impulsadas por IA, como la búsqueda semántica, las recomendaciones personalizadas o las consultas basadas en la intención, que requieren comprender el significado, el contexto y la similitud a través de datos no estructurados. Esto prepara el escenario de por qué los *embeddings*, la búsqueda vectorial y la extensibilidad de la base de datos se vuelven esenciales en los sistemas modernos impulsados por IA.

#### Aplicación de comercio electrónico impulsada por IA (con funciones de LLMs/GenAI)

A medida que extendemos la misma plataforma de comercio electrónico a una plataforma impulsada por IA, la naturaleza de la carga de trabajo cambia drásticamente. El sistema ya no se limita a recuperar filas o hacer cumplir la integridad transaccional; debe comprender la intención del usuario y el significado del producto, comparar elementos y generar contenido si es necesario. Este cambio introduce una combinación más rica de tipos de datos, nuevas formas de computación y una interacción más profunda entre las aplicaciones. Los siguientes puntos destacan cómo estas capacidades de IA cambian las responsabilidades de la base de datos:

- **Objetivo**: Proporcionar recomendaciones personalizadas, impulsar la búsqueda en lenguaje natural, generar descripciones de productos dinámicas y analizar el sentimiento de los clientes a partir de reseñas.
- **Datos**:
  - **Estructurados**: Los mismos datos de `product` y `sales_transaction` que en el modelo convencional.
  - **No estructurados**: Descripciones de productos (texto extenso), reseñas de clientes, imágenes de productos, consultas de búsqueda de usuarios (lenguaje natural) y posiblemente audio/video de interacciones de soporte al cliente.
  - **Derivados (*embeddings*)**: Representaciones vectoriales de descripciones de productos, reseñas de clientes y consultas de usuarios.
- **Interacciones con la base de datos**:
  - **Búsqueda en lenguaje natural**: Un usuario escribe *Show me comfortable shoes for hiking*. La base de datos debe comprender la intención, convertirla en un *embedding* (ya sea a través de una función de *embedding* dentro de la base de datos, manteniendo el modelo cerca de la base de datos, o a través de la capa de aplicación) y encontrar *embeddings* de productos semánticamente similares, no solo palabras clave.
  - **Motor de recomendaciones**: *Recommend products similar to what I just bought*. La base de datos busca el *embedding* del artículo comprado y encuentra los *embeddings* de productos de vecinos más cercanos.
  - **Análisis de sentimiento**: Ingiere reseñas de clientes, y la base de datos puede almacenar puntuaciones de sentimiento derivadas de LLMs, lo que permite consultas como *Find all products with negative sentiment reviews about durability*.
  - **IA generativa para descripciones de productos**: La base de datos puede almacenar plantillas y características sin procesar de los productos, que un LLM utiliza para generar descripciones únicas y atractivas.
- **Requisitos de la base de datos**: Todos los requisitos convencionales, además de un esquema flexible para datos no estructurados, indexación vectorial robusta (por ejemplo, HNSW e IVFFlat), búsqueda de similitud vectorial eficiente, escalabilidad para volúmenes de datos masivos y sólidas capacidades de integración con servicios externos de IA o funciones de IA dentro de la base de datos. (Para conocer los fundamentos de indexación y la guía de selección en PostgreSQL, consulta el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Diseño para Altos Volúmenes de Transacciones y Escritura de Código Transaccional Eficiente*).

En esencia, mientras que las aplicaciones convencionales se centran en gestionar y consultar fragmentos de información atómicos y bien definidos, las aplicaciones de IA profundizan en el significado matizado, las relaciones y los patrones dentro de conjuntos de datos vastos y diversos, exigiendo una base de datos que sea inherentemente consciente y esté optimizada para estas complejas tareas analíticas y semánticas.

---

### Identificación de vecinos más cercanos en espacios de búsqueda multidimensionales

Otra piedra angular de muchas aplicaciones modernas de IA, en particular aquellas que aprovechan el aprendizaje automático (*machine learning*) y el aprendizaje profundo (*deep learning*), es la capacidad de identificar de manera eficiente los vecinos más cercanos dentro de espacios de búsqueda multidimensionales. Este concepto es fundamental para tareas como los sistemas de recomendación, la detección de anomalías, el reconocimiento de imágenes y el procesamiento del lenguaje natural.

En esencia, un espacio de búsqueda multidimensional se refiere a datos donde cada elemento está representado no por un solo valor, sino por una colección de valores, o características (*features*), que lo describen. Cuando estas características son numéricas, se pueden considerar como coordenadas con dimensiones que van desde cientos hasta miles en un espacio geométrico de alta dimensionalidad. (Consulta el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, para obtener una introducción rápida sobre vectores y métricas de distancia). Por ejemplo, una imagen puede representarse mediante miles de características numéricas que describen sus colores, texturas y formas. El perfil de preferencias de un usuario podría representarse mediante un vector de sus calificaciones para diferentes géneros cinematográficos.

El problema del vecino más cercano se convierte entonces en: Dado un punto de datos específico (un "vector de consulta"), encontrar los puntos de datos en la base de datos que sean más similares. La similitud en este contexto se mide típicamente mediante métricas de distancia en el espacio multidimensional, como la distancia euclidiana o la similitud del coseno. Una distancia menor o una mayor similitud del coseno indica un mayor parecido semántico.

#### ¿Por qué es crucial la búsqueda de vecinos más cercanos para la IA?

Los sistemas modernos de IA no se basan solo en recuperar información, sino en encontrar lo que es más similar en significado, intención o comportamiento. Esto requiere comparar *embeddings* dimensionales (representaciones matemáticas de datos como texto, imágenes y productos) para determinar qué elementos están más cerca unos de otros en la búsqueda semántica. La búsqueda de vecinos más cercanos se convierte en una columna vertebral importante de las capacidades impulsadas por IA. Los siguientes ejemplos ilustran cómo se desarrolla esto en el mundo real, especialmente dentro del contexto del comercio electrónico:

- **Sistemas de recomendación**: Si a un usuario le gusta la película A, encontrar películas similares (sus vecinos más cercanos en un espacio de características de películas) es la base para recomendar nuevo contenido.
  - *Ejemplo de comercio electrónico*: Un cliente compra un bolso mensajero de cuero vintage. El sistema utiliza la búsqueda de vecinos más cercanos en los *embeddings* de productos para recomendar otros accesorios de estilo vintage, artículos de cuero o bolsos de trabajo que sean semánticamente similares, incluso si no comparten palabras clave exactas en sus descripciones. En la práctica, los sistemas de recomendación también aplican reglas comerciales como filtros de actualidad y "ya comprados" para evitar sugerir duplicados.
- **Búsqueda semántica**: En lugar de coincidencia de palabras clave, los usuarios pueden buscar el significado de una consulta. La consulta se convierte en un *embedding*, y luego los vecinos más cercanos en el espacio de *embeddings* representan documentos o conceptos semánticamente similares.
  - *Ejemplo de comercio electrónico*: Un cliente busca calzado duradero para aventuras al aire libre (*durable footwear for outdoor adventures*). La base de datos, mediante la búsqueda de vecinos más cercanos en los *embeddings* de zapatos, puede devolver resultados para botas de montaña, zapatillas de trail running o sandalias de trekking impermeables, comprendiendo la intención subyacente en lugar de solo coincidir con palabras literales.
- **Detección de anomalías**: Los puntos de datos que están lejos de sus vecinos más cercanos pueden considerarse anomalías o valores atípicos, útiles para la detección de fraude o la identificación de errores del sistema.
  - *Ejemplo de comercio electrónico*: El patrón de compra de un cliente se desvía repentinamente de su comportamiento habitual (por ejemplo, compra una gran cantidad de productos electrónicos de alto valor cuando normalmente compra ropa de bajo costo). Al representar el historial de compras como un vector, la búsqueda de vecinos más cercanos puede marcar esto como una anomalía para una posible investigación de fraude.
- **Agrupamiento (*clustering*) y clasificación**: Muchos algoritmos de aprendizaje automático se basan en identificar grupos de puntos de datos similares o clasificar nuevos datos según las características de sus vecinos más cercanos.
  - *Ejemplo de comercio electrónico*: Agrupación de reseñas de clientes en grupos de sentimiento positivo, sentimiento negativo o solicitudes de funciones mediante el análisis de la similitud de sus *embeddings*. Esto ayuda a los minoristas a identificar rápidamente temas comunes y puntos débiles de los productos.

#### Requisitos de bases de datos para aplicaciones de IA: búsqueda de vecinos más cercanos, embeddings e integración con LLMs

La necesidad de una búsqueda semántica eficiente en datos polimórficos, manejada a través de la búsqueda aproximada de vecinos más cercanos (*Approximate Nearest Neighbor* o ANN), junto con el papel crítico de los *embeddings* y la integración con LLMs, impacta profundamente en el diseño y los requisitos de las bases de datos.

Estas son las capacidades centrales de la base de datos para datos vectoriales:

- **Soporte de tipo de datos vectoriales**: La base de datos debe admitir un tipo de datos vectorial (por ejemplo, `VECTOR`, o `ARRAY` de flotantes) para almacenar las representaciones multidimensionales de los puntos de datos (*embeddings*). Esto es fundamental para una integración y un rendimiento perfectos.
- **Indexación especializada para altas dimensiones**: Los índices tradicionales B-tree o hash son ineficaces en espacios de alta dimensionalidad debido a la maldición de la dimensionalidad. Por lo tanto, las bases de datos necesitan índices especializados de vecinos más cercanos aproximados (ANN), tales como:
  - **Hierarchical Navigable Small World (HNSW)**: Un índice basado en grafos que es altamente eficiente para datos de alta dimensionalidad, ofreciendo una excelente velocidad de búsqueda y recuperación (*recall*).
  - **Inverted File Index (IVFFlat)**: Divide los datos en clústeres y busca solo en los clústeres relevantes, reduciendo el espacio de búsqueda.
  - **Índices especializados (Disk-ANN, ScaNN y similares a Faiss)**: Optimizados para diversos escenarios, incluidos grandes conjuntos de datos y necesidades de rendimiento específicas.
- **Funciones de distancia eficientes**: La base de datos debe proporcionar implementaciones altamente optimizadas de métricas de distancia comunes (por ejemplo, distancia euclidiana (L2), similitud del coseno y producto interno) para cálculos de similitud rápidos, cruciales para la búsqueda de vecinos más cercanos y la comprensión semántica.
- **Escalabilidad para el almacenamiento e indexación de vectores**: Al igual que los datos no estructurados, los vectores (*embeddings*) en sí mismos pueden consumir un almacenamiento significativo y sus índices pueden ser grandes. La base de datos necesita escalar para manejar terabytes/petabytes de datos vectoriales y admitir la indexación y búsqueda distribuidas, al tiempo que tiene en cuenta los objetivos de tiempo de recuperación (RTO) de copia de seguridad y recuperación a medida que crece el tamaño del clúster.
- **Búsqueda concurrente y actualizaciones de índices eficientes**: En aplicaciones de IA dinámicas, se agregan constantemente nuevos datos (y, por lo tanto, nuevos vectores/*embeddings*) y es posible que los datos existentes se actualicen. La base de datos debe admitir de manera eficiente búsquedas simultáneas de vecinos más cercanos y, al mismo tiempo, permitir actualizaciones continuas de índices sin una degradación significativa del rendimiento.

---

### Gestión de embeddings e integración con LLMs

El concepto de *embeddings* es fundamental para la IA moderna como una forma de almacenar la semántica de los datos y las solicitudes. Los *embeddings* son representaciones vectoriales densas de datos polimórficos, ya sean palabras, oraciones, párrafos, imágenes, audio o incluso documentos completos, que capturan su significado semántico. En esencia, transforman datos complejos y no estructurados en un formato numérico que las computadoras pueden entender y procesar matemáticamente.

#### ¿Qué son los embeddings?

Piensa en un *embedding* como una huella digital numérica única para una pieza de información. Por ejemplo, las palabras *king* y *queen* pueden tener vectores de *embedding* que están muy cerca unos de otros en un espacio multidimensional, lo que refleja su similitud semántica. La diferencia entre el vector *king* y el vector *queen* podría incluso representar la dimensión de género. Del mismo modo, el *embedding* de la imagen de un gato estaría numéricamente más cerca del *embedding* de la imagen de otro gato que del *embedding* de un automóvil. Por supuesto, los *embeddings* no son perfectos: las clases visualmente similares a veces pueden confundirse, por lo que la búsqueda de similitud debe validarse en el contexto del modelo y el conjunto de datos específicos.

Los *embeddings* suelen ser generados por sofisticados modelos de aprendizaje automático, sobre todo LLMs o modelos de *embeddings* especializados (por ejemplo, Word2Vec, BERT, los *embeddings* de OpenAI, etc.). Estos modelos aprenden a mapear los datos de entrada en un espacio vectorial de alta dimensionalidad donde las relaciones semánticas están representadas por distancias geométricas. PostgreSQL, con su extensión `PGVECTOR`, permite a los usuarios utilizar el tipo de datos `VECTOR`, lo que les permite almacenar vectores de alta dimensionalidad en él.

#### ¿Por qué son cruciales los embeddings y la integración con LLMs?

A medida que las aplicaciones de IA pasan de la coincidencia de palabras clave a la comprensión del significado, los *embeddings* se convierten en un puente importante entre las bases de datos y los LLMs. Traducen contenido generado por humanos, sin procesar y desordenado, en representaciones matemáticas estructuradas que capturan la intención, la similitud y el contexto, admitiendo búsquedas más ricas, recomendaciones más inteligentes y una experiencia de usuario más adaptativa. Los siguientes casos de uso destacan cómo esta integración transforma lo que un sistema impulsado por IA puede lograr:

- **Comprensión semántica**: Los *embeddings* permiten a las bases de datos representar el significado de los datos, no solo su forma literal. Esto permite la búsqueda semántica, donde una consulta como *Italian food* puede recuperar resultados de pizza o pasta, incluso si la palabra *Italian* no está presente en la descripción.
  - *Ejemplo de comercio electrónico*: Una plataforma de comercio electrónico utiliza *embeddings* de descripciones de productos. Un cliente busca *cozy knitwear*. El sistema, utilizando *embeddings*, entiende que *cozy* implica calidez y comodidad, y que *knitwear* implica suéteres, cárdigans y bufandas. Luego recupera productos cuyos *embeddings* están semánticamente cercanos a este concepto, incluso si los títulos de los productos no contienen explícitamente *cozy* o *knitwear*.
- **Búsqueda contextual (RAG)**: Los LLMs prosperan con el contexto. Al almacenar *embeddings* de documentos, conversaciones o bases de conocimiento, la base de datos puede proporcionar al LLM información relevante y semánticamente similar, mejorando la calidad y precisión de sus respuestas.
  - *Ejemplo de comercio electrónico*: Un cliente le pregunta a un chatbot impulsado por LLM en un sitio de comercio electrónico: *What's the return policy for electronics?*. El chatbot consulta la base de datos utilizando un *embedding* de esta pregunta. La base de datos, que almacena *embeddings* de artículos de ayuda y preguntas frecuentes, recupera los documentos más relevantes semánticamente sobre las políticas de devolución de productos electrónicos. Esta información luego se pasa al LLM, lo que le permite generar una respuesta precisa y específica basada en las políticas reales de la tienda.
- **Personalización**: Las preferencias del usuario, las interacciones históricas y los datos demográficos se pueden incrustar (*embed*). Al encontrar *embeddings* de vecinos más cercanos de usuarios a productos, se pueden ofrecer recomendaciones altamente personalizadas.
  - *Ejemplo de comercio electrónico*: El historial de navegación de un usuario y sus compras anteriores se convierten en un *embedding* de usuario. Las descripciones de los productos también se incrustan. Luego, el sistema encuentra productos cuyos *embeddings* están más cercanos al *embedding* del usuario, lo que genera recomendaciones de productos altamente personalizadas en la página de inicio o en campañas de marketing por correo electrónico.
- **Eficiencia con LLMs**: En lugar de enviar grandes cantidades de texto sin formato o datos de imágenes a un LLM para cada consulta, la base de datos puede almacenar *embeddings* precalculados. Esto reduce la carga computacional en el LLM y acelera la inferencia.
- **Base para GenAI**: Los *embeddings* son a menudo la entrada o salida de modelos generativos, guiando el proceso de generación para crear contenido nuevo y semánticamente coherente.
  - *Ejemplo de comercio electrónico*: Un LLM genera descripciones de productos únicas basadas en las características de un producto. Estas descripciones generadas se pueden incrustar y almacenar en la base de datos para la búsqueda semántica, lo que garantiza la coherencia y la capacidad de descubrimiento. O bien, un modelo generativo podría crear variaciones de imágenes de productos y sus *embeddings* podrían usarse para garantizar que las imágenes generadas sean visualmente similares al inventario existente.

#### Requisitos adicionales de base de datos para gestionar embeddings e integración con LLMs

Más allá de las capacidades vectoriales centrales, la gestión de *embeddings* y la integración con LLMs introduce requisitos adicionales:

- **Integración con servicios de generación de embeddings/LLMs**: La base de datos necesita formas optimizadas de integrarse con servicios externos o internos que generen *embeddings*. Esto puede implicar lo siguiente:
  - **Funciones dentro de la base de datos (*In-database functions*)**: Permitir a los usuarios llamar a modelos de *embeddings* externos directamente desde SQL.
  - **Carga de datos y ETL eficientes**: Herramientas para extraer datos sin procesar, enviarlos a un modelo de *embeddings* y luego cargar los vectores resultantes nuevamente en la base de datos.
  - **Conectividad API**: Conectores robustos para APIs de LLM populares (por ejemplo, OpenAI, Anthropic y Hugging Face) para tareas como RAG, donde la base de datos recupera el contexto relevante (a través de búsqueda vectorial) y lo pasa al LLM.
- **Capacidades de búsqueda híbrida**: La capacidad de combinar la búsqueda tradicional basada en palabras clave con la búsqueda semántica vectorial es vital. Por ejemplo, un usuario puede buscar *red dress* (palabra clave) pero también desear resultados que sean semánticamente similares a un vestido formal (búsqueda vectorial). La base de datos debe admitir la ejecución eficiente de tales consultas híbridas.
  - *Ejemplo de comercio electrónico*: Un cliente busca *cheap running shoes*. El sistema primero realiza una búsqueda por palabra clave de *running shoes* y luego refina los resultados buscando *embeddings* que sean semánticamente similares a *cheap* (por ejemplo, artículos de menor precio en la categoría de zapatillas para correr).
- **Transaccionalidad y consistencia para embeddings**: Para muchas aplicaciones de IA, es crucial que los *embeddings* se gestionen con la misma integridad y coherencia de datos que el resto de los datos. Si un documento se actualiza, su *embedding* debe regenerarse y actualizarse atómicamente.
- **Actualizaciones en tiempo real para embeddings**: En entornos dinámicos (por ejemplo, chat en vivo y sistemas de recomendación en tiempo real), llegan nuevos datos constantemente. La base de datos debe admitir una indexación eficiente en tiempo real de nuevos *embeddings* y una disponibilidad inmediata para la búsqueda.

En esencia, una base de datos para IA, especialmente con LLMs y GenAI, se transforma de un simple almacén de datos en un centro de información inteligente, capaz de comprender, organizar y recuperar información en función de su significado, todo impulsado por la presencia ubicua y la gestión eficiente de los *embeddings*.

---

### Resumen

En este capítulo, cubrimos los siguientes conceptos clave:

- **Requisitos distintos de bases de datos para aplicaciones de IA**: A diferencia de las aplicaciones convencionales que se ocupan principalmente de datos estructurados e integridad transaccional, las aplicaciones de IA, especialmente aquellas que aprovechan LLMs y GenAI, exigen bases de datos capaces de manejar grandes cantidades de datos no estructurados y polimórficos (texto, imágenes y audio), realizar búsquedas semánticas sofisticadas e integrarse sin problemas con modelos de IA.
- **Los vectores como base de las aplicaciones de IA basadas en GenAI y LLMs**: Este capítulo destacó que los vectores son las representaciones numéricas fundamentales que capturan el significado semántico de diversos tipos de datos. Estos arreglos numéricos multidimensionales son cruciales para tareas como la búsqueda semántica, los sistemas de recomendación y la comprensión contextual en LLMs.
- **Los vectores como tipo de datos en PostgreSQL**: De manera crítica, aprendiste que las extensiones de bases de datos especializadas (como `pgvector` para PostgreSQL) permiten que los vectores se traten como un tipo de datos nativo de primera clase dentro de una base de datos relacional tradicional. Esto permite el almacenamiento, indexación y consulta eficientes de *embeddings* vectoriales junto con datos estructurados convencionales, y permite a los desarrolladores utilizar operadores, funciones y agregados SQL familiares en las mismas consultas, simplificando la arquitectura y aprovechando la solidez de PostgreSQL.

Este capítulo ha sentado las bases para comprender las demandas únicas que la IA impone a los sistemas de bases de datos, enfatizando el papel fundamental de los datos polimórficos no estructurados, la búsqueda de vecinos más cercanos y la gestión de *embeddings* como elementos fundacionales para construir la próxima generación de aplicaciones inteligentes. PostgreSQL, a través de su extensibilidad inherente, está excepcionalmente bien posicionada para respaldar estos requisitos.

El siguiente capítulo demuestra cómo se puede aprovechar la extensión vectorial de PostgreSQL para integrarse con LLMs, generar *embeddings* a partir de datos estructurados y no estructurados, ejecutar búsquedas basadas en IA y generar recomendaciones.
