## Capítulo 21: Uniendo Todo

En este libro, hemos mostrado cómo utilizar PostgreSQL para crear soluciones empresariales modernas que incorporan capacidades transaccionales, analíticas y de IA. Las lecciones clave aprendidas son las siguientes:

- **Aprovechar PostgreSQL y su ecosistema** para organizar y gestionar datos polimórficos (números, caracteres, documentos JSON, *embeddings* de IA, datos SIG y tipos de datos personalizados) para manejar datos polimórficos de manera confiable y efectiva.
- **Mantener diferentes formatos de datos y sus operadores**, como búsquedas de texto, transacciones JSON o búsquedas de similitud basadas en vectores, bajo un mismo paraguas transaccional compatible con ACID para minimizar los dolores de cabeza de integración, la fragilidad arquitectónica, los retrasos de red y los desafíos operativos.
- **PostgreSQL es más que la distribución estándar de PostgreSQL** que puedes descargar de PostgreSQL.org. Cuenta con un vasto ecosistema de extensiones, como `pgvector` para *embeddings* de IA y cálculos de IA de vecinos más cercanos aproximados (*approximate nearest neighbour*), `pg_background` para procesamiento en segundo plano asincrónico y `plpgsql_check` para verificar procedimientos de bases de datos. Aprovecha estas extensiones para alcanzar todo el poder de PostgreSQL.
- **Al crear aplicaciones empresariales intensivas en datos híbridas a gran escala**, construye sobre la replicación de datos nativa de PostgreSQL para crear arquitecturas de datos efectivas y fáciles de administrar que aprovechen el principio de replicar datos de referencia, distribuir transacciones e integrar datos para análisis e IA.
- **Aprovecha estas mismas capacidades para crear servicios de datos modulares** optimizados para cargas de trabajo transaccionales, analíticas y de IA a través de modelos de datos específicos para cada carga de trabajo, estrategias de indexación y estrategias efectivas para evitar problemas de "vecinos ruidosos" (*noisy neighbors*).

La Figura 21.1 destaca los servicios de datos modulares, todos construidos en PostgreSQL, pero especializados para su carga de trabajo mediante el uso de diferentes enfoques de modelado y aprovechando los tipos de datos, operadores e índices extensibles de PostgreSQL:

*Figura 21.1: Servicios de datos con su especialización en modelado de datos*

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### Capacidades de PostgreSQL

En este libro, fuimos más allá del SQL estándar e introdujimos muchos conceptos de arquitectura y modelado de datos que consideramos clave para crear soluciones empresariales de alto rendimiento. La arquitectura adecuada proporciona una base sólida para un sistema escalable, y el modelo de datos correcto ayuda a los desarrolladores a evitar muchos problemas futuros de rendimiento y capacidad de administración.

#### Conceptos arquitectónicos: Polimórfico, versátil, híbrido

Tradicionalmente, las bases de datos eran vistas como monolitos y especialistas. Se utilizaban bases de datos especializadas para formatos de datos específicos: documentos JSON, pares clave-valor, búsqueda de texto y vectores de IA, teniendo cada una su propio software de base de datos especializado. De manera similar, los casos de uso analíticos y transaccionales se implementaban en productos de bases de datos adaptados a ellos.

En este libro, demostramos que un solo sistema de base de datos (PostgreSQL) puede almacenar datos SQL convencionales junto con JSON, KVP, SIG, texto y vectores de IA en la misma base de datos, y procesar transacciones bajo el mismo paraguas ACID, utilizando el mismo lenguaje de consulta y el mismo lenguaje de programación. Nos referimos a esto como el **polimorfismo de PostgreSQL**.

Diferentes casos de uso necesitan diferentes arquitecturas, pero todos pueden construirse sobre Postgres, y dentro de un caso de uso, una sola base de datos de PostgreSQL puede manejar múltiples formatos de datos. Así, vemos bases de datos transaccionales construidas sobre PostgreSQL que adoptan modelos de datos altamente normalizados. Las aplicaciones analíticas prosperan con esquemas en estrella, y las aplicaciones de IA adoptan con éxito *embeddings* de IA junto con datos SQL estándar. Esto refleja la **versatilidad de PostgreSQL**.

El **procesamiento híbrido**, donde las consultas basadas en SQL proporcionan decisiones estrictas e inequívocas de sí/no, en combinación con cálculos de vecinos más cercanos aproximados basados en IA que dan como resultado escalas continuas y flexibles, es una extensión natural de las capacidades de PostgreSQL.

Esto nos enseña que los servicios de datos específicos para cada caso de uso pueden ser polimórficos e integrar procesamiento híbrido, pero cada caso de uso se beneficia de un modelo de datos optimizado, y debido a que tiene diferentes patrones transaccionales y de consumo de recursos, debe aislarse de otros servicios para evitar el problema del vecino ruidoso (*noisy neighbor* o acaparador de recursos). PostgreSQL proporciona varias capacidades para lograr esto, tales como:

- **La replicación lógica con filtrado basado en filas y columnas** nos ayuda a crear servicios de datos modulares. Estos módulos pueden distribuirse geográficamente, particionarse para escalar con las necesidades del negocio y optimizarse para el caso de uso ([Capítulo 11](https://subscription.packtpub.com/book/data/9781806028474/11), *Alimentando el Almacén de Datos de PostgreSQL*).
- **PostgreSQL proporciona una amplia gama de lenguajes procedimentales**, que van desde PL/pgSQL (el lenguaje procedimental nativo) hasta PL/Python, e incluye PL/Tcl, PL/Perl, PL/R y muchos otros. Esto nos permite crear lógica enriquecida que opera justo al lado de los datos, evitando la latencia de la red, e incrusta la lógica de negocio dentro del modelo transaccional ACID ([Capítulo 6](https://subscription.packtpub.com/book/data/9781806028474/6), *Funciones y Procedimientos Almacenados*).
- **Las vistas SQL, las vistas materializadas y los disparadores** son un conjunto enriquecido de herramientas para transformar datos. Combinados con la replicación lógica, podemos mantener una representación transaccional de los datos sincronizada con un modelo de datos analítico o centrado en IA, sin utilizar herramientas de terceros y sin salir del ámbito de PostgreSQL. ¡Las mismas habilidades, la misma licencia, el mismo modelo operativo! ([Capítulo 12](https://subscription.packtpub.com/book/data/9781806028474/12), *Transformación de los Datos para Analítica*; [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas* y [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*).
- **Las canalizaciones de datos, el transporte asincrónico y los mecanismos de transformación** se utilizan para desacoplar sistemas distribuidos y aislarlos de los efectos de la red, ralentizaciones transitorias e interrupciones temporales. Se puede utilizar una combinación de disparadores, tablas y trabajadores en PostgreSQL para crear canalizaciones de datos robustas que se comuniquen con sistemas ajenos a PostgreSQL, como LLMs basados en la nube ([Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas* y [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*).
- **Procesamiento polimórfico e híbrido**: La lógica SQL, estricta e inequívoca, combinada con cálculos ANN que generan significado en una escala gradual, son la base para implementar soluciones de IA generativa en PostgreSQL ([Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*).
- **La IA agéntica** crea sistemas multicomponente que combinan agentes de IA con otros sistemas que actúan sobre la salida de la IA, también conocidos como actuadores. El Protocolo de Contexto de Modelo (*Model Context Protocol* o MCP) proporciona un marco seguro y auditable que permite a PostgreSQL actuar tanto como el agente de IA que genera recomendaciones como la herramienta que las ejecuta ([Capítulo 20](https://subscription.packtpub.com/book/data/9781806028474/20), *PostgreSQL y MCP: El Modelo para un Asistente de IA Robusto*).

#### Conceptos de gestión de datos

Tradicionalmente, PostgreSQL prosperó en aplicaciones transaccionales, donde se prefieren los modelos de datos normalizados. En este libro, mostramos que PostgreSQL es totalmente capaz de admitir modelos analíticos, como esquemas en estrella, y modelos de IA, como *embeddings* basados en vectores.

- **ACID es fundamental para las bases de datos transaccionales**. Mientras que la mayoría de los desarrolladores pueden creer que las transacciones ACID son rígidas e inflexibles, y que simplemente tienen éxito o fracasan, mostramos en el [Capítulo 5](https://subscription.packtpub.com/book/data/9781806028474/5), *Transacciones, Cumplimiento ACID y Normalización de Datos*, que PostgreSQL permite a los desarrolladores influir en los niveles de aislamiento y controlar los flujos de transacciones.
- **Los modelos de datos normalizados son clave para el rendimiento transaccional**. El [Capítulo 5](https://subscription.packtpub.com/book/data/9781806028474/5), *Transacciones, Cumplimiento ACID y Normalización de Datos*, enseña al desarrollador, paso a paso, cómo avanzar de 0FN a 3FN.
- **Los tipos de datos, operadores e índices son clave para cada base de datos**. Elegir el tipo de datos óptimo es esencial para cualquier caso de uso de base de datos. La regla de oro aconseja seleccionar un tipo de datos que represente con precisión el dominio, admita los operadores adecuados y acomode cómodamente el rango de datos esperado. El [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Diseño para Altos Volúmenes de Transacciones y Escritura de Código Transaccional Eficiente*, detalla los tipos de datos y los criterios de selección para SQL.
- **Los modelos de datos analíticos tienen requisitos diferentes**; por ejemplo, se centran en seleccionar, agrupar, agregar y analizar grandes conjuntos de datos. El [Capítulo 10](https://subscription.packtpub.com/book/data/9781806028474/10), *El Modelo de Datos Analítico*, proporciona una descripción general de los *data vaults* y detalla cómo implementar esquemas en estrella en PostgreSQL.
- **Los modelos de IA**, centrados en cálculos de vecinos más cercanos aproximados para respaldar la semántica y el significado, aprovechan los *embeddings* a menudo derivados de datos SQL. Los vectores e índices de IA se analizan en el [Capítulo 15](https://subscription.packtpub.com/book/data/9781806028474/15), *Requisitos de Bases de Datos para Casos de Uso de Inteligencia Artificial*.

---

### Innovaciones clave de PostgreSQL 18

PostgreSQL 18, lanzado el 25 de septiembre de 2025, fue un gran paso adelante para la comunidad de bases de datos de código abierto. Muchas de las mejoras se centran en el rendimiento, la seguridad y las mejoras internas (autenticación OAuth 2.0, validación del modo FIPS, obsolescencia de la autenticación MD5, aplicación paralela de transacciones de replicación lógica y E/S asincrónica) y no formaron parte del alcance de este libro.

Pudimos ilustrar las siguientes características en este libro:

- **Restricciones temporales** que controlan las superposiciones de rangos de fechas y horas.
- **Valores OLD/NEW en la cláusula RETURNING** para comandos `INSERT`, `UPDATE`, `DELETE` y `MERGE`, lo que permite rastrear y razonar sobre los valores antes y después de las transacciones.
- **UUIDv7**, un Identificador Único Universal ordenado por tiempo que reduce la fragmentación de índices y tiende a mejorar la recuperación en operaciones de lectura secuenciales.
- **Visibilidad de paralelismo en `pg_stat_statements`**: en nuestras estadísticas de consultas, mostramos cómo la instrumentación más reciente te ayuda a razonar sobre la ejecución paralela planificada frente a la realizada, lo cual es fundamental para diagnosticar *"¿por qué esta consulta dejó de escalar?"*.
- **La función `casefold()`**, a menudo utilizada en el contexto de la sentencia `LIKE`, proporciona una forma global y compatible con Unicode de trabajar con cadenas independientemente de mayúsculas y minúsculas.

PostgreSQL 18 contiene innovaciones adicionales dignas de mención, aunque no las cubrimos en este libro:

- **Columnas generadas virtuales** que calculan valores en el momento de la consulta y pueden simplificar los esquemas cuando un valor derivado está orientado principalmente a la lectura.
- **Exploración por saltos (*Skip scan*) para índices B-tree multicolumna**, que puede mejorar el uso del índice para ciertos patrones de consulta y reducir las exploraciones innecesarias.
- **Intercalación `PG_UNICODE_FAST` y compatibilidad con `LIKE` no determinista**, que amplían la comparación de cadenas y el comportamiento de coincidencia de patrones con corrección Unicode en diferentes idiomas.
- **Mejoras en las actualizaciones**, incluidas estadísticas preservadas del planificador y mejoras en `pg_upgrade`, que reducen la "caída de rendimiento posterior a la actualización" y aceleran las transiciones de versiones mayores.
- **Sumas de comprobación de página (*Page checksums*) habilitadas de forma predeterminada en `initdb`**, lo que fortalece la detección de corrupción de referencia en clústeres recién inicializados.

---

### PostgreSQL y la revolución de la IA

PostgreSQL se ha ganado los corazones y las mentes de los desarrolladores de aplicaciones transaccionales y ha logrado grandes avances en el espacio analítico. Una tendencia similar está ocurriendo en la IA. Si bien los desarrollos iniciales dependían de plataformas de datos especializadas como Pinecone o Weaviate, con el auge de la IA generativa, la extensión `pgvector` se ha convertido en el estándar para almacenar *embeddings* vectoriales en aplicaciones de Generación Aumentada por Recuperación (RAG). Esta tendencia está impulsada por desarrolladores que no querían poner en marcha una nueva base de datos de nicho solo para vectores; simplemente instalaron una extensión en su Postgres existente. La simplificación operativa y los modelos de licencia simplificados también contribuyeron a este desarrollo.

RAG hizo famosos a los vectores al resolver un problema muy específico: *"Encuentra fragmentos de texto relevantes y luego deja que el modelo responda"*. Eso es poderoso, pero está orientado principalmente a documentos; tu evidencia consiste en párrafos y pasajes. En el momento en que tus preguntas involucran precios, ofertas actuales, ventanas de validez, reglas de inventario, derechos de clientes o límites de cumplimiento, los documentos se convierten en un sustrato débil. Los datos relacionales son diferentes: están moldeados por restricciones, claves y gobernanza. Cuando incorporas *embeddings* a PostgreSQL, no solo recuperas "texto similar". Recuperas **verdad estructurada**, filas que ya viven dentro de las reglas de tu sistema.

En el [Capítulo 18](https://subscription.packtpub.com/book/data/9781806028474/18), *Enfoque Paso a Paso para Integrar LLMs con PostgreSQL para Crear Aplicaciones de IA Completas* y el [Capítulo 19](https://subscription.packtpub.com/book/data/9781806028474/19), *Patrones de Canalización de Embeddings de IA Listos para Producción*, describimos cómo se utiliza PostgreSQL, junto con `pgvector`, para desarrollar una aplicación de IA generativa que integra capacidades de LLM para transformar la salida de consultas SQL en lenguaje natural. Este chatbot es un primer paso para reducir la brecha entre la base de datos y el usuario humano.

Este es un cambio importante: el asistente no responde a partir de texto extraído (*scraped*) o documentos poco gobernados. Responde a partir de conjuntos de resultados SQL. Eso significa que el sistema puede imponer límites, reglas de unión y restricciones comerciales; luego, el trabajo del modelo se convierte en lo que debe ser: un traductor y un explicador, no un inventor. En otras palabras, la base de datos tiene la autoridad; el modelo se convierte en el narrador.

El [Capítulo 20](https://subscription.packtpub.com/book/data/9781806028474/20), *PostgreSQL y MCP: El Modelo para un Asistente de IA Robusto*, mostró cómo podemos dar el siguiente paso utilizando la IA para facilitar que los usuarios hagan preguntas. En lugar de exigir SQL para consultar la base de datos, mostramos cómo los usuarios pueden consultarla en lenguaje natural. El chatbot utiliza un LLM para generar SQL, verifica la seguridad del SQL generado, ejecuta la consulta y luego utiliza otro LLM para transformar la salida en lenguaje natural.

En este punto, un lector podría preguntarse razonablemente: *"¿No es este ya un asistente que utiliza herramientas? ¿Qué agrega realmente el MCP?"*. La respuesta es: lo que construimos es un patrón; el MCP es el contrato estándar que hace que ese patrón sea seguro de escalar a través de herramientas, equipos y sistemas. Nuestro "asistente de consultas gobernadas" valida SQL y devuelve filas de evidencia, lo cual es excelente, pero el contrato entre el modelo y la herramienta sigue siendo en gran medida implícito, incrustado en prompts y lógica de funciones. El MCP convierte ese contrato implícito en uno explícito: el modelo solicita una acción utilizando un esquema de herramienta definido; el sistema hace cumplir la política, se ejecuta bajo la identidad correcta y devuelve resultados tipados con metadatos de auditoría. Esa diferencia se vuelve decisiva en el momento en que agregas más herramientas, más conjuntos de datos, más roles y más presión de producción.

La integración subyacente de SQL y significado es fundamental. Mientras que las primeras aplicaciones de IA, centradas principalmente en la IA generativa y los chatbots, trabajaban con texto y documentos no estructurados, la perspectiva de trabajar con datos almacenados en bases de datos relacionales ayudará a los proyectos de IA a superar algunos de sus mayores obstáculos:

- **Calidad de los datos**: Los datos de texto y documentos, especialmente los datos extraídos de sitios web o repositorios internos de documentos, tienden a ser de menor calidad que los datos transaccionales administrados en bases de datos relacionales. Los datos relacionales han sido considerados de misión crítica durante muchos años, y las aplicaciones de IA que aprovechan estos datos tienden a tener una mayor confiabilidad. La integración de SQL en las consultas también proporciona límites estrictos, lo que reduce aún más la posibilidad de alucinaciones.
- **Gobernanza de datos**: Las bases de datos relacionales tienen control de acceso basado en roles (RBAC) incorporado y procesos de gobernanza sólidos. La IA que aprovecha estos datos se beneficiará enormemente de esa base.

Una forma útil de pensarlo es como **"dos verdades cooperando"**. La búsqueda semántica te da *recall* (recuperación); te ayuda a encontrar lo que se siente relevante. SQL te da precisión: impone lo que está permitido, actualizado, es válido y cumple las normas. Cuando estos dos trabajan juntos, obtienes asistentes que se sienten naturales de usar y confiables de operar.

Además de las ventajas de calidad y gobernanza, la integración de la IA y las bases de datos abre el uso conjunto para lo siguiente:

- **Bases de datos con IA**: Usar la IA para democratizar el acceso a los datos de modo que los analistas puedan hacer preguntas sin necesidad de un conocimiento detallado sobre el esquema o SQL.
- **Bases de datos para IA**: Usar datos que están almacenados en bases de datos relacionales para impulsar la IA generativa y aplicaciones agénticas más complejas.
- **IA para la base de datos**: Usar la IA para mejorar la base de datos en sí, por ejemplo, analizando la estructura de la base de datos en busca de índices faltantes o haciendo recomendaciones de optimización.

¡La conexión de las capacidades de IA con las propiedades y procesos tradicionales de las bases de datos empresariales abre posibilidades completamente nuevas!

Hasta ahora, nos hemos centrado en cómo la IA interactúa con la base de datos, pero el potencial es mayor: la IA puede usar información de una base de datos para generar instrucciones para cambiar otra base de datos. Por ejemplo, un análisis inteligente de ventas indica que ciertos precios deben reducirse, y la IA reenvía esa instrucción a la base de datos de precios de productos en lugar de informar a un analista y esperar que este actúe sobre la información; o la IA detecta un sentimiento negativo del cliente sobre el producto A y reduce su volumen de nuevos pedidos.

Aquí está el punto de inflexión: una vez que un sistema de IA puede actuar, no solo responder, necesitas una gobernanza que sobreviva al éxito. Los prompts son consejos. La producción necesita contratos. Necesitas entradas de herramientas tipadas, identidades de ejecución con permisos, cumplimiento de políticas y registros de auditoría que puedan decirte quién preguntó qué, qué herramienta se ejecutó, qué datos se devolvieron y qué sucedió a continuación. Esa es exactamente la brecha que el MCP fue diseñado para cerrar.

Aquí es donde interviene el MCP (ver [Capítulo 20](https://subscription.packtpub.com/book/data/9781806028474/20), *PostgreSQL y MCP: El Modelo para un Asistente de IA Robusto*). En lugar de que los agentes de IA interactúen directamente con las bases de datos y otros sistemas comerciales, el MCP proporciona una interfaz segura y auditada que controla sus acciones y coordina las acciones de los agentes y herramientas de IA.

---

### PostgreSQL: Una plataforma de datos para el futuro

PostgreSQL se ha ganado los corazones y las mentes de los desarrolladores empresariales y se ha consolidado como la base de datos número 1 utilizada y preferida por los desarrolladores (ver [Capítulo 1](https://subscription.packtpub.com/book/data/9781806028474/1), *Sistemas Empresariales Transaccionales, Analíticos y de IA y sus Requisitos de Bases de Datos*). Durante muchos años, PostgreSQL ha sido la elección indiscutible para aplicaciones transaccionales, pero históricamente no se consideraba para análisis a nivel empresarial. Sus capacidades introducidas en los últimos 10 años (vistas materializadas, `CUBE`, `ROLLUP`, `GROUPING SETS`, CTE recursivos con detección de ciclos, índices BRIN, etc.) lo convierten en un serio contendiente. Desarrollos importantes en el ecosistema de PostgreSQL, como TimescaleDB (una extensión de PostgreSQL para análisis en tiempo real de alto rendimiento en series temporales y datos de eventos), WarehousePG (solución de almacenamiento de datos con procesamiento paralelo masivo MPP basada en PostgreSQL), pg_lake (integración del lago de datos abierto Iceberg con PostgreSQL), ParadeDB (búsqueda tipo Elasticsearch con puntuación BM25 para PostgreSQL) y pg_duckdb (que integra el motor analítico columnar-vectorizado DuckDB en PostgreSQL), están ampliando los límites analíticos, permitiendo que las aplicaciones "hablen Postgres" para cargas de trabajo analíticas cada vez más grandes y complejas.

De manera similar, `pgvector` y las incorporaciones recientes que escalan `pgvector`, como Scalable Nearest Neighbors (SCANN) y Disk-based Approximate Nearest Neighbors (DiskANN), posicionan a PostgreSQL como la plataforma líder para operaciones de IA basadas en vectores. La capacidad única de PostgreSQL para combinar IA con SQL lo convierte en una opción poderosa para soluciones de IA.

Lo que hace que esta combinación sea inusualmente poderosa no es que PostgreSQL pueda "manejar vectores". Es que PostgreSQL puede manejar **vectores + uniones + restricciones + gobernanza en un solo lugar**. Puedes recuperar por significado, filtrar por política y devolver filas de evidencia con linaje, todo mientras mantienes el modelo operativo familiar para las empresas. Así es como la IA pasa de la experimentación a la adopción: no por ser ingeniosa, sino por ser gobernable.

Las aplicaciones empresariales modernas deben combinar capacidades transaccionales, analíticas y de IA de una manera casi en tiempo real, escalable, rentable y ágil. PostgreSQL, su ecosistema y las recientes innovaciones en analítica e IA posicionan a PostgreSQL como la opción número 1 para una plataforma de datos empresariales preparada para el futuro.

> [!TIP]
> **¡Simplemente usa Postgres y haz que las cosas sucedan!**
