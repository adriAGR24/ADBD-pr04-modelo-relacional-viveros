# ADBD-pr04-modelo-relacional-viveros

```sql
-- Creación de la base de datos
create database viveros;

-- Creación de tablas
create table if not exists viveros (
  id_vivero SERIAL primary key,
  nombre_vivero VARCHAR(100) not null,
  latitud DECIMAL(9,6) not null check(latitud between -90 and 90),
  longitud DECIMAL(9,6) not null check(longitud between -180 and 180),
  constraint UQ_viveros_coordenadas unique(latitud, longitud)
);

create table if not exists telefonos_viveros (
  id_vivero INT not null references viveros(id_vivero) on delete cascade,
  telefono VARCHAR(15) not null check (telefono ~ '^[+]?([0-9]{1,3})?[0-9]{6,12}$'),
  constraint pk_telefonos_viveros primary key (id_vivero, telefono)
);

create table if not exists zonas (
  id_zona SERIAL primary key,
  nombre_zona VARCHAR(50),
  latitud DECIMAL(9,6) not null check(latitud between -90 and 90),
  longitud DECIMAL(9,6) not null check(longitud between -180 and 180),
  superficie DECIMAL(10,2) not null check(superficie > 0),
  id_vivero INT not null references viveros(id_vivero) on delete cascade,
  constraint UQ_zonas_coordenadas unique(latitud, longitud),
  constraint UQ_nombre_zonas_viveros unique(nombre_zona, id_vivero);
);

create type tipo_producto as enum ('Planta', 'Jardinería', 'Decoración');

create table if not exists productos (
  id_producto SERIAL primary key,
  nombre_producto VARCHAR(100) not null unique,
  precio DECIMAL(10,2) not null check(precio >= 0),
  tipo tipo_producto not null
);

create table if not exists stock (
  id_producto INT not null references productos(id_producto) on delete cascade,
  id_zona INT not null references zonas(id_zona) on delete cascade,
  unidades INT not null check(unidades >= 0) default 0,
  constraint PK_stock primary key(id_producto, id_zona)
);

create table if not exists cliente_tajinaste_plus (
  dni_cliente VARCHAR(9) primary key check(dni_cliente ~ '^[0-9]{8}[A-Z]$'),
  nombre_cliente VARCHAR(100),
  apellidos_cliente VARCHAR(100)
);

create type cargo_empleado as enum ('Jardinero', 'Vendedor', 'Encargado', 'Administrativo');

create table if not exists empleados (
  dni_empleado VARCHAR(9) primary key check(dni_empleado ~ '^[0-9]{8}[A-Z]$'),
  nombre_empleado VARCHAR(100),
  apellidos_empleado VARCHAR(100),
  salario_mensual DECIMAL(8,2) not null check(salario_mensual >= 0),
  cargo cargo_empleado not null
);

create table if not exists tareas (
  id_tarea SERIAL primary key,
  nombre_tarea VARCHAR(100) not null unique,
  descripcion TEXT,
);

create table if not exists pedidos (
  id_pedido SERIAL primary key,
  dni_cliente VARCHAR(9) not null references cliente_tajinaste_plus(dni_cliente) on delete restrict,
  dni_empleado VARCHAR(9) references empleados(dni_empleado) on delete set null,
  fecha DATE not null check(fecha <= CURRENT_DATE)
);

create table if not exists articulos_pedidos (
  id_pedido INT not null references pedidos(id_pedido) on delete cascade,
  id_producto INT not null references productos(id_producto) on delete restrict,
  unidades INT not null check(unidades > 0),
  constraint pk_articulos_pedidos primary key (id_pedido, id_producto)
);

create table if not exists historial_trabajo (
  dni_empleado VARCHAR(9) not null references empleados(dni_empleado) on delete cascade,
  id_zona INT not null references zonas(id_zona) on delete cascade,
  id_tarea INT not null references tareas(id_tarea) on delete restrict,
  fecha_inicio TIMESTAMP not null,
  fecha_final TIMESTAMP not null,
  constraint PK_historial_trabajo primary key (dni_empleado, id_zona, id_tarea, fecha_inicio),
  constraint CK_fecha_trabajo check (fecha_final >= fecha_inicio)
);

-- Atributo calculado de cliente tajinaste plus: bonificación
create or replace view volumen_pedidos_mensual as
select 
  c.dni_cliente,
  DATE_TRUNC('month', p.fecha) as mes,
  SUM(a.unidades * pr.precio) as total_mensual
from cliente_tajinaste_plus c
join pedidos p on c.dni_cliente = p.dni_cliente
join articulos_pedidos a on p.id_pedido = a.id_pedido
join productos pr on a.id_producto = pr.id_producto
group by c.dni_cliente, DATE_TRUNC('month', p.fecha);

create or replace view bonificacion_clientes as
select
  v.dni_cliente,
  v.mes,
  case
    when v.total_mensual < 100 then 0
    when v.total_mensual < 500 then 0.02
    when v.total_mensual < 1000 then 0.05
    else 0.10
  end as porcentaje_bonificacion
from volumen_pedidos_mensual v;

-- Atributo calculado de zona: productividad
create or replace view productividad_zonas as
select
  z.id_zona,
  z.nombre_zona,
  DATE_TRUNC('month', h.fecha_inicio) as mes,
  COUNT(distinct h.id_tarea) as tareas_realizadas,
  SUM(extract(epoch from (h.fecha_final - h.fecha_inicio)) / 3600) as horas_trabajadas,
  ROUND(
    case
      when SUM(extract(epoch from (h.fecha_final - h.fecha_inicio)) / 3600) > 0
      then COUNT(distinct h.id_tarea) /
           SUM(extract(epoch from (h.fecha_final - h.fecha_inicio)) / 3600)
      else 0
    end,
    2) as productividad
from historial_trabajo h
join zonas z on z.id_zona = h.id_zona
group by z.id_zona, z.nombre_zona, DATE_TRUNC('month', h.fecha_inicio)
order by z.id_zona, mes;

-- Atributo calculado de empleado: productividad
create or replace view productividad_empleados as
select
  e.dni_empleado,
  e.nombre_empleado,
  e.apellidos_empleado,
  DATE_TRUNC('month', h.fecha_inicio) as mes,
  COUNT(distinct h.id_tarea) as tareas_realizadas,
  SUM(extract(epoch from (h.fecha_final - h.fecha_inicio)) / 3600) as horas_trabajadas,
  ROUND(
    COUNT(distinct h.id_tarea) /
    NULLIF(SUM(extract(epoch from (h.fecha_final - h.fecha_inicio)) / 3600), 0),
    2) as productividad
from historial_trabajo h
join empleados e on e.dni_empleado = h.dni_empleado
group by e.dni_empleado, e.nombre_empleado, e.apellidos_empleado, DATE_TRUNC('month', h.fecha_inicio)
order by mes, e.apellidos_empleado;

-- Inserción de filas
insert into viveros (nombre_vivero, latitud, longitud)
values
('Vivero La Esperanza', 28.487500, -16.315000),
('Vivero El Sauzal', 28.466700, -16.416700),
('Vivero Los Olivos', 28.400000, -16.550000),
('Vivero Tajinaste', 28.480000, -16.300000),
('Vivero La Cuesta', 28.458000, -16.290000);

insert into telefonos_viveros (id_vivero, telefono)
values
(1, '+34922123456'),
(1, '+34677888999'),
(2, '922111222'),
(3, '666000111'),
(4, '+34922333444'),
(5, '+34922555666');

insert into zonas (nombre_zona, latitud, longitud, superficie, id_vivero)
values
('Zona Norte', 28.488900, -16.314500, 1200.50, 1),
('Zona Sur', 28.485200, -16.317000, 950.75, 1),
('Zona Central', 28.467000, -16.420000, 800.00, 2),
(NULL, 28.401000, -16.552000, 650.00, 3),
(NULL, 28.482000, -16.299000, 1100.00, 4),
('Zona Experimental', 28.457500, -16.289500, 900.00, 5),
('Zona Central', 28.456000, -16.290500, 450.00, 5);

insert into productos (nombre_producto, precio, tipo)
values
('Rosa roja', 5.50, 'Planta'),
('Cactus del desierto', 7.20, 'Planta'),
('Abono orgánico 5kg', 12.00, 'Jardinería'),
('Maceta de cerámica', 8.75, 'Decoración'),
('Palmera canaria', 25.00, 'Planta'),
('Kit de jardinería básico', 18.50, 'Jardinería'),
('Fuente decorativa pequeña', 45.00, 'Decoración'),
('Tierra universal 10L', 6.50, 'Jardinería');

insert into stock (id_producto, id_zona, unidades)
values
(1, 1, 40),
(2, 1, 30),
(3, 2, 20),
(4, 2, 15),
(5, 3, 10),
(6, 3, 25),
(7, 4, 5),
(8, 5, 100),
(1, 6, 10),
(2, 7, 8),
(3, 7, 0);

insert into cliente_tajinaste_plus (dni_cliente, nombre_cliente, apellidos_cliente)
values
('12345678A', 'Laura', 'González Pérez'),
('23456789B', NULL, 'Ruiz Santana'),
('34567890C', 'Marta', NULL),
('45678901D', 'David', 'López Trujillo'),
('56789012E', NULL, NULL);

insert into empleados (dni_empleado, nombre_empleado, apellidos_empleado, salario_mensual, cargo)
values
('11111111A', 'Pedro', 'Rodríguez García', 1500.00, 'Jardinero'),
('22222222B', NULL, 'Morales Díaz', 1600.00, 'Vendedor'),
('33333333C', 'Luis', NULL, 1800.00, 'Encargado'),
('44444444D', 'Elena', 'Gómez Ortega', 1400.00, 'Jardinero'),
('55555555E', NULL, NULL, 1900.00, 'Administrativo'),
('66666666F', 'Carmen', 'López Ramos', 1550.00, 'Jardinero');

insert into tareas (nombre_tarea, descripcion)
values
('Riego', 'Riego automático o manual de plantas'),
('Poda', NULL),
('Fertilización', 'Aplicación de abonos orgánicos o químicos'),
('Limpieza', 'Mantenimiento general del área de trabajo'),
('Trasplante', 'Cambio de macetas o suelos para crecimiento');

insert into pedidos (dni_cliente, dni_empleado, fecha)
values
('12345678A', '22222222B', '2025-01-15'),
('23456789B', '22222222B', '2025-02-01'),
('34567890C', '33333333C', '2025-02-10'),
('45678901D', null, '2025-03-05'),
('56789012E', '22222222B', '2025-03-10'),
('12345678A', '33333333C', '2025-03-20');

insert into articulos_pedidos (id_pedido, id_producto, unidades)
values
(1, 1, 5),
(1, 3, 1),
(2, 4, 2),
(2, 5, 1),
(3, 2, 3),
(3, 8, 2),
(4, 7, 1),
(5, 6, 1),
(6, 1, 10),
(6, 5, 2);

insert into historial_trabajo (dni_empleado, id_zona, id_tarea, fecha_inicio, fecha_final)
values
('11111111A', 1, 1, '2025-01-10 08:00:00', '2025-01-10 12:00:00'),
('11111111A', 1, 2, '2025-01-11 08:00:00', '2025-01-11 10:30:00'),
('44444444D', 2, 3, '2025-01-12 07:30:00', '2025-01-12 11:30:00'),
('44444444D', 3, 4, '2025-01-13 09:00:00', '2025-01-13 13:00:00'),
('66666666F', 4, 1, '2025-01-14 08:00:00', '2025-01-14 14:00:00'),
('66666666F', 4, 2, '2025-01-15 08:30:00', '2025-01-15 12:00:00'),
('11111111A', 5, 5, '2025-02-01 09:00:00', '2025-02-01 11:30:00'),
('33333333C', 6, 4, '2025-02-05 08:00:00', '2025-02-05 15:00:00'),
('33333333C', 7, 3, '2025-02-10 07:00:00', '2025-02-10 12:00:00'),
('44444444D', 7, 5, '2025-02-11 09:30:00', '2025-02-11 11:00:00');

-- Visualización de las tablas
select * from viveros;
select * from telefonos_viveros;
select * from zonas;
select * from productos;
select * from stock;
select * from cliente_tajinaste_plus;
select * from empleados;
select * from tareas;
select * from pedidos;
select * from articulos_pedidos;
select * from historial_trabajo;
select * from volumen_pedidos_mensual;
select * from bonificacion_clientes;
select * from productividad_zonas;
select * from productividad_empleados;

-- Eliminación de filas

```

### Eliminación de un vivero

- **Configuración:**
  - En `telefonos_viveros`, la clave foránea `id_vivero` tiene `ON DELETE CASCADE`.
  - En `zonas`, la clave foránea `id_vivero` tiene `ON DELETE CASCADE`.
  - En `stock`, la clave foránea `id_zona` tiene `ON DELETE CASCADE`.
  - En `historial_trabajo`, la clave foránea `id_zona` tiene `ON DELETE CASCADE`.

- **Operación:**

```sql
delete from viveros
where id_vivero = 1;
```

- **Consecuencias:**
  - Todos los teléfonos pertenecientes al vivero con `id_vivero = 1` se eliminan. En este caso `+34922123456` y `+34677888999`.
  - Todas las zonas pertenecientes al vivero con `id_vivero = 1` se eliminan. En este caso la zona con `id_zona = 1` y `id_zona = 2`.
  - Todos los registros de `stock` relacionados con la zona 1 y 2 se eliminan. En este caso los registros del 1 al 4.
  - Todos los registros de `historial_trabajo` relacionados con la zona 1 y 2 se eliminan. En este caso los registros del 1 al 3.

### Eliminación de un producto

- **Configuración:**
  - En `stock`, la clave foránea `id_producto` tiene `ON DELETE CASCADE`.
  - En `articulos_pedidos`, la clave foránea `id_producto` tiene `ON DELETE RESTRICT`.

- **Operación:**

```sql
delete from productos
where id_producto = 6;
```

- **Consecuencias:**
  - Todos los registros de `articulos_pedidos` relacionados con el producto con `id_producto = 6` intentarán ser eliminados. En este caso el registro 8. Sin embargo, el sistema no lo permitirá y, por tanto, negará la eliminación del producto 6.
  - Todos los registros de `stock` relacionados con el producto con `id_producto = 6` se eliminarían. En este caso el registro 6. Sin embargo, al no permitirse la eliminación del producto 6, estos registros se mantendrán.

### Eliminación de un cliente

- **Configuración:**
  - En `pedidos`, la clave foránea `dni_cliente` tiene `ON DELETE RESTRICT`.

- **Operación:**

```sql
delete from cliente_tajinaste_plus
where dni_cliente = '12345678A';
```

- **Consecuencias:**
  - Todos los pedidos realizados por el cliente con `dni_cliente = 12345678A` intentarán ser eliminados. En este caso los pedidos con `id_pedido = 1` y `id_pedido = 6`. Sin embargo, el sistema no lo permitirá y, por tanto, negará la eliminación del cliente con `dni_cliente = 12345678A`.

### Eliminación de un pedido

- **Configuración:**
  - En `articulos_pedidos`, la clave foránea `id_pedido` tiene `ON DELETE CASCADE`.

- **Operación:**

```sql
delete from pedidos
where id_pedido = 2;
```

- **Consecuencias:**
  - Todos los registros de `articulos_pedidos` relacionados con el pedido con `id_pedido = 2` se eliminan. En este caso los registros 3 y 4.

### Eliminación de un empleado

- **Configuración:**
  - En `pedidos`, la clave foránea `dni_empleado` tiene `ON DELETE SET NULL`.
  - En `historial_trabajo`, la clave foránea `dni_empleado` tiene `ON DELETE CASCADE`.

- **Operación:**

```sql
delete from empleados
where dni_empleado = '33333333C';
```

- **Consecuencias:**
  - Todos los pedidos gestionados por el empleado con `dni_empleado = 33333333C` pondrán a `null` la clave foránea `dni_empleado`. En este caso, los pedidos con `id_pedido = 3` y `id_pedido = 6` ya no estarán gestionados por ningún empleado, pero seguirán existiendo.
  - Todos los registros de `historial_trabajo` relacionados con el empleado con `dni_empleado = 33333333C` se eliminan. En este caso los registros 8 y 9.

### Eliminación de una tarea

- **Configuración:**
  - En `historial_trabajo`, la clave foránea `id_tarea` tiene `ON DELETE RESTRICT`.

- **Operación:**

```sql
delete from tareas
where nombre_tarea = 'Trasplante';
```

**Consecuencia:**
  - Todos los registros de `historial_trabajo` relacionados con la tarea con `id_tarea = 5` intentarán ser eliminados. En este caso los registros 7 y 10. Sin embargo, el sistema no lo permitirá y, por tanto, negará la eliminación de la tarea con `nombre_tarea = Trasplante`.