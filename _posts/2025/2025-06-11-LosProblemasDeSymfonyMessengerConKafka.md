---
title: Los problemas de Symfony Messenger con Kafka
date: 2025-06-11 17:43:00 -0600
categories: [EventArchitecture, Kafka, Symfony]
tags: []
toc: true
---

# Los problemas de Symfony Messenger con Kafka

Después de pelearme durante mucho tiempo y en varias empresas, con Apache Kafka y Symfony Messenger en producción, llegué a una conclusión molesta: los transports existentes están diseñados para el problema equivocado.

No es que estén mal programados. Es que Kafka no es una cola de mensajes, y seguimos tratándolo como si lo fuera.

## El problema fundamental

Kafka es una plataforma de **streaming de eventos**. Redis, RabbitMQ, Amazon SQS son **colas de mensajes**. Son conceptos diferentes que resuelven problemas diferentes, pero Symfony Messenger los trata igual.

Esto crea una fricción arquitectural que se manifiesta en código feo, configuraciones duplicadas y soluciones que no escalan.

Imagínate que tienes un topic llamado `user_events` que contiene todos los eventos relacionados con usuarios: registros, actualizaciones, eliminaciones, cambios de plan, etc. En una arquitectura de eventos real, esto es lo normal. Los eventos relacionados van juntos.

Pero ahora resulta que tu servicio de facturación solo necesita procesar `plan_changed`, tu servicio de notificaciones solo quiere `user_registered`, y tu servicio de analytics los quiere todos.

Con los transports actuales tienes tres opciones, todas malas:

1. **Crear un transport por tipo de evento**: Terminas con 150 transports, cada uno escuchando un topic diferente. Pierdes la cohesión de tus streams de eventos.
2. **Procesar todo y filtrar en el handler**: Tu servicio de facturación tiene que deserializar eventos de registro de usuarios que nunca va a procesar. Ineficiente y ruidoso.
3. **Crear un transport genérico y hacer malabares**: Acabas con lógica de routing en lugares raros y código que nadie entiende seis meses después.

## La serialización entre aplicaciones

Si tu aplicación produce y consume sus propios mensajes, la serialización PHP de Symfony funciona perfectamente. El problema aparece cuando necesitas interoperabilidad entre aplicaciones diferentes.

Incluso si son dos aplicaciones Symfony distintas, la serialización PHP se rompe. El serializador necesita tener acceso a las mismas clases PHP que se usaron para crear el mensaje original. Si tu servicio de facturación intenta deserializar un evento que produjo tu servicio de usuarios, va a fallar porque no tiene la clase `App\Event\UserRegistered` del otro servicio.

La serialización JSON parece la solución obvia, pero implementarla bien es más complicado:

```php
// El servicio A produce un evento
$message = new UserRegistered($userId, $email);
// Se serializa a JSON... pero ¿cómo sabe el servicio B que es un UserRegistered?

// El mensaje llega al servicio B como JSON genérico
{"user_id": 123, "email": "test@example.com"}
// ¿Qué clase instanciar? ¿UserRegistered? ¿UserCreated? ¿UserSignedUp?
```

Sin metadatos sobre el tipo de mensaje, el servicio consumidor no puede saber qué hacer con el JSON que recibe.

## La configuración se multiplica

En Kafka real (no el que tienes en tu docker-compose local), la configuración es extensa. Autenticación SASL, SSL, configuraciones de performance, timeouts, etc.

Con los transports actuales,  normalmente repites esta configuración en cada transport:

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
      # ... 20 líneas más

order_events:
  options:
    config:
      security.protocol: SASL_SSL  # Otra vez lo mismo
      sasl.mechanisms: PLAIN
      sasl.username: "%env(KAFKA_USER)%"
      # ... las mismas 20 líneas
```

Esto es mantenimiento innecesario y fuente de bugs sutiles cuando cambias algo en un transport pero se te olvida en otro.

## Cómo debería funcionar

Después de demasiadas frustraciones, decidí escribir un transport que funcione como esperarías que funcionara Kafka en Symfony.

### Identificación explícita de mensajes

Los mensajes pueden llevar un identificador que se almacena en los headers de Kafka:

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

Cuando el mensaje llega al consumidor, el transport lee este header y puede mapear automáticamente el JSON a la clase PHP correcta.

### Consumo selectivo

Puedes configurar qué tipos de evento procesar del topic. Los que no estén configurados se auto-commitean (se ignoran):

```yaml
consumer:
  routing:
    - name: 'user_registered'
      class: 'App\Message\UserRegistered'
    - name: 'plan_changed'
      class: 'App\Message\PlanChanged'
  # user_updated, user_deleted, etc. se ignoran automáticamente
```

Esto permite que múltiples servicios consuman el mismo topic procesando subconjuntos diferentes de eventos, sin interferirse entre sí.

### Configuración global

Las configuraciones de Kafka se definen una vez a nivel global y se heredan:

```yaml
# config/packages/event_driven_kafka_transport.yaml
event_driven_kafka_transport:
  consumer:
    config:
      security.protocol: "%env(KAFKA_SECURITY_PROTOCOL)%"
      sasl.username: "%env(KAFKA_USER)%"
      # ... toda la configuración común

# config/packages/messenger.yaml - solo overrides específicos
framework:
  messenger:
    transports:
      billing_events:
        options:
          consumer:
            config:
              group.id: 'billing-service'  # Solo lo específico
```

### Multi-topic por defecto

Puedes producir el mismo evento a múltiples topics sin configuración adicional:

```yaml
options:
  topics: ['user_events', 'audit_events', 'analytics_events']
```

El evento se produce atómicamente a los tres topics.

## Los detalles técnicos

### Sistema de hooks

Para evitar acoplamiento, el transport usa un sistema de hooks. Implementas una interfaz que se ejecuta antes y después de producir/consumir:

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

El transport detecta automáticamente tu hook (sin configuración de servicios) y lo usa.

### Coexistencia

Para que puedas probarlo sin romper tu setup actual, usa un DSN diferente:

```bash
# Tu transport actual
KAFKA_DSN=kafka://localhost:9092

# El nuevo transport
KAFKA_EVENTS_DSN=ed+kafka://localhost:9092
```

Pueden coexistir sin problemas.

## ¿Vale la pena?

Desarrollar un transport custom lleva tiempo. Hay que entender cómo funciona Kafka por dentro, leer el código fuente de Symfony Messenger, y lidiar con edge cases que solo salen a la luz cuando tienes tráfico real.

¿Por qué no simplemente usar uno de los transports existentes y hacer algunos workarounds?

Porque los workarounds se acumulan. Empiezas con "solo vamos a filtrar estos eventos en el handler", luego "necesitamos duplicar esta configuración en tres sitios", después "vamos a crear un servicio que gestione el routing"... y al final tienes un sistema frankensteinian que nadie del equipo entiende completamente.

La diferencia la notas en velocidad de desarrollo. Con las herramientas correctas, añadir un nuevo tipo de evento son dos líneas de configuración. Con workarounds, es una tarde de trabajo y una pull request que toca cinco archivos diferentes.

Además, una vez que el transport está funcionando, se vuelve invisible. El resto del equipo simplemente lo usa sin pensar en las complejidades internas.

------

*El transport completo está disponible en [GitHub](https://github.com/AlvaroRosado/event-driven-kafka-messenger-transport). Si te encuentras peleándote con los mismos problemas, puede que te ahorre algunos dolores de cabeza.*
