---
name: shalom-api
description: Integra Shalom Perú (courier) por API REST desde cualquier agente de IA. Use when the user says "rastrear envío shalom", "seguir mi pedido shalom", "guía de shalom", "crear guía shalom pro", "agencia shalom más cercana", "cotizar envío shalom", "integrar shalom", "API shalom" or "shalom api peru" — or asks to add courier shipping, COD delivery tracking, agency lookup with coordinates, webhook delivery updates for WooCommerce/Shopify/n8n to any app. Covers tracking de envíos, catálogo de agencias, cotizaciones, DNI, creación de guías en Shalom Pro y webhooks firmados.
---

# Shalom API — Guía de integración para agentes de IA

Shalom API Perú expone el sistema del courier Shalom (shalom.com.pe) como API REST con respuestas JSON. Esta guía contiene todo lo necesario para integrarlo sin explorar: autenticación, endpoints esenciales, recetas paso a paso y reglas de operación.

- Base URL: `https://api.shalom-api.lat`
- Documentación pública completa: https://shalom-api.lat/docs (última actualización: 2026-09-08)
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
  "userId": "b7c9d1e2-4f6a-4c3b-9d2e-1a2b3c4d5e6f",
  "limit": 1000,
  "currentUsage": 137,
  "remaining": 863,
  "message": "API key válida"
}
```

### Rastrear envíos

#### `POST /track`

Devuelve el resultado de búsqueda y los estados de la guía (espejo del sistema de Shalom).

- `orderNumber` (requerido): Número de guía / orden (string de 8 dígitos, patrón ^[0-9]+$).
- `orderCode` (requerido): Código de seguridad (string de 4 caracteres).

```bash
curl -X POST "https://api.shalom-api.lat/track?orderNumber=66479331&orderCode=3KTH" \
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
  "search": { "..." : "resultado de búsqueda de Shalom (espejo)" },
  "statuses": [ { "..." : "estados de la guía (espejo)" } ]
}
```

#### `POST /track/batch`

Rastrea múltiples guías con control de flujo y concurrencia. Límite máximo de 50 órdenes por petición.

- `orders` (requerido): Array de objetos { orderNumber, orderCode }, máximo 50 items.

```bash
curl -X POST "https://api.shalom-api.lat/track/batch?orders={orders}" \
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

Respuesta (ejemplo):
```json
[
  { "search": { "..." : "..." }, "statuses": [ "..." ] },
  { "search": { "..." : "..." }, "statuses": [ "..." ] }
]
```

#### `GET /track/voucher`

Genera y descarga el comprobante del envío en formato imagen (por defecto) o PDF.

- `orderNumber` (requerido): Número de guía (8 dígitos).
- `orderCode` (requerido): Código de seguridad (4 caracteres).
- `format`: Formato de descarga: image (default) | pdf.

```bash
curl "https://api.shalom-api.lat/track/voucher?orderNumber=66479331&orderCode=3KTH&format=pdf" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
Content-Type: image/png

<binario — imagen o PDF del comprobante>
```

#### `GET /track/label`

Descarga el PDF de la etiqueta de envío usando el número y código de orden. Retorna PDF.

- `orderNumber` (requerido): Número de guía (8 dígitos).
- `orderCode` (requerido): Código de seguridad (4 caracteres).

```bash
curl "https://api.shalom-api.lat/track/label?orderNumber=66479331&orderCode=3KTH" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
Content-Type: application/pdf

<binario — PDF de la etiqueta>
```

### Agencias y cobertura

#### `GET /agencies`

Listado completo de agencias autorizadas omitiendo rutas aéreas de origen/destino. Filtro opcional por texto.

- `q`: Texto de búsqueda para filtrar por departamento, provincia o zona.

```bash
curl "https://api.shalom-api.lat/agencies?q=lima" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Lista de agencias minimal.",
  "total": 552,
  "query": "lima",
  "data": [
    {
      "ter_id": 392,
      "ter_abrebiatura": "MLV",
      "zona": "MALVINAS - JR. GARCIA VILLON",
      "ter_zona": "NORTE 1",
      "provincia": "LIMA",
      "departamento": "LIMA",
      "latitud": "-12.063...",
      "longitud": "-77.012...",
      "direccion": "JR. GARCIA VILLON 250, ...",
      "telefono": "(01) 500 7878",
      "hora_atencion": "LUNES A VIERNES - 8:00 AM A 8:00 PM",
      "hora_domingo": "DOMINGOS DE 8:00 AM A 5:00 PM",
      "estadoAgencia": "ATENDIENDO EN ESTE MOMENTO",
      "nombre": "LIMA / LIMA / LIMA / MALVINAS - JR. GARCIA VILLON",
      "lugar_over": "MALVINAS - JR. GARCIA VILLON",
      "ter_aereo": 1,
      "dep_id": 7,
      "prov_id": 1,
      "ubi_id": 70101,
      "...": "48 campos en total por agencia"
    }
  ]
}
```

#### `GET /agencies/search`

Busca por texto libre, departamento, provincia, disponibilidad aérea, u ordena por cercanía a coordenadas (near) en un radio específico.

- `q`: Texto de búsqueda libre.
- `departamento`: Filtrar por departamento.
- `provincia`: Filtrar por provincia.
- `aereo`: enum: true | false — filtrar por habilitación aérea.
- `near`: Coordenadas lat,lng (patrón numérico con signo) para ordenar por cercanía.
- `radius_km`: Radio de cobertura máximo en kilómetros.
- `per_page`: Límite de resultados a retornar (1–500, default 100).

```bash
curl "https://api.shalom-api.lat/agencies/search?q=lima&departamento=LIMA&provincia=LIMA&aereo=true&near=-12.046,-77.043&radius_km=5&per_page=5" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "total": 26,
  "returned": 2,
  "data": [
    {
      "ter_id": 392,
      "lugar_over": "MALVINAS - JR. GARCIA VILLON",
      "departamento": "LIMA",
      "provincia": "LIMA",
      "direccion": "JR. GARCIA VILLON 250, ...",
      "latitud": "-12.063...",
      "longitud": "-77.012...",
      "distancia_km": 0.71
    }
  ]
}
```

#### `GET /public/agencies`

Listado público de agencias para la landing de demostración. No requiere API key ni consume cuota.

- `q`: Texto de búsqueda para filtrar por departamento, provincia o zona.

```bash
curl "https://api.shalom-api.lat/public/agencies?q=lima" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Lista de agencias minimal.",
  "total": 552,
  "query": "",
  "data": [ { "ter_id": 3, "..." : "misma estructura que GET /agencies" } ]
}
```

#### `GET /public/agencies/search`

Búsqueda avanzada pública con los mismos filtros que GET /agencies/search. No requiere API key.

- `q`: Texto de búsqueda libre.
- `departamento`: Filtrar por departamento.
- `provincia`: Filtrar por provincia.
- `aereo`: enum: true | false.
- `near`: Coordenadas lat,lng para ordenar por cercanía.
- `radius_km`: Radio máximo en kilómetros.
- `per_page`: Límite de resultados (1–500, default 100).

```bash
curl "https://api.shalom-api.lat/public/agencies/search?q=lima&departamento=LIMA&provincia=LIMA&aereo=true&near=-12.046,-77.043&radius_km=5&per_page=5" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "total": 26,
  "returned": 2,
  "data": [ { "..." : "misma estructura que GET /agencies/search" } ]
}
```

### Ubicaciones (ubigeos)

#### `GET /locations/departments`

Obtiene todos los departamentos del Perú con cobertura de Shalom.

```bash
curl "https://api.shalom-api.lat/locations/departments" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "items": [
    { "id": 1, "name": "AMAZONAS", "ubi_id": 10101 },
    { "id": 7, "name": "LIMA", "ubi_id": 70101 }
  ]
}
```

#### `GET /locations/departments/{depId}/provinces`

Obtiene todas las provincias pertenecientes al departamento especificado por su ID.

- `{depId}` (requerido): ID del departamento (integer), ej. 15.

```bash
curl "https://api.shalom-api.lat/locations/departments/15/provinces?{depId}={{depId}}" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "items": [
    { "id": 1, "name": "LEONCIO PRADO", "ubi_id": 10101 }
  ]
}
```

#### `GET /locations/departments/{depId}/provinces/{provId}/districts`

Obtiene todos los distritos pertenecientes a la provincia y departamento especificados por sus IDs.

- `{depId}` (requerido): ID del departamento (integer).
- `{provId}` (requerido): ID de la provincia (integer), ej. 1.

```bash
curl "https://api.shalom-api.lat/locations/departments/15/provinces/1/districts?{depId}={{depId}}&{provId}={{provId}}" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "items": [
    { "id": 1, "name": "RUPA RUPA", "ubi_id": 10101 }
  ]
}
```

### Instancias (sesión Shalom Pro)

#### `POST /instances`

Crea una nueva instancia para el usuario autenticado con su API Key. No consume cuota.

- `name`: Nombre descriptivo de la instancia (ej. Sucursal Principal).

```bash
curl -X POST "https://api.shalom-api.lat/instances?name={name}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "name": "Mi Instancia"
}'
```

Request (JSON):
```json
{
  "name": "Mi Instancia"
}
```

Respuesta (ejemplo):
```json
{
  "status": "created",
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "name": "Mi Instancia",
  "message": "Instance created successfully"
}
```

#### `GET /instances`

Devuelve todas las instancias pertenecientes al usuario de la API Key proporcionada. No consume cuota.

```bash
curl "https://api.shalom-api.lat/instances" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "instances": [
    {
      "id": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
      "name": "Mi Instancia",
      "username": "usuario@mitienda.pe",
      "createdAt": "2026-08-01T10:30:00.000Z",
      "isLoggedIn": true
    }
  ]
}
```

#### `DELETE /instances`

Elimina la instancia y su sesión persistida del sistema. Requiere API key de instancia. No consume cuota.

- `instanceId` (requerido): ID de la instancia (en el body).

```bash
curl -X DELETE "https://api.shalom-api.lat/instances?instanceId={instanceId}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}'
```

Request (JSON):
```json
{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

Respuesta (ejemplo):
```json
{
  "status": "closed",
  "message": "Instance closed successfully"
}
```

#### `POST /instances/status` *(Shalom Pro)*

Verifica si la instancia está logueada en Shalom Pro. No consume cuota.

- `instanceId` (requerido): ID de la instancia (en el body).

```bash
curl -X POST "https://api.shalom-api.lat/instances/status?instanceId={instanceId}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}'
```

Request (JSON):
```json
{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

Respuesta (ejemplo):
```json
{
  "isLoggedIn": true,
  "username": "usuario@mitienda.pe",
  "url": "https://pro.shalom.pe"
}
```

#### `POST /instances/login` *(Shalom Pro)*

Realiza el login en pro.shalom.pe con navegador headless (resuelve reCAPTCHA v3). Las credenciales se guardan para auto-login futuro. No consume cuota.

- `instanceId` (requerido): ID de la instancia.
- `username` (requerido): Usuario/Email para iniciar sesión.
- `password` (requerido): Contraseña del usuario.

```bash
curl -X POST "https://api.shalom-api.lat/instances/login?instanceId={instanceId}&username={username}&password={password}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}'
```

Request (JSON):
```json
{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7",
  "username": "usuario@mitienda.pe",
  "password": "••••••••"
}
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Login successful",
  "url": "https://pro.shalom.pe"
}
```

#### `POST /instances/logout` *(Shalom Pro)*

Cierra la sesión de Shalom Pro y limpia la sesión persistida (incluye credenciales guardadas). No consume cuota.

- `instanceId` (requerido): ID de la instancia (en el body).

```bash
curl -X POST "https://api.shalom-api.lat/instances/logout?instanceId={instanceId}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}'
```

Request (JSON):
```json
{
  "instanceId": "7c9e6679-7425-40de-944b-e07fc1f90ae7"
}
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Logged out and session cleared"
}
```

### Crear envíos en Shalom Pro

#### `POST /account/register` *(Shalom Pro)*

Registra un envío individual directamente en la API de Shalom Pro.

- `instanceId` (requerido): ID de la instancia.
- `origen` (requerido): ID del terminal de origen (integer).
- `destino` (requerido): ID del terminal de destino (integer o string con prefijo "0" para aéreo, ej. "052").
- `documento` (requerido): DNI del destinatario (se usa para la creación automática).
- `name` (requerido): Nombres del destinatario.
- `firstname` (requerido): Primer apellido del destinatario.
- `lastname` (requerido): Segundo apellido del destinatario.
- `phone` (requerido): Teléfono del destinatario (integer).
- `content`: Nombre del producto (ej: SOBRE, PAQUETE XS). Automatiza tipo y costo.
- `cantidad`: Cantidad de bultos (integer).
- `clave`: Clave de recojo del envío.
- `declaracion_jurada`: enum: '' | 'Artículos de uso personal' | 'Documentos' | 'Ropa' | 'Electrodomésticos' — acepción para envío aéreo.
- `aereo`: enum: 0 | 1 — se autodetecta si destino empieza con 0.
- `costo`: Opcional si se envía content (se calculará automáticamente).

```bash
curl -X POST "https://api.shalom-api.lat/account/register?instanceId={instanceId}&origen={origen}&destino={destino}&documento={documento}&name={name}&firstname={firstname}&lastname={lastname}&phone={phone}&content={content}&cantidad={cantidad}&clave={clave}&declaracion_jurada={declaracion_jurada}&aereo=true&costo={costo}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc",
  "origen": 7,
  "destino": "052",
  "content": "PAQUETE XS",
  "cantidad": 1,
  "documento": "03891771",
  "name": "RONALD EDGAR",
  "firstname": "RANGEL",
  "lastname": "ANTON",
  "phone": 949916360,
  "clave": "1234",
  "declaracion_jurada": "Ropa"
}'
```

Request (JSON):
```json
{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc",
  "origen": 7,
  "destino": "052",
  "content": "PAQUETE XS",
  "cantidad": 1,
  "documento": "03891771",
  "name": "RONALD EDGAR",
  "firstname": "RANGEL",
  "lastname": "ANTON",
  "phone": 949916360,
  "clave": "1234",
  "declaracion_jurada": "Ropa"
}
```

Respuesta (ejemplo):
```json
{
  "...": "espejo de la respuesta de Shalom Pro con los datos de la orden creada"
}
```

#### `POST /account/register-bulk` *(Shalom Pro)*

Registra envíos masivos en Shalom a partir de una lista de shipments. Requiere estar logueado previamente.

- `instanceId` (requerido): ID de la instancia.
- `shipments` (requerido): Lista de envíos (mínimo 1). Cada item requiere: recipientDoc, recipientPhone, origin, destination, content.
- `securityCode`: Clave de seguridad de 4 dígitos.

```bash
curl -X POST "https://api.shalom-api.lat/account/register-bulk?instanceId={instanceId}&shipments={shipments}&securityCode={securityCode}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc",
  "shipments": [
    {
      "recipientDoc": "72845631",
      "recipientPhone": "987654321",
      "contactDoc": "72845631",
      "contactPhone": "987654321",
      "grr": "GRR-001",
      "origin": "LIMA",
      "destination": "PIURA",
      "content": "PAQUETE XS",
      "height": "10",
      "width": "20",
      "length": "30",
      "weight": "1.5",
      "quantity": "1"
    }
  ],
  "securityCode": "5858"
}'
```

Request (JSON):
```json
{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc",
  "shipments": [
    {
      "recipientDoc": "72845631",
      "recipientPhone": "987654321",
      "contactDoc": "72845631",
      "contactPhone": "987654321",
      "grr": "GRR-001",
      "origin": "LIMA",
      "destination": "PIURA",
      "content": "PAQUETE XS",
      "height": "10",
      "width": "20",
      "length": "30",
      "weight": "1.5",
      "quantity": "1"
    }
  ],
  "securityCode": "5858"
}
```

Respuesta (ejemplo):
```json
{
  "...": "espejo de la respuesta de Shalom Pro con el resultado por envío"
}
```

#### `POST /account/pending-shipments` *(Shalom Pro)*

Obtiene el espejo (mirror) de la respuesta de Shalom para los envíos que están en estado PENDIENTE.

- `instanceId` (requerido): ID de la instancia (en el body).

```bash
curl -X POST "https://api.shalom-api.lat/account/pending-shipments?instanceId={instanceId}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc"
}'
```

Request (JSON):
```json
{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc"
}
```

Respuesta (ejemplo):
```json
{
  "...": "espejo de la vista de envíos pendientes de Shalom Pro"
}
```

#### `POST /account/get-user` *(Shalom Pro)*

Obtiene el espejo (mirror) de la respuesta de Shalom para los datos del usuario autenticado en la instancia.

- `instanceId` (requerido): ID de la instancia (en el body).

```bash
curl -X POST "https://api.shalom-api.lat/account/get-user?instanceId={instanceId}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc"
}'
```

Request (JSON):
```json
{
  "instanceId": "2e656a02-7e37-4573-9d68-e76740d337dc"
}
```

Respuesta (ejemplo):
```json
{
  "...": "espejo del perfil del usuario Shalom autenticado"
}
```

### Cotizar tarifas y consultar DNI

#### `POST /account/quote`

Calcula el costo de un envío basado en origen y destino (ID o nombre del terminal). Respuesta cacheada ~5 minutos.

- `origin` (requerido): ID o nombre del terminal de origen (number | string), ej. 7.
- `destination` (requerido): ID o nombre del terminal de destino (number | string), ej. 582.

```bash
curl -X POST "https://api.shalom-api.lat/account/quote?origin={origin}&destination={destination}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "origin": 7,
  "destination": 582
}'
```

Request (JSON):
```json
{
  "origin": 7,
  "destination": 582
}
```

Respuesta (ejemplo):
```json
{
  "...": "espejo de la tarifa calculada por el sistema de Shalom (tarifa/mostrar)"
}
```

#### `GET /account/dni/{dni}`

Obtiene información de una persona por su número de DNI (espejo RENIEC).

- `{dni}` (requerido): Número de DNI (string de 8 dígitos), ej. 12345678.

```bash
curl "https://api.shalom-api.lat/account/dni/44273815?{dni}={{dni}}" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "...": "espejo de la información RENIEC del DNI consultado"
}
```

### Webhooks de tracking

#### `PUT /webhooks` *(Shalom Pro)*

Registra la URL de webhook de la cuenta y genera un secreto de firma. El secreto se devuelve completo solo aquí. Usa rotateSecret=true para rotarlo.

- `url` (requerido): URL destino (formato uri, https recomendado).
- `rotateSecret`: Regenerar el secreto de firma (boolean, default false).

```bash
curl -X PUT "https://api.shalom-api.lat/webhooks?url={url}&rotateSecret={rotateSecret}" \
  -H "x-api-key: TU_API_KEY" \
  -H "Content-Type: application/json" \
  -d '{
  "url": "https://tu-servidor.com/webhooks/shalom"
}'
```

Request (JSON):
```json
{
  "url": "https://tu-servidor.com/webhooks/shalom"
}
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Webhook configurado.",
  "webhook": {
    "url": "https://tu-servidor.com/webhooks/shalom",
    "secret": "whsec_9f2b7c4d1e2f3a4b5c6d7e8f9a0b1c2d3e4f5a6b7c8d9e0f",
    "enabled": true
  }
}
```

#### `GET /webhooks`

Devuelve la configuración de webhook de la cuenta (secreto enmascarado).

```bash
curl "https://api.shalom-api.lat/webhooks" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "configured": true,
  "webhook": {
    "url": "https://tu-servidor.com/webhooks/shalom",
    "enabled": true,
    "secretPreview": "whsec_9f2b7c4d…e0f"
  }
}
```

#### `DELETE /webhooks`

Elimina la configuración de webhook de la cuenta.

```bash
curl "https://api.shalom-api.lat/webhooks" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Webhook eliminado."
}
```

#### `POST /tracking/subscriptions`

Suscribe la cuenta a los cambios de estado de un envío. Registra el envío en el sistema de tracking en background y notifica vía webhook cuando cambie de estado.

- `orderNumber` (requerido): Número de guía (8 dígitos, patrón ^[0-9]+$).
- `orderCode` (requerido): Código de seguridad (4 caracteres).

```bash
curl -X POST "https://api.shalom-api.lat/tracking/subscriptions?orderNumber=66479331&orderCode=3KTH" \
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
  "success": true,
  "message": "Suscripción creada.",
  "subscription": {
    "id": "5f4e3d2c-1b0a-9988-7766-554433221100",
    "orderNumber": "66479331",
    "orderCode": "3KTH",
    "lastStatus": null,
    "active": true,
    "createdAt": "2026-09-08T12:00:00.000Z"
  }
}
```

#### `GET /tracking/subscriptions`

Lista las suscripciones de tracking de la cuenta.

```bash
curl "https://api.shalom-api.lat/tracking/subscriptions" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "total": 1,
  "data": [
    {
      "id": "5f4e3d2c-1b0a-9988-7766-554433221100",
      "orderNumber": "66479331",
      "orderCode": "3KTH",
      "lastStatus": "IN_TRANSIT",
      "active": true,
      "createdAt": "2026-09-08T12:00:00.000Z"
    }
  ]
}
```

#### `DELETE /tracking/subscriptions`

Cancela la suscripción de la cuenta a un envío.

- `orderNumber` (requerido): Número de guía (8 dígitos, querystring).
- `orderCode` (requerido): Código de seguridad (4 caracteres, querystring).

```bash
curl "https://api.shalom-api.lat/tracking/subscriptions?orderNumber=66479331&orderCode=3KTH" \
  -H "x-api-key: TU_API_KEY"
```

Respuesta (ejemplo):
```json
{
  "success": true,
  "message": "Suscripción eliminada."
}
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

### 3. Cotizar una tarifa

Solo requiere tu API key de usuario: `origin` y `destination` aceptan el ID (number) o el nombre del terminal:

```bash
curl -X POST "https://api.shalom-api.lat/account/quote" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"origin":7,"destination":582}'
```

La respuesta es el espejo de la tarifa calculada por Shalom y se cachea ~5 minutos.

### 4. Crear una guía en Shalom Pro

Flujo completo: instancia → login → registro del envío:

```bash
curl -X POST "https://api.shalom-api.lat/instances" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"name":"Mi tienda"}'
# guarda el instanceId (UUID) de la respuesta
curl -X POST "https://api.shalom-api.lat/instances/login" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"instanceId":"UUID","username":"usuario@tienda.pe","password":"secreto"}'
curl -X POST "https://api.shalom-api.lat/account/register" -H "x-api-key: TU_API_KEY" -H "Content-Type: application/json" -d '{"instanceId":"UUID","origen":7,"destino":"052","content":"PAQUETE XS","cantidad":1,"documento":"03891771","name":"RONALD EDGAR","firstname":"RANGEL","lastname":"ANTON","phone":949916360,"clave":"1234","declaracion_jurada":"Ropa"}'
```

Campos requeridos: instanceId, origen (int), destino (int o string con prefijo "0" para aéreo), documento, name, firstname, lastname, phone. `content` automatiza tipo y costo. Para lote usa `POST /account/register-bulk` con `shipments[]` (required: recipientDoc, recipientPhone, origin, destination, content).

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
- Instalar el nodo de n8n: https://shalom-api.lat/docs/instalar-n8n
- Instalar esta skill en otro agente: https://shalom-api.lat/docs/instalar-skill
- Directorio de agencias: https://shalom-api.lat/agencias
- Texto plano para LLMs: https://shalom-api.lat/llms.txt y https://shalom-api.lat/llms-full.txt

Shalom API Perú es una plataforma independiente compatible con Shalom Pro; no está afiliada a Shalom Empresarial S.A.C.
