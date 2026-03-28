# Procedimiento Almacenado de Recuperación Post Incidente

## 1. Descripción del incidente

Durante una ventana operativa en producción, se presentó una falla de sistema que dejó inactivos los triggers responsables de:

- actualizar saldos en `cuenta_bancaria`
- registrar eventos en `bitacora_operaciones`

Aunque los triggers fallaron, las transferencias continuaron insertándose en la tabla `transferencia`. Como resultado, el banco quedó con dos afectaciones críticas:

- **Desalineación contable**: movimientos registrados sin impacto consistente en saldos.
- **Pérdida de trazabilidad**: operaciones financieras sin evidencia completa de auditoría.

En sistemas bancarios reales esto representa un incidente mayor, porque compromete simultáneamente la integridad financiera y el cumplimiento normativo/auditable.

---

## 2. Objetivo de la solución

Implementar un procedimiento almacenado en PL/pgSQL para reconstruir, de forma controlada, el efecto de las transferencias afectadas por la caída de triggers.

Objetivos específicos:

1. Identificar transferencias registradas después del último evento confiable en bitácora.
2. Reaplicar el movimiento financiero (débito/crédito) cuenta por cuenta.
3. Registrar evidencia de reproceso en bitácora para recuperar trazabilidad.
4. Restablecer consistencia operativa antes de reabrir transacciones de negocio.

---

## 3. Estrategia de recuperación

La estrategia usa como punto de corte la última fecha registrada en bitácora:

- Se calcula `MAX(fecha_operacion)` de `bitacora_operaciones`.
- Se seleccionan transferencias con `fecha_creacion` mayor a ese valor.
- Esas transferencias se consideran candidatas a reconstrucción de saldo y auditoría.

Este patrón permite delimitar el alcance del incidente con un criterio temporal objetivo y repetible.

---

## 4. Uso de cursores

Se utiliza cursor en lugar de una consulta masiva por razones operativas y de control:

- Permite reprocesamiento **secuencial** y auditable transferencia por transferencia.
- Facilita trazabilidad detallada en caso de error durante una fila específica.
- Reduce riesgo de aplicar cambios masivos opacos en un incidente financiero.
- Hace más predecible la recuperación cuando se trabaja bajo contención.

En incidentes bancarios, el procesamiento controlado suele priorizarse sobre la velocidad de una operación masiva.

---

## 5. Lógica del procedimiento

### Paso a paso

1. **Selección de transferencias afectadas**
   - El cursor consulta `transferencia` filtrando por `fecha_creacion > MAX(fecha_operacion)` de `bitacora_operaciones`.

2. **Iteración con cursor**
   - Se abre el cursor y se recorre cada transferencia en orden cronológico.

3. **Actualización de saldo origen**
   - Se descuenta `v_monto` del `saldo_actual` de `v_cuenta_origen`.

4. **Actualización de saldo destino**
   - Se acredita `v_monto` al `saldo_actual` de `v_cuenta_destino`.

5. **Registro en bitácora**
   - Se inserta un registro en `bitacora_operaciones` con acción `RECONSTRUCCION_SALDO` para dejar evidencia de remediación.

---

## 6. Código SQL del procedimiento (PL/pgSQL)

```sql
CREATE OR REPLACE PROCEDURE public.reconstruir_saldos_post_incidente()
LANGUAGE plpgsql
AS $$
DECLARE
    -- Cursor filtrando solo por transferencias con estado 'APROBADA' (ID 12)
    cur_transferencias CURSOR FOR
        SELECT id_transferencia,
               cuenta_origen,
               cuenta_destino,
               monto
        FROM transferencia
        WHERE id_estado = 12
          AND fecha_creacion > (
            SELECT COALESCE(MAX(fecha_evento), '1900-01-01')
            FROM bitacora_operaciones
            WHERE accion = 'RECONSTRUCCION_SALDO'
        )
        ORDER BY fecha_creacion;

    v_id_transferencia INT;
    v_cuenta_origen VARCHAR;
    v_cuenta_destino VARCHAR;
    v_monto NUMERIC;

BEGIN
    RAISE NOTICE 'Iniciando reconstrucción de saldos basada en esquema detectado...';

    OPEN cur_transferencias;

    LOOP
        FETCH cur_transferencias INTO
            v_id_transferencia,
            v_cuenta_origen,
            v_cuenta_destino,
            v_monto;

        EXIT WHEN NOT FOUND;

        -- 1. Descontar de cuenta origen
        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual - v_monto
        WHERE numero_cuenta = v_cuenta_origen;

        -- 2. Sumar a cuenta destino
        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual + v_monto
        WHERE numero_cuenta = v_cuenta_destino;

        -- 3. Registrar en bitácora (Usando nombres de columna correctos del dump)
        INSERT INTO bitacora_operaciones (
            entidad_afectada,
            id_entidad,
            accion,
            usuario_responsable,
            fecha_evento,
            detalle
        )
        VALUES (
            'TRANSFERENCIA',
            v_id_transferencia,
            'RECONSTRUCCION_SALDO',
            NULL, -- Puedes poner el ID de un usuario administrador aquí
            CURRENT_TIMESTAMP,
            'Reproceso: Ajuste de saldo por fallo de trigger en transferencia ' || v_id_transferencia
        );

    END LOOP;

    CLOSE cur_transferencias;

    RAISE NOTICE 'Reconstrucción finalizada exitosamente.';
END;
$$;
```

---

## 7. Ejecución

Ejecutar el procedimiento con `CALL`:

```sql
CALL reconstruir_saldos_post_incidente();
```

Ejecución recomendada en ventana controlada:

```sql
BEGIN;
CALL reconstruir_saldos_post_incidente();
COMMIT;
```

---

## 8. Consideraciones importantes

### 8.1 Evitar reprocesamiento

- Definir y respetar marca de reproceso en bitácora (`RECONSTRUCCION_SALDO`).
- Validar antes de cada corrida que no se vuelvan a aplicar transferencias ya reconstruidas.

### 8.2 Control de concurrencia

- Ejecutar con transacciones operativas detenidas (modo contención).
- En escenarios de alta concurrencia, complementar con bloqueos (`FOR UPDATE`) para evitar carreras.

### 8.3 Impacto en sistemas reales

- Este tipo de corrección impacta saldos, conciliaciones y reportes regulatorios.
- Debe coordinarse con operaciones, riesgo, cumplimiento y auditoría interna.

### 8.4 Importancia de auditoría

- Toda acción de recuperación debe quedar registrada en bitácora con detalle y timestamp.
- La evidencia debe ser suficiente para reconstrucción forense y revisión de auditor externo.

---

## 9. Conclusión

Este procedimiento representa una práctica real de respuesta a incidentes en banca: contención, recuperación controlada y restablecimiento de trazabilidad.

Su diseño demuestra competencias clave de un DBA/desarrollador en producción:

- entendimiento de integridad transaccional
- capacidad de recuperación de datos financieros
- criterio de auditoría y cumplimiento
- documentación técnica orientada a incidentes reales

En entornos financieros, este enfoque no solo corrige datos: también restituye confianza operativa y evidencia auditada del proceso de remediación.

---

## 10. Guía de Pruebas: Reconstrucción de Saldos Post-Incidente

Esta guía permite probar el procedimiento almacenado `reconstruir_saldos_post_incidente`, validando que los saldos se ajusten correctamente tras un fallo de triggers.

### 10.1 Preparación del escenario (cuentas de prueba)

Se crean dos cuentas de prueba clonando estructura de una cuenta existente para respetar restricciones (`CHECK`, `FK`, formato de datos).

```sql
-- Insertar Cuenta Origen (inicia con 1000.00)
INSERT INTO public.cuenta_bancaria (
    numero_cuenta,
    tipo_cuenta,
    id_titular,
    tipo_titular,
    saldo_actual,
    moneda,
    id_estado,
    fecha_apertura
)
SELECT
    'TEST-01',
    tipo_cuenta,
    id_titular,
    tipo_titular,
    1000.00,
    moneda,
    id_estado,
    CURRENT_DATE
FROM public.cuenta_bancaria
LIMIT 1;

-- Insertar Cuenta Destino (inicia con 500.00)
INSERT INTO public.cuenta_bancaria (
    numero_cuenta,
    tipo_cuenta,
    id_titular,
    tipo_titular,
    saldo_actual,
    moneda,
    id_estado,
    fecha_apertura
)
SELECT
    'TEST-02',
    tipo_cuenta,
    id_titular,
    tipo_titular,
    500.00,
    moneda,
    id_estado,
    CURRENT_DATE
FROM public.cuenta_bancaria
LIMIT 1;
```

### 10.2 Creación de transferencia “huérfana”

Se inserta una transferencia con estado `APROBADA` (`id_estado = 12`) para simular una operación registrada sin impacto de trigger.

```sql
INSERT INTO public.transferencia (
    cuenta_origen,
    cuenta_destino,
    monto,
    id_estado,
    fecha_creacion,
    id_usuario_creador
)
VALUES (
    'TEST-01',
    'TEST-02',
    200.00,
    12,
    CURRENT_TIMESTAMP,
    1
);
```

### 10.3 Verificación pre-ejecución

Antes de ejecutar el procedimiento, los saldos deben permanecer sin cambios (`TEST-01 = 1000.00`, `TEST-02 = 500.00`).

```sql
SELECT numero_cuenta, saldo_actual
FROM public.cuenta_bancaria
WHERE numero_cuenta IN ('TEST-01', 'TEST-02');
```

### 10.4 Ejecución del procedimiento

```sql
CALL public.reconstruir_saldos_post_incidente();
```

### 10.5 Verificación de resultados (éxito)

Si el procedimiento se ejecutó correctamente:

| Cuenta  | Saldo Esperado | Acción Realizada   |
| ------- | -------------- | ------------------ |
| TEST-01 | 800.00         | Se restaron 200.00 |
| TEST-02 | 700.00         | Se sumaron 200.00  |

Consultar saldos finales:

```sql
SELECT numero_cuenta, saldo_actual
FROM public.cuenta_bancaria
WHERE numero_cuenta IN ('TEST-01', 'TEST-02');
```

Consultar bitácora:

```sql
SELECT *
FROM public.bitacora_operaciones
WHERE accion = 'RECONSTRUCCION_SALDO'
ORDER BY fecha_evento DESC
LIMIT 1;
```

### 10.6 Limpieza de datos

Una vez finalizada la prueba, eliminar los registros creados para mantener la base limpia.

```sql
DELETE FROM public.bitacora_operaciones
WHERE detalle LIKE '%TEST%';

DELETE FROM public.transferencia
WHERE cuenta_origen = 'TEST-01'
   OR cuenta_destino = 'TEST-02';

DELETE FROM public.cuenta_bancaria
WHERE numero_cuenta IN ('TEST-01', 'TEST-02');
```

> **Nota:** Si el `INSERT` de transferencia falla por `id_usuario_creador`, valida IDs disponibles con:

```sql
SELECT id_usuario
FROM public.usuario_sistema
LIMIT 1;
```
