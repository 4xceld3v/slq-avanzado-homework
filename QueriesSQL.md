# Consultas SQL para Base de Datos Bancaria

## Enunciado 1: Clientes con múltiples cuentas y sus saldos totales

**Necesidad:** El banco necesita identificar a los clientes que tienen más de una cuenta, mostrar cuántas cuentas tiene cada uno y el saldo total acumulado en todas sus cuentas, ordenados por saldo total de mayor a menor.

**Consulta SQL:**
```sql
SELECT cl.nombre AS nom_cliente, COUNT(cl.nombre) AS num_cuentas, SUM(cu.saldo) AS saldo_total FROM cliente cl
inner join cuenta cu on cl.id_cliente = cu.id_cliente
GROUP BY cl.nombre
HAVING COUNT(cl.nombre) > 1
ORDER BY SUM(cu.saldo) DESC;
```

## Enunciado 2: Comparativa entre depósitos y retiros por cliente

**Necesidad:** El departamento de análisis financiero necesita comparar los montos totales de depósitos y retiros realizados por cada cliente, para identificar patrones de comportamiento financiero.

**Consulta SQL:**
```sql
SELECT cl.nombre, tr.tipo_transaccion, SUM(tr.monto) AS total FROM cliente cl
inner join cuenta cu on cl.id_cliente = cu.id_cliente
inner join public.transaccion tr on cu.num_cuenta = tr.num_cuenta
WHERE tr.tipo_transaccion = 'deposito' OR tr.tipo_transaccion = 'retiro'
GROUP BY cl.nombre, tr.tipo_transaccion
ORDER BY tr.tipo_transaccion;
```

## Enunciado 3: Cuentas sin tarjetas asociadas

**Necesidad:** El departamento de tarjetas necesita identificar todas las cuentas que no tienen tarjetas asociadas para ofrecer productos específicos a estos clientes.

**Consulta SQL:**
```sql
SELECT c.num_cuenta, c.tipo_cuenta, c.saldo, c.fecha_apertura FROM cuenta c
LEFT JOIN tarjeta t ON c.num_cuenta = t.num_cuenta
WHERE t.num_cuenta IS NULL;
```

## Enunciado 4: Análisis de saldos promedio por tipo de cuenta y comportamiento transaccional

**Necesidad:** La gerencia necesita un análisis comparativo del saldo promedio entre cuentas de ahorro y corriente, pero solo considerando aquellas cuentas que han tenido al menos una transacción en los últimos 30 días.

**Consulta SQL:**
```sql
SELECT cu.num_cuenta, cu.tipo_cuenta, AVG(cu.saldo) AS prom_saldo FROM cuenta cu
INNER JOIN public.transaccion t on cu.num_cuenta = t.num_cuenta
WHERE cu.tipo_cuenta = 'ahorro' OR cu.tipo_cuenta = 'corriente'
AND t.fecha >= CURRENT_DATE - INTERVAL '30 days'
GROUP BY cu.num_cuenta, tipo_cuenta
ORDER BY cu.tipo_cuenta;
```

## Enunciado 5: Clientes con transferencias pero sin retiros en cajeros

**Necesidad:** El equipo de marketing digital necesita identificar a los clientes que utilizan transferencias pero no realizan retiros por cajeros automáticos, para dirigir campañas de banca digital.

**Consulta SQL:**
```sql
SELECT DISTINCT c.id_cliente, c.nombre, c.correo
FROM Cliente c
JOIN Cuenta cu ON c.id_cliente = cu.id_cliente
JOIN Transaccion t ON cu.num_cuenta = t.num_cuenta
WHERE t.tipo_transaccion = 'transferencia'
  AND t.num_cuenta NOT IN (
    SELECT DISTINCT t2.num_cuenta FROM Transaccion t2
    JOIN Retiro r ON t2.id_transaccion = r.id_transaccion
    WHERE t2.tipo_transaccion = 'retiro'
      AND r.canal = 'cajero'
  )
ORDER BY c.nombre;
```
