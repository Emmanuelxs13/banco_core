#  Banco Core – Sistema de Gestión Bancaria (PostgreSQL)

## Grupo 800 | ET0062 - BASES DE DATOS II | Emmanuel Berrio Jimenez

---

## 1. Descripción del Proyecto

Este proyecto consiste en el diseño e implementación de una base de datos relacional en PostgreSQL para soportar el core transaccional de una entidad bancaria.

El sistema gestiona:

- Clientes Persona Natural
- Clientes Empresa
- Usuarios del sistema con roles
- Cuentas bancarias
- Préstamos con flujo de aprobación
- Transferencias con control de autorización
- Bitácora de operaciones
- Catálogos centralizados

El diseño fue realizado bajo criterios de arquitectura profesional de bases de datos:
- Normalización (3FN)
- Integridad referencial estricta
- Indexación estratégica
- Separación de responsabilidades
- Preparación para escalabilidad

---

## 🧠 2. Interpretación del Enunciado

El enunciado describe un sistema bancario con:

- Múltiples tipos de clientes
- Múltiples roles con permisos distintos
- Productos financieros
- Operaciones con flujos de aprobación
- Registro obligatorio de auditoría

A partir de la narrativa se identificaron las siguientes entidades principales:

1. Cliente Persona Natural
2. Cliente Empresa
3. Usuario del Sistema
4. Cuenta Bancaria
5. Préstamo
6. Transferencia
7. Producto Bancario (Catálogo)
8. Estados (Catálogo)
9. Roles del Sistema (Catálogo)
10. Bitácora de Operaciones

Se transformó la narrativa de negocio en un modelo relacional estructurado.

---

## 🏗 3. Decisiones de Arquitectura

### 🔹 Separación de Clientes

Se decidió separar:
- cliente_persona
- cliente_empresa

Porque:
- Tienen atributos distintos.
- Permite escalabilidad futura.
- Evita columnas NULL innecesarias.

---

### 🔹 Tabla usuario_sistema Centralizada

Todos los usuarios (internos y externos) comparten una estructura común.

Se creó:
usuario_sistema

Con:
- id_rol (FK)
- id_estado (FK)
- tipo_relacion (PERSONA / EMPRESA)

Esto permite:
- Control de acceso por rol.
- Desactivar usuarios sin eliminarlos.
- Auditoría completa.

---

### 🔹 Catálogos Separados

Se crearon:
- rol_sistema
- estado_general
- producto_bancario

Ventajas:
- Evita hardcodeo.
- Permite crecimiento.
- Mejora integridad.

---

### 🔹 Flujos de Aprobación

Se modelaron mediante:
- Campos de estado (FK)
- id_usuario_creador
- id_usuario_aprobador
- fechas de aprobación

Esto permite:
- Trazabilidad completa.
- Auditoría.
- Control de responsabilidades.

---

### 🔹 Bitácora de Operaciones

Tabla:
bitacora_operaciones

Registra:
- Entidad afectada
- ID entidad
- Acción
- Usuario responsable
- Fecha
- Detalle

Permite:
- Cumplimiento normativo
- Auditoría financiera
- Historial completo

---

## ⚙️ 4. Integridad y Seguridad Implementada

Se aplicaron:

- PRIMARY KEYS
- FOREIGN KEYS
- UNIQUE constraints
- CHECK constraints
- Validación de mayoría de edad
- Validación de correo electrónico
- Restricción de saldo >= 0
- Restricción de montos positivos
- Indexación en campos críticos

---

## 📈 5. Índices Estratégicos

Se indexaron:

- Identificaciones
- NIT
- Estados
- Roles
- Relaciones FK
- Cuenta origen/destino
- Cliente solicitante
- Bitácora por entidad

Objetivo:
Optimizar búsquedas frecuentes y joins.

---

# 6. Consultas Implementadas

---

## 6.1 Subconsultas (3)

### Clientes con préstamos aprobados

```sql
SELECT nombre_completo
FROM cliente_persona
WHERE id_persona IN (
    SELECT id_cliente_solicitante
    FROM prestamo
    WHERE monto_aprobado IS NOT NULL
);
 ```

### Cuentas con saldo mayor al promedio
```sql
SELECT numero_cuenta, saldo_actual
FROM cuenta_bancaria
WHERE saldo_actual > (
    SELECT AVG(saldo_actual) FROM cuenta_bancaria
);
```

### Transferencias mayores al máximo del mes anterior
```sql
SELECT *
FROM transferencia
WHERE monto > (
    SELECT MAX(monto)
    FROM transferencia
    WHERE fecha_creacion >= date_trunc('month', CURRENT_DATE - interval '1 month')
);
```

### 6.2 Consultas con Filtro de Igualación (3)
```sql
SELECT * FROM cuenta_bancaria
WHERE moneda = 'COP';

SELECT * FROM prestamo
WHERE plazo_meses = 12;

SELECT * FROM usuario_sistema
WHERE id_rol = 2;
```

### 6.3 INNER JOIN (3)
```sql
SELECT t.id_transferencia, u.nombre_completo
FROM transferencia t
JOIN usuario_sistema u
ON t.id_usuario_creador = u.id_usuario;

SELECT p.id_prestamo, c.numero_cuenta
FROM prestamo p
JOIN cuenta_bancaria c
ON p.cuenta_destino_desembolso = c.numero_cuenta;

SELECT u.nombre_completo, r.nombre_rol
FROM usuario_sistema u
JOIN rol_sistema r
ON u.id_rol = r.id_rol;
```

### 6.4 LEFT JOIN (3)
```sql
SELECT c.numero_cuenta, t.id_transferencia
FROM cuenta_bancaria c
LEFT JOIN transferencia t
ON c.numero_cuenta = t.cuenta_origen;

SELECT p.id_prestamo, b.detalle
FROM prestamo p
LEFT JOIN bitacora_operaciones b
ON p.id_prestamo = b.id_entidad;

SELECT u.nombre_completo, t.id_transferencia
FROM usuario_sistema u
LEFT JOIN transferencia t
ON u.id_usuario = t.id_usuario_aprobador;
```

### 6.5 RIGHT JOIN (3)
```sql
SELECT t.id_transferencia, c.numero_cuenta
FROM transferencia t
RIGHT JOIN cuenta_bancaria c
ON t.cuenta_origen = c.numero_cuenta;

SELECT b.id_bitacora, p.id_prestamo
FROM bitacora_operaciones b
RIGHT JOIN prestamo p
ON b.id_entidad = p.id_prestamo;

SELECT t.id_transferencia, u.nombre_completo
FROM transferencia t
RIGHT JOIN usuario_sistema u
ON t.id_usuario_creador = u.id_usuario;
```
### Creado por:
Emmanuel Berrio Jimenez