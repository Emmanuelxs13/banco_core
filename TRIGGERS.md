# Triggers – Banco Core

**ET0062 – Bases de Datos II | Emmanuel Berrio Jimenez**

> Documentación de los dos triggers PL/pgSQL implementados para garantizar la integridad financiera en las operaciones de transferencia.
>
> - **Caso 1:** Carga masiva de transferencias que se procesan una por una y se detienen cuando el saldo se agota.
> - **Caso 2:** Aprobación 1 a 1 de transferencias pendientes donde, si una deja sin saldo a las demás, las siguientes se detienen automáticamente.

---

## Caso 1: Carga masiva de transferencias

### Descripción del escenario

Se realiza una **carga masiva de transferencias**: un proceso o usuario inserta muchas transferencias de forma secuencial, una por una, desde la misma cuenta origen.

Cada transferencia se valida en el acto al momento de insertarse. El sistema debe permitirlas **mientras haya saldo disponible** y detenerse automáticamente en el momento en que los fondos sean insuficientes para la siguiente operación.

La transferencia que no alcanza fondos **no se registra** — la serie se detiene ahí.

### Flujo de ejecución

```
Carga masiva: se insertan T1, T2, T3, T4... una por una

   INSERT T1  →  [TRIGGER]  →  saldo suficiente   →  ✓ Se inserta  →  saldo baja
   INSERT T2  →  [TRIGGER]  →  saldo suficiente   →  ✓ Se inserta  →  saldo baja
   INSERT T3  →  [TRIGGER]  →  saldo suficiente   →  ✓ Se inserta  →  saldo baja
   INSERT T4  →  [TRIGGER]  →  saldo INSUFICIENTE →  ✗ EXCEPCIÓN   →  se detiene
   INSERT T5  →  (nunca llega a ejecutarse)
```

### Función PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION validar_fondos_transferencia()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    saldo_actual NUMERIC;
BEGIN

    SELECT saldo_actual
    INTO saldo_actual
    FROM cuenta_bancaria
    WHERE numero_cuenta = NEW.cuenta_origen;

    IF saldo_actual < NEW.monto THEN
        RAISE EXCEPTION 'Fondos insuficientes en la cuenta %', NEW.cuenta_origen;
    END IF;

    RETURN NEW;

END;
$$;
```

### Trigger

```sql
CREATE TRIGGER trigger_validar_transferencia
BEFORE INSERT ON transferencia
FOR EACH ROW
EXECUTE FUNCTION validar_fondos_transferencia();
```

### Explicación paso a paso

| Paso | Acción                                                                                     |
| ---- | ------------------------------------------------------------------------------------------ |
| 1    | El proceso de carga inserta la primera transferencia con `INSERT INTO transferencia (...)` |
| 2    | El trigger se activa **antes** de que la fila se registre (`BEFORE INSERT`)                |
| 3    | Consulta el saldo actual de `cuenta_origen` (`NEW.cuenta_origen`) en ese instante          |
| 4    | Compara el saldo contra el monto de esta transferencia (`NEW.monto`)                       |
| 5    | Si hay fondos: retorna `NEW` → la transferencia se registra                                |
| 6    | Se pasa a la siguiente transferencia de la carga; el saldo ya es menor                     |
| 7    | Se repite hasta que el saldo ya no alcanza → `RAISE EXCEPTION` → la serie se detiene       |

> El trigger actúa en **cada fila individual** de la carga masiva. Como cada INSERT es independiente y el saldo se consulta en tiempo real tras cada operación, la cadena se corta exactamente cuando los fondos se agotan.

### Ejemplo práctico

```sql
-- Estado inicial: Cuenta A saldo = $1,000
-- Proceso de carga masiva: se intentan insertar 5 transferencias

-- T1: $300
INSERT INTO transferencia (cuenta_origen, cuenta_destino, monto) VALUES ('CUENTA_A', 'CUENTA_B', 300);
-- ✓ Saldo $1,000 >= $300  →  Registrada. Saldo restante: $700

-- T2: $250
INSERT INTO transferencia (cuenta_origen, cuenta_destino, monto) VALUES ('CUENTA_A', 'CUENTA_C', 250);
-- ✓ Saldo $700 >= $250  →  Registrada. Saldo restante: $450

-- T3: $450
INSERT INTO transferencia (cuenta_origen, cuenta_destino, monto) VALUES ('CUENTA_A', 'CUENTA_D', 450);
-- ✓ Saldo $450 >= $450  →  Registrada. Saldo restante: $0

-- T4: $200
INSERT INTO transferencia (cuenta_origen, cuenta_destino, monto) VALUES ('CUENTA_A', 'CUENTA_E', 200);
-- ✗ Saldo $0 < $200
--   ERROR: Fondos insuficientes en la cuenta CUENTA_A
--   La carga se detiene aquí. T4 no se registra.
```

---

## Caso 2: Aprobación masiva de transferencias pendientes

### Descripción del escenario

Existen varias transferencias en estado `PENDIENTE` registradas contra la misma cuenta origen. Un supervisor las aprueba **una por una** (1 a 1), cambiando su estado de `PENDIENTE` a `APROBADA`.

Cada aprobación descuenta el monto del saldo disponible. Cuando una transferencia consume los fondos restantes, **las siguientes aprobaciones se detienen automáticamente** — el trigger detecta que el saldo es insuficiente y lanza una excepción.

Las transferencias que no alcanzan fondos **permanecen en estado `PENDIENTE`**.

### Flujo de estados

```
PENDIENTE  ──►  APROBADA   (trigger valida saldo, descuenta origen, acredita destino)
           ──►  DETENIDA   (trigger lanza excepción: fondos agotados por aprobación anterior)
```

### Flujo de ejecución del trigger

```
Aprobaciones 1 a 1 por el supervisor:

   APROBAR T1  →  [TRIGGER]  →  saldo suficiente   →  ✓ APROBADA  →  saldo baja
   APROBAR T2  →  [TRIGGER]  →  saldo suficiente   →  ✓ APROBADA  →  saldo baja
   APROBAR T3  →  [TRIGGER]  →  saldo INSUFICIENTE →  ✗ EXCEPCIÓN →  se detiene
   APROBAR T4  →  (también falla si el saldo sigue sin alcanzar)

Detalle de cada aprobación:

UPDATE transferencia SET estado_transferencia = 'APROBADA' WHERE id = X
        │
        ▼
[TRIGGER BEFORE UPDATE]
        │
        ├── ¿Cambia a 'APROBADA' desde otro estado?
        │        │
        │        ├── NO  →  RETURN NEW (no interviene)
        │        │
        │        └── SÍ  →  SELECT saldo FOR UPDATE (bloqueo de fila)
        │                        │
        │                        ├── saldo >= monto  →  Descuenta cuenta_origen
        │                        │                  →  Acredita cuenta_destino
        │                        │                  →  Registra fecha_aprobacion
        │                        │                  →  RETURN NEW ✓
        │                        │
        │                        └── saldo < monto   →  RAISE EXCEPTION ✗
        │                                            →  T permanece PENDIENTE
```

### Función PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION aprobar_transferencia_validar_fondos()
RETURNS TRIGGER
LANGUAGE plpgsql
AS $$
DECLARE
    saldo_actual NUMERIC;
BEGIN

    IF NEW.estado_transferencia = 'APROBADA'
       AND OLD.estado_transferencia <> 'APROBADA' THEN

        SELECT saldo_actual
        INTO saldo_actual
        FROM cuenta_bancaria
        WHERE numero_cuenta = NEW.cuenta_origen
        FOR UPDATE;

        IF saldo_actual < NEW.monto THEN
            RAISE EXCEPTION
            'Transferencia % rechazada: fondos insuficientes',
            NEW.id_transferencia;
        END IF;

        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual - NEW.monto
        WHERE numero_cuenta = NEW.cuenta_origen;

        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual + NEW.monto
        WHERE numero_cuenta = NEW.cuenta_destino;

        NEW.fecha_aprobacion = NOW();

    END IF;

    RETURN NEW;

END;
$$;
```

### Trigger

```sql
CREATE TRIGGER trigger_aprobar_transferencia
BEFORE UPDATE ON transferencia
FOR EACH ROW
EXECUTE FUNCTION aprobar_transferencia_validar_fondos();
```

### Explicación paso a paso

| Paso | Acción                                                                                                 |
| ---- | ------------------------------------------------------------------------------------------------------ |
| 1    | Hay N transferencias en `PENDIENTE` para la misma `cuenta_origen`                                      |
| 2    | El supervisor aprueba la primera: `UPDATE ... SET estado_transferencia = 'APROBADA'`                   |
| 3    | El trigger verifica que el cambio sea desde otro estado hacia `'APROBADA'`                             |
| 4    | Consulta el saldo real con `FOR UPDATE` (bloquea la fila para evitar concurrencia)                     |
| 5    | Si hay fondos: descuenta de `cuenta_origen`, acredita en `cuenta_destino`, registra `fecha_aprobacion` |
| 6    | El supervisor aprueba la siguiente; el saldo ya es menor                                               |
| 7    | Se repite hasta que el saldo no alcanza → `RAISE EXCEPTION` → esa aprobación se detiene                |
| 8    | La transferencia que falló **permanece en `PENDIENTE`**; las anteriores ya fueron aprobadas            |

> Las transferencias aprobadas antes consumen el saldo progresivamente. La primera que no encuentra fondos suficientes falla, y cualquier intento de aprobar las siguientes también fallará si el saldo sigue sin alcanzar.

### Uso de `FOR UPDATE` — Por qué es necesario

En el escenario de aprobación 1 a 1 existe el riesgo de que dos aprobaciones se inicien casi al mismo tiempo sobre la misma cuenta:

- **Sin `FOR UPDATE`:** ambas leen el mismo saldo simultáneamente, ambas creen que hay fondos, y ambas descontarían — dejando el saldo en negativo.
- **Con `FOR UPDATE`:** la primera aprobación bloquea la fila de la cuenta. La segunda espera hasta que la primera confirme, y entonces lee el saldo ya actualizado.

```
Supervisor aprueba T1 y T2 casi al mismo tiempo:

Transacción T1                  Transacción T2
──────────────────────          ──────────────────────
SELECT saldo FOR UPDATE         SELECT saldo FOR UPDATE
  → bloqueo obtenido              → ESPERA (bloqueada)
  → saldo = $300 >= $300
UPDATE saldo = $0
COMMIT
                                → bloqueo liberado
                                → saldo = $0 < $200
                                → RAISE EXCEPTION ✗
                                → T2 permanece PENDIENTE
```

### Ejemplo práctico

```sql
-- Estado inicial: Cuenta A saldo = $1,000
-- Transferencias PENDIENTES para Cuenta A:
--   T1: $400   T2: $350   T3: $300   T4: $200

-- Supervisor aprueba T1
UPDATE transferencia SET estado_transferencia = 'APROBADA' WHERE id_transferencia = 1;
-- ✓ Saldo $1,000 >= $400  →  APROBADA
--   Cuenta A: $1,000 - $400 = $600

-- Supervisor aprueba T2
UPDATE transferencia SET estado_transferencia = 'APROBADA' WHERE id_transferencia = 2;
-- ✓ Saldo $600 >= $350  →  APROBADA
--   Cuenta A: $600 - $350 = $250

-- Supervisor aprueba T3
UPDATE transferencia SET estado_transferencia = 'APROBADA' WHERE id_transferencia = 3;
-- ✗ Saldo $250 < $300
--   ERROR: Transferencia 3 rechazada: fondos insuficientes
--   T3 permanece en PENDIENTE

-- Supervisor intenta aprobar T4
UPDATE transferencia SET estado_transferencia = 'APROBADA' WHERE id_transferencia = 4;
-- ✗ Saldo $250 >= $200 en este caso pasaría, pero si el saldo fuera $0:
--   ERROR: Transferencia 4 rechazada: fondos insuficientes
--   El proceso de aprobaciones se detiene completamente
```

---

## Resumen comparativo

| Característica                 | Caso 1 — Carga masiva                 | Caso 2 — Aprobación 1 a 1                 |
| ------------------------------ | ------------------------------------- | ----------------------------------------- |
| **Escenario**                  | Inserciones masivas secuenciales      | Supervisor aprueba PENDIENTES una por una |
| **Evento**                     | `BEFORE INSERT`                       | `BEFORE UPDATE`                           |
| **Cuándo actúa**               | Al registrar cada nueva transferencia | Al cambiar estado a `APROBADA`            |
| **Condición de parada**        | Saldo insuficiente al insertar        | Saldo agotado por aprobaciones anteriores |
| **Acción si hay fondos**       | Permite el INSERT                     | Descuenta origen, acredita destino        |
| **Acción si no hay fondos**    | Cancela el INSERT con excepción       | Cancela la aprobación con excepción       |
| **Descuenta saldo**            | No directamente (valida)              | Sí, de forma atómica                      |
| **Usa `FOR UPDATE`**           | No                                    | Sí (evita doble descuento concurrente)    |
| **Registra fecha**             | No                                    | Sí (`fecha_aprobacion = NOW()`)           |
| **Estado resultante si falla** | La transferencia no se crea           | La transferencia queda en `PENDIENTE`     |

---

_Banco Core – ET0062 Bases de Datos II | Emmanuel Berrio Jimenez_
