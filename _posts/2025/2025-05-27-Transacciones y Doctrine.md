---
title: Transacciones
date: 2025-05-21 15:43:00 -0600
categories: [Database]
tags: []
toc: true
---

En esta entrada vamos a profundizar un poco en como funcionan las transacciones a nivel de base de datos, problemas comunes y le echaremos un ojo a las funcionalidades no tan conocidas que podemos usar con doctrine.

## Estás hart@ de usarlas pero ...¿En sí qué es una transacción?

Es una secuencia de una o más operaciones ejecutadas como una unidad lógica, esto significa que una transacción asegura que todas las operaciones que en ella hacemos, se han ejecutado exitosamente, si alguna falla las operaciones implicadas se revienten haciendo un rollback.

## **Las propiedades ACID: Los pilares de las transacciones**

Para entender completamente las transacciones, es fundamental conocer las propiedades ACID, que garantizan la confiabilidad de las operaciones en base de datos:

![img](https://substackcdn.com/image/fetch/w_1456,c_limit,f_auto,q_auto:good,fl_progressive:steep/https%3A%2F%2Fsubstack-post-media.s3.amazonaws.com%2Fpublic%2Fimages%2Fbac45aff-4ebb-4654-94db-27a793f61309_1253x883.gif)

### **Atomicidad (Atomicity)**

Una transacción es una unidad indivisible de trabajo. Todas las operaciones dentro de una transacción se ejecutan completamente o no se ejecutan en absoluto. Si cualquier operación falla, toda la transacción se revierte.

### **Consistencia (Consistency)**

La base de datos debe mantenerse en un estado consistente antes y después de cada transacción. Todas las reglas de integridad, restricciones y triggers deben respetarse.

### **Aislamiento (Isolation)**

Las transacciones concurrentes no deben interferir entre sí. Cada transacción debe ejecutarse como si fuera la única operación en la base de datos.

### **Durabilidad (Durability)**

Una vez que una transacción se confirma, sus cambios son permanentes, incluso en caso de fallos del sistema.

## Aislamiento transaccional:

Cuando varias transacciones se ejecutan simultáneamente en una base de datos, pueden producirse varios problemas de concurrencia, como lecturas sucias, lecturas no repetibles y lecturas fantasma. Para gestionar estos problemas, las bases de datos SQL tienen diferentes niveles de aislamiento de transacciones, que controlan qué cambios en los datos son visibles para las transacciones que se ejecutan simultáneamente.

# Niveles de Aislamiento de Transacciones

## Comparación General

| Nivel de Aislamiento | Lectura Sucia | Non-Repeatable Read | Phantom Read | Anomalías de Serialización | Caso de Uso                                                  | Ejemplo Práctico                                             |
| -------------------- | ------------- | ------------------- | ------------ | -------------------------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **READ UNCOMMITTED** | ✅ Permite     | ✅ Permite           | ✅ Permite    | ✅ Permite                  | Reportes aproximados donde la precisión exacta no es crítica | Dashboard en tiempo real mostrando estadísticas generales de ventas |
| **READ COMMITTED**   | ❌ Previene    | ✅ Permite           | ✅ Permite    | ✅ Permite                  | Aplicaciones web típicas, operaciones CRUD básicas           | Sistema de e-commerce mostrando inventario actual            |
| **REPEATABLE READ**  | ❌ Previene    | ❌ Previene          | ✅ Permite    | ✅ Permite                  | Reportes financieros que requieren consistencia durante la ejecución | Generación de estados de cuenta bancarios                    |
| **SERIALIZABLE**     | ❌ Previene    | ❌ Previene          | ❌ Previene   | ❌ Previene                 | Transacciones críticas que requieren aislamiento total       | Transferencias bancarias entre cuentas                       |

## 

## Explicación de Anomalías

### 🔴 **Lectura Sucia (Dirty Read)**

Leer datos que han sido modificados por otra transacción pero aún no confirmados.

**Ejemplo:**

```sql
-- Sesión A
BEGIN;
INSERT INTO accounts (acctnum, balance) VALUES (12345, 100);
-- Aún no committed

-- Sesión B (con READ UNCOMMITTED)
SELECT * FROM accounts WHERE acctnum = 12345;  -- Ve el registro

-- Sesión A
ROLLBACK;  -- La cuenta nunca existió realmente
```

⚠️ **Nota PostgreSQL**: PostgreSQL no soporta realmente READ UNCOMMITTED - lo trata como READ COMMITTED.

### 🟡 **Non-Repeatable Read**

Una consulta repetida dentro de la misma transacción devuelve resultados diferentes.

**Ejemplo:**

```sql
-- Sesión A
BEGIN;
SELECT balance FROM accounts WHERE acctnum = 12345;  -- Lee $100

-- Sesión B
UPDATE accounts SET balance = 500 WHERE acctnum = 12345;
COMMIT;

-- Sesión A (misma transacción)
SELECT balance FROM accounts WHERE acctnum = 12345;  -- Lee $500
```

**Problema "Write After Read":**

```sql
-- Sesión A
BEGIN;
IF (SELECT balance FROM accounts WHERE acctnum = 12345) > 1000 THEN
    -- Otro proceso puede cambiar el balance aquí
    UPDATE accounts SET balance = balance * 1.1 WHERE acctnum = 12345;
END IF;
COMMIT;
```

### 🟠 **Phantom Read**

Nuevas filas aparecen o desaparecen entre consultas de la misma transacción.

**Ejemplo:**

```sql
-- Sesión A
BEGIN;
SELECT COUNT(*) FROM accounts WHERE balance > 0;  -- Cuenta 5

-- Sesión B
INSERT INTO accounts VALUES (12345, 100);
COMMIT;

-- Sesión A (misma transacción)
SELECT COUNT(*) FROM accounts WHERE balance > 0;  -- Cuenta 6
```

## **Tipos de Locks en MySQL y PostgreSQL**

Los locks (bloqueos) son fundamentales para manejar la concurrencia en bases de datos. Diferentes tipos de locks permiten controlar el acceso a recursos compartidos y prevenir condiciones de carrera.

### Shared Lock (Bloqueo Compartido)

Permite que múltiples transacciones lean el mismo recurso simultáneamente, pero previene escrituras.

```sql
SELECT * FROM users WHERE id = 123 FOR SHARE;
```

**Características:**

- ✅ Múltiples lecturas concurrentes permitidas
- ❌ Bloquea escrituras mientras esté activo
- 🎯 Ideal para: Reportes que requieren datos consistentes

### Exclusive Lock (Bloqueo Exclusivo)

Bloquea completamente el recurso, previniendo tanto lecturas como escrituras de otras transacciones.

```sql

SELECT * FROM products WHERE id = 456 FOR UPDATE;
```

**Características:**

- ❌ Bloquea todas las operaciones (lectura y escritura)
- 🎯 Ideal para: Actualizaciones críticas como inventario, transferencias

### Metadata Locks

Los metadata locks protegen la estructura de las tablas durante operaciones DDL (Data Definition Language).

###  Monitoring de Locks**

sql

```sql
-- PostgreSQL: Ver locks activos
SELECT 
    locktype,
    database,
    relation::regclass,
    page,
    tuple,
    transactionid,
    mode,
    granted,
    pid,
    query
FROM pg_locks 
JOIN pg_stat_activity ON pg_locks.pid = pg_stat_activity.pid
WHERE NOT granted;

-- MySQL: 
SELECT 
    object_type,
    object_schema,
    object_name,
    lock_type,
    lock_duration,
    lock_status,
    thread_id
FROM performance_schema.metadata_locks
WHERE lock_status = 'PENDING';

-- MariaDB: 
SELECT * FROM information_schema.innodb_locks;
SELECT * FROM information_schema.innodb_lock_waits;

```



## 🏁 Referencias

https://engineeringatscale.substack.com/p/acid-properties-databases-explained

https://medium.com/@aryadyas/understanding-database-transactions-how-they-work-and-where-they-matter-a00a65a94c14

https://www.thenile.dev/blog/transaction-isolation-postgres

https://antondevtips.com/blog/complete-guide-to-transaction-isolation-levels-in-sql

---
