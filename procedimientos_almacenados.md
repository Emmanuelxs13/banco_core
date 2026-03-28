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
CREATE OR REPLACE PROCEDURE reconstruir_saldos_post_incidente()
LANGUAGE plpgsql
AS $$
DECLARE
    -- Cursor
    cur_transferencias CURSOR FOR
        SELECT id_transferencia,
               cuenta_origen,
               cuenta_destino,
               monto,
               fecha_creacion
        FROM transferencia
        WHERE fecha_creacion > (
            SELECT COALESCE(MAX(fecha_operacion), '1900-01-01')
            FROM bitacora_operaciones
        )
        ORDER BY fecha_creacion;

    -- Variables para almacenar datos del cursor
    v_id_transferencia INT;
    v_cuenta_origen VARCHAR;
    v_cuenta_destino VARCHAR;
    v_monto NUMERIC;
    v_fecha TIMESTAMP;

BEGIN

    RAISE NOTICE 'Iniciando reconstrucción de saldos...';

    OPEN cur_transferencias;

    LOOP
        FETCH cur_transferencias INTO
            v_id_transferencia,
            v_cuenta_origen,
            v_cuenta_destino,
            v_monto,
            v_fecha;

        EXIT WHEN NOT FOUND;

        -- Descontar de cuenta origen
        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual - v_monto
        WHERE numero_cuenta = v_cuenta_origen;

        -- Sumar a cuenta destino
        UPDATE cuenta_bancaria
        SET saldo_actual = saldo_actual + v_monto
        WHERE numero_cuenta = v_cuenta_destino;

        -- Registrar en bitácora
        INSERT INTO bitacora_operaciones (
            entidad_afectada,
            id_entidad,
            accion,
            fecha_operacion,
            detalle
        )
        VALUES (
            'TRANSFERENCIA',
            v_id_transferencia,
            'RECONSTRUCCION_SALDO',
            NOW(),
            'Reproceso por incidente de triggers'
        );

    END LOOP;

    CLOSE cur_transferencias;

    RAISE NOTICE 'Reconstrucción finalizada correctamente';

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
