---
title: Apache Kafka para desarrolladores
date: 2025-05-21 15:43:00 -0600
categories: [EventArchitecture, Kafka]
tags: []
toc: true
---


Si alguna vez te has sentido perdido con Apache Kafka, no estás solo. Tal vez tu jefe lo mencionó como la solución a todos los problemas y solo asentiste, o estás en un proyecto con Kafka y cada error te deja desorientado. Incluso la documentación oficial puede resultar confusa. Si esto te suena, este artículo es para ti. Vamos a explicar Kafka de forma clara, sin tecnicismos innecesarios y con ejemplos prácticos.

## ¿Qué es Kafka y para qué sirve?

Kafka es el sistema que conecta aplicaciones modernas. Imagina una empresa donde cada departamento —ventas, contabilidad, logística, marketing— necesita saber qué pasa en los demás. Cuando hay una venta, contabilidad registra el ingreso, el almacén prepara el envío y marketing envía un correo. En lugar de que todos se comuniquen directamente (un desastre), Kafka actúa como un tablón de anuncios: cada área publica sus eventos y se suscribe a los que le interesan.

Esto no solo sirve para empresas, sino para microservicios, dispositivos IoT, análisis en tiempo real o procesamiento de datos masivos. Kafka organiza este flujo de información de manera rápida y confiable.

## Los elementos clave de Kafka

Para entender Kafka, necesitas conocer sus componentes principales:

- **Topics y particiones**: Donde se almacenan los mensajes.
- **Productores**: Quienes envían los mensajes.
- **Consumidores**: Quienes leen los mensajes.
- **Brokers**: Los servidores que gestionan el sistema.

Lo explicaremos con una metáfora sencilla: un almacén de documentos muy eficiente.

## El almacén de Kafka

Imagina que gestionas un almacén de documentos bien organizado. Este almacén es Kafka.

### Estanterías y cajones

Tienes estanterías especializadas para diferentes tipos de documentos: una para "Facturas", otra para "Pedidos", otra para "Alertas de inventario". Cada estantería es un **topic**. Cada una se divide en cajones numerados, que son las **particiones**. Por ejemplo, si esperas muchas facturas, creas una estantería con cinco cajones; si las alertas son pocas, dos cajones bastan.

Los cajones permiten que varios empleados trabajen en paralelo, cada uno en un cajón, sin interferencias. Más cajones significan más velocidad.

### El encargado

En la entrada hay un encargado (el **broker**) que lo controla todo:

- Recibe cualquier documento y sabe en qué estantería y cajón guardarlo.
- Nunca pierde nada.
- Ordena los documentos por orden de llegada.
- Recuerda qué documentos ha entregado a cada equipo.

En un sistema real, un clúster de Kafka tiene varios *brokers* que distribuyen la carga.

### Los repartidores

Cuando la empresa genera un documento (una venta, un pedido, una alerta), un repartidor (el **productor**) lo lleva al almacén. El encargado lo archiva en el cajón correcto, manteniendo el orden cronológico.

## Procesamiento de documentos

### El equipo de contabilidad

Cada semana, llega un equipo de contabilidad con tres personas que piden las facturas nuevas. El encargado asigna tareas:

- "María, revisa el cajón 1 de facturas."
- "Carlos, el cajón 2."
- "Ana, el cajón 3."

Cada uno trabaja en su cajón sin interferir. En Kafka, este equipo es un **Consumer Group**, y cada persona es un **Consumidor**. Los consumidores dividen el trabajo para procesar las particiones en paralelo.

### La memoria del encargado

El encargado sabe exactamente cuál fue el último documento entregado a cada equipo. Cuando regresan, retoman donde lo dejaron. En Kafka, esto es el **offset**: la posición exacta en la partición donde un consumidor dejó de leer.

## ¿Qué pasa si algo falla?

### Si alguien falta

Si un día Carlos no viene, el encargado redistribuye el trabajo:

- "María, además del cajón 1, hoy te encargas del cajón 2."
- "Ana, sigue con el cajón 3."

Nada se queda sin procesar. En Kafka, esto es el **rebalanceo automático**: si un consumidor falla, los demás se reparten sus particiones.

### Si se pierde un documento

Si un cajón se daña, el encargado usa copias de seguridad almacenadas en otros almacenes (otros *brokers*). En Kafka, estas **réplicas** aseguran que los datos estén duplicados para evitar pérdidas.

## Cómo se mantiene el orden

### Organización de documentos

Cada documento lleva una etiqueta (la **key** en Kafka), como el ID de un cliente. El encargado usa una fórmula para decidir en qué cajón va. La misma etiqueta siempre va al mismo cajón, garantizando que los documentos relacionados (por ejemplo, todas las facturas del cliente "12345") estén juntos y en orden cronológico.

### La regla del orden

Kafka solo garantiza orden dentro de una partición. Si necesitas orden global, usa una sola partición, pero esto limita el paralelismo, ya que solo un consumidor podrá procesarla. Es un equilibrio entre orden y velocidad:

- **Más particiones**: Mayor paralelismo, más velocidad.
- **Menos particiones**: Mayor orden, menos velocidad.

El número de particiones determina el máximo de consumidores que pueden trabajar en paralelo.

## Limpieza del almacén

El almacén no puede crecer infinitamente. El encargado usa dos métodos para mantenerlo manejable:

- **Por tiempo**: Los documentos antiguos (por ejemplo, de más de 7 días) se eliminan. En Kafka, esto se configura con `retention.ms`. Es ideal para logs o eventos temporales.
- **Por clave**: Se guarda solo el documento más reciente para cada etiqueta. Por ejemplo, el último estado de un pedido. Esto se activa con `cleanup.policy=compact` y es perfecto para datos como precios o ubicaciones.

## Consejos prácticos

1. **Elige bien las claves**:
  - Usar la misma clave envía todo a una partición, limitando el paralelismo.
  - Sin claves, Kafka distribuye mensajes aleatoriamente.
  - Usa claves que agrupen datos según tus necesidades.

2. **Planifica las particiones**:
  - Aumentar particiones es fácil; reducirlas es complicado.
  - Calcula cuántos consumidores necesitarás en el peor caso.

3. **Usa *topics* con criterio**:
  - Crea un *topic* por concepto claro (por ejemplo, "pedidos" o "transacciones").
  - Usa nombres descriptivos y consistentes.
  - No mezcles conceptos en un mismo *topic*.

4. **Gestiona versiones de mensajes**:
  - Los formatos de mensajes cambian con el tiempo.
  - Usa herramientas como Schema Registry con Avro o Protobuf para versionado.
  - Asegúrate de que los cambios sean compatibles hacia atrás y hacia adelante.

## Casos de uso

- **E-commerce**: Un *topic* "pedidos" registra compras. Los consumidores actualizan inventario, generan facturas, preparan envíos o envían recomendaciones.
- **Banca**: Un *topic* "transacciones" captura movimientos. Los consumidores detectan fraudes, registran en contabilidad o envían notificaciones.
- **IoT**: Un *topic* "sensores" recoge datos como temperatura. Los consumidores generan alertas, analizan históricos o alimentan modelos de machine learning.

## Resumen: Kafka vs. Almacén

| Concepto Kafka     | Metáfora del almacén      | Para qué sirve                    |
| ------------------ | ------------------------- | --------------------------------- |
| **Cluster**        | Almacén completo          | Sistema Kafka en conjunto         |
| **Broker**         | Encargado                 | Servidor que gestiona mensajes    |
| **Topic**          | Estantería                | Categoría de mensajes             |
| **Partition**      | Cajón                     | División para paralelismo         |
| **Producer**       | Repartidor                | Aplicación que envía mensajes     |
| **Consumer**       | Contable                  | Aplicación que lee mensajes       |
| **Consumer Group** | Equipo de contables       | Grupo de consumidores colaborando |
| **Offset**         | Último documento revisado | Posición de lectura               |
| **Key**            | Etiqueta del documento    | Criterio de particionamiento      |
| **Replication**    | Copias en otros almacenes | Redundancia para disponibilidad   |

