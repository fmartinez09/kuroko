---
title: "Serializabilidad Fuerte en FoundationDB: Detección de Conflictos Automática vs PostgreSQL en OpenAI"
description: ""
date: "May 30, 2025"
---

### Problema: Las Limitaciones de Snapshot Isolation (SI)

El nivel de aislamiento **Snapshot Isolation (SI)** es popular por su alta concurrencia y facilidad de implementación. Sin embargo, **NO garantiza serializabilidad**:

- **SI solo detecta conflictos de escritura (write-write conflicts)**: si dos transacciones escriben la misma clave, una falla.
- **No detecta conflictos de lectura (read-write conflicts)**: dos transacciones pueden leer datos obsoletos y escribir nuevas actualizaciones en otros lugares, causando inconsistencias (anomalías como "write skew").

**Ejemplo clásico (write skew)**:

```
T1: read(x), read(y), if x=0 and y=0 then write(x=1)
T2: read(x), read(y), if x=0 and y=0 then write(y=1)
```

Ambas transacciones leen `x=0, y=0` en su snapshot. Ambas deciden escribir: una en `x`, la otra en `y`. Ambas comitean exitosamente. El sistema finaliza con `x=1, y=1`, aunque la condición `x=0 && y=0` ya no se cumple. Este es un estado inconsistente.

Esto fue justamente lo que se discutió en [Hacker News](https://news.ycombinator.com/item?id=40544896) y el blog de **Concurrency Freaks**. El comentario clave fue:

> "Lo que realmente quieres es el par de garantías:- Lo que escribo no fue escrito por otro en paralelo (write-write conflict)- Lo que leo tampoco fue escrito por otro en paralelo (read-write conflict)."
> 

El mismo usuario comentó que **FoundationDB** es el sistema que mejor implementa esta idea, porque automáticamente rastrea reads/writes, y puedes manipular manualmente los sets de conflicto para optimizar según tus necesidades. Añadió incluso un ejemplo práctico:

```
beginTxn()
  if (x == 0 && y == 0) x = 1
  x = x; y = y;  // implicit read conflict
endTxn()

beginTxn()
  if (x == 0 && y == 0) y = 1
  x = x; y = y;  // implicit read conflict
endTxn()
```

Esto fuerza a que ambas transacciones entren en conflicto correctamente.

---

### Cómo lo resuelve FoundationDB

**FoundationDB implementa *Strict Serializability* mediante detección de conflictos automática en commit.**

### Registro de rangos de conflicto

En cada transacción:

- Las **lecturas** (`get`) registran un **read conflict range**.
- Las **escrituras** (`set`, `clear`) registran un **write conflict range**.

Esto ocurre de forma **automática**, pero puedes ajustarlo manualmente usando la API:

```
tr.add_read_conflict_range(start_key, end_key)
tr.add_write_conflict_range(start_key, end_key)
```

### Commit: Validación atómica

Cuando llamas a `tr.commit()`, FoundationDB ejecuta:

- **Verificación de write-write**: ¿Alguien más escribió en mis rangos de escritura?
- **Verificación de read-write**: ¿Alguien más escribió en mis rangos de lectura?

Si hay conflicto (alguien escribió después de mi `read_version`), **la transacción es abortada automáticamente**.

### API declarativa y flexible

- Si no quieres que una lectura falle aunque otro proceso escriba, puedes **eliminar el rango de conflicto**:

```
tr.add_read_conflict_range(key, key)  # Por defecto
tr.options.set_read_your_writes_disable()  # O eliminar explícitamente
```

Esto te permite controlar qué partes de tu transacción son "importantes" para el aislamiento.

---

### Caso OpenAI y PostgreSQL

Recientemente, se descubrió (en [Hacker News](https://news.ycombinator.com/item?id=40544896)) que **OpenAI** utiliza **PostgreSQL como base de datos central** para respaldar sus sistemas críticos. Esto es una decisión pragmática, basada en:

- Madurez, soporte y comunidad de PostgreSQL.
- Facilidad de integración con el stack de Azure.
- Capacidad de usar extensiones como Citus o Timescale.

Sin embargo, PostgreSQL con SI no tiene las garantías de consistencia estricta que ofrece FoundationDB. Por ejemplo:

- PostgreSQL no detecta **read-write conflicts**.
- La semántica de SI permite que dos transacciones lean datos desactualizados y comiteen cambios inconsistentes.

Esto significa que en escenarios de alta concurrencia (como OpenAI podría tener), podrían producirse **anomalías de consistencia** a menos que implementen lógicas adicionales o restrinjan patrones de acceso.

---

### Diagrama Simplificado

```
T0: (global read_version = 100)

T1 (read_version=100): read(x=0), read(y=0)
T2 (read_version=100): read(x=0), read(y=0)

T1: set(x=1)
T2: set(y=1)

T1.commit() -> verifica si x fue modificado desde 100 -> OK
T2.commit() -> verifica si y fue modificado desde 100 -> OK

FoundationDB: Ambos rangos (read+write) se verifican:

- T1: read(x,y), write(x) -> T2 modificó y -> conflicto read-write -> T2 ABORT
- T2: read(x,y), write(y) -> T1 modificó x -> conflicto read-write -> T1 ABORT

Resultado: Solo una transacción puede committear.
```

---

### Comparación: SI vs FoundationDB

| Característica | Snapshot Isolation (SI) | FoundationDB (Strict Serializability) |
| --- | --- | --- |
| Detección write-write conflict | Sí | Sí |
| Detección read-write conflict | No | Sí |
| Anomalías como write skew | Sí (pueden ocurrir) | No (previene) |
| API para control de conflictos | No | Sí (rangos explícitos) |
| Garantía de serializabilidad | No | Sí |

---

**FoundationDB no es solo una base de datos: es un marco para construir sistemas distribuidos correctos**.

- La detección de conflictos automáticos (read-write y write-write) es una primitiva fundamental.
- Su API permite **control total sobre el modelo de consistencia**.
- Esto es lo que lo hace **superior a SI**, y una herramienta **primordial para sistemas distribuidos**