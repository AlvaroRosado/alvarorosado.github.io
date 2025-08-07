---
title: DeadLocks
date: 2025-06-10 15:43:00 -0600
categories: [Database]
tags: []
toc: true
---

## ¿Qué es un Deadlock?

Un deadlock (bloqueo mutuo) es una situación en la que dos o más transacciones se bloquean mutuamente al intentar acceder a recursos que están siendo utilizados por las otras transacciones. Esto genera un ciclo de espera en el que ninguna puede continuar, ya que cada una espera que la otra libere los recursos que necesita.

Cuando MySQL detecta esta condición, automáticamente elige una de las transacciones como "víctima", la aborta y libera los locks que mantenía, permitiendo que las demás puedan continuar. La transacción abortada recibe el error `ERROR 1213 (40001): Deadlock found when trying to get lock`.

Los deadlocks son prácticamente inevitables en sistemas de bases de datos con alta concurrencia. Sin embargo, su impacto puede minimizarse de forma significativa mediante buenas prácticas de diseño, acceso ordenado a los datos y un monitoreo adecuado.


### Ejemplo con Actualizaciones Cruzadas

Consideremos una tabla de ejemplo para ilustrar un escenario típico de deadlock:

```sql
CREATE TABLE actor (
    actor_id INT PRIMARY KEY,
    first_name VARCHAR(50),
    last_name VARCHAR(50)
);

INSERT INTO actor VALUES 
(1, 'PENELOPE', 'GUINESS'),
(7, 'GRACE', 'MOSTEL');
```

**Sesión 1:**

```sql

`START TRANSACTION; 
UPDATE actor SET first_name = 'GUINESS' WHERE actor_id = 1; -- Adquiere un lock exclusivo en la fila actor_id = 1`

```

**Sesión 2:**

```sql

START TRANSACTION;
UPDATE actor SET first_name = 'MOSTEL' WHERE actor_id = 7;
-- Adquiere un lock exclusivo en la fila actor_id = 7

UPDATE actor SET last_name = 'PENELOPE' WHERE actor_id = 1;
-- Queda bloqueada esperando que Sesión 1 libere actor_id = 1

```

****Sesión 1 (continúa):**


```sql
UPDATE actor SET last_name = 'GRACE' WHERE actor_id = 7;

```

Ambas sesiones están esperando recursos que la otra ya posee.

<pre class="mermaid">
sequenceDiagram
    participant S1 as Sesión 1
    participant DB as Base de Datos
    participant S2 as Sesión 2

    S1->>DB: START TRANSACTION
    S1->>DB: UPDATE actor SET first_name = 'GUINESS' WHERE actor_id = 1
    DB-->>S1: Lock exclusivo en actor_id = 1

    S2->>DB: START TRANSACTION
    S2->>DB: UPDATE actor SET first_name = 'MOSTEL' WHERE actor_id = 7
    DB-->>S2: Lock exclusivo en actor_id = 7

    S2->>DB: UPDATE actor SET last_name = 'PENELOPE' WHERE actor_id = 1
    DB-->>S2: Espera (actor_id = 1 bloqueado por S1)
    Note over S2,DB: Sesión 2 bloqueada

    S1->>DB: UPDATE actor SET last_name = 'GRACE' WHERE actor_id = 7
    DB-->>S1: Espera (actor_id = 7 bloqueado por S2)
    Note over S1,DB: Sesión 1 bloqueada

    Note over S1,S2: Deadlock detectado
    DB->>S1: Aborta transacción (víctima)
    DB-->>S1: ERROR 1213: Deadlock found
    S2->>DB: Completa transacción
</pre>
<script src="https://cdn.jsdelivr.net/npm/mermaid@10.9.1/dist/mermaid.min.js"></script>


## Diferencia entre Deadlock y Lock Timeout

- **Deadlock**: Situación en la que dos o más transacciones se bloquean mutuamente, formando un ciclo de espera donde ninguna puede avanzar. MySQL detecta el ciclo y aborta una transacción, generando el error `1213: Deadlock found`.
-  **Lock Timeout**: Ocurre cuando una transacción espera demasiado por un recurso bloqueado por otra, sin formar un ciclo. Si la espera supera el límite configurado (`innodb_lock_wait_timeout`), MySQL termina la transacción con el error `1205: Lock wait timeout exceeded`.

-  **Diferencia clave**: Un deadlock implica un ciclo de dependencias cruzadas entre transacciones; un lock timeout es simplemente una espera prolongada por un recurso sin resolución.


## Consejos para investigar y evitar deadlocks

A continuación, se detallan prácticas recomendadas para identificar y mitigar deadlocks en bases de datos. Cada consejo incluye una explicación técnica y un ejemplo práctico.

1.  **Reducir la duración de las transacciones** Las transacciones de larga duración mantienen locks activos durante más tiempo, aumentando la probabilidad de deadlocks. Dividir transacciones complejas en unidades más pequeñas reduce el tiempo de retención de locks y minimiza conflictos.

2.  **Mantener un orden consistente en las operaciones** Los deadlocks ocurren frecuentemente cuando transacciones acceden a recursos en órdenes diferentes, creando ciclos de dependencias. Establecer un orden fijo para el acceso a tablas o filas previene estos ciclos al formar colas secuenciales.
  -   **Ejemplo**: Modificando el escenario de la tabla actor para acceder primero a actor_id = 1 y luego a actor_id = 7 en ambas sesiones:


3.  **Implementar lógica de reintento en la aplicación** Aunque los deadlocks no siempre se pueden evitar, agregar mecanismos de reintento permite que la aplicación reejecute la transacción abortada automáticamente. Priorice resolver la causa subyacente antes de depender exclusivamente de reintentos.


4.  **Revisar y ajustar niveles de aislamiento** Niveles de aislamiento altos generan más locks explícitos.


5.  **Optimizar con índices adecuados** Consultas sin índices apropiados escanean más registros, estableciendo locks innecesarios. Agregar índices bien diseñados reduce el número de locks y acelera las operaciones.
  -   **Ejemplo**: Para una tabla sin índice en producto_id:

      ```sql

      `   -- Consulta lenta sin índice
      UPDATE inventario SET stock = stock - 1 WHERE producto_id = 123;       `
      ```
      Agregue un índice:
      ```sql

      `   CREATE INDEX idx_producto_id ON inventario (producto_id);       `
      ```

      Esto limita los locks a filas específicas en lugar de rangos amplios.
6.  **Minimizar el impacto de locks en operaciones de esquema** Como administrador de bases de datos (DBA), evite operaciones que adquieran locks agresivos durante periodos prolongados.

Estas prácticas, combinadas con monitoreo continuo (por ejemplo, revisando logs de deadlocks en MySQL con SHOW ENGINE INNODB STATUS), ayudan a mitigar deadlocks efectivamente.
