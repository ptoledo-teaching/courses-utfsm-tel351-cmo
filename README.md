# CMO - Monitoreo y control de costos

Este laboratorio utiliza mediciones operacionales para detectar consumo innecesario, configurar una alerta y comprobar una medida de control de costos. El sistema sobre el cual se trabajará corresponde a una función Lambda que procesa archivos de telemetría almacenados en Amazon S3. El laboratorio consiste en medir el consumo inicial, estimar el costo mensual e identificar cómo reducir las lecturas sin alterar los resultados del servicio.

La actividad se realiza desde AWS Management Console, con la interfaz en inglés. Una plantilla autocontenida de CloudFormation proporciona la infraestructura y los datos. No se requiere desarrollar código, instalar herramientas ni escribir scripts. La limpieza se realiza después de guardar la evidencia y completar la evaluación.

## Resultados esperados

Al completar el laboratorio, será posible:

1. Ejecutar una prueba controlada y relacionar sus resultados con mediciones de consumo
2. Convertir las mediciones por archivo en una proyección mensual de uso y costo
3. Distinguir un aumento de demanda de un aumento del consumo necesario para obtener cada resultado
4. Transformar registros de la aplicación en una métrica y configurar una alerta operacional
5. Aplicar una medida de control sin impedir el procesamiento requerido
6. Comprobar el efecto de la medida utilizando cargas y resultados equivalentes
7. Diferenciar una señal de consumo, una estimación de costo y los cargos registrados por AWS

## Contexto

Las estaciones de la HoloNet envían archivos de telemetría con mediciones de sus enlaces. Un servicio lee cada archivo y produce un resumen con la cantidad de registros, los valores agregados y el hash del archivo de entrada. El resultado permite revisar el estado de cada estación sin requerir descargar ni examinar todos sus registros.

El equipo de desarrollo presupuestó el servicio suponiendo que cada archivo requeriría una sola lectura completa desde S3. Sin embargo, se sospecha que la configuración actual repite trabajo sin aportar resultados adicionales. Los resúmenes siguen siendo correctos, por lo que comprobar únicamente que la función termina sin errores no basta para detectar las lecturas redundantes.

El equipo debe medir el consumo, establecer una alerta que detecte lecturas adicionales y ajustar la configuración. Después del cambio, cada archivo debe seguir produciendo el mismo resumen, con todos sus registros incluidos. Omitir archivos, devolver resultados incompletos o detener el servicio no constituye una medida válida de ahorro.

## Arquitectura e interacción

La arquitectura utiliza dos funciones Lambda y un bucket privado de S3.

La función de pruebas prepara un conjunto de archivos de prueba e invoca a la función procesadora una vez por archivo. La procesadora carga el archivo desde S3 y realiza tres tareas: valida los datos de telemetría, calcula el resumen y comprueba la integridad de la entrada mediante su hash. Luego guarda el resumen en el bucket. La escritura de resultados en S3 no activa la función.

Cada prueba se identifica mediante un `runId`. Las entradas se crean bajo un prefijo nuevo, pero conservan el mismo contenido y los mismos identificadores lógicos entre pruebas. Esto permite comparar resultados equivalentes sin reutilizar salidas de una ejecución anterior. La herramienta no genera tráfico periódico y limita cada prueba a cinco archivos.

La función de pruebas también permite consultar y exportar mediciones. Consultar un resumen no vuelve a procesar las entradas. La plantilla incluye el dashboard `tel351-cmo` con un gráfico de invocaciones y errores de la procesadora. Durante la actividad se incorporarán la métrica de consumo, la alarma y su notificación por SNS.

## Evaluación

Este laboratorio cuenta con un sitio de evaluación en [lab13.shareddomain.link](https://lab13.shareddomain.link). Para iniciar una evaluación, ingresar el AWS account ID y el RUT normalizado (sin puntos ni guion y con `k` minúscula) que se utiliza como External ID del role `TEL351-Evaluator`. Mantener la managed policy `ReadOnlyAccess`, sin agregar permisos de escritura. El sitio inspecciona el stack `tel351-cmo` y sus recursos. No es necesario entregar un informe, capturas de pantalla ni correos de notificación.

Durante el laboratorio se procesa el mismo conjunto de archivos en dos pruebas. La primera utiliza la configuración inicial para medir el consumo y comprobar la activación de la alarma. La segunda permite verificar si el ajuste reduce las lecturas sin alterar los resultados. Al iniciar cada prueba, la función de pruebas devuelve un identificador (`runId`). Guardar el de cada ejecución e ingresarlo en el campo correspondiente del evaluador, que lo utiliza para localizar sus mediciones, resultados y logs en la cuenta.

Para comparar los costos, adjuntar las exportaciones de la misma estimación de AWS Pricing Calculator antes y después del ajuste. El evaluador obtiene los subtotales directamente de ambos archivos. Para las dos explicaciones breves utiliza un modelo de lenguaje en la cuenta docente y contrasta las afirmaciones con los datos de las pruebas.

| Criterio y comprobaciones | Puntos |
| --- | ---: |
| **Medición inicial.** Cinco archivos procesados, con sus resultados e invocaciones identificables (5); totales de lecturas y bytes leídos (5); GB-segundo facturados (5); explicación de la etapa responsable (3) y respaldo mediante la medición de un archivo afectado (2). | 20 |
| **Presupuesto inicial.** Proyección mensual de invocaciones y GB-segundo (5); proyección de GET, PUT y almacenamiento (5); costo mensual de Lambda (5) y de S3 (5) en la estimación exportada, de acuerdo con las mediciones y los supuestos del laboratorio. | 20 |
| **Métrica de consumo.** El filtro incluye los eventos que corresponden y excluye los demás (5); publica el número de lecturas, con el nombre y la unidad solicitados, y se observan muestras de la prueba (5). | 10 |
| **Alerta y notificación.** La alarma evalúa la métrica con la condición solicitada (5); la acción está habilitada y apunta al topic con una suscripción de correo confirmada (5); el historial muestra una transición a `ALARM` atribuible a las muestras de la prueba inicial (5). | 15 |
| **Dashboard.** El gráfico agregado muestra el máximo de lecturas por archivo en períodos de un minuto (5); el widget de estado identifica la alarma del laboratorio (5). | 10 |
| **Medida de control.** La configuración corrige el procesamiento sin cambiar el código ni la memoria (5); los cinco archivos se procesan con una lectura cada uno (5); se conservan las entradas, la cantidad de registros y el resultado de cada archivo (5). | 15 |
| **Comparación económica.** Estimación posterior y variación mensual en USD y porcentaje (5); comparación de las lecturas observadas (2) y del costo mensual estimado de Lambda y S3 en conjunto (3). | 10 |
| **Total** | **100** |

Cada comprobación de 5 puntos se calificará por separado. Será posible consultar avances sin haber completado todo el laboratorio. El resultado indicará el puntaje obtenido por criterio y, cuando exista una diferencia, qué dato o recurso no coincide. Si una consulta a AWS falla o una muestra reciente aún no está disponible, el sitio lo indicará para que se pueda repetir la consulta; no será necesario generar otra carga por ese motivo.

## Actividad

### 1. Desplegar y revisar la infraestructura

1. Ingresar a AWS Management Console y seleccionar **US East (N. Virginia)**, `us-east-1`
2. Descargar la plantilla [infrastructure.yaml](infrastructure.yaml) de este repositorio
3. Crear un stack desde **CloudFormation → Stacks → Create stack → With new resources (standard)**, utilizando **Upload a template file**
4. Utilizar `tel351-cmo` como nombre del stack, revisar los recursos IAM declarados y confirmar que CloudFormation puede crearlos
5. Esperar el estado `CREATE_COMPLETE` y abrir **Outputs**

Cada output contiene el nombre de un recurso del stack. La tabla indica qué hace cada recurso.

| Output | Función del recurso |
| --- | --- |
| `GeneratorFunctionName` | Inicia las pruebas y entrega las mediciones de la ejecución consultada |
| `ProcessorFunctionName` | Lee cada archivo de telemetría, calcula su resumen y lo guarda en S3 |
| `ResultsBucketName` | Almacena los archivos de entrada, sus resúmenes y las mediciones de cada prueba |
| `ProcessorLogGroupName` | Almacena los logs de las invocaciones de la función procesadora |
| `MeasurementsQueryName` | Consulta los logs en CloudWatch Logs Insights y muestra la duración facturada y la memoria asignada por invocación |

Identificar el recorrido de un archivo y distinguir los recursos del servicio de los utilizados para generar y consultar pruebas. Utilizar la misma región, memoria, arquitectura de ejecución y archivos de entrada en todas las pruebas. La procesadora utilizará ARM64 y 512 MB de memoria. Los log groups tendrán una retención de siete días.

No modificar recursos de otros laboratorios ni ampliar permisos para realizar la actividad.

### 2. Configurar el monitoreo antes de generar tráfico

La alerta debe distinguir un aumento de demanda de un aumento de lecturas por archivo. Procesar más archivos no debiera activarla si cada archivo requiere una sola lectura. El filtro solo procesa registros nuevos, por lo que debe quedar instalado antes de ejecutar la prueba inicial.

#### 2.1 Definir la métrica de consumo

1. Abrir en **CloudWatch → Log groups** el grupo indicado por `ProcessorLogGroupName` y crear un metric filter desde **Metric filters**
2. Diseñar y probar el patrón con los siguientes registros de ejemplo. Solo deben contribuir los eventos `file_processed` completados que informen una cantidad válida de lecturas. Utilizar los ejemplos en el editor del patrón, sin enviarlos al log group

```jsonl
{"event":"file_processed","fileId":"a","outcome":"COMPLETED","storageReads":2}
{"event":"file_processed","fileId":"b","outcome":"FAILED","storageReads":1}
{"event":"file_received","fileId":"c","outcome":"COMPLETED","storageReads":1}
{"event":"file_processed","fileId":"d","outcome":"COMPLETED"}
{"event":"file_processed","fileId":"e","outcome":"COMPLETED","storageReads":"invalid"}
```

3. Nombrar el filtro `tel351-cmo-storage-reads` y publicar la métrica `StorageReadsPerFile` en el namespace `TEL351/CMO`, con unidad `Count` y sin dimensiones. Elegir qué valor debe aportar cada coincidencia para que una ejecución con dos lecturas se distinga de otra con una

#### 2.2 Configurar la alarma y su notificación

1. Crear el topic SNS Standard `tel351-cmo-alerts` y suscribir una dirección de correo. Confirmar la suscripción desde el mensaje de SNS
2. Regresar a **Metric filters**, seleccionar el filtro recién creado y elegir **Create alarm**. Esta ruta permite definir la alarma sin buscar la métrica en el catálogo ni generar tráfico para que aparezca
3. Nombrar la alarma `tel351-cmo-storage-reads`. Utilizar períodos de un minuto y elegir la estadística, el umbral y la comparación que permitan detectar un solo archivo con más de una lectura, aunque los demás archivos del período requieran solo una. Configurar la evaluación sobre un período y tratar la ausencia de datos como no constitutiva de alarma
4. Asociar la acción **In alarm** al topic SNS y dejar las acciones habilitadas. No forzar el estado de la alarma ni publicar muestras artificiales

### 3. Ejecutar la prueba inicial y completar el dashboard

#### 3.1 Obtener la medición inicial

1. Abrir la función indicada en `GeneratorFunctionName` y, en **Test**, crear un evento privado llamado `cmo-generate`
2. Guardar el siguiente evento. Una vez creada la alarma y confirmada la suscripción, ejecutarlo una sola vez:

```json
{
  "action": "generate",
  "scenario": "baseline",
  "files": 5
}
```

3. Conservar el `runId` y comprobar la cantidad de archivos aceptados
4. Crear otro evento privado, llamado `cmo-summary`, con el siguiente contenido:

```json
{
  "action": "summary",
  "runId": "identificador-obtenido-al-generar"
}
```

5. Consultar el resumen con el `runId` inicial y comprobar que se completaron los cinco archivos. La respuesta contiene una tabla con una fila por archivo; la misma tabla se puede descargar como `measurements.csv` si se prefiere trabajar en una planilla

#### 3.2 Completar el dashboard

Mientras se publican las mediciones, abrir el dashboard `tel351-cmo`. El gráfico de invocaciones y errores ya está configurado. Incorporar dos widgets:

- Un gráfico que muestre el máximo de `StorageReadsPerFile` por minuto en la región del laboratorio
- Un widget de estado que identifique la alarma `tel351-cmo-storage-reads`

Si la serie todavía no aparece en el selector de métricas, avanzar con la interpretación y el presupuesto y completar el gráfico cuando esté disponible. En **Alarms → History**, identificar la transición a `ALARM` y contrastar el período que la provocó con la hora de la prueba inicial. El evaluador utilizará ese mismo `runId`; no es necesario que la alarma permanezca activa al momento de evaluar.

#### 3.3 Interpretar la medición inicial

Utilizar la tabla de `summary`. Si `measurementStatus` indica `PARTIAL`, repetir solo la consulta con el mismo `runId`, sin volver a ejecutar `generate`. Una celda vacía indica que el dato aún no está disponible; no equivale a cero. Si persiste una diferencia, utilizar el detalle de invocaciones (`invocations.csv`) y la consulta de Logs Insights indicada por `MeasurementsQueryName` para investigarla.

| Medición | Interpretación |
| --- | --- |
| `runId`, `fileId` y `requestId` | Relación entre prueba, archivo e invocación |
| `outcome` y `recordsProcessed` | Resultado y cantidad de registros incluidos |
| `inputDigest` y `resultDigest` | Hash del contenido del archivo de entrada y hash del contenido del resumen, respectivamente |
| `storageReads` | Total de lecturas de la entrada desde S3 para este archivo |
| `loadReads` | Lecturas de S3 para la carga inicial del archivo |
| `validationReads`, `summaryReads` e `integrityReads` | Lecturas adicionales de S3 durante la validación de los datos, el cálculo del resumen y la comprobación de integridad, respectivamente. Un cero indica que la etapa utilizó los datos en memoria, no que se omitió |
| `inputBytes` y `bytesRead` | Tamaño del archivo de entrada y total de bytes recibidos al leerlo, respectivamente |
| `resultObjectBytes` | Tamaño del resultado conservado en S3 |
| `billedDurationMs` y `memoryMB` | Duración facturada y memoria asignada a la invocación |

Calcular el total de lecturas, los bytes leídos y los GB-segundo, además de los promedios por archivo completado. Para obtener los GB-segundo de una invocación, multiplicar su duración facturada en segundos por la memoria asignada en GB: `(billedDurationMs / 1000) × (memoryMB / 1024)`.

La duración facturada se obtiene de los registros `REPORT` de Lambda. El tiempo medido dentro del código no la reemplaza. Los contadores de lecturas describen las llamadas de la aplicación; no constituyen un registro exhaustivo de facturación ni necesariamente incluyen reintentos internos del SDK.

**En el evaluador:** ingresar el `runId` inicial, el total de lecturas, los bytes leídos y los GB-segundo de la prueba. Explicar en dos frases qué etapa del procesamiento da cuenta del consumo observado. Para fundamentar el diagnóstico, identificar un archivo por su `fileId` y comparar el número de lecturas registradas con el esperado para ese archivo.

### 4. Proyectar el consumo y construir el presupuesto

Proyectar el consumo mensual a partir de las mediciones por archivo de la prueba inicial. Utilizar los siguientes supuestos:

| Supuesto | Valor |
| --- | --- |
| Demanda mensual | 100.000 archivos nuevos |
| Composición de la demanda | Los cinco tipos de archivo de la prueba, en igual proporción |
| Invocaciones del servicio | Una invocación de la procesadora por archivo |
| Almacenamiento | S3 Standard, originales y resultados conservados durante 30 días |
| Estado del servicio | Operación estable, llegadas uniformes y mes de 30 días |
| Escrituras del flujo presupuestado | Una carga de entrada y una escritura de resultado por archivo |
| Lecturas de entradas | Las observadas en la prueba |
| Descargas de resultados hacia Internet | No se incluyen en este escenario |
| Región y modalidad | `us-east-1`, uso bajo demanda |

1. Calcular las invocaciones, los GB-segundo, las lecturas y escrituras mensuales, y el volumen promedio almacenado
2. Abrir [AWS Pricing Calculator](https://calculator.aws/#/) y utilizar la interfaz en inglés. Crear una estimación con dos entradas: una para AWS Lambda y otra para Amazon S3
3. Utilizar la arquitectura ARM64 y calcular el costo en USD antes de créditos, descuentos y beneficios de la cuenta. Excluir el Free Tier en la estimación de Lambda. Para el almacenamiento, expresar el volumen en GB utilizando 1 GB = 1.073.741.824 bytes
4. En **My estimate → Export → CSV**, guardar la estimación inicial. Mantener abierta la estimación para actualizarla después de aplicar la medida de control

```text
lecturas mensuales = archivos mensuales × promedio de lecturas por archivo
GB-segundo mensuales = archivos mensuales × promedio de GB-segundo por archivo
almacenamiento estable en GB = archivos mensuales × promedio de bytes almacenados por archivo (entrada + resultado) / 1.073.741.824
```

El presupuesto calculado en esta sección comprende el procesamiento y almacenamiento del servicio. No incluye la función de pruebas, sus exportaciones, el monitoreo, SNS ni la navegación por consola. Identificarlo como una estimación parcial, no como el costo total de la solución o el monto por pagar.

Contabilizar las lecturas de la función procesadora como solicitudes a S3, no como transferencias de datos hacia Internet. Revisar las condiciones de transferencia aplicables a los servicios en la misma región. Los supuestos de la demanda no autorizan generar 100.000 archivos: el consumo mensual se obtiene por extrapolación.

**En el evaluador:** ingresar las cantidades mensuales proyectadas: invocaciones, GB-segundo, solicitudes GET, solicitudes PUT y GB de almacenamiento. Adjuntar el CSV inicial de AWS Pricing Calculator. Utilizar cantidades sin redondear en los cálculos intermedios. El evaluador compara por separado los subtotales mensuales de Lambda y S3.

### 5. Aplicar una medida de control

1. En la procesadora, abrir **Configuration → Environment variables**. La variable `REUSE_INPUT_AT` permite reutilizar la entrada en memoria en una etapa: `validation`, `summary` o `integrity`. El valor inicial, `none`, mantiene el comportamiento de referencia
2. A partir de las mediciones iniciales, decidir en qué etapa aplicar el ajuste y anticipar cómo debe cambiar la métrica
3. Cambiar únicamente el valor de `REUSE_INPUT_AT` y guardar la configuración
4. Esperar a que finalice la actualización e iniciar la prueba en un nuevo período de la alarma, para distinguir las muestras anteriores de las posteriores al ajuste
5. Ejecutar `cmo-generate` por segunda vez, conservar el nuevo `runId` y consultar `cmo-summary` con ese identificador
6. Comprobar que se completaron todos los archivos y comparar sus resultados por `fileId`

Para cada `fileId`, comprobar que `inputDigest` (hash del archivo de entrada) coincide en ambas pruebas. Luego, comparar `recordsProcessed` y `resultDigest` (hash del resumen): ambos deben mantenerse después del ajuste. El cálculo de los hashes excluye los identificadores y las fechas propios de cada ejecución.

No modificar código, entradas, memoria, permisos ni mecanismos de registro. No reducir la cantidad de archivos, deshabilitar la función, alterar los umbrales ni borrar evidencias para simular un ahorro.

**En el evaluador:** ingresar el `runId` posterior al ajuste. El sitio revisará la configuración de la procesadora y comparará las dos ejecuciones por `fileId`, no por el orden de las filas. Observar únicamente `OK` sin tráfico no demuestra que se haya reducido el consumo.

### 6. Comparar el consumo y el costo estimado

1. Volver a la estimación abierta en AWS Pricing Calculator. A partir de la tabla de la segunda ejecución, actualizar solo los consumos que cambiaron. Mantener la demanda, las tarifas y los demás supuestos del presupuesto inicial
2. Exportar la estimación actualizada como un segundo CSV. Calcular la diferencia entre el costo mensual inicial y el posterior (`inicial − posterior`), en USD y como porcentaje del inicial, utilizando los totales mensuales en USD de los dos CSV. Un resultado negativo indica un aumento. Si el costo inicial es cero, indicar que no corresponde calcular el porcentaje
3. Separar la reducción de lecturas observada de las variaciones de duración de Lambda. Una muestra pequeña no permite garantizar que toda diferencia de tiempo provenga de la corrección

El número de lecturas puede disminuir sin que el costo total caiga en la misma proporción. El almacenamiento, las escrituras y las invocaciones pueden mantenerse, y el monitoreo puede representar una parte importante del costo de una aplicación pequeña. No se exige un porcentaje predeterminado de ahorro ni se debe presentar la proyección como un cargo ya registrado por AWS.

**En el evaluador:** adjuntar el CSV posterior e ingresar la diferencia en USD. Ingresar el porcentaje de variación con un decimal; si el costo mensual inicial es cero, escribir `NA` en ese campo. Comparar en dos frases las lecturas observadas y el costo mensual estimado de Lambda y S3 en conjunto antes y después del ajuste.

## Limpieza

Realizar la limpieza después de consultar el resultado y completar la evaluación. Una vez eliminados los recursos, el evaluador ya no podrá inspeccionarlos.

1. No iniciar nuevas pruebas y comprobar que las iniciadas hayan finalizado
2. Eliminar la alarma `tel351-cmo-storage-reads` y el metric filter creado manualmente
3. Eliminar la suscripción y el topic `tel351-cmo-alerts`
4. Vaciar únicamente el bucket indicado en `ResultsBucketName`
5. Eliminar el stack `tel351-cmo` y comprobar que finalice su eliminación
6. Comprobar que se eliminaron el dashboard y los demás recursos proporcionados por el stack
7. Eliminar los eventos privados de prueba que permanezcan en la consola

Eliminar únicamente los recursos creados para este laboratorio. Si la eliminación del stack falla, revisar **Events**, resolver la causa y reintentar. Las estimaciones de AWS Pricing Calculator pueden conservarse. Las métricas históricas no se eliminan manualmente y dejarán de recibir muestras al retirar sus fuentes.
