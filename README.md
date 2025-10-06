#  EventFlow – Tarea 2 NoSQL 2025

##  Descripción General

**EventFlow** es una plataforma de gestión y venta de entradas para eventos, diseñada bajo una **arquitectura de microservicios** con un enfoque **políglota NoSQL**.  
El sistema está compuesto por tres microservicios principales:  
- **Servicio de Usuarios**  
- **Servicio de Eventos**  
- **Servicio de Reservas y Pagos**  



##  Arquitectura General del Sistema

La arquitectura se basa en **microservicios independientes**, comunicados mediante REST API y desplegados en contenedores Docker.

###  Componentes
| Microservicio | Función Principal | Base de Datos | Tipo de NoSQL |
|----------------|------------------|----------------|----------------|
| Usuarios | Maneja perfiles e historial de compras | MongoDB | Documental |
| Eventos | Administra información y aforo de eventos | MongoDB | Documental |
| Reservas y Pagos | Orquesta las transacciones y procesamientos de compra | Redis | Clave-Valor (en memoria) |



##  Justificación del Diseño

### 1. Enfoque General y Requerimientos de Consistencia

**Objetivo:** cumplir los requerimientos CAP definidos en la consigna.

| Tipo de Operación | Prioridad | Enfoque Adoptado |
|--------------------|------------|------------------|
| Lecturas (Usuarios y Eventos) | Alta disponibilidad y tolerancia a particiones (AP) | **MongoDB** con consistencia eventual |
| Escrituras (Reservas y Pagos) | Alta consistencia, sin doble venta (CP) | **Redis** con operaciones atómicas y transacciones ACID |

**Razonamiento:**  
- Las consultas de usuarios y eventos requieren rapidez y escalabilidad, por lo que se prioriza **disponibilidad** sobre consistencia estricta.  
- Las operaciones de reserva y pago requieren **consistencia fuerte** para evitar duplicaciones; Redis permite operaciones atómicas (INCR/DECR) que garantizan integridad en tiempo real.



### 2. Modelado de Datos (NoSQL)

####  Servicio de Usuarios (MongoDB)

{
  "_id": ObjectId,
  "tipo_documento": "DNI",
  "nro_documento": "12345678",
  "nombre": "Juan",
  "apellido": "Pérez",
  "email": "juan@example.com",
  "historial_compras": [
    { "evento_id": "evt123", "fecha_compra": "2025-10-01", "entradas": 2 }
  ]
}
Datos embebidos: se incluyen las compras recientes para lecturas rápidas.

Referencias: los IDs de reservas se almacenan para mantener escalabilidad y evitar duplicación.
---
Servicio de Eventos (MongoDB)
{
  "_id": ObjectId,
  "nombre": "Concierto Rock",
  "fecha": "2025-12-01T21:00:00Z",
  "lugar": "Estadio Centenario",
  "aforo_total": 1000,
  "entradas_disponibles": 850,
  "reservas_activas": ["res1", "res2"]
}


Modelo mixto: datos estáticos embebidos (nombre, fecha) + campos dinámicos actualizables (entradas disponibles).
Servicio de Reservas y Pagos (Redis)

Estructura tipo key-value:

evento:evt123:entradas → 850
reserva:usr001:evt123 → "pendiente"


Redis gestiona el inventario en tiempo real mediante contadores atómicos y bloqueos ligeros para evitar sobreventa.


###3. Patrón SAGA (Orquestación de Transacciones Distribuidas)

El Servicio de Reservas y Pagos actúa como orquestador central, coordinando los pasos de la transacción.

✅ Flujo de Transacción Exitosa

Validar usuario (GET /api/usuarios/{id})

Verificar disponibilidad de entradas (GET /api/eventos/{id})

Decrementar contador en Redis (DECR evento:evt123:entradas)

Procesar pago (simulación)

Confirmar reserva (POST /api/reservas)

Actualizar historial del usuario (MongoDB)

Responder éxito al cliente

🔁 Transacciones de Compensación (en caso de fallo)

Si falla el pago → se ejecuta INCR evento:evt123:entradas (libera entradas).

Si falla la actualización del historial → se revierte pago y se libera la reserva.

###4. Patrón Chain of Responsibility (Lógica Interna)

Dentro del servicio de Reservas y Pagos, las solicitudes se procesan mediante una cadena de manejadores:

ValidadorDeDatos → ValidadorDeInventario → ProcesadorDePago → ActualizadorDeHistorial


Cada manejador implementa una interfaz común (Handler) y pasa el control al siguiente si la validación tiene éxito.
Esto permite una lógica modular, extensible y desacoplada.

### 5 Tecnologías de Despliegue

Docker Compose:

mongodb

redis

eventflow-app (FastAPI o Node.js)

Red interna: comunicación entre servicios.

Volúmenes persistentes: para datos de MongoDB y Redis.

Ejemplo de ejecución:
docker-compose up -d
curl -X POST http://localhost:8000/api/usuarios \
     -H "Content-Type: application/json" \
     -d '{"nombre":"Juan","apellido":"Pérez","email":"juan@example.com"}'
