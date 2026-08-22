## Capítulo 17: Razonamiento Multimodal: Combinación de Vectores de IA con SQL Estándar

A medida que las cargas de trabajo de IA se generalizan, plantean un nuevo desafío para las bases de datos. En la era de la IA, las bases de datos no solo almacenan datos y manejan transacciones comerciales; también deben comprender el significado del contenido polimórfico que almacenan y aprovechar su semántica para obtener nuevos conocimientos.

Como vimos en el capítulo anterior, los vectores son una herramienta poderosa para capturar el significado incrustado de los datos, y los cálculos basados en vectores son los componentes básicos para los motores de recomendación y muchas otras aplicaciones que dependen del procesamiento semántico basado en IA.

En este capítulo, mostraremos que combinar el procesamiento semántico basado en IA con operaciones relacionales en PostgreSQL crea un motor de razonamiento unificado capaz de impulsar la búsqueda, la personalización, la detección de anomalías, las recomendaciones y los sistemas de decisión asistidos por IA a escala.

Exploraremos el razonamiento multimodal, los diseños de consultas híbridas de alto rendimiento, las estrategias de indexación, los modelos de clasificación (*ranking*), los casos de uso del mundo real y sus implicaciones arquitectónicas. Al final, comprenderás por qué SQL y los vectores deben estar juntos, y por qué separarlos en diferentes sistemas introduce complejidad innecesaria, costos y sobrecarga cognitiva.

Este capítulo cubre los siguientes temas:

- El poder del razonamiento multimodal
- Las razones por las cuales los vectores y los datos SQL pertenecen a la misma plataforma
- Pautas para diseñar consultas híbridas que abarquen vectores y datos SQL
- Ejemplos del mundo real
- Una discusión arquitectónica de sistemas más simples con gobernanza unificada

> [!NOTE]
> **Tu compra incluye una copia gratuita en PDF + paquete de código**  
> Tu compra incluye una copia en PDF sin DRM de este libro, el paquete de código y extras exclusivos adicionales. Consulta la sección de beneficios gratuitos con tu libro en el Prefacio para desbloquearlos instantáneamente y maximizar tu aprendizaje.

---

### El poder del razonamiento multimodal

Las bases de datos vectoriales introdujeron una capacidad radicalmente nueva: las máquinas ahora pueden trabajar con el significado. Los vectores se utilizan para almacenar y procesar la semántica incrustada (también conocida como significado) de datos, palabras, textos e imágenes.

Los vectores convierten palabras, descripciones y contexto en matemáticas, haciendo de la "similitud" algo que podemos computar. En lugar de buscar coincidencias exactas de números y palabras, podemos hacer coincidir la intención, el tono y el concepto, incluso cuando la redacción difiere.

Pero el significado por sí solo no es suficiente, porque la similitud es solo el primer paso. Los sistemas reales no se limitan a encontrar cosas. Deben decidir qué es válido, seguro, disponible y relevante en este preciso momento.

Un resultado que es una coincidencia semánticamente perfecta, pero que está agotado, fuera de región, con sobreprecio o retirado del mercado no es inteligencia: es fricción vestida con una bata de laboratorio.

Imagina pedirle al sistema: *"Muéstrame atuendos masculinos para una ocasión formal"*. Una base de datos vectorial devolverá artículos con descripciones semánticamente similares. Encontrará artículos con el mismo caso de uso (ocasión formal), características comparables (ropa para hombre) o un estilo coincidente (formal) porque sus *embeddings* se encuentran cercanos en el espacio vectorial.

Pero las aplicaciones reales no se detienen ahí. Preguntan lo siguiente:

- **Atuendos masculinos para una ocasión formal y en stock**: La similitud está bien, pero el inventario es un requisito estricto en el comercio electrónico. El cliente solo puede comprar lo que está disponible ahora, por lo que la consulta debe unirse a la tabla de inventario y filtrar por artículos vendibles.
- **Atuendos masculinos para una ocasión formal con un precio inferior a 500 $**: El significado no tiene presupuesto. Los clientes sí. El sistema debe respetar las restricciones de precios, a menudo utilizando reglas de precios actuales (no rangos vencidos), promociones y lógica de precios regional.
- **Atuendos masculinos disponibles en mi región**: La relevancia no es global; es local. La elegibilidad de envío, la cobertura del almacén, las restricciones regulatorias y los catálogos regionales determinan lo que realmente significa "disponible".
- **Atuendos masculinos, pero solo modelos con certificación ecológica**: Los artículos similares no siempre son artículos aceptables. Las certificaciones, las etiquetas de cumplimiento y los atributos de sostenibilidad a menudo se almacenan como metadatos estructurados y deben aplicarse como filtros estrictos.

En todos los escenarios anteriores, la IA nos ayudó a identificar productos que coincidían con la intención del usuario, y el procesamiento SQL nos ayudó a conectarnos con la realidad del negocio. Eso es el razonamiento multimodal: combinar la inteligencia semántica con la toma de decisiones relacional.

#### Lo que los vectores aportan

El [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, nos mostró que podemos usar vectores para convertir el significado de una oración, por ejemplo, la descripción de un producto o un ticket de soporte, en un conjunto de números que representan su significado. Esos números actúan como la huella digital semántica.

La huella digital semántica y los cálculos vectoriales basados en IA nos ayudan a identificar lo siguiente:

- **Similitud de significado**: Ayuda al sistema a reconocer que "zapatillas para correr" y "zapatillas de jogging" son esencialmente la misma idea, aunque las palabras difieran.
- **Relación contextual**: No solo sabe lo que significa una palabra, sino también qué suele acompañarla. Por ejemplo, aprende que "cancelación de ruido" a menudo va con "auriculares", o que "sendero" a menudo va con "agarre" y "resistente al agua".
- **Estructuras latentes**: Los vectores pueden captar patrones que son difíciles de describir con reglas, por ejemplo, agrupar quejas de clientes que suenan diferentes pero en realidad corresponden al mismo problema. O encontrar productos que "se ven similares" cuando se usan vectores de imágenes.
- **Representaciones numéricas para el razonamiento de LLM**: La IA moderna (por ejemplo, los LLM) trabaja con números. Los vectores son una forma limpia de alimentar el significado en los flujos de trabajo de IA, de modo que el sistema pueda recuperar, comparar, clasificar y razonar.

Los vectores nos dicen qué cosas se sienten similares entre sí. Esto significa que el sistema puede encontrar "primos cercanos" de un concepto, no solo coincidencias exactas de palabras clave.

Los vectores nos ayudan a responder preguntas difusas, tales como:

- *"Muéstrame productos como este."*
- *"Encuentra problemas similares en tickets de soporte."*
- *"¿Qué contenido coincide con la intención de esta consulta?"*

Los vectores hacen que la base de datos se parezca menos a un bibliotecario estricto y más a un asistente de tienda servicial que entiende lo que querías decir.

#### Lo que SQL aporta

Si los vectores son la intuición de la base de datos ("esto se siente similar"), entonces SQL es su disciplina ("solo muestra lo que es válido, permitido y procesable"). SQL es el lenguaje que nos permite hacer preguntas claras y exactas y obtener respuestas confiables y repetibles.

Esto es lo que SQL nos brinda:

- **Filtros de precisión**: Actúan como casillas de verificación al comprar: solo portátiles de menos de 500 $, solo talla 42, solo en stock. SQL hace esto con condiciones `WHERE`.
- **Condiciones estrictas**: Reglas que deben ser verdaderas, no "suficientemente cercanas". Por ejemplo, el precio debe ser actual, el estado debe ser `AVAILABLE` y la región debe ser `ASIA`. Si no coincide, no aparece.
- **Agregaciones**: SQL puede resumir datos, como *¿Cuántos pedidos hubo hoy? ¿Cuál es el precio promedio? ¿Ventas totales por marca?* Esas son las agregaciones `COUNT`, `AVG`, `SUM`, etc., que revisamos en el [Capítulo 13](https://subscription.packtpub.com/book/data/9781806028474/13), *Funciones Analíticas de PostgreSQL*.
- **Agrupaciones y resúmenes (*rollups*)**: Pueden organizar resúmenes por categorías, como ventas por ciudad, problemas por línea de productos e inventario por almacén. Estos son los operadores de agrupación discutidos en el [Capítulo 13](https://subscription.packtpub.com/book/data/9781806028474/13), *Funciones Analíticas de PostgreSQL*.
- **Uniones (*joins*) a través de datos diversos**: Este es el superpoder de SQL: conectar información relacionada almacenada en diferentes tablas. Por ejemplo, combinar producto, precio actual, inventario, marca y país de origen en un solo resultado. Sin los operadores `JOIN`, tendríamos hechos dispersos. Con los joins, obtenemos una historia completa.
- **Seguridad y gobernanza**: SQL puede controlar quién puede ver qué. La seguridad a nivel de fila (*Row-Level Security*) es como decir: "Solo puedes ver los clientes de tu región". La seguridad a nivel de columna filtra las propiedades a las que no tienes acceso.
- **Auditoría**: El registro de auditoría de la base de datos es como llevar un registro de quién preguntó qué y cuándo, esencial para industrias reguladas y para la confianza interna.
- **Transacciones y restricciones**: Las transacciones compatibles con ACID garantizan que las actualizaciones se realicen de forma segura: o todo tiene éxito o nada cambia. Las restricciones imponen reglas sobre los datos: el precio no puede ser negativo, el producto debe tener un precio antes de poder venderse y cada producto debe tener un identificador único.
- **Décadas de comportamiento de rendimiento predecible**: Las bases de datos SQL (especialmente PostgreSQL) se han optimizado para manejar grandes volúmenes de datos y picos de tráfico. SQL proporciona herramientas maduras: índices, planes de consulta, optimizadores, monitores: elementos que mantienen los sistemas estables bajo presión.

SQL responde preguntas específicas y devuelve datos sólidos. Mientras que los vectores y la IA nos ayudan a encontrar elementos semánticamente similares, SQL se asegura de que los valores sean correctos, seguros, conformes a las normativas y utilizables en el mundo real.

#### Razonamiento multimodal: Vectores y SQL

Aquí es donde las cosas se ponen interesantes. Los vectores ayudan al sistema a comprender el significado; SQL garantiza que la respuesta se ajuste a reglas estrictas y rápidas del mundo real. Cuando combinamos ambas técnicas, surge la magia práctica, y los "resultados interesantes" se convierten en "resultados que realmente se pueden usar".

Podemos solicitar: *"Encuentra productos similares a esta descripción, pero solo los que cuesten menos de 500 $ en este momento, solo las versiones ecológicas, y clasifícalos de modo que las mejores coincidencias y el mejor valor aparezcan primero"*.

Esto es lo que sucede en términos simples:

- Los vectores manejan la parte de "similar a esto" (como un asistente inteligente que comprende la intención).
- SQL maneja las partes de "menos de 500 $", "el precio debe ser actual" y "solo con certificación ecológica" (como una política estricta de la tienda que mantiene la honestidad de los resultados).
- La clasificación (*ranking*) mezcla tanto la "coincidencia más cercana" como la "mejor oferta".

O podemos solicitar: *"Muéstrame productos que coincidan con esta consulta, pero solo productos electrónicos, solo de esta marca, y elimina cualquier producto que haya sido retirado del mercado"*.

Esto es lo que sucede en términos simples:

- Los vectores encuentran artículos que se sienten relacionados con tu búsqueda.
- SQL asegura que solo veas lo que es relevante para la categoría, la marca y las reglas de seguridad.
- Los artículos retirados del mercado quedan excluidos, sin importar cuán "similares" sean, porque la seguridad prevalece sobre la similitud.

La IA estrecha el campo conceptual. Reduce un catálogo enorme a "cosas que parecen relevantes".

SQL afina la acción operativa. Aplica las restricciones reales: precio, disponibilidad, cumplimiento normativo, seguridad, región y permisos, y permite al usuario realizar el pedido, pagar y completar la compra.

Las consultas multimodales nos ayudan a transformar datos sin procesar en decisiones comerciales ejecutables.

---

### Por qué los vectores y SQL pertenecen a la misma plataforma

Al desarrollar un sistema moderno que combina procesamiento basado en IA y consultas SQL estándar, los desarrolladores y arquitectos se debaten sobre si los *embeddings* y el procesamiento vectorial basado en IA deben residir en la base de datos central (donde se gestionan conceptos convencionales como productos, precios, inventario, clientes y pedidos) o si el procesamiento de IA debe delegarse en un sistema especializado.

Muchos proyectos pioneros habilitados para IA optaron por crear un almacén vectorial (*vector store*) independiente para los *embeddings*, los cálculos de vecinos más cercanos y otros procesos de IA. Elegir una tecnología especializada, de última generación e innovadora parecía atractivo, como comprar una herramienta brillante para un problema nuevo.

Pero aquí está la trampa: dividir el procesamiento de datos entre múltiples componentes fractura la arquitectura y conlleva desventajas significativas. La arquitectura dividida crea lo siguiente:

- **Dos sistemas**: Una base de datos para datos empresariales (productos, precios y clientes) y otra base de datos para *embeddings* (significado/similitud).
- **Dos modelos de consistencia**: Los datos (por ejemplo, productos) deben actualizarse en varios lugares. Cuando las descripciones de los productos cambian en la base de datos central, ¿cambian también los *embeddings* correspondientes? De repente, la verdad puede quedar desincronizada.
- **Dos modelos de seguridad**: Los usuarios, permisos y reglas de auditoría deben mantenerse en múltiples lugares. Alguien podría tener acceso a los *embeddings* pero no a los datos reales, o viceversa. Eso es un dolor de cabeza de gobernanza.
- **Dos perfiles de rendimiento**: Un sistema es rápido para transacciones y uniones; el otro es rápido para la búsqueda por similitud. Pero la carga de trabajo real requiere ambos a la vez, lo que a menudo implica llamadas de red adicionales y mayor latencia.
- **Dos dominios de falla**: Si cualquiera de los dos sistemas se ralentiza o falla, toda la aplicación se rompe o se degrada. La arquitectura dividida duplicó los lugares donde las cosas pueden salir mal. Las copias de seguridad y la recuperación, la alta disponibilidad y las actualizaciones sin tiempo de inactividad se convierten en una pesadilla.
- **Dos dominios de administración**: Dos conjuntos de licencias, dos ciclos de parches, dos conjuntos de habilidades, dos procedimientos de copia de seguridad y recuperación, y así sucesivamente.

Y luego aparece el costo humano:

- Los científicos de datos generan *embeddings* en un sistema.
- La aplicación lee metadatos (precio, stock y categoría) de otro.
- El equipo de operaciones debe ejecutar y monitorear ambos.
- Los auditores deben verificar las políticas en dos mundos.
- Los desarrolladores deben capacitarse en ambas tecnologías.
- El sistema necesita licencias y soporte para dos tecnologías.

Esas son muchas piezas móviles. Una decisión aparentemente obvia de utilizar tecnología especializada se ha convertido en una carga pesada que complica la vida de todos.

La arquitectura extensible de PostgreSQL nos permite evitar estos problemas. La extensión `pgvector` introducida en el [Capítulo 16](https://subscription.packtpub.com/book/data/9781806028474/16), *Vectores e Indexación para IA con pgvector*, facilita almacenar y trabajar con vectores de IA de hasta 16.000 dimensiones (e indexar hasta 2.000 dimensiones) en PostgreSQL, y gestionar el procesamiento de IA y el SQL estándar en el mismo contexto transaccional, con la misma herramienta, los mismos procedimientos operativos, en el mismo hardware y bajo la misma licencia permisiva.

Postgres elimina la división. Y cuando eliminamos la división, eliminamos toda una categoría de problemas: trabajos de sincronización, seguridad desajustada, latencia adicional e investigaciones de "¿por qué los resultados son incorrectos hoy?".

#### La localidad gana

La localidad es una forma de decir: mantén juntas las cosas relacionadas. Si los datos del producto (nombre, descripción, precio, stock y región) están en un lugar y el *embedding* del producto (su significado) está en otro lugar, estamos constantemente transportando información de un lado a otro.

Pero cuando los *embeddings* se almacenan junto con los datos operativos en la misma base de datos PostgreSQL, la vida se vuelve más simple y más rápida.

Esto es lo que significa en términos sencillos:

- **Sin duplicación**: No duplicamos los detalles del producto en dos bases de datos. Esto crea una única fuente de verdad. Menos margen de error, menos desorden.
- **Sin trabajos de sincronización**: No necesitamos crear canalizaciones de datos ni tareas cron para mantener actualizado el almacén vectorial. Nada de pánico a las 2 AM por "¿se ejecutó la sincronización?". Nada de "acabamos de actualizar las descripciones de los productos para la venta de Black Friday, ¿puedes volver a ejecutar la sincronización?".
- **Sin vectores obsoletos**: Obsoleto significa desactualizado. Si la descripción de un producto cambia, su *embedding* también debería cambiar. Mantenerlos juntos hace que sea mucho menos probable que el sistema busque el significado de ayer utilizando los datos SQL de hoy.
- **Sin código personalizado entre sistemas para fusionar resultados de búsqueda**: Si todos los datos están en la misma base de datos, podemos usar `JOIN` para conectar el *embedding*, el precio actual, el inventario y la marca; no hay necesidad de código personalizado para fusionar los resultados de la base de datos SQL con los resultados de la base de datos de IA. Si los *embeddings* están en otro lugar, te ves obligado a hacer esa conexión a través de redes y código de aplicación: costoso de desarrollar, lento, frágil y difícil de depurar.
- **Sin explosión de complejidad**: Dos sistemas no solo duplican el trabajo, lo multiplican: monitoreo, guías de operación (*runbooks*), capacitación, control de acceso, copias de seguridad, actualizaciones, manejo de fallas, administración de costos, resolución de problemas... todo multiplicado por dos.
- **Todo vive bajo un mismo paraguas transaccional**: PostgreSQL trata los datos y su *embedding* como una sola unidad consistente. Las transacciones siempre son compatibles con ACID, las reglas de seguridad son consistentes y las consultas pueden tomar decisiones de una sola vez.

En resumen, cuando el significado (datos vectoriales) y los metadatos (datos estructurados) conviven, la arquitectura deja de luchar contra ti.

#### Mejor consistencia

La consistencia simplemente significa que el sistema no debe contradecirse a sí mismo y que las reglas de consistencia deben aplicarse automáticamente. Cualquier transacción inconsistente debe ser rechazada de inmediato. La consistencia está en el corazón de las definiciones ACID que revisamos en el [Capítulo 5](https://subscription.packtpub.com/book/data/9781806028474/5), *Transacciones, Cumplimiento ACID y Normalización de Datos*.

Los *embeddings* (vectores) se crean a partir de datos comerciales SQL, como descripciones de productos, notas de clientes o informes de incidentes, y deben mantenerse sincronizados. De lo contrario, si los datos SQL cambian pero el *embedding* no lo hace, la búsqueda y las recomendaciones comienzan a utilizar significados obsoletos con los datos SQL actuales, y el sistema se vuelve inconsistente.

PostgreSQL (y, de hecho, cualquier base de datos relacional compatible con ACID) ayuda a abordar este problema:

- **Las actualizaciones de productos y de *embeddings* pueden ser compatibles con ACID**, garantizando un modelo de todo o nada. Cuando actualizamos la descripción de un producto y regeneramos su *embedding*, Postgres puede asegurar que ambos cambios ocurran en la misma transacción y se hagan visibles al mismo tiempo. Esto evita inconsistencias donde descripciones nuevas coexistan con *embeddings* antiguos.
- **Las políticas de seguridad se aplican de manera uniforme**, garantizando que las mismas reglas de acceso protejan a ambos. Si un usuario no tiene permiso para ver productos, clientes o registros específicos, tampoco debería poder buscar en *embeddings* que los revelen indirectamente.
- **Las auditorías y los registros reflejan una única realidad**, proporcionando una sola pista de auditoría documental, una sola línea de tiempo y una sola verdad, sin necesidad de conciliar marcas de tiempo ni fusionar datos de registros y auditorías.

En resumen, PostgreSQL hace que sea mucho más fácil mantener el significado y los datos alineados, seguros y demostrables.

#### Mejor rendimiento

El rendimiento se reduce principalmente a una cosa: cuánto trabajo puede hacer el sistema y qué tan lejos deben viajar los datos:

- **Los saltos de red son costosos**: Un salto de red ocurre cuando la aplicación debe saltar de un sistema a otro a través de la red (por ejemplo, desde tu base de datos principal a una base de datos vectorial y viceversa). Incluso si cada salto toma solo milisegundos, se acumula rápidamente, especialmente a escala.
- **Transmitir *embeddings* entre sistemas es muy costoso**: Los *embeddings* son listas grandes de números, ¡y un solo vector con 2.000 dimensiones ocupará 8 KB, incluso si la descripción original tiene solo 256 caracteres! Si mantienes el *embedding* en un sistema separado, a menudo terminas moviendo muchos de esos datos, ya sea para sincronizar, buscar o unir con datos comerciales. Eso genera costos de ancho de banda, latencia y dificultades operativas.

Cuando SQL y los vectores se ejecutan dentro del mismo motor de PostgreSQL, suceden tres cosas buenas:

1. **Los cálculos de distancia ocurren cerca de la memoria**: Las matemáticas de similitud (qué tan cerca están dos vectores) ocurren justo donde residen los datos SQL. No hay necesidad de enviar el *embedding* a través de la red, lo que significa menos retraso y menos sobrecarga.
2. **Los filtros reducen los conjuntos de candidatos de manera eficiente**: El planificador de SQL puede usar estadísticas para reducir rápidamente el espacio de búsqueda. Por ejemplo, aplicando primero filtros SQL (`category = Electronics`, `current price < $500`, `in stock`, `region = ASIA`), y luego calculando la similitud solo en el conjunto de datos más pequeño.
3. **Las operaciones de unión evitan la latencia entre sistemas**: Las respuestas reales generalmente requieren uniones entre múltiples conjuntos de datos: *embeddings*, productos, precios actuales, inventario disponible y otros atributos. Si los vectores están en otro sistema, te ves obligado a realizar uniones lentas y complejas en el código de la aplicación. Si todo está en PostgreSQL, las uniones son locales y rápidas: una sola consulta, un solo plan de ejecución.

En resumen, mantener los vectores y SQL juntos reduce los desplazamientos, reduce el trabajo y reduce las esperas. Menos "obtener y ensamblar", más "consultar y decidir".

#### Mejor gobernanza

La gobernanza es una forma de decir: ¿podemos confiar en este sistema, demostrar lo que sucedió y controlar quién puede hacer qué? En industrias reguladas como la banca, los seguros, la salud, las telecomunicaciones y el gobierno, no solo necesitamos respuestas: necesitamos respuestas explicables y demostrables.

Por eso les importan los siguientes aspectos:

- **Linaje claro (*Clear lineage*)**: El linaje explica el origen de un resultado. ¿Qué tablas, qué filas, qué transformaciones y qué *embeddings* se utilizaron para derivar la respuesta? El linaje es la capacidad de rastrear un resultado hasta su origen, casi como un recibo de los datos.
- **Pistas de auditoría (*Audit trails*)**: Una pista de auditoría es un registro que documenta quién consultó qué, quién modificó qué, cuándo y desde dónde. Esto importa cuando los reguladores hacen preguntas o cuando algo sale mal y se requiere rendición de cuentas.
- **Resultados reproducibles**: Cuando volvemos a ejecutar la misma consulta con los mismos datos, deberíamos obtener el mismo resultado. Eso es crucial para el cumplimiento normativo, las investigaciones y las revisiones de imparcialidad.
- **Acceso controlado**: No todos deberían ver todo. Los sistemas empresariales necesitan un control detallado, a veces hasta filas y columnas específicas, para que los usuarios solo vean aquello a lo que tienen autorización.

Y aquí está la simple verdad: una sola base de datos es intrínsecamente más fácil de gobernar porque tratamos con un único lugar para permisos, registros, políticas, copias de seguridad, retención de datos y revisiones de cumplimiento.

Cuando los *embeddings* residen fuera de la base de datos, la gobernanza se convierte en un problema de dos mundos: dos conjuntos de políticas, dos pistas de auditoría y el doble de posibilidades de pasar algo por alto.

*Figura 17.1: Las ventajas de integrar vectores de IA y SQL*

> [!NOTE]
> **Descarga las imágenes en color**  
> Tu compra incluye una copia en PDF en color y sin DRM de este libro, ideal para ver imágenes en color, capturas de pantalla y diagramas. Consulta la sección de beneficios gratuitos con tu libro al final del Prefacio para desbloquear tu copia en PDF.

La Figura 17.1 resume por qué los vectores de IA y SQL deben administrarse en la misma plataforma siempre que sea posible: mejor localidad, consistencia, gobernanza y rendimiento. Los vectores dentro de PostgreSQL no son una moda pasajera. Lo hemos visto con datos SIG (*GIS*) y JSON, y con la búsqueda de texto completo; la extensibilidad inherente de PostgreSQL absorbe nuevas capacidades y proporciona un entorno de gestión de datos coherente y confiable.

---

### Diseño de consultas híbridas de alto rendimiento

Ahora que hemos establecido la necesidad de administrar vectores de IA y SQL en el mismo entorno, entremos en la sala de máquinas y descubramos cómo hacer que estas consultas y transacciones híbridas multimodales sean rápidas y prácticas.

Una consulta híbrida es simplemente una declaración que hace lo siguiente:

- Utiliza vectores para encontrar cosas que son similares en significado.
- Utiliza SQL para garantizar que los resultados sean válidos en el mundo real (precio, stock, categoría, región y permisos).

Esto es similar a lo que vimos en el [Capítulo 14](https://subscription.packtpub.com/book/data/9781806028474/14), *Análisis de Documentos de Texto en PostgreSQL*, donde combinamos vectores de búsqueda de texto con consultas SQL convencionales.

Las consultas híbridas de IA y SQL con mejor rendimiento suelen seguir la misma receta:

1. **Restringir el conjunto de datos con filtros SQL**: Primero, reduce el espacio de búsqueda con reglas simples: "Solo Electrónica", "Solo precio actual inferior a 500 $", "Solo en stock" y "Solo mi región". Esto es económico y rápido, como reducir la búsqueda a un solo pasillo de la tienda antes de comenzar a comparar productos.
2. **Clasificar las filas supervivientes con similitud vectorial**: Ahora que la lista es más pequeña, usa vectores para clasificar lo que queda por significado más cercano. Esta es la parte computacionalmente costosa, por lo que deseas aplicarla a menos elementos.
3. **Opcionalmente combinar métricas de SQL y vectores en una puntuación final**: A veces, lo "más similar" no es suficiente. Es posible que deseemos resultados que también sean más baratos, más populares, más nuevos o con mejores calificaciones. Para lograr esto, creamos una puntuación combinada: 70% de similitud + 30% de valor comercial.
4. **Devolver resultados clasificados**: Finalmente, devuelve los mejores resultados, incluyendo opcionalmente detalles como la puntuación de distancia, el precio, la categoría y el motivo por el cual coincidió. Esos metadatos ayudan a la depuración, el ajuste y permiten al usuario confiar en los resultados.

Este patrón parece sencillo, pero ejecutarlo bien depende de comprender cómo PostgreSQL ejecuta realmente una consulta entre bastidores, porque el orden de las operaciones importa. Si accidentalmente hacemos que PostgreSQL calcule distancias vectoriales para todo antes de filtrar, la consulta puede ralentizarse significativamente. Pero si filtramos primero y clasificamos después, el rendimiento se mantiene fluido, incluso a escala.

Una buena consulta híbrida puede parecer magia para el usuario, pero construirla es ingeniería básica: cuidadosa, ordenada y eficiente.

#### Principio 1: Filtrar temprano, clasificar después

La búsqueda vectorial es potente, pero también es un trabajo pesado para la computadora. La búsqueda vectorial significa que la base de datos realiza operaciones matemáticas para medir cuán similar es cada fila a la consulta, ¡y cada fila y consulta puede constar de hasta 4.000 números! Si le pedimos que haga esas operaciones en un conjunto de datos no restringido, es como pedirle a alguien que compare tu rostro con el de cada habitante de la ciudad antes de decidir quién se parece a ti.

Los filtros SQL, por otro lado, son más económicos y rápidos, especialmente cuando pueden aprovechar índices. Son como decir: "Primero mira solo a las personas de este vecindario". Es por eso que es muy recomendable filtrar temprano y clasificar después.

En los siguientes ejemplos, el parámetro posicional `$1` representa el *embedding* del texto de la consulta.

A continuación se muestra una ilustración del patrón correcto, donde filtramos primero:

```sql
SELECT * FROM products WHERE category = 'Electronics' ORDER BY embedding <-> $1 LIMIT 20;
```

Esto es lo que sucede aquí:

1. Primero, SQL aplica el filtro `Electronics` y reduce el posible conjunto de resultados.
2. Luego, la similitud vectorial se calcula solo para esas filas.
3. Después, limitamos el conjunto de resultados a los 20 mejores.

Esto significa que la comparación de vectores y el cálculo de distancia, computacionalmente intensivos, se realizan en un conjunto de datos mucho más pequeño (consulta el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Diseño para Altos Volúmenes de Transacciones y Escritura de Código Transaccional Eficiente*, para obtener detalles sobre el orden de procesamiento de los bloques de construcción de una consulta SQL).

Este es un ejemplo de un patrón incorrecto, donde se clasifica todo:

```sql
SELECT * FROM products ORDER BY embedding <-> $1 LIMIT 20;
```

Esto puede parecer inofensivo, pero es costoso:

- Postgres debe calcular la distancia (es decir, la similitud con `$1`) para cada producto.
- Todos los productos se ordenarán según la métrica de distancia que acabamos de calcular.
- Solo después de hacer todo ese trabajo podrá seleccionar los 20 mejores.

Este proceso se volverá extremadamente lento a medida que crezca la tabla.

Una analogía simple explica la diferencia entre los dos enfoques:

- **Consulta buena**: *"Primero ve al pasillo de Electrónica, luego encuentra los 20 artículos más similares"*.
- **Consulta mala**: *"Escanea toda la tienda, calcula la similitud para cada artículo, luego elige 20"*.

Por eso es fundamental filtrar temprano y clasificar después para mantener las búsquedas híbridas rápidas, escalables y asequibles.

#### Principio 2: Dejar que cada índice haga su trabajo

Los índices de bases de datos son como el índice temático de un libro:

- Sin un índice temático, el lector debe hojear cada página para encontrar lo que busca.
- Con un índice temático, puede saltar directamente a la sección correcta, minimizando el tiempo de búsqueda.

En búsquedas híbridas que involucran SQL y vectores, nos encontramos haciendo dos tipos diferentes de preguntas:

- **Preguntas estructuradas (reglas rápidas de sí/no)**, como `category = Electronics`, `price < 500`, `current = true`.
- **Preguntas de significado (similitud)**, como *"Encuentra productos similares"*.

PostgreSQL sobresale en ambos aspectos, pero utiliza índices diferentes para cada uno:

- **Índice B-tree/BRIN para filtros estructurados**, como *"Encuentra todo lo que esté en el pasillo de electrónica"*. Esto es rápido porque los índices B-tree y BRIN están diseñados para coincidencias exactas y rangos.
- **Índice IVFFlat o HNSW para similitud vectorial**, como *"Dentro de ese pasillo, encuentra artículos que sean como un altavoz portátil Sonos"*. Este índice está diseñado para búsquedas rápidas de vecinos más cercanos en el espacio vectorial.

Aquí hay algunos ejemplos:

```sql
CREATE INDEX idx_products_category ON products(category);
```
*Define un índice B-tree (el índice predeterminado en PostgreSQL) en la columna `category` de la tabla `products`.*

```sql
CREATE INDEX idx_products_embedding ON products USING ivfflat (embedding vector_l2_ops) WITH (lists = 200);
```
*Crea un índice IVFFlat en la columna `embedding` para la tabla `products`, lo optimiza para búsquedas L2 (distancia euclidiana) y proporciona 200 depósitos (*buckets*) para agrupar elementos.*

El planificador de consultas de PostgreSQL los utiliza conjuntamente cuando la estructura de la consulta lo permite:

- `idx_products_category` puede ayudar a podar el conjunto de datos y reducir los resultados a la categoría correcta (como caminar directamente al pasillo de Electrónica).
- `idx_products_embedding` se utiliza para clasificar a los candidatos y encontrar rápidamente las coincidencias más cercanas dentro de ese conjunto más pequeño sobre las filas restantes (como comparar artículos similares solo en ese pasillo, no en toda la tienda).

El resultado combina velocidad y relevancia:

- **Velocidad** porque no buscamos a ciegas en toda la tabla.
- **Relevancia** porque utilizamos la similitud semántica para clasificar lo que realmente importa.

En resumen, no obligues a un solo índice a hacer dos trabajos. Deja que los índices SQL manejen las reglas de sí/no y enfoca los índices vectoriales en el significado.

#### Principio 3: Usar JOIN para construir inteligencia multitable

El operador `JOIN` es la forma en que la base de datos conecta información relacionada almacenada en diferentes tablas. He aquí un ejemplo:

- Una tabla tiene productos (nombre, descripción, *embedding*).
- Otra tabla tiene inventario (niveles de stock).
- Otra tabla tiene usuarios (sus preferencias, ubicación, nivel de membresía).

Una respuesta real generalmente necesita piezas de todas ellas, y ahí es donde usamos las uniones.

##### Ejemplo 1

En este ejemplo, queremos productos similares, pero solo si están en stock y disponibles para la venta (*"Muéstrame productos similares a este... pero solo los que estén en stock"*):

```sql
SELECT p.product_id, p.name, inv.stock_level, p.embedding <-> $1 AS distance FROM products p JOIN inventory inv ON inv.product_id = p.product_id WHERE inv.stock_level > 0 ORDER BY distance LIMIT 10;
```

El SQL anterior hace tres cosas:

1. Encuentra productos que son "cercanos en significado" (similitud vectorial).
2. Verifica el inventario para mantener solo los artículos que están en stock.
3. Devuelve las mejores coincidencias que están disponibles para su compra.

Para obtener detalles sobre el orden de ejecución de SQL, consulta el [Capítulo 7](https://subscription.packtpub.com/book/data/9781806028474/7), *Diseño para Altos Volúmenes de Transacciones y Escritura de Código Transaccional Eficiente*.

Combinar los resultados de ambas búsquedas en PostgreSQL es fácil y devuelve un resultado útil. Esto se complica cuando ambas tablas se almacenan en sistemas diferentes. Si los *embeddings* residen en una base de datos vectorial separada, el código de la aplicación debe implementar los siguientes pasos engorrosos:

1. Pedir a la base de datos vectorial los "productos más similares".
2. Llevar esos IDs de vuelta a PostgreSQL.
3. Unirlos con el inventario disponible utilizando una unión hash o una unión por combinación codificada de forma personalizada.
4. Si demasiados artículos están agotados, volver a la base de datos vectorial para obtener más opciones posibles.

Este enfoque es más lento, más desordenado y más difícil de mantener consistente. Cuando los vectores de IA viven dentro de PostgreSQL junto con el resto de los datos sobre productos, precios, inventarios y transacciones, se trata de una sola consulta, un solo plan y una sola verdad.

##### Ejemplo 2

Este ejemplo requiere personalización: hacer coincidir los productos con el gusto de un usuario. Una consulta de personalización podría pedir *"Selecciona productos que coincidan con las preferencias de este usuario"* (`u.preference_vec` en la siguiente consulta hipotética):

```sql
SELECT p.product_id, p.name, p.embedding <-> u.preference_vec AS distance FROM users u JOIN products p ON TRUE WHERE u.user_id = 42 ORDER BY distance LIMIT 15;
```

Reescribamos esto en un lenguaje sencillo:

- El usuario tiene un vector de preferencias: `u.preference_vec` (la huella digital de sus gustos).
- Cada producto tiene un vector de *embedding*: `p.embedding` (su huella digital característica).
- La base de datos los compara y devuelve las coincidencias más cercanas. Este es el patrón central detrás de los sistemas de personalización: *"A las personas como tú también les gustó..."* o *"Recomendado para ti..."*. Puedes combinar fácilmente esto con uniones adicionales para limitar por precio objetivo, disponibilidad de inventario, etc.

En resumen, los joins son la forma de convertir la similitud vectorial en inteligencia del mundo real, porque la respuesta final casi siempre depende de más factores que la simple similitud. También necesita considerar otros hechos concretos, como inventario, precio, región, elegibilidad, cumplimiento y contexto del usuario.

#### Principio 4: Utilizar modelos de puntuación híbridos

A veces, "lo más similar" no es lo mismo que "lo mejor". Una búsqueda vectorial puede decirte qué artículos son los más cercanos en significado a tu consulta. Pero en aplicaciones reales, también te importan las señales comerciales como popularidad, precio, margen, calificaciones, frescura (artículos más nuevos), riesgo y marcas de cumplimiento.

Un modelo de puntuación híbrido es simplemente una forma de combinar múltiples cálculos de similitud impulsados por IA con otras calificaciones impulsadas por SQL en una puntuación de clasificación final.

Piénsalo como juzgar un concurso con múltiples criterios:

- **60%**: ¿Qué tan bien coincide con lo que el usuario quiere decir?
- **40%**: ¿Qué tan rentable es para el negocio?

La siguiente consulta hipotética ilustra cómo se podrían combinar múltiples modelos de puntuación:

```sql
SELECT p.product_id, p.name, (0.6 * (1 - (p.embedding <-> $1))) + (0.4 * p.margin_score) AS final_score FROM products p WHERE p.category = 'Swimwear' ORDER BY final_score DESC LIMIT 10;
```

El SQL anterior hace lo siguiente:

1. Encuentra trajes de baño que son similares a lo que se solicitó.
2. También impulsa las prendas con alto margen.
3. Luego, combina ambos en un solo número llamado `final_score`.
4. Finalmente, ordena por esa puntuación y devuelve los 10 mejores.

Los resultados no son solo "la coincidencia más cercana", son "la coincidencia más cercana y con mayor probabilidad de ser una recomendación rentable".

Este método te permite incorporar ambos elementos:

- **Cercanía semántica** (similitud vectorial: lo que se siente como una coincidencia).
- **Relevancia comercial** (señales derivadas de SQL: lo que es confiable, valioso o rentable).

Y esta idea se aplica mucho más allá del comercio minorista:

- **Sugerencias de reparación**: Problema similar + mayor riesgo primero.
- **Reclamaciones de seguros**: Reclamación similar + mayor riesgo de fraude primero.
- **Rutas**: Destino similar + menor costo + mayor seguridad.

Postgres proporciona una solución elegante porque puede calcular la similitud, aplicar reglas, combinar puntuaciones y devolver resultados clasificados, todo en una sola consulta y en un solo motor.

#### Principio 5: Empujar el cómputo a la base de datos

Con demasiada frecuencia, los datos se trasladan a la aplicación y es allí donde se realizan muchos cálculos. Alentamos a los desarrolladores a pensar de manera diferente: no hagas que la aplicación haga el trabajo de la base de datos. Empuja muchos cálculos a la base de datos cerca de los datos, especialmente aquellos donde el planificador puede aprovechar estadísticas para crear un plan óptimo que minimice la E/S y la CPU.

Un error común (el antipatrón) se ve así:

1. Realizar la "búsqueda de similitud" en una base de datos vectorial separada.
2. Extraer los mejores resultados (IDs) hacia tu aplicación.
3. Aplicar reglas de negocio en el código (precio, stock, región y categoría).
4. Obtener más detalles de tu base de datos relacional principal.
5. Reclasificar todo nuevamente en la aplicación.

Este enfoque funciona, pero es muy ineficiente y costoso. Causa lo siguiente:

- **Mayor latencia**: Los datos rebotan entre sistemas: `app -> vector DB -> app -> Postgres -> app`. Cada salto añade tiempo.
- **Inconsistencia**: La base de datos vectorial y la base de datos relacional pueden desincronizarse. Podrías recomendar algo que está agotado o al que ese usuario no tiene acceso.
- **Complejidad del código**: Tu aplicación se convierte en un mini motor de consultas que maneja filtrado, unión, clasificación y reintentos. Esa lógica se duplica en varios servicios, lo que dificulta las pruebas.
- **Sobrecarga operativa**: Estás ejecutando y monitoreando dos bases de datos y sus canalizaciones de sincronización. Más bloques de construcción significan más cosas que pueden fallar.

El patrón correcto es hacerlo en una sola consulta SQL en PostgreSQL.

En ese patrón ocurre lo siguiente:

- SQL maneja las reglas estrictas (categoría, precio actual, stock, región y permisos).
- Los vectores de IA manejan la clasificación por similitud.
- PostgreSQL maneja las uniones y la puntuación.
- La base de datos devuelve los resultados finales clasificados.

¿Por qué es preferible este patrón?

- **El planificador y el optimizador hacen el trabajo**: PostgreSQL utiliza sus estadísticas y motor de planificación para crear el plan más eficiente, lo que incluye qué índices usar, qué filtros aplicar primero y cómo unir tablas.
- **La base de datos proporciona garantías transaccionales ACID**: Los resultados son consistentes con el estado actual de la base de datos: sin *embeddings* obsoletos ni precios desactualizados.
- **La arquitectura se vuelve más saludable**: Tenemos menos piezas móviles y menos integraciones personalizadas con procesos dispares. Esto viene acompañado de una menor carga operativa y una gobernanza mucho más sencilla.

*Figura 17.2: Cinco principios del patrón de diseño para consultas híbridas de alto rendimiento*

La Figura 17.2 resume los cinco patrones de diseño para consultas híbridas de alto rendimiento:

1. Filtrar temprano, clasificar después.
2. Dejar que cada índice haga su trabajo.
3. Usar joins para construir inteligencia multitable.
4. Usar modelos de puntuación híbridos.
5. Empujar el cómputo a la base de datos.

En resumen: un solo motor, una sola consulta, una sola verdad. Deja que la aplicación haga las preguntas, deja que PostgreSQL haga el trabajo pesado con los datos y obtén las respuestas.

---

### Ejemplos y patrones del mundo real

Hasta ahora, hemos hablado de cómo deberían hacerse las cosas. Ahora, veamos el código correspondiente para algunos ejemplos: detección de fraude en el comercio minorista, análisis de registros de mantenimiento y predicción de abandono de clientes (*churn*) en telecomunicaciones.

#### Comercio minorista: Búsqueda semántica y restricciones de inventario

En casos de uso reales, la gente no pide *"Muéstrame artículos similares"* y se detiene ahí. Piden artículos que sean similares y que también cumplan con restricciones de la vida real, como tamaño, precio y disponibilidad. Estos son patrones que verás en todas partes, no solo en el comercio minorista.

Por ejemplo, un usuario dice: *"Muéstrame zapatillas de entrenamiento similares a esta, en mi talla, por menos de 150 $"*.

Esta consulta hipotética hace exactamente eso:

```sql
SELECT name, price, size, stock_count, description_vec <-> $1 AS distance FROM shoes WHERE size = 11 AND price < 150 AND stock_count > 0 ORDER BY distance LIMIT 12;
```

Repasemos cada una de las cláusulas de la consulta:

- `description_vec <-> $1`: Esta es la parte de "coincidencia de significado". Compara la descripción del calzado del usuario (convertida en un vector) con el vector de descripción de cada zapato y mide qué tan similares son.
- `WHERE size = 11`: Solo muestra zapatos en la talla del usuario. Sin resultados inútiles.
- `AND price < 150`: Solo muestra zapatos dentro del presupuesto.
- `AND stock_count > 0`: Solo muestra zapatos que están disponibles para comprar en este momento.
- `ORDER BY distance`: Coloca los zapatos más similares primero.
- `LIMIT 12`: Devuelve solo los 12 mejores resultados, como una página de búsqueda limpia.

Esto es poderoso porque el resultado está listo para su uso inmediato por parte de la aplicación. No hay necesidad de recuperar resultados y luego filtrarlos en el código de la aplicación. La base de datos ya devuelve las respuestas correctas, de forma rápida y limpia.

En otras palabras, los vectores encuentran el "concepto adecuado". SQL garantiza que se ajuste a las "reglas del mundo real".

#### Banca: Similitud en casos de fraude

La detección y el análisis de fraudes es una de las principales áreas de aplicación del aprendizaje automático y la IA. En la banca, los investigadores a menudo quieren responder a una pregunta como: *"¿Hemos visto casos de fraude como este antes?"*.

La parte de "como este" es compleja porque las notas sobre fraudes se redactan en lenguaje natural. Dos casos pueden describir el mismo patrón utilizando palabras totalmente diferentes. Ahí es donde los vectores ayudan.

Aquí hay una consulta hipotética que compara el *embedding* del caso de fraude actual (representado como `$1`) con casos de fraude conocidos y sus *embeddings*:

```sql
SELECT case_id, amount, region_code, notes_vec <-> $1 AS distance FROM fraud_cases WHERE amount > 10000 AND status = 'OPEN' AND region_code = 'W1' ORDER BY similarity ASC LIMIT 20;
```

Repasemos las cláusulas de la consulta:

- `notes_vec <-> $1 AS distance`: Esto compara el significado de las notas del caso actual (el vector `$1`) con los *embeddings* de notas de casos pasados almacenados como vectores. Encuentra casos que suenan diferentes pero significan lo mismo y están semánticamente cerca entre sí, es decir, tienen una distancia euclidiana pequeña (`<->`).
- `WHERE amount > 10000`: Solo examina casos de alto valor (porque lo que está en juego es mayor).
- `AND status = 'OPEN'`: Solo muestra casos que aún están activos y necesitan atención.
- `AND region_code = 'W1'`: Solo incluye casos de una región específica (tal vez debido a reglas locales o al alcance de la investigación).
- `ORDER BY distance ASC LIMIT 20;`: Muestra primero los 20 casos más similares.

Esto es cercanía semántica más reglas estrictas en un solo lugar:

- Los vectores ayudan a los investigadores a encontrar patrones rápidamente.
- SQL asegura que los resultados cumplan con las restricciones operativas y regulatorias (región, estado, umbrales).

Esto no se trata solo de encontrar casos similares; se trata de encontrar casos relevantes y procesables.

#### Manufactura: Análisis de registros de mantenimiento

Los registros de mantenimiento combinan datos de IoT (como temperatura, vibraciones, horas de funcionamiento, velocidad de rotación, etc.) y texto ingresado por los técnicos durante la instalación, el mantenimiento y la reparación. Dos técnicos pueden describir el mismo problema utilizando palabras completamente diferentes.

En la manufactura, a menudo recibimos solicitudes para encontrar problemas similares e identificar soluciones exitosas, por ejemplo: *"Estamos experimentando este fallo ahora mismo. ¿Hemos visto algo similar antes y qué lo solucionó?"*.

Los vectores de IA nos ayudarán a manejar la entrada de texto manual y a hacer coincidir patrones de datos.

Esta consulta hipotética va un paso más allá: también prefiere registros recientes, porque una solución de la semana pasada suele ser más relevante que una de hace cinco años:

```sql
SELECT log_id, event_timestamp, (0.7 * (1 - (log_vec <-> $1))) + (0.3 * freshness_score(event_timestamp)) AS score FROM maintenance_logs WHERE equipment_type = 'AXLE' ORDER BY score DESC LIMIT 5;
```

Repasemos los detalles de la consulta:

- `WHERE equipment_type = 'AXLE'`: Solo busca registros para el mismo tipo de equipo. Un problema de eje no debe compararse con un problema de ventilador de enfriamiento.
- `(1 - (log_vec <-> $1))`: Esta es la parte de "similitud". La base de datos compara el significado del informe de falla actual (`$1`) con el vector de cada registro. Más similar equivale a una puntuación más alta.
- `freshness_score(event_timestamp)`: Esto otorga puntos adicionales a los registros más nuevos. Es un "impulso por novedad".
- `0.7 * similarity + 0.3 * freshness`: Esto crea una puntuación combinada: 70% basada en la similitud y 30% basada en eventos recientes.
- `ORDER BY score DESC LIMIT 5`: Devuelve las cinco coincidencias más útiles, similares y recientes.

La combinación de búsqueda semántica con datos SQL nos ayudó a encontrar informes que pertenecen a la pieza específica, están descritos de manera similar por los técnicos y son recientes y probablemente relevantes.

#### Telecomunicaciones: Análisis de abandono de clientes (*Churn*)

El abandono de clientes (*churn*), es decir, los clientes que cancelan su contrato y se registran con otro proveedor, es una métrica clave en los negocios de suscripción. Los servicios de telefonía móvil son altamente competitivos, y una pregunta comercial común es: *"¿Qué clientes se parecen a este perfil... y están en riesgo de irse?"*.

Esta consulta hipotética hace exactamente eso utilizando vectores para la similitud y SQL para la acción empresarial:

```sql
SELECT c.customer_id, c.behavior_vec <-> $1 AS sim, churn_probability FROM customers c WHERE plan_type = 'PREMIUM' ORDER BY sim, churn_probability DESC LIMIT 10;
```

Analicemos los componentes de la consulta:

- `c.behavior_vec <-> $1 AS sim`: Cada cliente tiene un "vector de comportamiento" que representa patrones como uso, quejas, pagos, actividad en la aplicación, etc. `$1` es el vector del tipo de perfil de cliente que te interesa. La base de datos los compara y encuentra a los clientes cuyo comportamiento es más similar.
- `churn_probability`: Esto se ha definido en otro lugar en función de las tendencias de uso recientes, los pagos atrasados y el comportamiento de las quejas.
- `WHERE plan_type = 'PREMIUM'`: Solo busca clientes Premium (tal vez porque estás ejecutando una campaña de retención para ellos).
- `ORDER BY sim, churn_probability DESC`: Primero muestra los clientes más parecidos al perfil. Entre ellos, coloca a los que tienen una mayor "probabilidad de abandono" (más propensos a irse) en la parte superior, para que puedas actuar donde más importa.
- `LIMIT 10`: Devuelve los 10 objetivos principales.

Esto es poderoso porque no es solo analítica. Es un insumo directo para la toma de decisiones:

- Los vectores responden *"¿Quién más se comporta así?"*.
- SQL agrega *"Solo Premium"*.
- La puntuación de abandono agrega *"¿Quién necesita atención primero?"*.

Eso es IA y SQL en un solo flujo, una sola consulta que produce directamente una lista corta procesable para ofertas de retención, contacto directo o mejoras en el servicio.

---

### Arquitectura: Por qué esto es importante

En las secciones anteriores, revisamos los cinco principios para consultas híbridas de alto rendimiento y examinamos algunos ejemplos de consultas de comercio minorista, manufactura, banca y telecomunicaciones. Sin embargo, resolver este problema (o evitarlo) no se trata solo de escribir mejores consultas; se trata de construir sistemas que no se desmoronen cuando aparezcan el tráfico real, los datos reales y los plazos reales.

#### Los sistemas más simples escalan mejor

Cuando mantenemos todo en un solo sistema (PostgreSQL manejando tanto SQL como vectores), la vida se vuelve más fácil de formas muy prácticas:

- **Más fácil de implementar**: Utilizamos una única plataforma de base de datos, no dos. Menos piezas móviles significa menos sorpresas de "funcionó en staging pero falló en producción".
- **Más fácil de proteger**: Un solo lugar para administrar usuarios, roles, permisos y reglas de acceso. No hay necesidad de duplicar políticas en un almacén vectorial y una base de datos relacional esperando que se mantengan consistentes.
- **Más fácil de monitorear**: Un solo conjunto de paneles, registros, alertas y herramientas de ajuste del rendimiento. Cuando algo va lento, puedes encontrar la causa más rápido en lugar de jugar al "ping-pong de culpables" entre sistemas.
- **Menos puntos de falla**: Cada sistema adicional añade otro lugar donde las cosas pueden romperse: enlaces de red, canalizaciones de sincronización, autenticación, actualizaciones, copias de seguridad. Un solo sistema reduce la superficie de fallas, mejorando así la confiabilidad.
- **Menos código personalizado**: Todas las consultas se ejecutan en una sola base de datos; no es necesario desarrollar código personalizado que combine los resultados de diferentes bases de datos. Tampoco hay canalizaciones de datos para sincronizar múltiples sistemas de origen.

En resumen, las arquitecturas simples escalan mejor porque tienen menos partes móviles.

#### Gobernanza unificada

La gobernanza es un tema crítico en las soluciones de IA. La gobernanza se reduce a preguntas simples: ¿podemos demostrar lo que sucedió, controlar quién puede ver qué y mantenernos conformes a la normativa?

Cuando tus datos y tus *embeddings* viven en una sola base de datos, la gobernanza se vuelve mucho más simple porque solo hay un lugar para administrar y verificar.

Por eso a los auditores les encantan las arquitecturas simples:

- **Un solo sistema de registro**: Hay un registro único de actividad: quién consultó qué, quién modificó qué y cuándo sucedió. No hay necesidad de unir registros de múltiples sistemas y adivinar la historia completa.
- **Un solo motor de políticas**: Todas las reglas (permisos, seguridad a nivel de fila y políticas de retención) se definen y aplican en un solo lugar. No estás intentando mantener alineados dos sistemas diferentes.
- **Un solo mecanismo de acceso**: Los usuarios se autentican una vez, los roles se gestionan una vez y el acceso se concede una vez. No hay confusión de "tienen acceso aquí pero no allá".

A los equipos de riesgos les encantan los sistemas que no proliferan porque cada sistema adicional añade otro conjunto de credenciales, otro lugar donde los datos podrían filtrarse, otro alcance de auditoría y otro riesgo de actualización o mala configuración.

Un solo sistema es más fácil de confiar, más fácil de controlar y más fácil de demostrar ante auditorías.

#### El modelo relacional de PostgreSQL sigue estando preparado para el futuro

La razón por la que PostgreSQL sigue prosperando después de décadas se remonta a las decisiones de diseño iniciales tomadas por Michael Stonebraker y su equipo hace más de 30 años: el modelo objeto-relacional y su extensibilidad inherente han permitido a PostgreSQL mantenerse relevante al agregar nuevas capacidades sin sacrificar sus puntos fuertes fundamentales: confiabilidad, consistencia, seguridad y SQL.

A lo largo de los años, con una combinación de tecnología central y extensiones, PostgreSQL ha integrado nuevos e importantes estilos de datos y cargas de trabajo, tales como los siguientes:

- **JSON(B)** para almacenar y manejar datos semiestructurados y flexibles sin necesidad de una base de datos completamente nueva.
- **Búsqueda de texto completo** para buscar documentos y descripciones directamente en la base de datos.
- **Series temporales** para manejar datos de sensores, métricas y flujos de eventos de forma natural.
- **Almacenamiento en columnas** para permitir consultas analíticas más rápidas en grandes conjuntos de datos.
- **Replicación lógica** para copiar y sincronizar datos de forma segura y eficiente entre sistemas.
- **Clústeres distribuidos** que escalan horizontalmente para alta disponibilidad y cargas de trabajo globales.
- **Vectores de IA** para trabajar con el significado y la similitud, impulsando la búsqueda y recuperación de IA en el mismo lugar que los datos operativos.

Detrás de todo eso subyace una idea simple: el futuro favorece a las arquitecturas que integran, no a los diseños que fragmentan. Por lo general, las soluciones integradas tienden a ser más efectivas y eficientes que las arquitecturas fragmentadas con integraciones personalizadas, ya que los desafíos de fragmentación resultantes a menudo superan las ganancias de eficiencia locales.

En términos sencillos, las herramientas que mantienen todo conectado y consistente ganarán porque reducen la complejidad. Las herramientas que te obligan a unir múltiples plataformas tecnológicas para responder a una sola pregunta seguirán aumentando los costos, los riesgos y los retrasos.

---

### Resumen

Este capítulo demostró que combinar vectores y SQL dentro de PostgreSQL no solo es posible, sino estratégicamente ventajoso.

Los vectores aportan significado. SQL aporta estructura. Juntos, permiten el razonamiento multimodal.

Exploramos cómo las consultas híbridas permiten que las aplicaciones piensen semánticamente respetando la lógica comercial codificada en modelos relacionales. Estudiamos indexación, estrategias de filtrado, clasificación híbrida, uniones vectoriales y patrones del mundo real en el comercio minorista, las finanzas, las telecomunicaciones y la manufactura.

El mensaje es simple: la IA no elimina la necesidad de SQL. La IA hace que SQL sea más importante. Y SQL hace que la IA sea verdaderamente utilizable. Con `pgvector` y PostgreSQL, obtienes un motor único y unificado que puede recuperar, razonar, filtrar, clasificar y decidir, todo dentro de la comodidad y estabilidad de una plataforma relacional confiable. Esta fusión marca un nuevo capítulo en la evolución de las bases de datos: una base de datos que no solo almacena datos, sino que también los comprende.

El próximo capítulo nos enseña cómo generar *embeddings* para conjuntos de datos y consultas conectando PostgreSQL a LLMs alojados y cómo almacenar los vectores resultantes en PostgreSQL.
