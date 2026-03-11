# Banco Core – Sistema de Gestión Bancaria (PostgreSQL)

**Grupo 800 | ET0062 – Bases de Datos II**  
**Emmanuel Berrio Jimenez**

---

## Tabla de Contenidos

1. [Descripción del Proyecto](#1-descripción-del-proyecto)
2. [Tecnologías Utilizadas](#2-tecnologías-utilizadas)
3. [Importación de la Base de Datos](#3-importación-de-la-base-de-datos)
4. [Modelo de Datos](#4-modelo-de-datos)
5. [Interpretación del Ejercicio](#5-interpretación-del-ejercicio)
6. [Integridad de Datos](#6-integridad-de-datos)
7. [Índices Estratégicos](#7-índices-estratégicos)
8. [Triggers Implementados](#8-triggers-implementados)
9. [Consultas SQL](#9-consultas-sql)
10. [Conclusión](#10-conclusión)

---

## 1. Descripción del Proyecto

Diseño e implementación de una base de datos relacional en **PostgreSQL 18** para soportar el core transaccional de una entidad bancaria.

| Módulo                | Descripción                                                    |
| --------------------- | -------------------------------------------------------------- |
| **Catálogos**         | Roles, estados y productos bancarios normalizados              |
| **Clientes**          | Persona natural y empresa con atributos diferenciados          |
| **Usuarios**          | Control de acceso unificado con roles y estados                |
| **Cuentas bancarias** | Ahorros, corriente y empresarial con restricciones de saldo    |
| **Préstamos**         | Flujo completo de solicitud, aprobación y desembolso           |
| **Transferencias**    | Movimientos entre cuentas con control de fondos y autorización |
| **Bitácora**          | Registro de auditoría de todas las operaciones críticas        |

---

## 2. Tecnologías Utilizadas

| Tecnología     | Versión              | Uso                                                   |
| -------------- | -------------------- | ----------------------------------------------------- |
| **PostgreSQL** | 18.0                 | Motor de base de datos relacional                     |
| **pgAdmin**    | 4                    | Administración visual y restauración de backups       |
| **SQL**        | Estándar SQL:2016    | DDL, DML, consultas y restricciones                   |
| **PL/pgSQL**   | Nativa de PostgreSQL | Lógica de negocio en triggers y funciones almacenadas |
| **pg_dump**    | 18.0                 | Generación del backup en formato custom               |

---

## 3. Importación de la Base de Datos

### Requisitos previos

- PostgreSQL 18 instalado y en ejecución
- pgAdmin 4 instalado
- Archivo `banco_core_db` (backup en formato custom de `pg_dump`)

### Paso 1 — Crear la base de datos

1. Abrir **pgAdmin 4**
2. En el panel izquierdo, click derecho sobre **Databases** → **Create** → **Database...**
3. Completar los campos:
   - **Database:** `banco_core`
   - **Owner:** `postgres`
4. Click en **Save**

### Paso 2 — Restaurar el backup

1. Click derecho sobre la base de datos `banco_core` → **Restore...**
2. Configurar la ventana de restauración:

| Campo         | Valor                                  |
| ------------- | -------------------------------------- |
| **Format**    | `Custom or tar`                        |
| **Filename**  | Seleccionar el archivo `banco_core_db` |
| **Role name** | `postgres`                             |

3. Ir a la pestaña **Restore options** y activar:
   - ✅ Pre-data
   - ✅ Data
   - ✅ Post-data
4. Click en **Restore**

### Paso 3 — Verificar la restauración

Expandir en el panel izquierdo:

```
banco_core
  └── Schemas
        └── public
              ├── Tables        → 10 tablas
              ├── Functions     → 2 funciones PL/pgSQL
              └── Sequences     → secuencias por cada tabla
```

> **Importante:** El archivo `banco_core_db` fue generado con `pg_dump -F c` (formato custom binario). **No es un archivo `.sql` plano.** Debe restaurarse obligatoriamente con la opción **Restore...** de pgAdmin o con `pg_restore` desde la terminal. No usar **Query Tool** para este archivo.

### Alternativa por terminal

```bash
pg_restore -U postgres -d banco_core -F c --clean banco_core_db
```

---

## 4. Modelo de Datos

### Diagrama de tablas

```
rol_sistema ◄──── usuario_sistema ────► estado_general
                       │
              ┌────────┴─────────┐
              ▼                  ▼
       cliente_persona     cliente_empresa
              │                  │
              └────────┬─────────┘
                       ▼
              producto_bancario
                       │
              ┌────────┼────────┐
              ▼        ▼        ▼
       cuenta_bancaria prestamo transferencia
              │        │        │
              └────────┴────────┘
                       │
                       ▼
              bitacora_operaciones
```

### Descripción de cada tabla

#### `rol_sistema`

Catálogo centralizado de roles de acceso del sistema. Controla qué tipo de operaciones puede realizar cada usuario. Evita hardcodeo de roles en la lógica de aplicación.

| Columna      | Tipo           | Descripción                 |
| ------------ | -------------- | --------------------------- |
| `id_rol`     | `integer` (PK) | Identificador único del rol |
| `nombre_rol` | `varchar(100)` | Nombre descriptivo del rol  |

**Roles disponibles:**

| ID  | Rol                 | Descripción                        |
| --- | ------------------- | ---------------------------------- |
| 1   | CLIENTE_PERSONA     | Cliente persona natural            |
| 2   | CLIENTE_EMPRESA     | Cliente empresa                    |
| 3   | EMPLEADO_VENTANILLA | Operador de sucursal               |
| 4   | EMPLEADO_COMERCIAL  | Asesor comercial                   |
| 5   | EMPLEADO_EMPRESA    | Empleado de empresa cliente        |
| 6   | SUPERVISOR_EMPRESA  | Supervisor de empresa              |
| 7   | ANALISTA_INTERNO    | Analista de crédito                |
| 8   | BACKOFFICE          | Operaciones internas               |
| 9   | ADMIN_SISTEMA       | Administrador del sistema          |
| 10  | AUDITOR             | Auditor con acceso de solo lectura |

---

#### `estado_general`

Catálogo polimórfico de estados reutilizables para múltiples entidades. Un solo catálogo soporta los estados de usuarios, cuentas, préstamos y transferencias, diferenciados por un campo `tipo_entidad`.

| Columna         | Tipo           | Descripción                    |
| --------------- | -------------- | ------------------------------ |
| `id_estado`     | `integer` (PK) | Identificador único del estado |
| `nombre_estado` | `varchar(50)`  | Nombre del estado              |
| `tipo_entidad`  | `varchar(50)`  | Entidad a la que aplica        |

**Estados por entidad:**

| Entidad       | Estados                                                                                                            |
| ------------- | ------------------------------------------------------------------------------------------------------------------ |
| USUARIO       | ACTIVO, INACTIVO, BLOQUEADO, PENDIENTE_VERIFICACION, EN_REVISION, ELIMINADO                                        |
| CUENTA        | ACTIVA, INACTIVA, CERRADA, SUSPENDIDA, EMBARGADA, PENDIENTE_APERTURA, CONGELADA                                    |
| PRESTAMO      | EN_ESTUDIO, PREAPROBADO, APROBADO, RECHAZADO, DESEMBOLSADO, EN_MORA, CANCELADO, REFINANCIADO, PENDIENTE_DOCUMENTOS |
| TRANSFERENCIA | PENDIENTE, APROBADA, RECHAZADA, EN_REVISION, CANCELADA, PROGRAMADA, PROCESADA, FALLIDA                             |

---

#### `producto_bancario`

Catálogo de productos que el banco ofrece (ahorros, corriente, CDT, crédito hipotecario, etc.). El campo `requiere_aprobacion` indica si el producto necesita aprobación de un empleado autorizado antes de activarse.

| Columna               | Tipo               | Descripción                           |
| --------------------- | ------------------ | ------------------------------------- |
| `codigo_producto`     | `varchar(20)` (PK) | Código único del producto             |
| `nombre_producto`     | `varchar(100)`     | Nombre comercial                      |
| `requiere_aprobacion` | `boolean`          | Indica si requiere aprobación interna |

---

#### `cliente_persona`

Almacena los datos propios de clientes que son personas naturales. Separada de `cliente_empresa` para mantener una estructura limpia sin columnas NULL y facilitar validaciones específicas como mayoría de edad o formato de identificación.

| Columna                 | Tipo                    | Descripción                  |
| ----------------------- | ----------------------- | ---------------------------- |
| `id_persona`            | `integer` (PK)          | Identificador único          |
| `nombre_completo`       | `varchar(200)`          | Nombre completo              |
| `numero_identificacion` | `varchar(30)` (UNIQUE)  | Cédula o pasaporte           |
| `fecha_nacimiento`      | `date`                  | Para validar mayoría de edad |
| `correo_electronico`    | `varchar(150)` (UNIQUE) | Correo de contacto           |
| `telefono`              | `varchar(15)`           | CHECK: 7–15 dígitos          |
| `direccion`             | `text`                  | Dirección de residencia      |
| `ciudad`                | `varchar(100)`          | Ciudad                       |

---

#### `cliente_empresa`

Almacena los datos específicos de empresas clientes. Maneja atributos exclusivos como NIT, razón social y representante legal que no aplican para personas naturales.

| Columna               | Tipo                    | Descripción                |
| --------------------- | ----------------------- | -------------------------- |
| `id_empresa`          | `integer` (PK)          | Identificador único        |
| `razon_social`        | `varchar(200)`          | Nombre legal de la empresa |
| `nit`                 | `varchar(30)` (UNIQUE)  | NIT de la empresa          |
| `representante_legal` | `varchar(200)`          | Nombre del representante   |
| `correo_electronico`  | `varchar(150)` (UNIQUE) | Correo corporativo         |
| `telefono`            | `varchar(15)`           | CHECK: 7–15 dígitos        |
| `ciudad`              | `varchar(100)`          | Ciudad de operación        |

---

#### `usuario_sistema`

Tabla central de acceso al sistema. Unifica bajo un mismo modelo tanto a clientes (persona o empresa) como a empleados internos. Cada usuario tiene un rol, un estado y una referencia al tipo de entidad relacionada.

| Columna              | Tipo                            | Descripción                             |
| -------------------- | ------------------------------- | --------------------------------------- |
| `id_usuario`         | `integer` (PK)                  | Identificador único                     |
| `nombre_completo`    | `varchar(200)`                  | Nombre para visualización               |
| `correo_electronico` | `varchar(150)` (UNIQUE)         | Correo de acceso                        |
| `contrasena_hash`    | `text`                          | Contraseña hasheada                     |
| `id_rol`             | `integer` (FK → rol_sistema)    | Rol asignado                            |
| `id_estado`          | `integer` (FK → estado_general) | Estado actual                           |
| `tipo_relacion`      | `varchar(10)`                   | CHECK: PERSONA \| EMPRESA               |
| `id_referencia`      | `integer`                       | ID en cliente_persona o cliente_empresa |
| `fecha_creacion`     | `timestamp`                     | Fecha de registro                       |

---

#### `cuenta_bancaria`

Registra las cuentas financieras asociadas a clientes. Soporta titulares persona o empresa. Incluye restricciones de saldo no negativo y estado activo para operaciones.

| Columna           | Tipo                                   | Descripción                            |
| ----------------- | -------------------------------------- | -------------------------------------- |
| `numero_cuenta`   | `varchar(30)` (PK)                     | Número único de cuenta                 |
| `tipo_cuenta`     | `varchar(50)`                          | Ahorros, corriente, empresarial        |
| `saldo_actual`    | `numeric(18,2)`                        | Saldo disponible (CHECK ≥ 0)           |
| `moneda`          | `varchar(10)`                          | COP, USD, EUR                          |
| `id_estado`       | `integer` (FK → estado_general)        | Estado de la cuenta                    |
| `tipo_titular`    | `varchar(10)`                          | CHECK: PERSONA \| EMPRESA              |
| `id_titular`      | `integer`                              | FK a cliente_persona o cliente_empresa |
| `codigo_producto` | `varchar(20)` (FK → producto_bancario) | Producto asociado                      |
| `fecha_apertura`  | `date`                                 | Fecha de apertura                      |

---

#### `prestamo`

Gestiona el ciclo completo de un préstamo: desde la solicitud inicial hasta el desembolso, con trazabilidad del usuario que lo creó y del analista que lo aprobó. Soporta tanto clientes persona como empresa.

| Columna                     | Tipo                                 | Descripción                            |
| --------------------------- | ------------------------------------ | -------------------------------------- |
| `id_prestamo`               | `integer` (PK)                       | Identificador único                    |
| `tipo_cliente`              | `varchar(10)`                        | CHECK: PERSONA \| EMPRESA              |
| `id_cliente_solicitante`    | `integer`                            | FK a cliente_persona o cliente_empresa |
| `monto_solicitado`          | `numeric(18,2)`                      | Monto pedido por el cliente            |
| `monto_aprobado`            | `numeric(18,2)`                      | Monto aprobado por el banco            |
| `tasa_interes`              | `numeric(5,2)`                       | Tasa aplicada                          |
| `plazo_meses`               | `integer`                            | Plazo en meses                         |
| `id_estado`                 | `integer` (FK → estado_general)      | Estado actual del préstamo             |
| `cuenta_destino_desembolso` | `varchar(30)` (FK → cuenta_bancaria) | Cuenta donde se acredita               |
| `id_usuario_creador`        | `integer` (FK → usuario_sistema)     | Quien registró la solicitud            |
| `id_usuario_aprobador`      | `integer` (FK → usuario_sistema)     | Quien aprobó                           |
| `fecha_solicitud`           | `timestamp`                          | Fecha de la solicitud                  |
| `fecha_aprobacion`          | `timestamp`                          | Fecha de aprobación                    |

---

#### `transferencia`

Registra los movimientos de fondos entre cuentas. Implementa un flujo de aprobación donde la transferencia pasa por estado PENDIENTE antes de ser APROBADA. Los triggers garantizan que el saldo siempre sea suficiente.

| Columna                | Tipo                                 | Descripción                  |
| ---------------------- | ------------------------------------ | ---------------------------- |
| `id_transferencia`     | `integer` (PK)                       | Identificador único          |
| `cuenta_origen`        | `varchar(30)` (FK → cuenta_bancaria) | Cuenta debitada              |
| `cuenta_destino`       | `varchar(30)` (FK → cuenta_bancaria) | Cuenta acreditada            |
| `monto`                | `numeric(18,2)`                      | CHECK: monto > 0             |
| `id_estado`            | `integer` (FK → estado_general)      | Estado de la transferencia   |
| `id_usuario_creador`   | `integer` (FK → usuario_sistema)     | Quien la registró            |
| `id_usuario_aprobador` | `integer` (FK → usuario_sistema)     | Quien la aprobó              |
| `fecha_creacion`       | `timestamp`                          | Fecha de registro            |
| `fecha_aprobacion`     | `timestamp`                          | Fecha de aprobación          |
| `descripcion`          | `text`                               | Concepto de la transferencia |

---

#### `bitacora_operaciones`

Tabla de auditoría que registra toda operación significativa realizada en el sistema. Permite rastrear qué usuario realizó qué acción, sobre qué entidad y en qué momento. Es inmutable por diseño.

| Columna                  | Tipo                             | Descripción                     |
| ------------------------ | -------------------------------- | ------------------------------- |
| `id_bitacora`            | `integer` (PK)                   | Identificador único             |
| `entidad_afectada`       | `varchar(100)`                   | Nombre de la tabla afectada     |
| `id_entidad`             | `integer`                        | ID del registro afectado        |
| `accion`                 | `varchar(50)`                    | INSERT, UPDATE, APPROVE, REJECT |
| `id_usuario_responsable` | `integer` (FK → usuario_sistema) | Ejecutor de la acción           |
| `fecha_operacion`        | `timestamp`                      | Momento exacto de la operación  |
| `detalle`                | `text`                           | Descripción detallada           |

---

## 5. Interpretación del Ejercicio

### Separación de clientes persona y empresa

La decisión de crear dos tablas independientes (`cliente_persona` y `cliente_empresa`) responde a una diferencia estructural en los datos:

- Una **persona natural** tiene: cédula, fecha de nacimiento, nombre completo.
- Una **empresa** tiene: NIT, razón social, representante legal, fecha de constitución.

Fusionar estos datos en una sola tabla hubiera generado columnas con valores NULL en el 50% de los registros, violando la 3FN y dificultando la aplicación de restricciones `UNIQUE` y `CHECK` específicas por tipo.

### Usuario sistema centralizado

`usuario_sistema` resuelve el problema de tener múltiples tipos de actores (clientes y empleados) accediendo al mismo sistema bajo reglas distintas. La columna `tipo_relacion` junto con `id_referencia` actúa como una clave foránea polimórfica que apunta a `cliente_persona` o `cliente_empresa` según corresponda. Esto permite:

- Desactivar un usuario sin eliminar sus datos históricos
- Asignar múltiples roles sin duplicar registros
- Mantener un único punto de control de acceso

### Gestión de cuentas y préstamos

Las cuentas bancarias soportan múltiples tipos de titular mediante el patrón `tipo_titular + id_titular`, evitando la necesidad de FK separadas para persona y empresa. Los préstamos replican este patrón con `tipo_cliente + id_cliente_solicitante`.

Los préstamos tienen un flujo de estados explícito:

```
EN_ESTUDIO → PREAPROBADO → APROBADO → DESEMBOLSADO
                        ↘ RECHAZADO
```

El campo `cuenta_destino_desembolso` asegura que al aprobar un préstamo, los fondos se acrediten directamente a una cuenta del cliente.

### Gestión de transferencias y control de fondos

Las transferencias implementan un flujo de dos fases:

```
PENDIENTE → (validación de fondos) → APROBADA
                                  ↘ RECHAZADA
```

Esta separación es clave: permite que un supervisor revise transferencias antes de ejecutarlas, y garantiza mediante triggers que al momento de la aprobación los fondos sigan disponibles, independientemente del tiempo transcurrido desde la solicitud.

### Bitácora como garantía de auditoría

En un sistema bancario, el cumplimiento normativo exige trazabilidad completa. La `bitacora_operaciones` registra automáticamente cada operación crítica (apertura de cuentas, aprobación de préstamos, transferencias ejecutadas), permitiendo reconstruir la historia de cualquier entidad en cualquier momento.

### Normalización aplicada

El modelo cumple con la **Tercera Forma Normal (3FN)**:

- **1FN:** Todos los atributos son atómicos, sin grupos repetitivos.
- **2FN:** Todos los atributos dependen completamente de la PK (no de parte de ella).
- **3FN:** No existen dependencias transitivas; los catálogos (`rol_sistema`, `estado_general`, `producto_bancario`) centralizan los valores de dominio.

---

## 6. Integridad de Datos

### Primary Key

Todas las tablas tienen PK definida. Las tablas con columna `id` usan secuencias automáticas (`SERIAL` / `SEQUENCE` con `nextval`). `cuenta_bancaria` usa `numero_cuenta` como PK natural dado que es un identificador único de negocio.

### Foreign Key

Todas las relaciones entre tablas están respaldadas por FK explícitas:

| Tabla                  | FK                          | Referencia                          |
| ---------------------- | --------------------------- | ----------------------------------- |
| `usuario_sistema`      | `id_rol`                    | `rol_sistema.id_rol`                |
| `usuario_sistema`      | `id_estado`                 | `estado_general.id_estado`          |
| `cuenta_bancaria`      | `id_estado`                 | `estado_general.id_estado`          |
| `cuenta_bancaria`      | `codigo_producto`           | `producto_bancario.codigo_producto` |
| `prestamo`             | `id_estado`                 | `estado_general.id_estado`          |
| `prestamo`             | `cuenta_destino_desembolso` | `cuenta_bancaria.numero_cuenta`     |
| `prestamo`             | `id_usuario_creador`        | `usuario_sistema.id_usuario`        |
| `prestamo`             | `id_usuario_aprobador`      | `usuario_sistema.id_usuario`        |
| `transferencia`        | `cuenta_origen`             | `cuenta_bancaria.numero_cuenta`     |
| `transferencia`        | `cuenta_destino`            | `cuenta_bancaria.numero_cuenta`     |
| `transferencia`        | `id_estado`                 | `estado_general.id_estado`          |
| `transferencia`        | `id_usuario_creador`        | `usuario_sistema.id_usuario`        |
| `transferencia`        | `id_usuario_aprobador`      | `usuario_sistema.id_usuario`        |
| `bitacora_operaciones` | `id_usuario_responsable`    | `usuario_sistema.id_usuario`        |

### Unique

| Tabla             | Columnas con restricción UNIQUE               |
| ----------------- | --------------------------------------------- |
| `cliente_persona` | `numero_identificacion`, `correo_electronico` |
| `cliente_empresa` | `nit`, `correo_electronico`                   |
| `usuario_sistema` | `correo_electronico`                          |
| `cuenta_bancaria` | `numero_cuenta` (es la PK)                    |

### Check

| Tabla             | Restricción       | Condición                                                |
| ----------------- | ----------------- | -------------------------------------------------------- |
| `cliente_persona` | Mayoría de edad   | `fecha_nacimiento <= CURRENT_DATE - INTERVAL '18 years'` |
| `cliente_persona` | Teléfono válido   | `char_length(telefono) BETWEEN 7 AND 15`                 |
| `cliente_empresa` | Teléfono válido   | `char_length(telefono) BETWEEN 7 AND 15`                 |
| `usuario_sistema` | Tipo de relación  | `tipo_relacion IN ('PERSONA', 'EMPRESA')`                |
| `cuenta_bancaria` | Tipo de titular   | `tipo_titular IN ('PERSONA', 'EMPRESA')`                 |
| `cuenta_bancaria` | Saldo no negativo | `saldo_actual >= 0`                                      |
| `prestamo`        | Tipo de cliente   | `tipo_cliente IN ('PERSONA', 'EMPRESA')`                 |
| `transferencia`   | Monto positivo    | `monto > 0`                                              |

---

## 7. Índices Estratégicos

Los índices se aplicaron en campos de búsqueda frecuente y columnas usadas en `JOIN`, `WHERE` y `ORDER BY`:

| Índice                               | Tabla                  | Columna(s)                     | Justificación                             |
| ------------------------------------ | ---------------------- | ------------------------------ | ----------------------------------------- |
| `idx_cliente_persona_identificacion` | `cliente_persona`      | `numero_identificacion`        | Búsqueda frecuente por documento          |
| `idx_cliente_empresa_nit`            | `cliente_empresa`      | `nit`                          | Identificación de empresas en tiempo real |
| `idx_usuario_sistema_rol`            | `usuario_sistema`      | `id_rol`                       | Filtros por tipo de usuario               |
| `idx_usuario_sistema_estado`         | `usuario_sistema`      | `id_estado`                    | Obtener usuarios activos/bloqueados       |
| `idx_cuenta_bancaria_titular`        | `cuenta_bancaria`      | `id_titular, tipo_titular`     | Consultar cuentas por dueño               |
| `idx_cuenta_bancaria_estado`         | `cuenta_bancaria`      | `id_estado`                    | Filtrar cuentas activas                   |
| `idx_prestamo_solicitante`           | `prestamo`             | `id_cliente_solicitante`       | Historial de préstamos por cliente        |
| `idx_prestamo_estado`                | `prestamo`             | `id_estado`                    | Préstamos en mora o en revisión           |
| `idx_transferencia_cuenta_origen`    | `transferencia`        | `cuenta_origen`                | Historial de débitos por cuenta           |
| `idx_transferencia_cuenta_destino`   | `transferencia`        | `cuenta_destino`               | Historial de créditos por cuenta          |
| `idx_transferencia_estado`           | `transferencia`        | `id_estado`                    | Filtrar transferencias pendientes         |
| `idx_bitacora_entidad`               | `bitacora_operaciones` | `entidad_afectada, id_entidad` | Auditoría por entidad específica          |

---

## 8. Triggers Implementados

### Caso 1 — Control de fondos en transferencias masivas (BEFORE INSERT)

#### Problema

Un cliente puede intentar registrar múltiples transferencias consecutivas en un período corto. Si el sistema no valida el saldo en el momento exacto del INSERT, una cuenta podría comprometerse más allá de su saldo real antes de que llegue la aprobación.

#### Solución

Un trigger `BEFORE INSERT` sobre la tabla `transferencia` verifica que la cuenta origen tenga fondos suficientes en el instante en que se registra cada nueva transferencia pendiente. Si el saldo disponible es menor al monto solicitado, el INSERT se cancela con una excepción.

#### Función PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION public.validar_fondos_transferencia()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    saldo_actual NUMERIC;
BEGIN
    -- Obtener el saldo actual de la cuenta origen con bloqueo de fila
    SELECT saldo_actual
    INTO saldo_actual
    FROM cuenta_bancaria
    WHERE numero_cuenta = NEW.cuenta_origen
    FOR UPDATE;

    -- Si la cuenta no existe, rechazar la operación
    IF NOT FOUND THEN
        RAISE EXCEPTION 'La cuenta origen % no existe.', NEW.cuenta_origen;
    END IF;

    -- Si no hay fondos suficientes, cancelar el INSERT
    IF saldo_actual < NEW.monto THEN
        RAISE EXCEPTION
            'Fondos insuficientes en cuenta %. Saldo disponible: %, Monto requerido: %',
            NEW.cuenta_origen, saldo_actual, NEW.monto;
    END IF;

    RETURN NEW;
END;
$$;
```

#### Trigger

```sql
CREATE TRIGGER tg_validar_fondos_transferencia
BEFORE INSERT ON public.transferencia
FOR EACH ROW
EXECUTE FUNCTION public.validar_fondos_transferencia();
```

#### Lógica del trigger

1. Antes de insertar la transferencia, se consulta el saldo actual de la cuenta origen.
2. Se usa `FOR UPDATE` para bloquear la fila durante la transacción y evitar condiciones de carrera (race conditions).
3. Si el saldo es insuficiente, se lanza una excepción que aborta el INSERT.
4. Si hay fondos, el registro entra en estado `PENDIENTE` y la validación de descuento real ocurre al momento de la aprobación (Caso 2).

---

### Caso 2 — Aprobación de transferencias con control de fondos concurrente (BEFORE UPDATE)

#### Problema

Cuando hay múltiples transferencias en estado `PENDIENTE` para la misma cuenta origen, al ir aprobándolas una por una puede ocurrir que la primera consuma todo el saldo disponible y no quede nada para las siguientes. Sin control, el sistema podría aprobar transferencias sin fondos reales.

#### Solución

Un trigger `BEFORE UPDATE` que se activa cuando una transferencia cambia su estado a `APROBADA`. En ese momento, verifica nuevamente el saldo real de la cuenta y, si es suficiente, lo descuenta. Si no hay fondos, cambia automáticamente el estado a `RECHAZADA` en lugar de aprobarla.

#### Función PL/pgSQL

```sql
CREATE OR REPLACE FUNCTION public.aprobar_transferencia_validar_fondos()
RETURNS trigger
LANGUAGE plpgsql
AS $$
DECLARE
    saldo_actual NUMERIC;
    id_estado_aprobada INTEGER;
    id_estado_rechazada INTEGER;
BEGIN
    -- Solo actuar cuando el estado cambia a APROBADA
    IF NEW.id_estado = OLD.id_estado THEN
        RETURN NEW;
    END IF;

    -- Obtener el ID del estado APROBADA
    SELECT id_estado INTO id_estado_aprobada
    FROM estado_general
    WHERE nombre_estado = 'APROBADA' AND tipo_entidad = 'TRANSFERENCIA';

    -- Solo proceder si el nuevo estado es APROBADA
    IF NEW.id_estado != id_estado_aprobada THEN
        RETURN NEW;
    END IF;

    -- Obtener el ID del estado RECHAZADA para uso en caso de fondos insuficientes
    SELECT id_estado INTO id_estado_rechazada
    FROM estado_general
    WHERE nombre_estado = 'RECHAZADA' AND tipo_entidad = 'TRANSFERENCIA';

    -- Bloquear la fila de la cuenta origen para evitar concurrencia
    SELECT saldo_actual
    INTO saldo_actual
    FROM cuenta_bancaria
    WHERE numero_cuenta = NEW.cuenta_origen
    FOR UPDATE;

    -- Verificar si hay saldo suficiente al momento de la aprobación
    IF saldo_actual < NEW.monto THEN
        -- No hay fondos: rechazar automáticamente
        NEW.id_estado := id_estado_rechazada;
        RETURN NEW;
    END IF;

    -- Hay fondos: descontar de la cuenta origen y acreditar en la destino
    UPDATE cuenta_bancaria
    SET saldo_actual = saldo_actual - NEW.monto
    WHERE numero_cuenta = NEW.cuenta_origen;

    UPDATE cuenta_bancaria
    SET saldo_actual = saldo_actual + NEW.monto
    WHERE numero_cuenta = NEW.cuenta_destino;

    -- Registrar en bitácora
    INSERT INTO bitacora_operaciones (entidad_afectada, id_entidad, accion, detalle, fecha_operacion)
    VALUES (
        'transferencia',
        NEW.id_transferencia,
        'APPROVE',
        format('Transferencia aprobada: %s → %s por $%s', NEW.cuenta_origen, NEW.cuenta_destino, NEW.monto),
        NOW()
    );

    RETURN NEW;
END;
$$;
```

#### Trigger

```sql
CREATE TRIGGER tg_aprobar_transferencia_validar_fondos
BEFORE UPDATE ON public.transferencia
FOR EACH ROW
EXECUTE FUNCTION public.aprobar_transferencia_validar_fondos();
```

#### Lógica del trigger

1. Se activa únicamente cuando el campo `id_estado` de una transferencia cambia a `APROBADA`.
2. Se aplica `FOR UPDATE` sobre la cuenta origen para serializar el acceso concurrente — si dos aprobaciones ocurren en paralelo, la segunda espera a que la primera libere el bloqueo.
3. Si el saldo es suficiente: se descuenta de la cuenta origen y se acredita en la cuenta destino de forma atómica.
4. Si el saldo ya no es suficiente (fue consumido por una transferencia anterior): el estado de la transferencia cambia automáticamente a `RECHAZADA`, sin abortar la transacción.
5. Se registra la operación en `bitacora_operaciones` para trazabilidad completa.

#### Escenario de ejemplo

```
Cuenta A: saldo = $1,000
  Transferencia T1: $700 → PENDIENTE
  Transferencia T2: $500 → PENDIENTE

Aprobar T1: saldo $1,000 >= $700 ✓ → APROBADA → Cuenta A: saldo = $300
Aprobar T2: saldo $300 < $500 ✗  → Cambia a RECHAZADA automáticamente
```

---

## 9. Consultas SQL

### 9.1 Subconsultas

Las subconsultas permiten filtrar con base en el resultado de otra consulta sin necesidad de un JOIN explícito.

**Clientes persona con al menos un préstamo aprobado**

```sql
-- Retorna los clientes que tienen monto_aprobado definido (préstamo aprobado)
SELECT nombre_completo
FROM public.cliente_persona
WHERE id_persona IN (
    SELECT id_cliente_solicitante
    FROM public.prestamo
    WHERE monto_aprobado IS NOT NULL
      AND tipo_cliente = 'PERSONA'
);
```

**Cuentas con saldo superior al promedio del sistema**

```sql
-- Útil para detectar cuentas de alto valor para productos premium
SELECT numero_cuenta, tipo_cuenta, saldo_actual
FROM public.cuenta_bancaria
WHERE saldo_actual > (
    SELECT AVG(saldo_actual)
    FROM public.cuenta_bancaria
    WHERE id_estado = (
        SELECT id_estado FROM public.estado_general
        WHERE nombre_estado = 'ACTIVA' AND tipo_entidad = 'CUENTA'
    )
);
```

**Transferencias que superan el monto máximo del mes anterior**

```sql
-- Detección de transferencias inusualmente altas para análisis de riesgo
SELECT id_transferencia, cuenta_origen, cuenta_destino, monto, fecha_creacion
FROM public.transferencia
WHERE monto > (
    SELECT MAX(monto)
    FROM public.transferencia
    WHERE fecha_creacion >= date_trunc('month', CURRENT_DATE - INTERVAL '1 month')
      AND fecha_creacion <  date_trunc('month', CURRENT_DATE)
);
```

---

### 9.2 Filtros de Igualación

Consultas directas utilizando `WHERE` con condiciones de igualdad exacta.

```sql
-- Todas las cuentas en pesos colombianos
SELECT numero_cuenta, tipo_cuenta, saldo_actual
FROM public.cuenta_bancaria
WHERE moneda = 'COP';

-- Préstamos con plazo exacto de 12 meses
SELECT id_prestamo, monto_aprobado, tasa_interes
FROM public.prestamo
WHERE plazo_meses = 12;

-- Usuarios con rol CLIENTE_EMPRESA (id_rol = 2)
SELECT id_usuario, nombre_completo, correo_electronico
FROM public.usuario_sistema
WHERE id_rol = 2;

-- Transferencias en estado PENDIENTE
SELECT id_transferencia, cuenta_origen, cuenta_destino, monto
FROM public.transferencia
WHERE id_estado = (
    SELECT id_estado FROM public.estado_general
    WHERE nombre_estado = 'PENDIENTE' AND tipo_entidad = 'TRANSFERENCIA'
);
```

---

### 9.3 INNER JOIN

Devuelve únicamente los registros que tienen coincidencia en ambas tablas.

```sql
-- Transferencias con nombre del usuario que las registró
-- Solo muestra transferencias que tienen un usuario creador registrado
SELECT
    t.id_transferencia,
    t.cuenta_origen,
    t.cuenta_destino,
    t.monto,
    u.nombre_completo AS creado_por
FROM public.transferencia t
INNER JOIN public.usuario_sistema u ON t.id_usuario_creador = u.id_usuario;
```

```sql
-- Préstamos con la cuenta de destino de desembolso
-- Muestra solo préstamos que tienen cuenta de desembolso asignada
SELECT
    p.id_prestamo,
    p.monto_aprobado,
    p.plazo_meses,
    c.numero_cuenta,
    c.tipo_cuenta
FROM public.prestamo p
INNER JOIN public.cuenta_bancaria c ON p.cuenta_destino_desembolso = c.numero_cuenta;
```

```sql
-- Usuarios con su rol y estado actual
-- Visión unificada de acceso al sistema
SELECT
    u.id_usuario,
    u.nombre_completo,
    r.nombre_rol,
    e.nombre_estado AS estado
FROM public.usuario_sistema u
INNER JOIN public.rol_sistema r   ON u.id_rol    = r.id_rol
INNER JOIN public.estado_general e ON u.id_estado = e.id_estado;
```

---

### 9.4 LEFT JOIN

Devuelve todos los registros de la tabla izquierda aunque no tengan coincidencia en la derecha (el lado derecho sale como NULL).

```sql
-- Todas las cuentas y sus transferencias salientes
-- Incluye cuentas sin ninguna transferencia registrada
SELECT
    c.numero_cuenta,
    c.tipo_cuenta,
    c.saldo_actual,
    t.id_transferencia,
    t.monto
FROM public.cuenta_bancaria c
LEFT JOIN public.transferencia t ON c.numero_cuenta = t.cuenta_origen
ORDER BY c.numero_cuenta;
```

```sql
-- Todos los préstamos y su registro en bitácora
-- Identifica préstamos sin ningún evento auditado
SELECT
    p.id_prestamo,
    p.monto_aprobado,
    b.accion,
    b.fecha_operacion,
    b.detalle
FROM public.prestamo p
LEFT JOIN public.bitacora_operaciones b
    ON b.id_entidad = p.id_prestamo
   AND b.entidad_afectada = 'prestamo';
```

```sql
-- Todos los usuarios y las transferencias que han aprobado
-- Muestra NULL donde el usuario nunca aprobó una transferencia
SELECT
    u.id_usuario,
    u.nombre_completo,
    COUNT(t.id_transferencia) AS transferencias_aprobadas
FROM public.usuario_sistema u
LEFT JOIN public.transferencia t ON u.id_usuario = t.id_usuario_aprobador
GROUP BY u.id_usuario, u.nombre_completo
ORDER BY transferencias_aprobadas DESC;
```

---

### 9.5 RIGHT JOIN

Devuelve todos los registros de la tabla derecha aunque no tengan coincidencia en la izquierda (el lado izquierdo sale como NULL).

```sql
-- Todas las cuentas con sus transferencias salientes
-- Inverso del LEFT JOIN anterior: garantiza que ninguna cuenta quede fuera
SELECT
    t.id_transferencia,
    t.monto,
    c.numero_cuenta,
    c.tipo_cuenta,
    c.saldo_actual
FROM public.transferencia t
RIGHT JOIN public.cuenta_bancaria c ON t.cuenta_origen = c.numero_cuenta;
```

```sql
-- Todos los préstamos con su bitácora de auditoría
-- Muestra préstamos aunque no tengan eventos en bitácora
SELECT
    b.id_bitacora,
    b.accion,
    b.fecha_operacion,
    p.id_prestamo,
    p.monto_aprobado
FROM public.bitacora_operaciones b
RIGHT JOIN public.prestamo p
    ON b.id_entidad = p.id_prestamo
   AND b.entidad_afectada = 'prestamo';
```

```sql
-- Todos los usuarios con las transferencias donde son creadores
-- Incluye usuarios que jamás han creado una transferencia
SELECT
    u.id_usuario,
    u.nombre_completo,
    t.id_transferencia,
    t.monto,
    t.fecha_creacion
FROM public.transferencia t
RIGHT JOIN public.usuario_sistema u ON t.id_usuario_creador = u.id_usuario
ORDER BY u.id_usuario;
```

---

## 10. Conclusión

**Banco Core** demuestra que es posible implementar la lógica de negocio crítica de un sistema financiero directamente en la capa de base de datos, obteniendo garantías de consistencia que ninguna capa de aplicación podría ofrecer por sí sola.

### Lo que este proyecto demuestra

| Aspecto                 | Implementación                                                            |
| ----------------------- | ------------------------------------------------------------------------- |
| **Modelado relacional** | 10 tablas en 3FN con relaciones explícitas y catálogos normalizados       |
| **Integridad de datos** | PK, FK, UNIQUE y CHECK cubren todos los invariantes de negocio            |
| **Reglas de negocio**   | Dos triggers PL/pgSQL con control transaccional y bloqueo de concurrencia |
| **Auditoría**           | Bitácora automática de operaciones críticas                               |
| **Escalabilidad**       | Índices estratégicos e independencia por tipo de entidad                  |
| **Simulación real**     | Flujos completos de préstamos y transferencias con estados y aprobadores  |

### Aprendizajes clave

- El uso de `FOR UPDATE` en los triggers es indispensable en sistemas bancarios para evitar que dos transacciones concurrentes lean el mismo saldo y ambas crean que hay fondos suficientes.
- Separar los datos de negocio (clientes, cuentas) de los datos de control (estados, roles) mediante catálogos normalizados reduce el acoplamiento y facilita la evolución del sistema.
- Los triggers `BEFORE` permiten modificar o cancelar operaciones antes de que afecten la base de datos, siendo la herramienta correcta para validaciones que deben ser atómicas con la operación que las dispara.

---

_Banco Core — Emmanuel Berrio Jimenez | ET0062 Bases de Datos II | Grupo 800_
