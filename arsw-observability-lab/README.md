# Lab 06 - Observabilidad de Microservicios con Grafana, Prometheus, Loki y OpenTelemetry

## Descripción

En este laboratorio implementé una solución completa de observabilidad para un microservicio Spring Boot. El objetivo fue comprender cómo monitorear el comportamiento interno de un sistema distribuido a través de métricas, logs y trazas, usando herramientas como Prometheus, Grafana y Loki.

## Arquitectura implementada

La arquitectura que construí conecta los siguientes componentes:

- **Spring Boot + Actuator + Micrometer**: expone métricas en `/actuator/prometheus`
- **Prometheus**: recolecta las métricas cada 5 segundos
- **Loki + Promtail**: centraliza los logs del contenedor
- **Grafana**: visualiza métricas y logs en dashboards y alertas
- **Docker Compose**: orquesta todos los servicios

## Tecnologías utilizadas

- Java 21
- Spring Boot 4.1.0
- Docker y Docker Compose
- Prometheus
- Grafana
- Loki
- Promtail

## Cómo ejecutar el proyecto

### Requisitos previos

- Docker Desktop instalado y corriendo
- Java 21
- Maven 3.8+

### Pasos

1. Clonar el repositorio:
```bash
git clone https://github.com/carljob/Lab06-Observabilidad.git
cd Lab06-Observabilidad/arsw-observability-lab
```

2. Compilar la aplicación:
```bash
cd app/demoobservability-demo
mvn package -DskipTests
cd ../..
```

3. Levantar el entorno:
```bash
docker compose up -d --build
```

4. Verificar que todos los contenedores estén corriendo:
```bash
docker ps
```

5. Acceder a los servicios:
    - Aplicación: http://localhost:8081/actuator/health
    - Prometheus: http://localhost:9090
    - Grafana: http://localhost:3000 (admin/admin)

## Endpoints disponibles

| Método | Endpoint | Descripción |
|--------|----------|-------------|
| POST | /orders | Crear un pedido |
| GET | /orders/{id} | Consultar un pedido |
| GET | /orders/simulate-latency | Simular latencia |
| GET | /orders/simulate-error | Simular error 500 |

## Métricas expuestas

- `orders_total` — total de pedidos creados
- `orders_failed_total` — total de pedidos fallidos
- `http_server_requests_seconds_count` — conteo de solicitudes HTTP
- `jvm_memory_used_bytes` — memoria usada por la JVM
- `process_cpu_usage` — uso de CPU del proceso

---

## Actividad 1 — Diagnóstico de observabilidad

### Incidente 1: Aumento de errores HTTP 500

| Campo | Descripción |
|-------|-------------|
| Incidente observado | Aumento de errores HTTP 500 en el servicio de pedidos |
| Métrica afectada | `orders_failed_total` y `http_server_requests_seconds_count{status="500"}` |
| Logs relacionados | `ERROR OrderController: Error simulado en el servicio de pedidos` |
| Endpoint involucrado | `GET /orders/simulate-error` |
| Posible causa | Excepción no controlada lanzada intencionalmente en el controlador |
| Impacto para el usuario | El usuario recibe una respuesta 500 Internal Server Error |
| Acción correctiva propuesta | Agregar manejo de excepciones específicas y validar entradas antes de procesar |
| Alerta que debería existir | `sum(rate(http_server_requests_seconds_count{status="500"}[1m])) > 0` |

### Incidente 2: Aumento de latencia

| Campo | Descripción |
|-------|-------------|
| Incidente observado | Tiempo de respuesta elevado en el servicio de pedidos |
| Métrica afectada | `http_server_requests_seconds_sum / http_server_requests_seconds_count` |
| Logs relacionados | `WARN OrderController: Simulando latencia artificial de X ms` |
| Endpoint involucrado | `GET /orders/simulate-latency` |
| Posible causa | Procesamiento lento simulado con Thread.sleep entre 500ms y 3000ms |
| Impacto para el usuario | Respuestas lentas que degradan la experiencia del sistema |
| Acción correctiva propuesta | Identificar el cuello de botella, optimizar el procesamiento y agregar timeouts |
| Alerta que debería existir | `latencia promedio > 1 segundo` |

### Incidente 3: Creación de pedidos

| Campo | Descripción |
|-------|-------------|
| Incidente observado | Aumento de solicitudes POST al endpoint de pedidos |
| Métrica afectada | `orders_total`, `http_server_requests_seconds_count{uri="/orders"}` |
| Logs relacionados | `INFO OrderController: Pedido creado correctamente. orderId=ORD-xxx` |
| Endpoint involucrado | `POST /orders` |
| Posible causa | Aumento normal de actividad de negocio |
| Impacto para el usuario | Sin impacto negativo, operación exitosa |
| Acción correctiva propuesta | Monitorear el crecimiento para escalar el servicio si es necesario |
| Alerta que debería existir | Alerta si `orders_total` deja de incrementar durante un período de alta demanda |

---

## Actividad 2 — Diseño de dashboard

El dashboard que construí se llama **ARSW - Observabilidad de Microservicios** y contiene 8 paneles:

| Panel | Fuente de datos | Consulta | Qué permite analizar | Decisión técnica que soporta |
|-------|----------------|----------|----------------------|------------------------------|
| Estado del Servicio | Prometheus | `up{job="observability-demo"}` | Si el servicio está disponible para Prometheus | Decidir si hay que reiniciar el servicio |
| Solicitudes HTTP por Endpoint | Prometheus | `sum by (uri, method, status) (rate(http_server_requests_seconds_count[1m]))` | Volumen de tráfico por endpoint | Identificar endpoints con mayor carga |
| Latencia Promedio | Prometheus | `sum(rate(..._sum[1m])) / sum(rate(..._count[1m]))` | Tiempo promedio de respuesta | Detectar degradación del rendimiento |
| Errores HTTP 500 | Prometheus | `sum(rate(http_server_requests_seconds_count{status="500"}[1m]))` | Tasa de errores internos | Escalar o corregir el servicio ante fallos |
| Pedidos Creados | Prometheus | `orders_total` | Total de pedidos exitosos | Medir actividad de negocio |
| Pedidos Fallidos | Prometheus | `orders_failed_total` | Total de errores en pedidos | Detectar problemas en el flujo de negocio |
| Memoria JVM | Prometheus | `sum(jvm_memory_used_bytes{application="observability-demo"})` | Consumo de memoria de la app | Detectar memory leaks o necesidad de más recursos |
| Uso de CPU | Prometheus | `process_cpu_usage{application="observability-demo"}` | Carga del proceso Java | Decidir si escalar horizontalmente |

---

## Actividad 3 — Observabilidad en arquitectura de e-commerce

Para una plataforma de comercio electrónico con los servicios `order-service`, `payment-service`, `inventory-service`, `notification-service` y `shipping-service`, propongo la siguiente estrategia de observabilidad:

### Métricas importantes por servicio

- **order-service**: `orders_created_total`, `orders_failed_total`, latencia de creación
- **payment-service**: `payments_processed_total`, `payments_failed_total`, tiempo de procesamiento
- **inventory-service**: `stock_queries_total`, `stock_updates_total`, disponibilidad de productos
- **notification-service**: `notifications_sent_total`, `notifications_failed_total`
- **shipping-service**: `shipments_created_total`, latencia de despacho

### Logs que deberían registrarse

- Cada transacción iniciada y completada con su ID
- Errores con stack trace completo
- Timeouts entre servicios
- Cambios de estado en pedidos y pagos

### Trazas necesarias

El flujo completo de una compra debería trazarse así:

Cliente → order-service → payment-service → inventory-service → shipping-service → notification-service

Esto permite identificar en qué servicio ocurrió la mayor latencia o el error original.

### Alertas recomendadas

| Alerta | Condición | Severidad |
|--------|-----------|-----------|
| Servicio caído | `up == 0` | Crítica |
| Errores en pagos | `payments_failed_total > 0` | Crítica |
| Latencia elevada | `latencia > 2s` | Alta |
| Stock bajo | `stock_available < umbral` | Media |
| Notificaciones fallidas | `notifications_failed_total > 5` | Media |

### Respuestas a preguntas clave

**¿Qué señales permitirían detectar un problema en pagos?**
Incremento en `payments_failed_total`, errores HTTP 500 en `payment-service`, logs con mensajes de timeout o rechazo, y trazas que muestren que el flujo se detiene en ese servicio.

**¿Qué señales permitirían detectar lentitud en inventario?**
Aumento en la latencia promedio de `inventory-service`, logs con tiempos de consulta elevados, y trazas que muestren que el mayor tiempo ocurre en ese componente.

**¿Qué señales permitirían saber si las notificaciones fallan?**
Incremento en `notifications_failed_total`, logs con errores de envío, y ausencia de logs de confirmación de entrega.

**¿Qué métrica usaría para medir disponibilidad?**
`up{job="<nombre-servicio>"}` — un valor de 1 indica disponible, 0 indica caído.

**¿Qué métrica usaría para medir experiencia del usuario?**
La latencia promedio del endpoint principal (`http_server_requests_seconds_sum / http_server_requests_seconds_count`) combinada con la tasa de errores HTTP 500.

---

## Alertas configuradas en Grafana

| Alerta | Consulta | Interpretación |
|--------|----------|----------------|
| Servicio Caído | `up{job="observability-demo"} == 0` | Prometheus no puede consultar la aplicación |
| Errores HTTP 500 | `sum(rate(http_server_requests_seconds_count{status="500"}[1m])) > 0` | La aplicación está generando errores internos |
| Latencia Elevada | `sum(rate(..._sum[1m])) / sum(rate(..._count[1m])) > 1` | La latencia promedio supera un segundo |