# Luciano Vernazza — Portfolio

🇬🇧 [Read in English](README.md)

**Project Manager · Data Migrations & Technical Onboarding** en una plataforma SaaS de gestión para clínicas de cirugía plástica y estética · Argentina

Trabajo en el punto donde una clínica deja su sistema anterior y empieza a operar en una nueva plataforma. Ese pasaje incluye mover años de pacientes, turnos y documentos clínicos, y armar los formularios que la clínica va a usar desde el primer día. Mi trabajo es que ese proceso sea predecible. Para eso diseño y construyo herramientas que reemplazan pasos manuales, detectan problemas antes de que lleguen al cliente y le dan a leadership datos concretos para decidir.

> Todos los nombres de personas y clientes fueron anonimizados. Las métricas que corresponden a personas se muestran solo como porcentajes.

---

## Índice

1. [Contexto: cómo es una migración](#1-contexto-cómo-es-una-migración)
2. [Prevalidation App (Flask)](#2-prevalidation-app-flask)
3. [Notebooks de prevalidación en Azure Databricks](#3-notebooks-de-prevalidación-en-azure-databricks)
4. [TAP Mapper — CLI + Claude Skill](#4-tap-mapper--cli--claude-skill)
5. [Intake Form Editor](#5-intake-form-editor)
6. [EHR Template Converter](#6-ehr-template-converter)
7. [Pipeline de Fillable PDFs](#7-pipeline-de-fillable-pdfs)
8. [Consent Form Batch Uploader](#8-consent-form-batch-uploader)
9. [Automatización con n8n](#9-automatización-con-n8n)
10. [Migration Project Tracker](#10-migration-project-tracker)
11. [Project Database Updater — Claude Skill](#11-project-database-updater--claude-skill)
12. [Analytics de performance y KPIs](#12-analytics-de-performance-y-kpis)
13. [Matcher de archivos clínicos huérfanos](#13-matcher-de-archivos-clínicos-huérfanos)
14. [Scripts de soporte](#14-scripts-de-soporte)
15. [Documentación y knowledge transfer](#15-documentación-y-knowledge-transfer)
16. [Stack técnico](#16-stack-técnico)

---

## 1. Contexto: cómo es una migración

Cada migración pasa por cuatro etapas:

- **Staging:** una primera carga de prueba en la cuenta del cliente.
- **Gap:** los datos nuevos que se generaron desde el staging.
- **Go-Live:** la carga definitiva.
- **Files & Media:** los documentos clínicos, imágenes y adjuntos.

Los datos llegan desde muchos EHRs distintos y casi nunca vienen limpios: CSVs con comillas y comas mal puestas, encodings mezclados, columnas con nombres distintos según el sistema, valores inconsistentes y, en algunos casos, decenas de miles de tipos de turno diferentes.

Cuando empecé, buena parte de este proceso dependía de pasos manuales, de conocimiento que tenían pocas personas y de herramientas que solo podía usar alguien técnico. Cada proyecto de esta página ataca una parte concreta de ese problema.

| Etapa | Herramientas |
|---|---|
| Recepción y prevalidación de datos | Prevalidation App, notebooks de Databricks |
| Mapping de turnos | TAP Mapper |
| Formularios (intake, consent, templates clínicos) | Intake Form Editor, EHR Template Converter, Pipeline de Fillable PDFs, Batch Uploader |
| Files & Media | Matcher de archivos clínicos, reportes de validación |
| Seguimiento de proyectos | n8n, Migration Project Tracker, Project Database Updater |
| Leadership | Exportador de Linear, dashboards de performance, KPIs de formularios |

---

## 2. Prevalidation App (Flask)

**Stack:** Python · Flask · pandas · openpyxl · Construida con Claude Code

### El problema
Antes de migrar, hay que revisar los archivos del cliente para encontrar campos vacíos o valores que no tienen equivalente en la plataforma (un género escrito de cinco formas distintas, teléfonos con menos dígitos, fechas de nacimiento imposibles). Ese chequeo se hacía en notebooks de Azure Databricks, con varios problemas:

- Había que saber configurar widgets y elegir la notebook correcta según cuántos archivos mandaba el cliente.
- El mapping de columnas se hacía a mano en cada proyecto.
- El mapping de tipos de turno era un proceso separado, en otro archivo.
- Los Product Specialists, que validan ese mapping con el cliente, no podían participar sin pedirle a alguien técnico que corriera todo.

### La intención
Que cualquier persona del equipo, técnica o no, pudiera correr una prevalidación completa sin tocar código. También quería que el proceso fuera siempre el mismo, para que los resultados no dependieran de quién lo hacía, y que el Product Specialist pudiera entrar en su parte sin esperar a nadie.

### Cómo se resolvió
Construí una app web local que guía el proceso en cinco pasos, con dos roles:

- **Project Manager:** acceso completo, desde la creación del cliente hasta la descarga de reportes.
- **Product Specialist:** entra directo al mapping de turnos y ve los pasos anteriores en modo lectura.

**El flujo:**

1. **Carga de archivos.** La app detecta sola qué configuración corresponde:

   | Archivos | Modo |
   |---|---|
   | Patients | Patients-Only |
   | Patients + appointments | Standard |
   | Patients + 2 de appointments | Dual Appointments (columnas `[A]` / `[B]`) |
   | 2 de patients | Dual Patients |

2. **Column mapping.** Las columnas del cliente se asocian a los 24 campos estándar de pacientes y 15 de turnos. La app sugiere el mapping usando los nombres de columna habituales de cada EHR.
3. **Resultados.** Cada campo queda clasificado como ok, vacío o en conflicto. Los conflictos se abren en un modal y desde ahí se crean mapping rules ("M", "Masc" y "male" van a Male). Por diseño, un valor en conflicto no puede ser destino de una regla.
4. **Mapping de turnos**, integrado con el TAP Mapper (sección 4).
5. **Reportes:** un Word para el cliente, CSVs con los registros problemáticos, el archivo de mapping de turnos y un resumen de hallazgos en Excel.

La parte más difícil fue el parsing:

- **Que coincidiera con producción.** El mismo CSV daba resultados distintos según el programa con que se abriera, y descubrimos que LibreOffice interpretaba las comillas de otra manera. La app replica cómo lee el archivo el sistema de producción.
- **Archivos sucios.** Detecta el encoding (UTF-8, UTF-8 con BOM, UTF-16) y el delimitador. Para Excel corruptos prueba tres engines en cadena: openpyxl, xlrd y `read_html`.
- **Tests realistas.** Generé 14 CSVs de prueba a partir de mapping rules reales de clientes, para que la app enfrentara los mismos problemas que manda cada EHR.

### El impacto
La prevalidación dejó de depender de saber usar Databricks. El flujo es siempre el mismo y el Product Specialist trabaja en su parte sin intermediarios. El mapping de turnos pasó a ser un paso dentro del mismo proceso, en vez de un archivo aparte. La app se presentó en una demo al CTO, que pidió el repositorio para armar el pipeline de deploy y llevarla del uso local a un servicio del equipo. Cuando el código se perdió por accidente, se pudo reconstruir gracias a un documento de contexto de 16 secciones.

---

## 3. Notebooks de prevalidación en Azure Databricks

**Stack:** PySpark · pandas · SQL · Unity Catalog

### El problema
Las notebooks eran la herramienta de prevalidación del equipo, pero fallaban justo con los clientes más difíciles:

- **No contemplaban todos los formatos de envío.** Algunos EHRs mandan los turnos en dos archivos separados (tipo y estado en uno, el resto en otro).
- **Pandas descartaba sin avisar las filas rotas por comillas mal puestas.** La validación decía que todo estaba bien porque las filas problemáticas directamente no existían.
- **Algunos Excel hacían fallar la conversión a Spark** por tipos de datos mezclados.
- **Ciertas consultas se colgaban indefinidamente** cuando el catálogo del cliente estaba mal configurado.
- **El resumen de hallazgos tardaba muchísimo** con archivos grandes, porque se escribía celda por celda.

### La intención
Que las notebooks fueran confiables con cualquier archivo y nunca dieran un "todo ok" falso. También quería que no se colgaran y que cubrieran todas las variantes de envío que manejábamos, sin cambiar la lógica de validación que ya funcionaba.

### Cómo se resolvió
- **Parser propio para CSVs dañados.** Conserva todas las filas, aunque estén rotas, para que los valores desplazados aparezcan como conflictos en lugar de desaparecer.
- **Lectura de Excel como texto** (`dtype=str`) y fallback entre engines para archivos corruptos.
- **Timeout de 60 segundos** con `ThreadPoolExecutor` en las consultas a Spark. Si el catálogo está mal, la notebook avisa en lugar de colgarse.
- **Límite de filas en las hojas de detalle del Excel**, con referencia al CSV completo.
- **Tipos y propósitos de turno leídos desde la tabla real del cliente**, en vez de desde tablas que no existían en todos los catálogos.
- **Tres variantes mantenidas en paralelo:** estándar, una adaptada a un EHR específico y una para clientes que mandan dos archivos de turnos. Todos los cambios se aplican de forma consistente en las tres.

### El impacto
Las notebooks dejaron de dar falsos positivos con archivos mal formados, que eran justamente los que más riesgo tenían de romper una migración. Dejaron de colgarse y pasaron a cubrir clientes que antes había que procesar a mano. Además, todas las lecciones de parsing que salieron de este trabajo se trasladaron después a la Prevalidation App.

---

## 4. TAP Mapper — CLI + Claude Skill

**Stack:** Python · SQLite · openpyxl

### El problema
Cada clínica nombra sus turnos a su manera. Hay que traducir cada tipo y propósito de turno del sistema anterior a los valores estándar de la plataforma ("Botox f/u 2wk" pasa a ser un follow-up de neurotoxinas, por ejemplo). Algunos clientes tienen cientos de valores y otros, más de 10.000.

Se hacía a mano, fila por fila, y eso traía tres problemas:
- Llevaba muchísimo tiempo.
- Dos personas mapeaban el mismo valor de forma distinta.
- Las correcciones que hacían los especialistas no se reutilizaban en el siguiente cliente.

### La intención
Automatizar la mayor parte del mapping y que la herramienta aprendiera de cada proyecto, para que el siguiente cliente arrancara con más mapeado. Además, que quedara claro qué filas necesitaban revisión humana y por qué.

### Cómo se resolvió
- **CLI con una base de conocimiento en SQLite** que se alimenta de los mappings históricos. Las correcciones de los Product Specialists pesan tres veces más que el resto, porque son la referencia validada.
- **Cuatro comandos:** `map` (mapea un archivo nuevo y aprende), `import` (carga históricos), `corrections` (aprende de correcciones) y `status`.
- **Lógica de matching en varias pasadas:** primero términos específicos de procedimientos, después patrones clínicos más amplios, después lenguaje genérico de seguimiento y al final los casos administrativos. Los turnos combinados van al procedimiento principal.
- **Colores por nivel de confianza:** alta, a revisar o sin match. Cada fila dudosa lleva un comentario que explica la decisión, para que el especialista revise solo lo necesario.
- **Una Claude Skill** para usarlo en lenguaje natural, en inglés o en español.

### El impacto
Los archivos de mapping salen con entre 90% y 100% de cobertura automática, incluso en clientes con más de 10.000 propósitos. La revisión humana se concentra en las filas marcadas. El especialista recibe un archivo explicado en lugar de una planilla en blanco, y cada corrección mejora el resultado del cliente siguiente. Se usó en más de 10 clientes.

---

## 5. Intake Form Editor

**Stack:** Flask · React

### El problema
Los formularios de intake (los que completa el paciente antes de la consulta) se guardan como documentos JSON en una base NoSQL. Para cambiar una pregunta, agregar un campo o hacerlo obligatorio había que editar el JSON a mano en un editor de código y subirlo con Postman.

Eso traía varios problemas:
- Solo podía hacerlo alguien técnico.
- Un error de sintaxis rompía el formulario.
- No había forma de ver cómo iba a quedar antes de publicarlo.
- Cada cambio pedido por un cliente se volvía un pequeño proyecto.

### La intención
Que editar un formulario fuera tan simple como editar un documento. Quería ver el resultado mientras se edita, sin tocar JSON y sin riesgo de romper la estructura.

### Cómo se resolvió
Un editor visual con un parser bidireccional: lee el JSON real, lo convierte en un estado editable y, al exportar, genera el JSON completo, incluidos los campos técnicos.

- **Importación** desde archivos `.json` o `.http`, descartando automáticamente los headers HTTP.
- **Sidebar de secciones.** Los campos se agrupan por fila, así que dos campos en la misma fila se ven uno al lado del otro, igual que en el formulario real.
- **Edición de cada campo:** tipo (texto, fecha, teléfono, sí/no, dropdown, checkbox, firma, grilla, subtítulo), pregunta, obligatoriedad, opciones, posición y a qué sección de la historia clínica se mapea la respuesta.
- **Rich text** para los textos de consentimiento.
- **Preview** en iPhone, iPad, Android y desktop.
- **Borradores** para retomar el trabajo después.
- **API REST** con CRUD de formularios y endpoints de importación y exportación.

Se validó haciendo un round-trip completo (importar y volver a exportar) con un formulario real de más de 150 campos y 12 secciones, sin pérdida de datos.

### El impacto
Editar un formulario dejó de requerir conocimientos de JSON y dejó de tener riesgo de romperlo. El preview permite revisar el resultado antes de entregarlo, y los cambios que pide un cliente se resuelven en el editor, sin pasar por código. Además, abre la posibilidad de que personas no técnicas del equipo hagan ajustes que antes tenían que escalar.

---

## 6. EHR Template Converter

**Stack:** Claude Skill · Python · poppler · JavaScript

### El problema
Las clínicas mandan sus plantillas de notas clínicas en PDF y había que recrearlas en el editor de la plataforma. Antes de construir nada hice una investigación técnica (documentación interna, conversaciones del equipo y tickets) y encontré:

- **El proceso era totalmente manual.** Cada PDF se reconstruía click por click en un editor con limitaciones (por ejemplo, tablas de 10×10 como máximo).
- **No había API** para cargar plantillas de forma programática.
- **No había forma de copiar plantillas entre cuentas.** Era un problema concreto para grupos de clínicas que comparten plantillas.
- **El equipo creaba las plantillas para casi todos los clientes**, así que el cuello de botella era nuestro.

### La intención
Pasar de reconstruir cada plantilla a mano a convertirla automáticamente. También quería poder procesar lotes enteros y copiar plantillas entre cuentas sin rehacerlas.

### Cómo se resolvió
- **Ingeniería inversa del editor.** Con pruebas sistemáticas descubrí qué HTML sobrevive cuando se pega en el editor TinyMCE:
  - checkboxes nativos
  - campos rellenables no editables
  - colores con atributos `data-mce-style`
  - imágenes en base64
  - fondos de celda (se pierden, así que se resolvió envolviendo el contenido en un `span` con color)
- **Una Claude Skill** que recibe una carpeta de PDFs y devuelve una carpeta de HTMLs listos para pegar. Rasteriza cada página, extrae las imágenes embebidas y genera el HTML respetando esas reglas.
- **Un bookmarklet** que extrae la plantilla abierta en una cuenta como archivo HTML, para pegarla en otra.
- La skill quedó empaquetada para que otras personas del equipo la instalen y la usen.

### El impacto
La creación de plantillas pasó de reconstruir cada PDF a mano a convertir lotes completos y revisar el resultado. Copiar plantillas entre cuentas, que antes significaba rehacerlas, ahora es exportar y pegar. Y el conocimiento de qué acepta el editor quedó documentado en la skill, en vez de depender de la memoria de quien lo descubrió.

---

## 7. Pipeline de Fillable PDFs

**Stack:** Python · PyMuPDF

### El problema
Los consentimientos informados llegan como PDFs planos. Para que la plataforma los complete sola con los datos del paciente (nombre, fecha de nacimiento, proveedor, fecha del turno) y los firme digitalmente, cada campo tiene que existir como campo de formulario con el nombre exacto de la macro correspondiente.

Hacerlo a mano en un editor de PDF, formulario por formulario, llevaba muchísimo tiempo, y los clientes suelen mandar decenas de consentimientos juntos.

### La intención
Procesar lotes enteros de consentimientos de una vez. El objetivo era detectar automáticamente dónde va cada dato y cada firma, sin tapar texto del documento original y sin cambiar su diseño.

### Cómo se resolvió
Un pipeline de cinco etapas sobre una skill interna de generación de campos:

1. **Extracción.** Detecta las líneas para completar, sean underscores o líneas dibujadas, y las asocia con su etiqueta, esté arriba, abajo o al costado.
2. **Limpieza.** Elimina falsos positivos del cuerpo del texto y asocia etiquetas que quedan en bloques separados.
3. **Firmas.** Resuelve el patrón típico de firma sobre una línea con la etiqueta debajo.
4. **Solapamientos.** Ajusta cada campo para que no tape texto.
5. **Aplicación y verificación en lote.** Genera los PDFs finales y comprueba que todo quedó bien ubicado.

### El impacto
Lotes de 40 consentimientos, con más de 300 campos entre texto, firmas y checkboxes, se procesan en una sola corrida. Lo que antes era trabajo manual, formulario por formulario, pasó a ser una revisión del resultado. El diseño original de cada formulario se conserva y los campos quedan listos para el autocompletado de la plataforma.

---

## 8. Consent Form Batch Uploader

**Stack:** Python CLI · REST API

### El problema
Una vez preparados, los consentimientos había que subirlos uno por uno a la cuenta del cliente desde la interfaz web. Con lotes de decenas de formularios, eso llevaba mucho tiempo y era fácil equivocarse.

### La intención
Subir una carpeta completa de una sola vez, con la posibilidad de simular la carga antes de hacerla.

### Cómo se resolvió
Retomé una herramienta que había empezado un colega y la convertí en un CLI interactivo:
- Se arrastra la carpeta a la terminal.
- Se busca la cuenta del cliente por nombre o por ID.
- Se hace login y se ve un resumen antes de confirmar.
- Opciones: `--dry-run` para simular, demora entre subidas, filtros por extensión y estado inicial de los formularios.
- Incluye un set de 10 PDFs de prueba.

### El impacto
El flujo de carga en lote quedó resuelto del lado de la herramienta. Su uso en producción quedó bloqueado por un error de la API de la plataforma, que está identificado y documentado para el equipo de desarrollo.

---

## 9. Automatización con n8n

**Stack:** n8n · Linear · Notion · Slack · Fellow.ai

### El problema
La información de cada migración vivía en varios lugares: las tareas en Linear, el seguimiento en Notion, la comunicación en Slack y las decisiones en las reuniones. Mantener todo sincronizado era manual. Esto generaba varios problemas:
- Los estados quedaban desactualizados.
- Nadie se enteraba a tiempo cuando una tarea se bloqueaba.
- Los project managers del lado del cliente tenían que preguntar en qué estaba cada proyecto.

### La intención
Que Linear fuera la única fuente de verdad y que todo lo demás se actualizara solo. Las notificaciones relevantes tenían que llegar a Slack sin que nadie las escribiera.

### Cómo se resolvió
**Workflows en producción:**
- Etiquetado automático de issues en Linear
- Mensajes automáticos en Slack
- Avisos cuando una migración está lista para validar
- Transiciones de estado de proyectos
- Creación automática de sub-tareas según el EHR de origen

**Workflows diseñados:**
- Un router que sincroniza Linear con Notion
- Manejo de transiciones de fase
- Sincronización del calendario de hitos
- Registro automático de desde cuándo está bloqueada una tarea (cada 15 minutos)
- Un resumen diario en Slack
- Alertas de hitos próximos

**Decisiones de arquitectura:**
- La fase del proyecto se deriva de las tareas activas, no del estado del proyecto, porque el estado se actualizaba tarde.
- Cada issue se sincroniza 1:1 como tarea, lo que eliminó un workflow entero.
- Se usan los nodos nativos de Notion, con el ID de Linear como clave para evitar duplicados.

Para los workflows pendientes dejé una especificación de mapeo de campos, un documento de diseño y un brief para que Claude Code los construya de forma autónoma.

### El impacto
Las tareas repetitivas de etiquetado, aviso y cambio de estado dejaron de hacerse a mano. El equipo se entera de los cambios en Slack en el momento. Y quedó definida la arquitectura completa para que el seguimiento de proyectos en Notion se actualice solo a partir de Linear. En el camino pasé de usar n8n por primera vez a diseñar un sistema de varios workflows coordinados.

---

## 10. Migration Project Tracker

**Stack:** Notion

### El problema
Los project managers que acompañan a cada cliente no trabajan en Linear y no tenían una vista simple de en qué etapa estaba cada migración ni qué hitos venían. La información existía, pero repartida y en un formato pensado para developers.

### La intención
Una vista única del estado de todas las migraciones, adaptada a quien la mira: calendarios para los project managers y detalle de tareas para developers.

### Cómo se resolvió
Diseñé desde cero un sistema en Notion:
- **Dos bases relacionadas:** proyectos y tareas.
- **Una base de hitos** y un calendario general.
- **Vistas específicas:** un calendario por cada project manager y vistas de detalle para developers.

Se alimenta automáticamente desde Linear con los workflows de n8n y avisa en Slack.

### El impacto
Cada project manager ve en su calendario los hitos de sus clientes sin pedir actualizaciones. Developers y project managers miran la misma información, cada uno en el formato que le sirve, y desaparecen las preguntas de "¿en qué está tal migración?".

---

## 11. Project Database Updater — Claude Skill

**Stack:** Claude Skill · MCP (Notion, Linear, Fellow)

### El problema
Todo el equipo de implementación usa una base compartida de Notion con el estado de cada cliente. La sección de migración de datos tiene 24 campos: estado, fechas de staging y go-live, aprobación del mapping de turnos, recepción del export completo, etc. Se completaban a mano, así que en la práctica estaban vacíos o desactualizados, y otros equipos tomaban decisiones con información vieja.

### La intención
Que esos campos se mantuvieran al día sin trabajo manual, sacando la información de donde realmente está: el estado de las tareas en Linear y lo que se decide en las reuniones.

### Cómo se resolvió
Una Claude Skill que cruza Linear, los resúmenes de reuniones de Fellow y Notion:
- **Modo `initial`:** completa el historial de todos los proyectos activos.
- **Modo `weekly`:** actualiza solo lo que cambió en la semana.
- **Aprobación previa:** propone los cambios en una tabla y espera aprobación antes de aplicarlos.

A partir de la primera corrida se incorporaron a la skill reglas aprendidas:
- Siempre se consulta Linear primero, aunque el campo de Notion esté vacío.
- Una fecha vacía no significa que el evento no ocurrió.
- Ciertas etiquetas son equivalentes.
- Las reuniones se revisan en cada corrida.

### El impacto
Se hizo el backfill completo de todos los proyectos activos y desde entonces la base se actualiza cada semana. Los demás equipos consultan datos que reflejan el estado real. La misma integración se usó para reasignar en una sola operación decenas de proyectos cuando cambió el equipo de project managers, conservando el resto de su información.

---

## 12. Analytics de performance y KPIs

**Stack:** Python · Linear GraphQL API · React · PowerPoint · Notion

### El problema
Leadership necesitaba responder preguntas concretas: cuánto tarda una migración, cuánto retrabajo hay, si el equipo de formularios tiene capacidad para más clientes y dónde se traba el trabajo. Los datos estaban en Linear, pero la exportación estándar no servía:

- **Los tiempos no reflejaban el trabajo real.** Se calculaban en horas corridas, incluyendo noches y fines de semana.
- **Faltaban issues.** Los archivados quedaban afuera.
- **No había forma directa de contar los formularios** adjuntos a cada tarea.

### La intención
Tener métricas confiables y comparables en el tiempo para evaluar el desempeño del equipo, detectar cuellos de botella y reportar a leadership con datos y no con impresiones.

### Cómo se resolvió
- **Exportador de Linear vía GraphQL:**
  - Filtra del lado del servidor para no traer datos de más.
  - Recorre el historial completo de cada issue para calcular cuánto estuvo en cada estado.
  - Mide en horas hábiles e incluye los issues archivados.
  - Cuenta los archivos adjuntos leyendo los embeds de la descripción, incluidos los que están dentro de ZIPs.
- **Dashboards de performance** en React y presentaciones en PowerPoint con duración de proyectos, tiempo por tipo de tarea y rondas de validación.
- **KPIs de formularios en Notion:**
  - Limpieza del dataset: exclusión de outliers con IQR y de tareas que saltaron estados.
  - Gráficos de ingreso de tareas, volumen completado, eficiencia por tipo de formulario y backlog.
- **Un artifact en React** que consulta Linear a través de MCP y exporta a CSV sin manejar API keys.

### El impacto
- **Evaluaciones de desempeño con datos.** En una de ellas, los dashboards mostraron una reducción de entre 60% y 80% en el retrabajo a lo largo del tiempo.
- **El cuello de botella en formularios estaba en la espera en cola, no en el tiempo de trabajo.** Lo mostró el análisis de capacidad, y cambió la discusión de "trabajar más rápido" a "cómo se asigna el trabajo".
- **Seguimiento continuo.** El dashboard de KPIs quedó como herramienta permanente.

---

## 13. Matcher de archivos clínicos huérfanos

**Stack:** Python · pandas

### El problema
En la etapa de Files & Media de una migración aparecieron cientos de documentos clínicos (notas en RTF y correos en EML) que no se podían asociar a ningún paciente, así que no se podían migrar. El primer intento, cruzar por la ruta del archivo, no encontró ninguna coincidencia.

### La intención
Recuperar la mayor cantidad posible de esos archivos asociándolos a su paciente, y entender con precisión por qué el resto no se podía asociar, para informar al cliente con datos.

### Cómo se resolvió
Construí el script de cruce y dos scripts de diagnóstico:
- **El primer diagnóstico** confirmó que las rutas tenían el mismo formato en ambos archivos, así que el problema no era de formato.
- **El segundo** encontró la clave real: el ID numérico del documento, que aparece en el nombre del archivo y en el índice de documentos del sistema de origen.

Con eso, el script cruza en dos pasos: archivo → paciente en el índice de documentos, y paciente → ID en la nueva plataforma.

### El impacto
La identificación de archivos en el índice de origen pasó de 0% a cerca de 95%. El análisis también mostró que el cuello de botella real estaba en el segundo paso: muchos nombres de pacientes no coincidían con el reporte demográfico. Eso transformó un "no se pueden migrar" en un problema concreto y acotado para resolver con el cliente.

---

## 14. Scripts de soporte

Herramientas más chicas que resolvieron necesidades puntuales del día a día:

- **Generador de CSVs de test.** Para probar la Prevalidation App necesitaba datos que fallaran como fallan los reales. Revisé las mapping rules documentadas de varios clientes y generé 14 archivos sintéticos que reproducen los problemas típicos de cada EHR. Permitió testear la app sin usar datos de pacientes.
- **Contador de archivos en SFTP.** Para confirmar que un cliente subió todo antes de arrancar Files & Media hacía falta contar archivos en servidores con miles de carpetas. El script recorre el servidor de forma recursiva y pide las credenciales en tiempo de ejecución, sin argumentos, para que no queden en el historial de la terminal.
- **Merger de Excels.** Algunos sistemas exportan los datos partidos en muchos archivos. El script los une validando antes que todos tengan las mismas columnas, para no generar un archivo combinado corrupto sin darse cuenta.
- **Reportes de validación de exports.** Los reportes de Files & Media siguen un template en Word que no se puede alterar. Se completan editando directamente el XML del documento, lo que conserva el formato exacto y evita las inconsistencias de hacerlo a mano.

---

## 15. Documentación y knowledge transfer

### El problema
Mucho del conocimiento operativo (cómo se hacen ciertos tipos de migración, qué pedirle a cada EHR, cómo explicarle el proceso a un cliente) estaba en la cabeza de pocas personas o disperso en grabaciones de reuniones. Cuando una persona clave dejó el equipo, ese riesgo se volvió urgente.

### La intención
Convertir ese conocimiento en documentación que cualquiera pudiera seguir, y tener material listo tanto para el equipo interno como para clientes y ventas.

### Cómo se resolvió
- **Runbooks operativos** para tres tipos de migración, construidos a partir del historial de reuniones grabadas.
- **Guías de migración para clientes y ventas**, en dos versiones: una técnica y completa, y otra simplificada con diagramas propios para clientes que no necesitan el detalle.
- **Planes de proyecto y presentaciones de kickoff.**
- **Documentos de contexto** que permiten reconstruir o continuar una app en una nueva sesión de Claude Code.

### El impacto
El traspaso de funciones se hizo sin perder el conocimiento operativo. Ventas pudo usar las guías en conversaciones con prospectos, y el equipo tiene una referencia común en lugar de depender de preguntarle a la persona que "sabe".

---

## 16. Stack técnico

| Área | Tecnologías |
|---|---|
| Lenguajes | Python, JavaScript/React, SQL, PowerShell |
| Librerías | pandas, openpyxl, PyMuPDF, reportlab, paramiko, Flask |
| Datos | Azure Databricks (PySpark, Unity Catalog), SQLite |
| Automatización | n8n, Claude Skills, Claude Code, MCP |
| Herramientas | Linear (GraphQL), Notion, Slack, Fellow.ai |
