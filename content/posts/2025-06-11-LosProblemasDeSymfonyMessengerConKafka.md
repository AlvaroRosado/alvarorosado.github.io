---
title: Los problemas de Symfony Messenger con Kafka
date: 2025-06-10 15:43:00 -0600
categories: [EventArchitecture, Kafka]
tags: []
toc: true
---

Tras años trabajando con Apache Kafka y Symfony Messenger en entornos de producción, he llegado a una conclusión frustrante: los *transports* actuales de Symfony Messenger no están diseñados para aprovechar al máximo las capacidades de Kafka. No es que estén mal implementados, sino que abordan el problema desde una perspectiva equivocada. Kafka es una plataforma de *streaming* de eventos, no una cola de mensajes como RabbitMQ o Amazon SQS. Sin embargo, Symfony Messenger los trata de forma indistinta, lo que genera fricciones arquitecturales, código complicado y soluciones que no escalan bien.

## El problema de base

Kafka está diseñado para manejar **eventos en streaming**, mientras que Redis, RabbitMQ o SQS son **colas de mensajes**. Aunque parecen similares, sus propósitos y dinámicas son distintos. Symfony Messenger, al tratar a Kafka como una cola más, introduce problemas que se reflejan en configuraciones redundantes, lógica confusa y un mantenimiento innecesariamente complejo.

Imagina un *topic* en Kafka llamado `user_events`, que agrupa todos los eventos relacionados con usuarios: registros, actualizaciones, cambios de plan, eliminaciones, etc. Esto es típico en una arquitectura basada en eventos, donde los eventos relacionados coexisten en un mismo *stream*. Ahora, supongamos que tienes varios servicios:

- El servicio de facturación solo necesita procesar eventos de tipo `plan_changed`.
- El servicio de notificaciones solo quiere los eventos `user_registered`.
- El servicio de analíticas necesita todos los eventos.

Con los *transports* actuales de Symfony Messenger, te enfrentas a tres opciones, todas problemáticas:

1. **Crear un *transport* por tipo de evento**: Esto lleva a tener decenas de *transports*, cada uno escuchando un *topic* diferente, lo que fragmenta la cohesión del *stream* de eventos.
2. **Procesar todo y filtrar en el *handler***: El servicio de facturación termina deserializando eventos irrelevantes, como registros de usuarios, lo que es ineficiente y complica la lógica.
3. **Usar un *transport* genérico con lógica de enrutamiento personalizada**: Esto resulta en código confuso, con lógica de enrutamiento dispersa que se vuelve inmantenible con el tiempo.

## La interoperabilidad y la serialización

Si tu aplicación produce y consume sus propios mensajes, la serialización PHP nativa de Symfony Messenger funciona sin problemas. Sin embargo, cuando varias aplicaciones (incluso si todas usan Symfony) necesitan interoperar, la cosa se complica. La serialización PHP depende de que el consumidor tenga acceso a las mismas clases PHP que generaron el mensaje. Por ejemplo, si el servicio de usuarios produce un evento `App\Event\UserRegistered` y el servicio de facturación intenta deserializarlo, fallará si no tiene esa clase exacta.

Una solución aparente es usar JSON para serializar los mensajes, pero esto introduce otro problema:

```php
// El servicio A produce un evento
$message = new UserRegistered($userId, $email);

// Se serializa a JSON, pero...
// El mensaje llega al servicio B como:
{"user_id": 123, "email": "test@example.com"}

// ¿Cómo sabe el servicio B qué clase instanciar? ¿UserRegistered? ¿UserCreated?
```

Sin metadatos que indiquen el tipo de mensaje, el consumidor no tiene forma de interpretar correctamente el JSON recibido.

## Configuraciones redundantes

En un entorno real de Kafka (no en un *docker-compose* local), las configuraciones son extensas: autenticación SASL, SSL, ajustes de rendimiento, *timeouts*, etc. Con los *transports* actuales, estas configuraciones se repiten para cada *transport*, lo que genera duplicación innecesaria:

```yaml
user_events:
  options:
    config:
      security.protocol: SASL_SSL
      sasl.mechanisms: PLAIN
      sasl.username: "%env(KAFKA_USER)%"
      sasl.password: "%env(KAFKA_PASS)%"
      auto.offset.reset: earliest
      enable.auto.commit: false
      # ... otras 20 líneas de configuración

order_events:
  options:
    config:
      security.protocol: SASL_SSL  # Repetido
      sasl.mechanisms: PLAIN      # Repetido
      sasl.username: "%env(KAFKA_USER)%"  # Repetido
      # ... las mismas 20 líneas
```

Esta repetición no solo es tediosa, sino que también es una fuente de errores sutiles, como olvidar actualizar un parámetro en un *transport* cuando cambias otro.

## Una solución mejor

Cansado de estas limitaciones, desarrollé un *transport* personalizado para Symfony Messenger que se alinea mejor con la naturaleza de Kafka.

### Identificación clara de mensajes

Los mensajes incluyen un identificador explícito que se almacena en los *headers* de Kafka:

```php
class UserRegistered
{
    public function __construct(
        public readonly string $userId,
        public readonly string $email
    ) {}

    public function identifier(): string
    {
        return 'user_registered';
    }
}
```

Cuando el mensaje llega al consumidor, el *transport* lee este identificador y mapea automáticamente el JSON a la clase PHP correspondiente.

### Consumo selectivo de eventos

Puedes especificar qué tipos de eventos procesar desde un *topic*. Los eventos no configurados se ignoran automáticamente (*auto-commit*):

```yaml
consumer:
  routing:
    - name: user_registered
      class: App\Message\UserRegistered
    - name: plan_changed
      class: App\Message\PlanChanged
    # Otros eventos como user_updated o user_deleted se ignoran
```

Esto permite que múltiples servicios consuman el mismo *topic*, procesando solo los eventos relevantes sin interferencias.

### Configuración centralizada

Las configuraciones comunes de Kafka se definen una sola vez y se heredan en todos los *transports*:

```yaml
# config/packages/event_driven_kafka_transport.yaml
event_driven_kafka_transport:
  consumer:
    config:
      security.protocol: "%env(KAFKA_SECURITY_PROTOCOL)%"
      sasl.username: "%env(KAFKA_USER)%"
      # ... configuración común

# config/packages/messenger.yaml
framework:
  messenger:
    transports:
      billing_events:
        options:
          consumer:
            config:
              group.id: billing-service  # Solo configuraciones específicas
```

### Soporte nativo para múltiples *topics*

Puedes enviar un evento a varios *topics* sin configuraciones adicionales:

```yaml
options:
  topics:
    - user_events
    - audit_events
    - analytics_events
```

El evento se publica atómicamente en todos los *topics* especificados.

## Detalles técnicos

### Sistema de *hooks*

Para mantener el código desacoplado, el *transport* utiliza *hooks* que se ejecutan antes y después de producir o consumir mensajes:

```php
class EventHook implements KafkaTransportHookInterface
{
    public function beforeProduce(Envelope $envelope): Envelope
    {
        $message = $envelope->getMessage();
        if ($message instanceof Message) {
            return $envelope->with(new KafkaIdentifierStamp($message->identifier()));
        }
        return $envelope;
    }
}
```

El *transport* detecta automáticamente estos *hooks* sin necesidad de configuraciones adicionales.

### Compatibilidad con configuraciones existentes

Para facilitar la transición, el *transport* usa un DSN diferente, permitiendo que coexista con los *transports* actuales:

```bash
# Transport actual
KAFKA_DSN=kafka://localhost:9092

# Nuevo transport
KAFKA_EVENTS_DSN=ed+kafka://localhost:9092
```

## ¿Vale la pena el esfuerzo?

Desarrollar un *transport* personalizado requiere tiempo y un conocimiento profundo de Kafka y del código fuente de Symfony Messenger. Además, hay que lidiar con casos extremos que solo aparecen en producción con tráfico real. Entonces, ¿por qué no usar los *transports* existentes y aplicar parches?

Porque los parches se acumulan. Empiezas filtrando eventos en el *handler*, luego duplicas configuraciones, después creas servicios para gestionar el enrutamiento... y terminas con un sistema frágil y difícil de mantener. Con un *transport* bien diseñado, agregar un nuevo tipo de evento se reduce a un par de líneas de configuración, en lugar de una tarde de trabajo y una *pull request* que modifica múltiples archivos.

Cuando el *transport* está bien implementado, se vuelve transparente para el equipo. Los desarrolladores pueden usarlo sin preocuparse por las complejidades internas, lo que acelera el desarrollo y reduce errores.

---

*El código completo del *transport* está disponible en [GitHub](https://github.com/AlvaroRosado/event-driven-kafka-messenger-transport). Si estás lidiando con los mismos problemas, podría ahorrarte muchos dolores de cabeza.*
