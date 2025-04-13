# Consultas NoSQL para Sistema Bancario en MongoDB

A continuación se presentan 5 enunciados de consultas basados en las colecciones NoSQL del sistema bancario, junto con las soluciones utilizando operaciones avanzadas de MongoDB.

## 1. Análisis de Saldos por Tipo de Cuenta

**Enunciado:** El departamento financiero necesita un informe que muestre el saldo total, promedio, máximo y mínimo por cada tipo de cuenta (ahorro y corriente) para evaluar la distribución de fondos en el banco.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  { $unwind: "$cuentas" },
  {
    $group: {
      _id: "$cuentas.tipo_cuenta",
      total_saldo: { $sum: "$cuentas.saldo" },
      promedio_saldo: { $avg: "$cuentas.saldo" },
      maximo_saldo: { $max: "$cuentas.saldo" },
      minimo_saldo: { $min: "$cuentas.saldo" },
      cantidad_cuentas: { $sum: 1 }
    }
  },
  {
    $project: {
      _id: 0,
      tipo_cuenta: "$_id",
      total_saldo: { $round: ["$total_saldo", 2] },
      promedio_saldo: { $round: ["$promedio_saldo", 2] },
      maximo_saldo: 1,
      minimo_saldo: 1,
      cantidad_cuentas: 1
    }
  }
]);
```

## 2. Patrones de Transacciones por Cliente

**Enunciado:** El equipo de análisis de comportamiento necesita identificar los patrones de transacciones de cada cliente, mostrando la cantidad y el monto total de transacciones por tipo (depósito, retiro, transferencia) para cada cliente.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  {
    $lookup: {
      from: "clientes",
      localField: "cliente_ref",
      foreignField: "_id",
      as: "cliente_info"
    }
  },
  { $unwind: "$cliente_info" },
  {
    $group: {
      _id: {
        cliente_id: "$cliente_info._id",
        nombre: "$cliente_info.nombre",
        cedula: "$cliente_info.cedula",
        tipo_transaccion: "$tipo_transaccion"
      },
      cantidad_transacciones: { $sum: 1 },
      monto_total: { $sum: "$monto" },
      promedio_monto: { $avg: "$monto" }
    }
  },
  {
    $group: {
      _id: {
        cliente_id: "$_id.cliente_id",
        nombre: "$_id.nombre",
        cedula: "$_id.cedula"
      },
      transacciones: {
        $push: {
          tipo: "$_id.tipo_transaccion",
          cantidad: "$cantidad_transacciones",
          monto_total: { $round: ["$monto_total", 2] },
          promedio_monto: { $round: ["$promedio_monto", 2] }
        }
      },
      total_transacciones: { $sum: "$cantidad_transacciones" },
      monto_total_general: { $sum: "$monto_total" }
    }
  },
  {
    $project: {
      _id: 0,
      cliente_id: "$_id.cliente_id",
      nombre: "$_id.nombre",
      cedula: "$_id.cedula",
      transacciones: 1,
      total_transacciones: 1,
      monto_total_general: { $round: ["$monto_total_general", 2] }
    }
  },
  { $sort: { "monto_total_general": -1 } }
]);
```

## 3. Clientes con Múltiples Tarjetas de Crédito

**Enunciado:** El departamento de riesgo crediticio necesita identificar a los clientes que poseen más de una tarjeta de crédito, mostrando sus datos personales, cantidad de tarjetas y el detalle de cada una.

**Consulta MongoDB:**
```javascript
db.clientes.aggregate([
  { $unwind: "$cuentas" },
  { $unwind: "$cuentas.tarjetas" },
  { $match: { "cuentas.tarjetas.tipo_tarjeta": "credito" } },
  {
    $group: {
      _id: "$_id",
      nombre: { $first: "$nombre" },
      cedula: { $first: "$cedula" },
      correo: { $first: "$correo" },
      cantidad_tarjetas: { $sum: 1 },
      tarjetas: { $push: "$cuentas.tarjetas" }
    }
  },
  { $match: { cantidad_tarjetas: { $gte: 1 } } },
  {
    $project: {
      _id: 0,
      cliente_id: "$_id",
      nombre: 1,
      cedula: 1,
      correo: 1,
      cantidad_tarjetas: 1,
      tarjetas: {
        $map: {
          input: "$tarjetas",
          as: "tarjeta",
          in: {
            numero_tarjeta: "$$tarjeta.numero_tarjeta",
            tipo_tarjeta: "$$tarjeta.tipo_tarjeta",
            fecha_emision: "$$tarjeta.fecha_emision",
            fecha_expiracion: "$$tarjeta.fecha_expiracion",
            cuenta_asociada: "$$tarjeta.cuenta_asociada"
          }
        }
      }
    }
  }
]);
```

## 4. Análisis de Medios de Pago más Utilizados

**Enunciado:** El departamento de marketing necesita conocer cuáles son los medios de pago más utilizados para depósitos, agrupados por mes, para orientar sus campañas promocionales.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  {
    $match: {
      tipo_transaccion: "deposito"
    }
  },
  {
    $addFields: {
      mes: { $month: { $toDate: "$fecha" } },
      año: { $year: { $toDate: "$fecha" } }
    }
  },
  {
    $group: {
      _id: {
        año: "$año",
        mes: "$mes",
        medio_pago: "$detalles_deposito.medio_pago"
      },
      cantidad: { $sum: 1 },
      monto_total: { $sum: "$monto" }
    }
  },
  {
    $sort: {
      "_id.año": 1,
      "_id.mes": 1,
      cantidad: -1
    }
  },
  {
    $group: {
      _id: {
        año: "$_id.año",
        mes: "$_id.mes"
      },
      medios_pago: {
        $push: {
          medio: "$_id.medio_pago",
          cantidad: "$cantidad",
          monto_total: { $round: ["$monto_total", 2] }
        }
      },
      total_depositos: { $sum: "$cantidad" }
    }
  },
  {
    $project: {
      _id: 0,
      año: "$_id.año",
      mes: "$_id.mes",
      total_depositos: 1,
      medios_pago: 1,
      medio_mas_utilizado: { $arrayElemAt: ["$medios_pago", 0] }
    }
  }
]);
```

## 5. Detección de Cuentas con Transacciones Sospechosas

**Enunciado:** El departamento de seguridad necesita identificar cuentas con patrones de transacciones sospechosas, definidas como aquellas que tienen más de 3 retiros en un mismo día con un monto total superior a 1,000,000 COP.

**Consulta MongoDB:**
```javascript
db.transacciones.aggregate([
  {
    $match: {
      tipo_transaccion: "retiro",
      fecha: {
        $exists: true,
        $ne: null
      }
    }
  },
  {
    $addFields: {
      fecha_iso: { $toDate: "$fecha" },
      dia: { $dateToString: { format: "%Y-%m-%d", date: { $toDate: "$fecha" } } }
    }
  },
  {
    $group: {
      _id: {
        num_cuenta: "$num_cuenta",
        dia: "$dia"
      },
      cantidad_retiros: { $sum: 1 },
      monto_total: { $sum: "$monto" },
      transacciones: { $push: "$$ROOT" }
    }
  },
  {
    $match: {
      $and: [
        { cantidad_retiros: { $gt: 3 } },
        { monto_total: { $gte: 1000000 } }
      ]
    }
  },
  {
    $lookup: {
      from: "clientes",
      localField: "_id.num_cuenta",
      foreignField: "cuentas.num_cuenta",
      as: "cliente"
    }
  },
  { $unwind: "$cliente" },
  {
    $project: {
      _id: 0,
      numero_cuenta: "$_id.num_cuenta",
      fecha: "$_id.dia",
      cantidad_retiros: 1,
      monto_total: 1,
      cliente: {
        nombre: "$cliente.nombre",
        cedula: "$cliente.cedula",
        correo: "$cliente.correo"
      },
      transacciones: {
        $map: {
          input: "$transacciones",
          as: "t",
          in: {
            fecha: "$$t.fecha",
            monto: "$$t.monto",
            descripcion: "$$t.descripcion",
            canal: "$$t.detalles_retiro.canal"
          }
        }
      }
    }
  },
  { $sort: { monto_total: -1 } }
]); // No hay transacciones sospechosas
```
