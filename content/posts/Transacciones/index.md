---
title: Transacciones
date: 2025-05-27 17:00:00 -0600
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

| Nivel de Aislamiento | Lectura Sucia | Non-Repeatable Read | Phantom Read | Caso de Uso                                                  | Ejemplo Práctico                                             |
| -------------------- | ------------- | ------------------- | ----------- | ------------------------------------------------------------ | ------------------------------------------------------------ |
| **READ UNCOMMITTED** | ✅ Permite     | ✅ Permite           | ✅ Permite   | Reportes aproximados donde la precisión exacta no es crítica | Dashboard en tiempo real mostrando estadísticas generales de ventas |
| **READ COMMITTED**   | ❌ Previene    | ✅ Permite           | ✅ Permite   | Aplicaciones web típicas, operaciones CRUD básicas           | Sistema de e-commerce mostrando inventario actual            |
| **REPEATABLE READ**  | ❌ Previene    | ❌ Previene          | ✅ Permite   | Reportes financieros que requieren consistencia durante la ejecución | Generación de estados de cuenta bancarios                    |
| **SERIALIZABLE**     | ❌ Previene    | ❌ Previene          | ❌ Previene  | Transacciones críticas que requieren aislamiento total       | Transferencias bancarias entre cuentas                       |

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

### Monitoring de Locks

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

## **Comportamiento Completo de Locks en MySQL según Nivel de Aislamiento**

MySQL InnoDB implementa diferentes estrategias de bloqueo para cada nivel de aislamiento. El nivel por defecto es **REPEATABLE READ** y los locks se aplican automáticamente en **todas las operaciones DML** (INSERT, UPDATE, DELETE), no solo en consultas explícitas con FOR UPDATE.

### **REPEATABLE READ (Nivel por Defecto)**

**Características principales:**
- Las lecturas consistentes usan snapshots
- Utiliza gap locks y next-key locks automáticamente
- Se aplica en SELECT FOR UPDATE/SHARE, UPDATE, DELETE e INSERT

**Comportamiento en SELECT FOR UPDATE/SHARE:**

```sql
-- Índice único: solo record lock
SELECT * FROM users WHERE id = 123 FOR UPDATE;
-- Solo bloquea el registro específico

-- Búsqueda por rango: gap + record locks
SELECT * FROM orders WHERE status = 'pending' FOR UPDATE;
-- Bloquea registros existentes + gaps entre ellos
```

**Comportamiento en UPDATE:**

```sql
-- Tabla: products (id, category, price)
-- Datos: (1,'electronics',100), (2,'electronics',300), (3,'electronics',500)

-- Sesión A
UPDATE products SET price = 250 WHERE category = 'electronics';
-- Aplica automáticamente:
-- 1. Next-key locks en todos los registros con category='electronics'
-- 2. Gap locks entre esos registros
-- 3. Gap lock después del último registro del rango
```

**Comportamiento en INSERT:**

```sql
-- Los INSERTs adquieren insert intention locks
INSERT INTO products VALUES (4, 'electronics', 350);

-- Si hay gap locks activos en el rango donde va el nuevo registro:
-- 1. INSERT espera hasta que se liberen los gap locks
-- 2. Luego aplica record lock en el nuevo registro
-- 3. Pueden ocurrir deadlocks con otros INSERTs concurrentes
```

**Comportamiento en DELETE:**

```sql
-- DELETE usa next-key locks igual que UPDATE
DELETE FROM orders WHERE status = 'cancelled';
-- Bloquea:
-- 1. Todos los registros que coinciden (record locks)
-- 2. Gaps entre esos registros (gap locks)
-- 3. Previene INSERTs de nuevos registros 'cancelled'
```

### **READ COMMITTED**

**Diferencias clave:**
- NO usa gap locks en operaciones normales
- Solo record locks en registros que realmente modifica
- Libera locks de registros que no cumplen condiciones WHERE

**Comportamiento mejorado en UPDATE:**

```sql
-- Tabla sin índices
CREATE TABLE t (a INT NOT NULL, b INT) ENGINE = InnoDB;
INSERT INTO t VALUES (1,2),(2,3),(3,2),(4,3),(5,2);

-- REPEATABLE READ (por defecto)
UPDATE t SET b = 5 WHERE b = 3;
-- Escanea y bloquea TODAS las filas, retiene todos los locks

-- READ COMMITTED  
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;
UPDATE t SET b = 5 WHERE b = 3;
-- Escanea todas las filas pero:
-- 1. Libera locks de filas que NO modifica
-- 2. Solo retiene locks de filas actualizadas
-- 3. Permite mayor concurrencia
```

**Semi-consistent reads en READ COMMITTED:**

```sql
-- Sesión A (READ COMMITTED)
UPDATE accounts SET balance = balance + 100 WHERE status = 'active';

-- Sesión B (concurrent)
UPDATE accounts SET status = 'inactive' WHERE id = 123;

-- En READ COMMITTED:
-- 1. Sesión A lee la versión committed más reciente para evaluar WHERE
-- 2. Si el registro ya no cumple la condición, lo omite sin bloquear
-- 3. Si cumple, adquiere lock y procede con UPDATE
```

### **READ UNCOMMITTED**

**Comportamiento de locks:**
- Los `SELECT` statements se ejecutan de manera **no bloqueante**
- Pueden leer versiones anteriores (no committed) de las filas
- Estas lecturas **no son consistentes** - dirty reads
- Para el resto de operaciones funciona como **READ COMMITTED**

```sql
-- Sesión A
BEGIN;
UPDATE accounts SET balance = 5000 WHERE id = 123;
-- NO hace commit aún

-- Sesión B (READ UNCOMMITTED)
SET TRANSACTION ISOLATION LEVEL READ UNCOMMITTED;
SELECT balance FROM accounts WHERE id = 123;  
-- Puede leer 5000 (valor no committed)
-- Si A hace ROLLBACK, nunca existió realmente ese valor

-- Para INSERTs, UPDATEs, DELETEs: comportamiento igual que READ COMMITTED
UPDATE accounts SET balance = 1000 WHERE id = 456;
-- Solo bloquea registros que modifica, libera locks de no-matches
```

### **SERIALIZABLE**

**Comportamiento especial según autocommit:**

**Con autocommit deshabilitado:**
```sql
SET autocommit = 0;

-- Los SELECT planos se convierten automáticamente en SELECT FOR SHARE
SELECT * FROM products WHERE category = 'electronics';
-- InnoDB lo convierte implícitamente a:
SELECT * FROM products WHERE category = 'electronics' FOR SHARE;

-- Esto significa que adquiere shared locks y puede bloquear otras transacciones
```

**Con autocommit habilitado (por defecto):**
```sql
SET autocommit = 1;  -- Valor por defecto

SELECT * FROM products WHERE category = 'electronics';
-- El SELECT es su propia transacción
-- Se ejecuta como lectura consistente no bloqueante
-- NO necesita bloquear otras transacciones
-- Puede ser serializado sin problemas de concurrencia
```

**Forzar bloqueo en SERIALIZABLE:**
```sql
-- Para forzar que un SELECT bloque si otras transacciones 
-- han modificado las filas seleccionadas:
SET autocommit = 0;
SELECT * FROM products WHERE category = 'electronics';
-- Ahora SÍ bloquea (convierte a FOR SHARE automáticamente)
```

**Excepción para Grant Tables:**
```sql
-- Las operaciones DML que leen de las tablas de grants de MySQL
-- NO adquieren read locks en las grant tables, independientemente 
-- del nivel de aislamiento

-- Ejemplo: esta consulta NO bloquea las tablas mysql.user, mysql.db, etc.
SELECT u.name 
FROM users u 
JOIN mysql.user mu ON u.username = mu.User 
WHERE u.status = 'active';

-- Esto es para evitar bloqueos en operaciones de seguridad/permisos
```

### **Configuración de Niveles de Aislamiento**

```sql
-- Para la sesión actual
SET TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Para todas las conexiones siguientes
SET GLOBAL TRANSACTION ISOLATION LEVEL READ COMMITTED;

-- Al iniciar el servidor
mysqld --transaction-isolation=READ-COMMITTED

-- Verificar nivel actual
SELECT @@transaction_isolation;
```

### **Consideraciones Importantes**

**⚠️ Mixing Locking and Non-locking Statements:**
```sql
-- NO recomendado en REPEATABLE READ
BEGIN;
SELECT balance FROM accounts WHERE id = 123;        -- Non-locking
UPDATE accounts SET balance = 1000 WHERE id = 123;  -- Locking
-- Estados de tabla inconsistentes
```

**🎯 Caso de uso con índices:**
```sql
CREATE TABLE t (a INT NOT NULL, b INT, c INT, INDEX (b)) ENGINE = InnoDB;

-- Sesión A
UPDATE t SET b = 3 WHERE b = 2 AND c = 3;
-- Solo considera la columna indexada (b) para locks

-- Sesión B se bloquea porque también usa el índice en columna b
UPDATE t SET b = 4 WHERE b = 2 AND c = 4;
```

## **Gap Locks y Next-Key Locks: Prevención de Phantom Reads**

Los gap locks y next-key locks son mecanismos específicos de InnoDB para prevenir phantom reads y mantener la consistencia en REPEATABLE READ.

### **¿Qué son los Gap Locks?**

Un **gap lock** bloquea el espacio entre registros de índice, NO los registros en sí mismos. Su único propósito es **prevenir que otras transacciones inserten nuevos registros** en ese gap.

```sql
-- Ejemplo con índice en columna 'id'
-- Registros existentes: id = 10, 20, 30
SELECT * FROM users WHERE id > 15 AND id < 25 FOR UPDATE;

-- Gap locks aplicados:
-- Gap antes de id=20: (15, 20) 
-- Previene INSERT de id=16,17,18,19
-- NO bloquea el registro id=20 en sí
```

**Características importantes:**
- ✅ Múltiples transacciones pueden tener gap locks en el mismo gap
- ✅ Gap locks son **compatibles entre sí**
- ❌ **Solo previenen INSERTs**, no afectan SELECTs ni UPDATEs de registros existentes

### **¿Qué son los Next-Key Locks?**

Un **next-key lock** combina:
- **Record lock** en el registro actual
- **Gap lock** en el gap antes del registro

```sql
-- Registros: id = 10, 20, 30, 40
SELECT * FROM users WHERE id <= 25 FOR UPDATE;

-- Next-key locks aplicados:
-- (-∞, 10] + record lock en id=10
-- (10, 20] + record lock en id=20  
-- (20, 30] - solo gap lock (no incluye id=30)
```

### **Ejemplos Prácticos Detallados**

**Ejemplo 1: Gap Lock en Acción**

```sql
CREATE TABLE products (
    id INT PRIMARY KEY,
    category VARCHAR(50),
    price DECIMAL(10,2),
    INDEX idx_price (price)
);

INSERT INTO products VALUES 
(1, 'electronics', 100.00),
(2, 'electronics', 300.00),  
(3, 'electronics', 500.00);

-- Sesión A
BEGIN;
SELECT * FROM products WHERE price > 200 AND price < 400 FOR UPDATE;
-- Aplica gap lock en el rango (200, 300) y (300, 400)

-- Sesión B - Esta INSERT se BLOQUEA
INSERT INTO products VALUES (4, 'electronics', 250.00);
-- Se bloquea porque 250 está en el gap (200, 300)

-- Sesión C - Esta INSERT funciona
INSERT INTO products VALUES (5, 'electronics', 600.00);  
-- OK porque 600 no está en ningún gap bloqueado
```

**Ejemplo 2: Next-Key Lock Completo**

```sql
-- Datos: user_id = 5, 10, 15, 20
BEGIN;
SELECT * FROM users WHERE user_id <= 12 FOR UPDATE;

-- Next-key locks aplicados:
-- (-∞, 5] : gap (-∞, 5) + record lock en 5
-- (5, 10]  : gap (5, 10) + record lock en 10  
-- (10, 15) : solo gap lock hasta 12

-- Otros comportamientos:
-- ✅ SELECT user_id=15 permitido (no tiene record lock)
-- ❌ INSERT user_id=7 bloqueado (está en gap (5,10))
-- ❌ UPDATE user_id=10 bloqueado (tiene record lock)
```

### **INSERT y Gap Locks:**

```sql
-- Datos existentes: id = 10, 20, 30
-- Sesión A
UPDATE products SET price = 200 WHERE id BETWEEN 15 AND 25;
-- Aplica gap locks en (10,20) y (20,30)

-- Sesión B - estos INSERTs se BLOQUEAN
INSERT INTO products VALUES (12, 'test', 100);  -- Bloqueado por gap (10,20)  
INSERT INTO products VALUES (22, 'test', 100);  -- Bloqueado por gap (20,30)

-- Sesión C - este INSERT funciona
INSERT INTO products VALUES (35, 'test', 100);  -- OK, fuera de gaps bloqueados
```

### **Casos Especiales y Optimizaciones**

**Búsqueda por Clave Única**
```sql
-- Índice único en email
SELECT * FROM users WHERE email = 'user@example.com' FOR UPDATE;
-- Solo record lock, NO gap lock
-- InnoDB sabe que es único, no hay riesgo de phantoms
```

**Búsqueda que No Encuentra Registros**
```sql
-- No hay registros con price = 250
SELECT * FROM products WHERE price = 250 FOR UPDATE;
-- Aplica gap lock en el gap donde estaría price=250
-- Previene INSERT de price=250 hasta commit
```

### **Cuándo se Desactivan los Gap Locks**

Los gap locks se **desactivan** automáticamente en:

1. **READ COMMITTED**: Solo para statement locking normal
2. **Foreign Key Constraints**: Siempre activos para verificación de integridad
3. **Duplicate Key Checking**: Siempre activos para prevenir duplicados

```sql
-- En READ COMMITTED, esto NO usa gap locks
UPDATE products SET price = 200 WHERE category = 'electronics';

-- Pero esto SÍ usa gap locks (foreign key constraint)
INSERT INTO order_items (product_id) VALUES (999);
-- Verifica que product_id=999 existe y no se puede eliminar
```

### **Monitoring Completo de Locks**

```sql
-- Ver todos los locks activos (no solo FOR UPDATE)
SELECT 
    object_schema,
    object_name,
    lock_type,
    lock_mode,
    lock_status,
    lock_data,
    engine_transaction_id
FROM performance_schema.data_locks;

-- Ver gap locks específicamente
SELECT * FROM performance_schema.data_locks 
WHERE lock_type = 'GAP' OR lock_type = 'REC_NOT_GAP';

-- Ver locks en espera (deadlock detection)
SELECT
  r.trx_id waiting_transaction,
  r.trx_mysql_thread_id waiting_thread,
  r.trx_query waiting_query,
  b.trx_id blocking_transaction,
  b.trx_mysql_thread_id blocking_thread,
  b.trx_query blocking_query
FROM
  information_schema.innodb_lock_waits w
    JOIN information_schema.innodb_trx b ON
    b.trx_id = w.blocking_trx_id
    JOIN information_schema.innodb_trx r ON
    r.trx_id = w.requesting_trx_id;

-- InnoDB status para análisis detallado
SHOW ENGINE INNODB STATUS\G
```

### **Recomendaciones por Caso de Uso**

| Escenario | Nivel Recomendado | Razón |
|-----------|-------------------|-------|
| **E-commerce con alta concurrencia** | READ COMMITTED | Menos deadlocks en INSERTs de productos/órdenes |
| **Sistemas financieros** | REPEATABLE READ | Consistencia crítica en transferencias |
| **Aplicaciones con muchos INSERTs concurrentes** | READ COMMITTED | Evita bloqueos por gap locks |
| **Reportes que requieren consistencia total** | REPEATABLE READ o SERIALIZABLE | Snapshot consistente durante toda la consulta |
| **Sistemas de auditoría** | SERIALIZABLE | Máxima consistencia |

## Referencias

https://engineeringatscale.substack.com/p/acid-properties-databases-explained

https://medium.com/@aryadyas/understanding-database-transactions-how-they-work-and-where-they-matter-a00a65a94c14

https://www.thenile.dev/blog/transaction-isolation-postgres

https://antondevtips.com/blog/complete-guide-to-transaction-isolation-levels-in-sql

---
