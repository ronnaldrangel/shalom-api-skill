---
name: shalom-api
description: Integra Shalom Perú (courier) por API REST desde cualquier agente de IA. Use when the user says "rastrear envío shalom", "seguir mi pedido shalom", "guía de shalom", "crear guía shalom pro", "agencia shalom más cercana", "cotizar envío shalom", "integrar shalom", "API shalom" or "shalom api peru" — or asks to add courier shipping, COD delivery tracking, agency lookup with coordinates, webhook delivery updates for WooCommerce/Shopify/n8n to any app. Covers tracking de envíos, catálogo de agencias, cotizaciones, DNI, creación de guías en Shalom Pro y webhooks firmados.
---

# Shalom API — Guía de integración para agentes de IA

Shalom API Perú expone el sistema del courier Shalom (shalom.com.pe) como API REST con respuestas JSON. Esta guía contiene todo lo necesario para integrarlo sin explorar: autenticación, endpoints esenciales, recetas paso a paso y reglas de operación.

- Base URL: `https://api.shalom-api.lat`
- Documentación pública completa: https://shalom-api.lat/docs (última actualización: 2026-09-01)
- Directorio de agencias con direcciones y coordenadas: https://shalom-api.lat/agencias

## Autenticación

Toda petición (salvo `GET /public/agencies`) requiere el header `x-api-key: sk_...`. La key se solicita por WhatsApp al responsable del servicio y tiene una cuota mensual según el plan. Valida siempre la key antes de operar:

```bash
curl "https://api.shalom-api.lat/validate" -H "x-api-key: TU_API_KEY"
# => { "valid": true, "limit": 1000, "currentUsage": 137, "remaining": 863 }
```

Rate limit global: 1000 peticiones por minuto. Al superarlo la API responde `429` con `{ limit, currentUsage, remaining }`.

## Endpoints esenciales

### Autenticación y API keys

#### `GET /validate`

Confirma que la key es válida y devuelve el límite mensual de tu plan y el consumo acumulado del mes. Úsala como health check de tu integración antes de operar.

```bash
curl "https://api.shalom-api.lat/validate" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "valid": true,
  "userId": "b7c9d1e2-...",
  "limit": 1000,
  "currentUsage": 137,
  "remaining": 863
}
```

### Rastrear envíos

#### `POST /track`

Devuelve el estado actual y la línea de tiempo de movimientos de la guía.

```bash
curl -X POST "https://api.shalom-api.lat/track" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "orderNumber": "66479331",
  "orderCode": "3KTH"
}'
```

Request (JSON):
```json
{
  "orderNumber": "66479331",
  "orderCode": "3KTH"
}
```

Respuesta (ejemplo):
```json
{
  "orderNumber": "66479331",
  "orderCode": "3KTH",
  "status": "IN_TRANSIT",
  "estados": [
    { "estado": "REGISTRADO", "fecha": "2026-08-30 09:12" },
    { "estado": "EN RUTA", "fecha": "2026-08-31 22:40" }
  ]
}
```

#### `POST /track/batch`

Hasta 50 guías por petición, procesadas con concurrencia controlada. Ideal para sincronizar pedidos de un ecommerce de una sola vez.

```bash
curl -X POST "https://api.shalom-api.lat/track/batch" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "orders": [
    { "orderNumber": "66479331", "orderCode": "3KTH" },
    { "orderNumber": "66479332", "orderCode": "9ABC" }
  ]
}'
```

Request (JSON):
```json
{
  "orders": [
    { "orderNumber": "66479331", "orderCode": "3KTH" },
    { "orderNumber": "66479332", "orderCode": "9ABC" }
  ]
}
```

#### `GET /track/voucher`

Genera el comprobante del envío como imagen PNG o PDF listo para adjuntar en el correo de confirmación.

- `orderNumber` (requerido): Número de guía (8 dígitos).
- `orderCode` (requerido): Código de seguridad (4 caracteres).
- `format`: image (por defecto) o pdf.

```bash
curl "https://api.shalom-api.lat/track/voucher" \
  -H "x-api-key: TU_API_KEY"
```

#### `GET /track/label`

Etiqueta de rotulación en PDF para imprimir y pegar en el paquete. Requiere el ose_id que devuelve el rastreo.

- `orderNumber` (requerido): Número de guía (8 dígitos).
- `orderCode` (requerido): Código de seguridad (4 caracteres).

```bash
curl "https://api.shalom-api.lat/track/label" \
  -H "x-api-key: TU_API_KEY"
```

### Agencias y cobertura

#### `GET /agencies`

Listado completo del catálogo (sin rutas aéreas de origen/destino). Filtro opcional por texto.

- `q`: Filtra por departamento, provincia o zona.

```bash
curl "https://api.shalom-api.lat/agencies" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "total": 552,
  "data": [
    {
      "ter_id": 355,
      "lugar_over": "TINGO MARÍA - LEONCIO PRADO",
      "departamento": "HUANUCO",
      "provincia": "LEONCIO PRADO",
      "direccion": "AV. TITO JAIME 914, RUPA RUPA- LEONCIO PRADO - HUANUCO, REF. ...",
      "telefono": "(01) 500 7878",
      "latitud": "-9.292...",
      "longitud": "-75.997...",
      "ter_aereo": 1
    }
  ]
}
```

#### `GET /agencies/search`

Combina texto libre, filtros geográficos y orden por cercanía dentro de un radio. Es el endpoint que alimenta el directorio y el mapa de este sitio.

- `q`: Texto libre: provincia, departamento, zona, nombre o dirección.
- `departamento`: Filtra por departamento (parcial permitido).
- `provincia`: Filtra por provincia (parcial permitido).
- `aereo`: true | false — solo agencias con cobertura aérea.
- `near`: Coordenadas lat,lng para ordenar por cercanía.
- `radius_km`: Radio máximo en km (requiere near).
- `per_page`: Límite de resultados (1–500, por defecto 100).

```bash
curl "https://api.shalom-api.lat/agencies/search" \
  -H "x-api-key: TU_API_KEY"
```

#### `GET /public/agencies`

La misma lista del catálogo sin autenticación, pensada para pruebas y demos. Es la fuente de datos de /agencias en este sitio.

```bash
curl "https://api.shalom-api.lat/public/agencies" \
  -H "x-api-key: TU_API_KEY"
```

### Crear envíos en Shalom Pro

#### `POST /instances` *(Shalom Pro)*

Registra una cuenta Shalom Pro (usuario y contraseña) como instancia de tu usuario. La plataforma gestiona el login con navegador headless y mantiene las cookies de sesión.

```bash
curl -X POST "https://api.shalom-api.lat/instances" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "name": "Tienda Demo",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}'
```

Request (JSON):
```json
{
  "name": "Tienda Demo",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}
```

#### `POST /instances/login` *(Shalom Pro)*

Fuerza el login en pro.shalom.pe y guarda la sesión. No es necesario si guardaste credenciales: la plataforma auto-recupera la sesión cuando expira.

```bash
curl -X POST "https://api.shalom-api.lat/instances/login" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "uuid-de-la-instancia",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}'
```

Request (JSON):
```json
{
  "instanceId": "uuid-de-la-instancia",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}
```

#### `POST /account/register` *(Shalom Pro)*

Crea una guía individual en Shalom Pro: resuelve destinatario por DNI, calcula la tarifa y confirma la orden. Devuelve el número de guía generado.

```bash
curl -X POST "https://api.shalom-api.lat/account/register" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "uuid-de-la-instancia",
  "origen": 7,
  "destino": 582,
  "destinatario": {
    "dni": "44273815",
    "nombre": "María Quispe",
    "telefono": "999888777",
    "direccion": "Av. Los Próceres 1450"
  },
  "productos": [{ "descripcion": "Zapatillas", "cantidad": 1 }]
}'
```

Request (JSON):
```json
{
  "instanceId": "uuid-de-la-instancia",
  "origen": 7,
  "destino": 582,
  "destinatario": {
    "dni": "44273815",
    "nombre": "María Quispe",
    "telefono": "999888777",
    "direccion": "Av. Los Próceres 1450"
  },
  "productos": [{ "descripcion": "Zapatillas", "cantidad": 1 }]
}
```

#### `POST /account/register-bulk` *(Shalom Pro)*

Registra una lista de envíos en una sola petición con auto-resolución de agencias, DNI/RENIEC, tarifa y costo. Pensado para sincronizar todas las órdenes pagadas del día.

```bash
curl -X POST "https://api.shalom-api.lat/account/register-bulk" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "uuid-de-la-instancia",
  "securityCode": "código-de-seguridad",
  "shipments": [
    { "destinatario": "...", "direccion": "...", "productos": [] }
  ]
}'
```

Request (JSON):
```json
{
  "instanceId": "uuid-de-la-instancia",
  "securityCode": "código-de-seguridad",
  "shipments": [
    { "destinatario": "...", "direccion": "...", "productos": [] }
  ]
}
```

#### `POST /account/pending-shipments` *(Shalom Pro)*

Devuelve la lista de envíos pendientes de la cuenta conectada, espejo de la vista de pendientes de Shalom Pro.

```bash
curl -X POST "https://api.shalom-api.lat/account/pending-shipments" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "uuid-de-la-instancia"
}'
```

Request (JSON):
```json
{
  "instanceId": "uuid-de-la-instancia"
}
```

### Cotizar tarifas y consultar DNI

#### `POST /account/quote` *(Shalom Pro)*

Calcula la tarifa entre origen y destino (IDs de agencia del catálogo). Respuesta cacheada ~5 minutos.

```bash
curl -X POST "https://api.shalom-api.lat/account/quote" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "uuid-de-la-instancia",
  "origin": 7,
  "destination": 582
}'
```

Request (JSON):
```json
{
  "instanceId": "uuid-de-la-instancia",
  "origin": 7,
  "destination": 582
}
```

#### `GET /account/dni/{dni}`

Valida un DNI de 8 dígitos contra RENIEC y devuelve nombres y apellidos. Úsalo para autocompletar el destinatario en tu checkout.

- `{dni}` (requerido): DNI de 8 dígitos.

```bash
curl "https://api.shalom-api.lat/account/dni/44273815" \
  -H "x-api-key: TU_API_KEY"
```

### Webhooks de tracking

#### `PUT /webhooks` *(Shalom Pro)*

Configura (o rota) la URL de destino. La respuesta incluye el secreto whsec_ completo una única vez: guárdalo, después solo se muestra enmascarado.

```bash
curl -X PUT "https://api.shalom-api.lat/webhooks" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://mitienda.pe/api/shalom-webhook"
}'
```

Request (JSON):
```json
{
  "url": "https://mitienda.pe/api/shalom-webhook"
}
```

Respuesta (ejemplo):
```json
{
  "url": "https://mitienda.pe/api/shalom-webhook",
  "secret": "whsec_9f2b7c...completo-solo-aqui"
}
```

#### `POST /tracking/subscriptions`

Empieza a seguir una guía: el sistema la polla en segundo plano y dispara el webhook en cada cambio de estado.

```bash
curl -X POST "https://api.shalom-api.lat/tracking/subscriptions" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "orderNumber": "66479331",
  "orderCode": "3KTH"
}'
```

Request (JSON):
```json
{
  "orderNumber": "66479331",
  "orderCode": "3KTH"
}
```

#### `GET /tracking/subscriptions`

Devuelve las guías suscritas por tu usuario con su último estado conocido.

```bash
curl "https://api.shalom-api.lat/tracking/subscriptions" \
  -H "x-api-key: TU_API_KEY"
```

#### `DELETE /tracking/subscriptions`

Deja de seguir una guía: acepta query params orderNumber y orderCode.

- `orderNumber` (requerido): Número de guía (8 dígitos).
- `orderCode` (requerido): Código de seguridad (4 caracteres).

```bash
curl "https://api.shalom-api.lat/tracking/subscriptions" \
  -H "x-api-key: TU_API_KEY"
```

## Recetas

### 1. Rastrear una guía

`orderNumber` son 8 dígitos y `orderCode` 4 caracteres alfanuméricos.

```bash
curl -X POST "https://api.shalom-api.lat/track" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"orderNumber":"66479331","orderCode":"3KTH"}'
```

Devuelve estado actual y línea de tiempo. `404` significa guía inexistente o aún no registrada.

### 2. Encontrar la agencia más cercana a un cliente

Usa la búsqueda avanzada con coordenadas del comprador y guarda `ter_id`, `direccion` y `lugar_over` del primer resultado:

```bash
curl "https://api.shalom-api.lat/agencies/search?near=-12.046,-77.043&radius_km=10&per_page=5" -H "x-api-key: TU_API_KEY"
# data[0] => { ter_id: 651, lugar_over: "AV  NICOLAS DUENAS CDRA. 5", direccion: "..." }
```

El catálogo se actualiza a diario: cachea la lista hasta 24 h en tu app en lugar de consultarla en cada request.

### 3. Cotizar una tarifa (Shalom Pro)

Requiere instancia (cuenta Shalom Pro conectada). `origin` y `destination` son `ter_id` del catálogo de agencias:

```bash
curl -X POST "https://api.shalom-api.lat/account/quote" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"instanceId":"UUID","origin":7,"destination":582}'
```

### 4. Crear una guía en Shalom Pro

Flujo completo: instancia → login → registro del envío:

```bash
curl -X POST "https://api.shalom-api.lat/instances" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"name":"Mi tienda","username":"usuario@tienda.pe","password":"secreto"}'
# guarda el instanceId (UUID) de la respuesta
curl -X POST "https://api.shalom-api.lat/instances/login" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"instanceId":"UUID","username":"usuario@tienda.pe","password":"secreto"}'
curl -X POST "https://api.shalom-api.lat/account/register" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"instanceId":"UUID","origen":7,"destino":582,"destinatario":{"dni":"44273815","nombre":"María Quispe","telefono":"999888777","direccion":"Av. Los Próceres 1450"}}'
```

El registro resuelve destinatario por DNI, tarifa y agencia automáticamente. Rastrea después con `POST /track` usando la guía devuelta.

### 5. Verificar la firma de un webhook
Los webhooks llegan con header `X-Shalom-Signature: t=<unix>,v1=<hex>` donde `v1 = HMAC-SHA256("<t>.<rawBody>", whsec_...)`. Verifica en tu endpoint antes de procesar:

```javascript
import { createHmac, timingSafeEqual } from 'node:crypto';

function verifySignature(rawBody, header, secret) {
  const [tPart, v1Part] = header.split(',');
  const t = tPart.replace('t=', '');
  const v1 = v1Part.replace('v1=', '');
  const expected = createHmac('sha256', secret).update(`${t}.${rawBody}`).digest('hex');
  return timingSafeEqual(Buffer.from(v1), Buffer.from(expected));
}
```

Registra la URL con `PUT /webhooks` (el secreto `whsec_` se muestra completo una sola vez) y suscribe guías con `POST /tracking/subscriptions`.

## Manejo de errores

Todos los errores devuelven JSON `{ "error": "mensaje" }`:

| Código | Significado | Qué hacer |
| --- | --- | --- |
| 400 | Petición malformada | Corrige el campo indicado en el mensaje |
| 401 | API key ausente o inválida | Verifica el header `x-api-key` |
| 403 | Plan expirado o sin permisos | Revisa el plan de la key |
| 404 | Recurso inexistente (guía, DNI, ruta) | Valida los datos antes de reintentar |
| 429 | Rate limit o cuota agotada | Backoff exponencial; consulta `GET /validate` |
| 500 | Error interno o del sistema origen | Reintenta con backoff |

## Reglas de operación

- Valida la API key con `GET /validate` antes de la primera operación del flujo.
- Usa SOLO los endpoints listados en esta guía; no inventes rutas.
- `orderNumber`: 8 dígitos. `orderCode`: 4 caracteres. DNI: 8 dígitos. Lote de tracking: máximo 50 guías.
- Cachea el catálogo de agencias hasta 24 h; se actualiza a diario en el origen.
- Para seguimiento continuo usa webhooks (`PUT /webhooks` + `POST /tracking/subscriptions`), no polling.
- Las operaciones marcadas *(Shalom Pro)* requieren `instanceId` y credenciales del usuario final.
- Ante `429` aplica backoff exponencial y consulta `GET /validate` para ver la cuota restante.

## Recursos

- Índice de documentación: https://shalom-api.lat/docs
- Guías de integración: https://shalom-api.lat/integraciones/n8n · /woocommerce · /shopify
- Directorio de agencias: https://shalom-api.lat/agencias
- Texto plano para LLMs: https://shalom-api.lat/llms.txt y https://shalom-api.lat/llms-full.txt

Shalom API Perú es una plataforma independiente compatible con Shalom Pro; no está afiliada a Shalom Empresarial S.A.C.
