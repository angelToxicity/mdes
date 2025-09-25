# Guía de Integración — APIs de Tokenización
Basado en definiciones Swagger (JSDoc) del archivo adjunto

Fecha: 2025-09-25

--------------------------------------------------------------------------------

## 1) Resumen

Este documento describe, de forma organizada y explicativa, las APIs expuestas para:
- Tokenizar datos de tarjeta
- Destokenizar (recuperar) datos previamente tokenizados

Incluye:
- Descripción de todos los schemas definidos en el archivo.
- Un ejemplo único de request y de response para cada endpoint.
- Estructura de respuesta 200 basada en la Imagen 1 para tokenización.
- Modelo de errores basado en la Imagen 2.
- ACTUALIZACIÓN: Respuesta 200 de destokenización ajustada según la Imagen 3.

--------------------------------------------------------------------------------

## 2) Información general

- Base path sugerido: /tokenization
- Formato: application/json
- Autenticación: bearerAuth (JWT u otro Bearer Token)

Encabezado requerido:
```
Authorization: Bearer <token>
```

--------------------------------------------------------------------------------

## 3) Modelos (Schemas)

A continuación, se listan todos los schemas definidos en el archivo Swagger (.js):

### 3.1) thTokenizarCard
Objeto de entrada para tokenizar.

Campos:
- cardNumber (string): PAN o número de tarjeta.
- holderCard (string): Nombre del titular.
- expirationCard (string): Fecha de expiración (formato acordado, ej. MMYY o MM/YY).
- codBank (string): Código o identificador de banco.

Ejemplo:
```json
{
  "cardNumber": "5412345678901234",
  "holderCard": "JUAN PEREZ",
  "expirationCard": "1227",
  "codBank": "001"
}
```

### 3.2) thDetokenizeData
Objeto de entrada para destokenizar.

Campos:
- token (string): Token previamente emitido.

Ejemplo:
```json
{
  "token": "9237398539653612"
}
```

### 3.3) Modelo de respuesta 200 — Tokenizar (basado en Imagen 1)
Se adopta la estructura de éxito (HTTP 200) de la Imagen 1 para el endpoint de tokenización:

```json
{
  "response": {
    "token": "9237398539653612",
    "stored_at": 1735310199684
  }
}
```

- response.token (string): Token generado.
- response.stored_at (number): Marca de tiempo (epoch ms) del registro.

### 3.4) Modelo de respuesta 200 — Destokenizar (basado en Imagen 3)
La Imagen 3 muestra que el servicio construye un objeto `card` con:
- `token_value` y `token_count` desde `cards[0].token_value` y `cards[0].token_count`.
- `card_pan`, `card_expiry`, `card_holder` desde `msgDesCryt.split(';')` en los índices [2], [3], [1].

Respuesta 200 de referencia:
```json
{
  "token_value": "9237398539653612",
  "card_pan": "5412345678901234",
  "card_expiry": "1227",
  "card_holder": "JUAN PEREZ",
  "token_count": 1
}
```

Notas:
- La respuesta de destokenización ya no usa el contenedor `response` de la Imagen 1. Ahora devuelve directamente el objeto plano anterior.
- Si requieres compatibilidad hacia atrás, puedes envolver este objeto dentro de `response`, pero la implementación mostrada en la Imagen 3 envía `res.status(200).send(card)` sin contenedor.

### 3.5) Modelo de error (basado en Imagen 2)
Estructura de error:

```json
{
  "msg": "unauthorized 02"
}
```

Ejemplos sugeridos por estado:
- 401 Unauthorized:
  ```json
  { "msg": "unauthorized 02" }
  ```
- 403 Forbidden:
  ```json
  { "msg": "forbidden" }
  ```
- 500 Internal Server Error:
  ```json
  { "msg": "unexpected error" }
  ```

--------------------------------------------------------------------------------

## 4) Endpoints

A menos que se indique lo contrario, todos los endpoints requieren:
- Header: `Authorization: Bearer <token>`
- Content-Type: `application/json`

### 4.1) POST /tokenization/tokenizeData
- Summary: Tokenizar
- Tag: TOKENIZACION
- Description: Tokenizar

Request body (schema: thTokenizarCard):
```json
{
  "cardNumber": "5412345678901234",
  "holderCard": "JUAN PEREZ",
  "expirationCard": "1227",
  "codBank": "001"
}
```

Ejemplo cURL:
```bash
curl -X POST "<BASE_URL>/tokenization/tokenizeData" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "cardNumber": "5412345678901234",
    "holderCard": "JUAN PEREZ",
    "expirationCard": "1227",
    "codBank": "001"
  }'
```

Response 200 (basado en Imagen 1):
```json
{
  "response": {
    "token": "9237398539653612",
    "stored_at": 1735310199684
  }
}
```

Posibles errores:
- 401 Unauthorized:
  ```json
  { "msg": "unauthorized 02" }
  ```
- 403 No se pudo tokenizar la información:
  ```json
  { "msg": "forbidden" }
  ```
- 500 Error inesperado:
  ```json
  { "msg": "unexpected error" }
  ```

---

### 4.2) POST /tokenization/detokenizeData
- Summary: Destokenizar
- Tag: TOKENIZACION
- Description: Destokenizar una tarjeta

Request body (schema: thDetokenizeData):
```json
{
  "token": "9237398539653612"
}
```

Ejemplo cURL:
```bash
curl -X POST "<BASE_URL>/tokenization/detokenizeData" \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{
    "token": "9237398539653612"
  }'
```

Response 200 (ACTUALIZADO — basado en Imagen 3):
```json
{
  "token_value": "9237398539653612",
  "card_pan": "5412345678901234",
  "card_expiry": "1227",
  "card_holder": "JUAN PEREZ",
  "token_count": 1
}
```

Posibles errores:
- 401 Unauthorized:
  ```json
  { "msg": "unauthorized 02" }
  ```
- 403 No se pudo destokenizar la información:
  ```json
  { "msg": "forbidden" }
  ```
- 500 Error inesperado:
  ```json
  { "msg": "unexpected error" }
  ```

--------------------------------------------------------------------------------

## 5) Recomendaciones

- Idempotencia: utiliza identificadores de correlación si tu backend lo soporta.
- Seguridad:
  - Asegura la obtención y validación del Bearer Token.
  - Evita incluir PANs completos en logs.
- Validaciones de entrada:
  - thTokenizarCard: valida formato de `cardNumber` y `expirationCard`.
  - thDetokenizeData: valida presencia y formato de `token`.
- Errores: mantén la estructura `{ "msg": "..." }` para consistencia con la Imagen 2.

--------------------------------------------------------------------------------

## 6) Ejemplos de integración rápida

Tokenizar (fetch/JS):
```js
await fetch(`${BASE_URL}/tokenization/tokenizeData`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    cardNumber: '5412345678901234',
    holderCard: 'JUAN PEREZ',
    expirationCard: '1227',
    codBank: '001'
  })
});
```

Destokenizar (fetch/JS):
```js
await fetch(`${BASE_URL}/tokenization/detokenizeData`, {
  method: 'POST',
  headers: {
    'Authorization': `Bearer ${token}`,
    'Content-Type': 'application/json'
  },
  body: JSON.stringify({
    token: '9237398539653612'
  })
}).then(r => r.json()).then(card => {
  // card = { token_value, card_pan, card_expiry, card_holder, token_count }
});
```

--------------------------------------------------------------------------------

Fin del documento.