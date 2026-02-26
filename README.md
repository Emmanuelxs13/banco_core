# Banco Core – Sistema de Gestión Bancaria (PostgreSQL)

**Grupo 800 | ET0062 - Bases de Datos II**  
**Emmanuel Berrio Jimenez**

---

## 1. Descripción del Proyecto

Diseño e implementación de una base de datos relacional en **PostgreSQL 18** para soportar el core transaccional de una entidad bancaria.

| Módulo | Descripción |
|---|---|
| Clientes | Persona natural y empresa |
| Usuarios | Control de acceso por roles |
| Cuentas bancarias | Ahorros, corriente, empresarial |
| Préstamos | Flujo de solicitud, aprobación y desembolso |
| Transferencias | Con control de autorización |
| Bitácora | Registro de auditoría de operaciones |
| Catálogos | Roles, estados y productos centralizados |

**Criterios de diseño:** Normalización (3FN) · Integridad referencial · Indexación estratégica · Escalabilidad

---

## 2. Importación de la Base de Datos

### Requisitos
- PostgreSQL 18
- pgAdmin 4

### Pasos

**1. Crear la base de datos**

- En pgAdmin 4, click derecho en **Databases** → **Create** → **Database**
- Nombre: `banco_core`
- Owner: `postgres`
- Click en **Save**

**2. Restaurar el archivo**

- Click derecho sobre `banco_core` → **Restore...**
- **Filename:** seleccionar el archivo `banco_core_db` (sin extensión)
- **Format:** `Custom or tar`
- **Role name:** `postgres`
- En la pestaña **Restore options**, activar:
  - Pre-data
  - Data
  - Post-data
- Click en **Restore**

**3. Verificar**

- Expandir `banco_core` → **Schemas** → **public** → **Tables**
- Deben aparecer las 10 tablas del sistema

> **Nota:** El archivo `banco_core_db` fue generado con `pg_dump` en formato `custom`. No es un archivo `.sql` plano; debe restaurarse con la opción **Restore...** de pgAdmin, no con **Query Tool**.

---

## 3. Modelo de Datos

### Tablas del sistema

| Tabla | Descripción | PK |
|---|---|---|
| `rol_sistema` | Catálogo de roles de usuario | `id_rol` |
| `estado_general` | Catálogo de estados por tipo de entidad | `id_estado` |
| `producto_bancario` | Catálogo de productos del banco | `codigo_producto` |
| `cliente_persona` | Clientes persona natural | `id_persona` |
| `cliente_empresa` | Clientes empresa | `id_empresa` |
| `usuario_sistema` | Usuarios con rol y estado | `id_usuario` |
| `cuenta_bancaria` | Cuentas asociadas a clientes | `numero_cuenta` |
| `prestamo` | Préstamos con flujo de aprobación | `id_prestamo` |
| `transferencia` | Transferencias entre cuentas | `id_transferencia` |
| `bitacora_operaciones` | Registro de auditoría | `id_bitacora` |

### Roles del sistema (`rol_sistema`)

| ID | Rol |
|---|---|
| 1 | CLIENTE_PERSONA |
| 2 | CLIENTE_EMPRESA |
| 3 | EMPLEADO_VENTANILLA |
| 4 | EMPLEADO_COMERCIAL |
| 5 | EMPLEADO_EMPRESA |
| 6 | SUPERVISOR_EMPRESA |
| 7 | ANALISTA_INTERNO |
| 8 | BACKOFFICE |
| 9 | ADMIN_SISTEMA |
| 10 | AUDITOR |

### Estados generales (`estado_general`)

| Tipo | Estados |
|---|---|
| USUARIO | ACTIVO, INACTIVO, BLOQUEADO, PENDIENTE_VERIFICACION, EN_REVISION, ELIMINADO |
| CUENTA | ACTIVA, INACTIVA, CERRADA, SUSPENDIDA, EMBARGADA, PENDIENTE_APERTURA, CONGELADA |
| PRESTAMO | EN_ESTUDIO, PREAPROBADO, APROBADO, RECHAZADO, DESEMBOLSADO, EN_MORA, CANCELADO, REFINANCIADO, PENDIENTE_DOCUMENTOS |
| TRANSFERENCIA | PENDIENTE, APROBADA, RECHAZADA, EN_REVISION, CANCELADA, PROGRAMADA, PROCESADA, FALLIDA |

---

## 4. Decisiones de Arquitectura

### Separación de tipos de cliente

Se crearon tablas independientes `cliente_persona` y `cliente_empresa` porque tienen atributos distintos (NIT vs. identificación, representante legal, etc.), evitando columnas NULL innecesarias y facilitando la escalabilidad.

### Usuario centralizado

`usuario_sistema` unifica acceso de clientes y empleados bajo un mismo modelo con `id_rol`, `id_estado` y `tipo_relacion (PERSONA | EMPRESA)`. Permite desactivar usuarios sin eliminarlos y mantener auditoría completa.

### Catálogos normalizados

`rol_sistema`, `estado_general` y `producto_bancario` centralizan los valores evitando hardcodeo y asegurando integridad referencial en todas las entidades.

### Flujos de aprobación

Préstamos y transferencias incluyen `id_usuario_creador`, `id_usuario_aprobador` y fechas de aprobación como FK, garantizando trazabilidad completa del ciclo de vida de cada operación.

### Bitácora de operaciones

`bitacora_operaciones` registra entidad afectada, ID, acción, usuario responsable, fecha y detalle. Soporta cumplimiento normativo y auditoría financiera.

---

## 5. Integridad y Restricciones

| Tipo | Detalle |
|---|---|
| PRIMARY KEY | Todas las tablas |
| FOREIGN KEY | Relaciones entre todas las entidades |
| UNIQUE | Identificación, NIT, correo electrónico, número de cuenta |
| CHECK | Mayoría de edad, formato de teléfono (7–15 dígitos), saldo ≥ 0, monto > 0 |
| CHECK | `tipo_titular` ∈ {PERSONA, EMPRESA}, `tipo_cliente` ∈ {PERSONA, EMPRESA} |

---

## 6. Índices Estratégicos

Campos indexados para optimizar búsquedas frecuentes y joins:

- `numero_identificacion`, `nit`
- `id_estado`, `id_rol`
- FK de titular, solicitante y aprobador
- `cuenta_origen`, `cuenta_destino` en transferencias
- `entidad_afectada` en bitácora

---

## 7. Consultas Implementadas

### 7.1 Subconsultas

**Clientes con préstamos aprobados**
```sql
SELECT nombre_completo
FROM cliente_persona
WHERE id_persona IN (
    SELECT id_cliente_solicitante
    FROM prestamo
    WHERE monto_aprobado IS NOT NULL
);
```

**Cuentas con saldo mayor al promedio**
```sql
SELECT numero_cuenta, saldo_actual
FROM cuenta_bancaria
WHERE saldo_actual > (
    SELECT AVG(saldo_actual) FROM cuenta_bancaria
);
```

**Transferencias mayores al máximo del mes anterior**
```sql
SELECT *
FROM transferencia
WHERE monto > (
    SELECT MAX(monto)
    FROM transferencia
    WHERE fecha_creacion >= date_trunc('month', CURRENT_DATE - interval '1 month')
);
```

---

### 7.2 Filtro de Igualación

```sql
-- Cuentas en pesos colombianos
SELECT * FROM cuenta_bancaria
WHERE moneda = 'COP';

-- Préstamos a 12 meses
SELECT * FROM prestamo
WHERE plazo_meses = 12;

-- Usuarios con rol CLIENTE_EMPRESA
SELECT * FROM usuario_sistema
WHERE id_rol = 2;
```

---

### 7.3 INNER JOIN

```sql
-- Transferencias con el nombre del usuario que las creó
SELECT t.id_transferencia, u.nombre_completo
FROM transferencia t
JOIN usuario_sistema u ON t.id_usuario_creador = u.id_usuario;

-- Préstamos con la cuenta de desembolso
SELECT p.id_prestamo, c.numero_cuenta
FROM prestamo p
JOIN cuenta_bancaria c ON p.cuenta_destino_desembolso = c.numero_cuenta;

-- Usuarios con su rol
SELECT u.nombre_completo, r.nombre_rol
FROM usuario_sistema u
JOIN rol_sistema r ON u.id_rol = r.id_rol;
```

---

### 7.4 LEFT JOIN

```sql
-- Cuentas y sus transferencias salientes (incluye cuentas sin transferencias)
SELECT c.numero_cuenta, t.id_transferencia
FROM cuenta_bancaria c
LEFT JOIN transferencia t ON c.numero_cuenta = t.cuenta_origen;

-- Préstamos y su historial en bitácora
SELECT p.id_prestamo, b.detalle
FROM prestamo p
LEFT JOIN bitacora_operaciones b ON p.id_prestamo = b.id_entidad;

-- Usuarios y transferencias que han aprobado
SELECT u.nombre_completo, t.id_transferencia
FROM usuario_sistema u
LEFT JOIN transferencia t ON u.id_usuario = t.id_usuario_aprobador;
```

---

### 7.5 RIGHT JOIN

```sql
-- Transferencias por cuenta (incluye cuentas sin transferencias salientes)
SELECT t.id_transferencia, c.numero_cuenta
FROM transferencia t
RIGHT JOIN cuenta_bancaria c ON t.cuenta_origen = c.numero_cuenta;

-- Bitácora de cada préstamo (incluye préstamos sin registro)
SELECT b.id_bitacora, p.id_prestamo
FROM bitacora_operaciones b
RIGHT JOIN prestamo p ON b.id_entidad = p.id_prestamo;

-- Transferencias por usuario creador (incluye todos los usuarios)
SELECT t.id_transferencia, u.nombre_completo
FROM transferencia t
RIGHT JOIN usuario_sistema u ON t.id_usuario_creador = u.id_usuario;
```

---

*Creado por Emmanuel Berrio Jimenez — ET0062 Bases de Datos II*
