# API Affiliate Firmoyo — Crear y enviar envelope desde template

Documento técnico para integradores. Describe el endpoint que permite crear y enviar uno o varios *envelopes* (sobres de firma) a partir de un *template* preexistente en Firmoyo.

> **Audiencia:** desarrollador backend que va a consumir el servicio desde su aplicación.
> **Versión del documento:** 1.0 — 2026-05-12
> **Owner:** equipo Firmoyo (backend)

---

## 1. Información general

| Item | Valor |
|---|---|
| **Método HTTP** | `POST` |
| **Ambiente DEV** | `https://api-dev.firmoyo.digiyo.id` |
| **Ambiente PROD** | `TODO: confirmar URL de producción` |
| **Path** | `/affiliate-firmoyo/affiliate/create-and-send-envelope-base-on-template` |
| **Content-Type** | `application/json` |
| **Charset** | `UTF-8` |
| **Naturaleza** | Idempotente (vía header `Idempotency-Key`) |
| **Operación** | Crea N envelopes en una sola llamada (batch) y los envía al destinatario por el canal indicado. |

URL completa de DEV:

```
POST https://api-dev.firmoyo.digiyo.id/affiliate-firmoyo/affiliate/create-and-send-envelope-base-on-template
```

---

## 2. Autenticación

La autenticación se realiza vía API Key en un header propietario.

| Header | Obligatorio | Descripción |
|---|---|---|
| `X-RshkMichi-ApiKey` | Sí | API Key entregada por el equipo Firmoyo. Identifica al *affiliate* (cliente integrador). **No se debe versionar en repos públicos ni exponer en clientes (front, mobile).** |

Ejemplo (DEV — **no usar en producción**):

```
X-RshkMichi-ApiKey: ec9f4832ff33232a4de1@affiliate-firmoyo-do-not-use-in-production@34cd9e947729523b66h3
```

> El sufijo `do-not-use-in-production` indica que esta key está rate-limited y/o sandbox. Solicitar key de PROD al equipo Firmoyo antes del go-live.

**Buenas prácticas:**

- Almacenar la key en un secret manager (Vault, AWS Secrets Manager, GCP Secret Manager) y leerla desde variables de entorno.
- Rotar la key periódicamente y ante cualquier sospecha de exposición.
- No loguear el header en sistemas de observabilidad (enmascarar al menos los últimos N caracteres).

---

## 3. Idempotencia

El endpoint soporta idempotencia mediante el header `Idempotency-Key`.

| Header | Obligatorio | Descripción |
|---|---|---|
| `Idempotency-Key` | Sí (recomendado) | Identificador único generado por el cliente para cada request lógico. Si se reintenta el mismo request con la misma key, el servidor devuelve la respuesta original sin volver a crear envelopes. |

**Recomendaciones para el cliente:**

- Generar la key como `UUID v4` por cada request lógico (no por cada reintento HTTP).
- Persistir la key junto a la operación de negocio (ej.: en una tabla `outbox`) para poder reintentar de forma segura ante fallos de red, timeouts o 5xx.
- Conservar la misma key durante toda la cadena de reintentos del mismo request lógico.
- TTL de la key del lado del servidor: **TODO: confirmar (típicamente 24h–7 días).**

Ejemplo:

```
Idempotency-Key: 20d93d96-2d94-421f-a00b-929dfe84a413
```

---

## 4. Headers requeridos (resumen)

```
Content-Type:        application/json
X-RshkMichi-ApiKey:  <api-key>
Idempotency-Key:     <uuid-v4>
```

---

## 5. Request body

El body acepta un objeto con un `templateCode` y una lista `entries`. Cada entrada en `entries` representa **un envelope a crear**, lo que permite enviar múltiples envelopes en una sola llamada (batch).

### 5.1 Estructura

```json
{
  "templateCode": "b45f9bcd-f061-4132-8836-a0b79590c7b4",
  "entries": [
    {
      "envelopeName": "Bases y condiciones",
      "externalId": "CONTRATO-2026-000123",
      "messageMail": "Firma el documento",
      "companyEmail": "company@gmaidsl.ds",
      "data": {
        "nroSocio": "Nro43221",
        "cedula": "2311122",
        "nombreCompleto": "Juan Carlos Benitez",
        "domicilio": "Italia casi Fernando de la Mora"
      },
      "recipients": [
        {
          "userType": "CI-PY",
          "fullName": "Juan Carlos Benitez",
          "userValue": "5333186",
          "deliveryType": "EMAIL",
          "mail": "mail@gmail.com",
          "signType": "SIMPLE_SIGN",
          "verificationType": "LINK_ONLY"
        }
      ]
    }
  ]
}
```

### 5.2 Campos de nivel raíz

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `templateCode` | `string` (UUID) | Sí | Identificador del template previamente cargado en Firmoyo. Define la plantilla del documento, sus campos y la posición de las firmas. **TODO: documentar cómo obtener el `templateCode` (panel admin / endpoint de listado).** |
| `entries` | `array<Entry>` | Sí | Lista de envelopes a generar. Mínimo 1. **TODO: confirmar máximo permitido por request (sugerido ≤ 100).** |

### 5.3 Objeto `Entry`

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `envelopeName` | `string` | Sí | Nombre legible del envelope. Aparece en notificaciones y dashboard. |
| `externalId` | `string` | No | **ID propio del consumidor** para correlacionar el envelope con la entidad de negocio en su sistema (ej.: ID de contrato, número de operación). Si se envía, Firmoyo lo persiste y lo retorna tal cual en `results[].externalId`. **Recomendado** para trazabilidad y conciliación. |
| `messageMail` | `string` | Sí (si `deliveryType=EMAIL`) | Mensaje incluido en el cuerpo del mail al destinatario. |
| `companyEmail` | `string` (email) | Sí | Email de la empresa que envía el envelope (remitente lógico / contacto). |
| `data` | `object<string,string>` | Sí | Diccionario clave-valor con los datos que se inyectan en el template. Las claves deben coincidir con las variables definidas en el `templateCode`. |
| `recipients` | `array<Recipient>` | Sí | Lista de destinatarios firmantes. Mínimo 1. **Soporta múltiples firmantes en paralelo** (todos pueden firmar simultáneamente, sin orden). |

> **Importante sobre `data`:** las claves disponibles dependen 100% del template. Si una clave del template no está presente en `data`, el envelope puede fallar o renderizar vacío. Coordinar con el dueño del template las claves esperadas.

### 5.4 Objeto `Recipient`

| Campo | Tipo | Obligatorio | Descripción |
|---|---|---|---|
| `userType` | `enum` | Sí | Tipo de identificación del firmante. Ejemplo visto: `CI-PY` (Cédula de Identidad Paraguay). **TODO: documentar todos los valores soportados.** |
| `fullName` | `string` | Sí | Nombre y apellido completos del firmante. |
| `userValue` | `string` | Sí | Valor del documento de identificación, según `userType`. Para `CI-PY` corresponde al número de cédula. |
| `deliveryType` | `enum` | Sí | Canal de envío. Ejemplo visto: `EMAIL`. **TODO: documentar otros canales (`SMS`, `WHATSAPP`, etc.).** |
| `mail` | `string` (email) | Condicional | Obligatorio si `deliveryType=EMAIL`. |
| `signType` | `enum` | Sí | Tipo de firma a aplicar. Ejemplo visto: `SIMPLE_SIGN`. **TODO: documentar otros tipos (`ADVANCED_SIGN`, `QUALIFIED_SIGN`, etc.).** |
| `verificationType` | `enum` | Sí | Mecanismo de verificación previo a la firma. Ejemplo visto: `LINK_ONLY` (basta con abrir el link). **TODO: documentar otros (`OTP_SMS`, `OTP_EMAIL`, `BIOMETRIC`, etc.).** |

> **Nota:** los valores de los enums listados arriba son los observados en el ejemplo de referencia. **Coordinar con el equipo Firmoyo el catálogo completo** antes de habilitar nuevos flujos.

### 5.5 Múltiples firmantes (firma en paralelo)

Un envelope puede tener **N recipients** y todos firman **en paralelo**: cada uno recibe su link de firma de forma independiente y puede firmar en cualquier orden. El envelope queda en estado terminal "firmado" cuando **todos** los recipients completaron su firma.

Ejemplo de envelope con dos firmantes:

```json
{
  "templateCode": "b45f9bcd-f061-4132-8836-a0b79590c7b4",
  "entries": [
    {
      "envelopeName": "Contrato bilateral",
      "externalId": "CONTRATO-2026-000124",
      "messageMail": "Firma el contrato adjunto",
      "companyEmail": "company@empresa.com",
      "data": { "nroContrato": "C-000124" },
      "recipients": [
        {
          "userType": "CI-PY",
          "fullName": "Juan Carlos Benitez",
          "userValue": "5333186",
          "deliveryType": "EMAIL",
          "mail": "juan@example.com",
          "signType": "SIMPLE_SIGN",
          "verificationType": "LINK_ONLY"
        },
        {
          "userType": "CI-PY",
          "fullName": "María González",
          "userValue": "4221177",
          "deliveryType": "EMAIL",
          "mail": "maria@example.com",
          "signType": "SIMPLE_SIGN",
          "verificationType": "LINK_ONLY"
        }
      ]
    }
  ]
}
```

> **Importante:** la firma es **siempre paralela**. No hay soporte (al día de hoy) para firma secuencial con orden definido. Si el flujo de negocio lo requiere, debe orquestarse desde el lado del consumidor (crear el envelope del segundo firmante recién cuando el primero firmó, polling mediante).

---

## 6. Response

### 6.1 Estructura

```json
{
  "totalRequested": 1,
  "successCount": 1,
  "failureCount": 0,
  "templateCode": "b45f9bcd-f061-4132-8836-a0b79590c7b4",
  "results": [
    {
      "success": true,
      "envelopeCode": "f2eaa248-a6bf-4e24-a22a-2c53533a942f",
      "externalId": "CONTRATO-2026-000123",
      "errorMessage": null,
      "errorCode": null,
      "envelopeName": "Bases y condiciones"
    }
  ],
  "idempotencyKey": "20d93d96-2d94-421f-a00b-929dfe84a413"
}
```

### 6.2 Campos de respuesta

| Campo | Tipo | Descripción |
|---|---|---|
| `totalRequested` | `int` | Cantidad de entries enviados en el request. |
| `successCount` | `int` | Cantidad de envelopes creados con éxito. |
| `failureCount` | `int` | Cantidad de envelopes que fallaron. |
| `templateCode` | `string` (UUID) | Eco del template usado. |
| `results` | `array<Result>` | Resultado individual por cada entry, **en el mismo orden** del request. |
| `idempotencyKey` | `string` (UUID) | Eco de la key usada en el header. |

### 6.3 Objeto `Result`

| Campo | Tipo | Descripción |
|---|---|---|
| `success` | `boolean` | `true` si el envelope se creó y envió correctamente. |
| `envelopeCode` | `string` (UUID) \| `null` | ID del envelope en Firmoyo. **Persistirlo** para consultas posteriores y trazabilidad. `null` si `success=false`. |
| `externalId` | `string` \| `null` | Eco del `externalId` enviado en el request (si se envió). Permite correlacionar la respuesta con la entidad de negocio del consumidor sin necesidad de mantener un mapeo extra. `null` si no se envió en el request. |
| `errorMessage` | `string` \| `null` | Descripción del error cuando `success=false`. |
| `errorCode` | `string` \| `null` | Código de error machine-readable cuando `success=false`. **TODO: documentar catálogo de `errorCode`.** |
| `envelopeName` | `string` | Eco del `envelopeName` del request, útil para correlacionar. |

> **Importante:** este endpoint puede devolver **HTTP 200** aunque `failureCount > 0`. El consumidor **debe iterar `results`** y manejar errores parciales por cada entry.

---

## 7. Códigos HTTP esperados

| Status                     | Significado | Acción del cliente |
|----------------------------|---|---|
| `200 OK`                   | Request procesado. Revisar `successCount` / `failureCount` para resultados parciales. | Procesar `results` uno por uno. |
| `400 Bad Request`          | Body mal formado, JSON inválido o campo obligatorio faltante. | Revisar payload. **No reintentar** sin corregir. |
| `401 Unauthorized`         | API Key faltante o inválida. | Revisar header `X-RshkMichi-ApiKey`. **No reintentar.** |
| `(Falta)403 Forbidden`     | API Key sin permisos sobre el template o el affiliate. | Contactar al equipo Firmoyo. |
| `(Falta)404 Not Found`     | `templateCode` inexistente. | Validar el código contra el panel/listado. |
| `(Falta)409 Conflict`      | Conflicto de idempotencia (misma `Idempotency-Key` con body distinto). | Generar nueva `Idempotency-Key` o respetar el body original. |
| `(Falta)422 Unprocessable Entity` | Body válido pero datos inconsistentes (enums no soportados, claves faltantes para el template, etc.). | Revisar `data` y `recipients`. |
| `(Falta)429 Too Many Requests`    | Rate limit excedido. | Backoff exponencial con jitter. **TODO: confirmar límites.** |
| `5xx`                      | Error del lado de Firmoyo. | Reintentar con la **misma** `Idempotency-Key` y backoff. |

> **TODO:** confirmar con el equipo Firmoyo el listado oficial de status y el formato de respuesta de error global (cuando el request falla *antes* de procesar entries individuales).

---

## 8. Manejo de errores recomendado

Pseudocódigo de referencia:

```pseudo
response = POST(...)

if response.status in [400, 401, 403, 404, 409, 422]:
    log_error(response)
    raise NonRetryableError(response)

if response.status == 429 or response.status >= 500:
    schedule_retry(same idempotency_key, backoff_exponencial_con_jitter)
    return

# 200 OK
for entry_request, entry_result in zip(request.entries, response.results):
    if entry_result.success:
        persist(entry_request.business_id, entry_result.envelopeCode)
    else:
        record_failure(entry_request, entry_result.errorCode, entry_result.errorMessage)
```

**Reglas clave:**

1. **Reintentar siempre con la misma `Idempotency-Key`** ante errores transitorios (5xx, 429, timeouts de red).
2. **Nunca reintentar** ante 4xx semánticos (400, 401, 403, 404, 422) sin corregir el request.
3. Persistir `envelopeCode` apenas se reciba: es la única forma de trazar el envelope luego.
4. Tratar el resultado entry-por-entry: éxito a nivel HTTP **no implica** éxito a nivel envelope.

---

## 9. Consulta de estado del envelope (polling)

El servicio **no expone webhooks**: el consumidor debe consultar el estado del envelope vía polling.

> **TODO:** documentar el endpoint de consulta de estado (`GET /envelope/{envelopeCode}` o similar), los estados posibles (`PENDING`, `SIGNED`, `REJECTED`, `EXPIRED`, etc.) y la frecuencia recomendada de polling.

**Recomendación general** mientras se confirma:

- Polling con backoff: cada 1 min los primeros 10 min, luego cada 5 min hasta 1h, luego cada 30 min hasta TTL del envelope.
- Persistir el último estado conocido para evitar reprocesar transiciones ya vistas.
- Considerar un job programado (cron / scheduler) que recorra los envelopes en estado no terminal.

---

## 10. Ejemplos

### 10.1 cURL

```bash
curl -X POST "https://api-dev.firmoyo.digiyo.id/affiliate-firmoyo/affiliate/create-and-send-envelope-base-on-template" \
  -H "Content-Type: application/json" \
  -H "X-RshkMichi-ApiKey: ec9f4832ff33232a4de1@affiliate-firmoyo-do-not-use-in-production@34cd9e947729523b66h3" \
  -H "Idempotency-Key: 20d93d96-2d94-421f-a00b-929dfe84a413" \
  -d '{
    "templateCode": "b45f9bcd-f061-4132-8836-a0b79590c7b4",
    "entries": [
      {
        "envelopeName": "Bases y condiciones",
        "externalId": "CONTRATO-2026-000123",
        "messageMail": "Firma el documento",
        "companyEmail": "company@gmaidsl.ds",
        "data": {
          "nroSocio": "Nro43221",
          "cedula": "2311122",
          "nombreCompleto": "Juan Carlos Benitez",
          "domicilio": "Italia casi Fernando de la Mora"
        },
        "recipients": [
          {
            "userType": "CI-PY",
            "fullName": "Juan Carlos Benitez",
            "userValue": "5333186",
            "deliveryType": "EMAIL",
            "mail": "mail@gmail.com",
            "signType": "SIMPLE_SIGN",
            "verificationType": "LINK_ONLY"
          }
        ]
      }
    ]
  }'
```

### 10.2 Node.js (axios)

```javascript
import axios from "axios";
import { randomUUID } from "crypto";

const client = axios.create({
  baseURL: process.env.FIRMOYO_BASE_URL, // https://api-dev.firmoyo.digiyo.id
  headers: {
    "Content-Type": "application/json",
    "X-RshkMichi-ApiKey": process.env.FIRMOYO_API_KEY,
  },
  timeout: 15_000,
});

export async function createEnvelopes(payload, idempotencyKey = randomUUID()) {
  const { data } = await client.post(
    "/affiliate-firmoyo/affiliate/create-and-send-envelope-base-on-template",
    payload,
    { headers: { "Idempotency-Key": idempotencyKey } },
  );
  return data;
}
```

### 10.3 Java (OkHttp)

```java
OkHttpClient client = new OkHttpClient.Builder()
    .callTimeout(Duration.ofSeconds(15))
    .build();

Request request = new Request.Builder()
    .url("https://api-dev.firmoyo.digiyo.id/affiliate-firmoyo/affiliate/create-and-send-envelope-base-on-template")
    .addHeader("Content-Type", "application/json")
    .addHeader("X-RshkMichi-ApiKey", System.getenv("FIRMOYO_API_KEY"))
    .addHeader("Idempotency-Key", UUID.randomUUID().toString())
    .post(RequestBody.create(jsonPayload, MediaType.get("application/json")))
    .build();

try (Response response = client.newCall(request).execute()) {
    // Validar status, parsear body, iterar results
}
```

---

## 11. Ambientes y testing

| Ambiente | URL | Uso |
|---|---|---|
| **DEV** | `https://api-dev.firmoyo.digiyo.id` | Único ambiente disponible para integración y pruebas. |
| **PROD** | TODO: confirmar | Producción. |

> **Importante — no hay sandbox de testing automatizado.** Todas las llamadas al ambiente DEV **disparan envíos reales** por el canal indicado (ej.: emails reales al destinatario). Implicancias para el consumidor:
>
> - Usar **direcciones de mail propias y controladas** durante el desarrollo y pruebas (mailboxes del equipo, alias, mailtrap-like propios). **Nunca** usar mails de clientes reales en pruebas.
> - Considerar el costo / cuota de envíos al diseñar la batería de tests automatizados.
> - Para pruebas unitarias / CI: **mockear el cliente HTTP** y no llamar al endpoint real. Reservar las llamadas reales para tests de integración manuales o programados con baja frecuencia.
> - Coordinar con el equipo Firmoyo si se necesita resetear / limpiar envelopes de prueba en DEV.

---

## 12. Checklist de integración

- [ ] API Key de DEV cargada en variable de entorno.
- [ ] `Idempotency-Key` generada como UUID v4 y persistida junto al request lógico.
- [ ] `externalId` propio enviado en cada entry para correlación con la entidad de negocio.
- [ ] Manejo de 4xx vs 5xx implementado (no reintentar 4xx semánticos).
- [ ] Backoff exponencial con jitter en 5xx / 429.
- [ ] Iteración de `results` para detectar errores parciales.
- [ ] `envelopeCode` persistido por cada entry exitosa.
- [ ] Mapeo `externalId` ↔ `envelopeCode` persistido para conciliación.
- [ ] Logs sin filtrar la API Key.
- [ ] Job de polling para consultar estado de envelopes.
- [ ] Tests unitarios con cliente HTTP mockeado (no llamar al endpoint real).
- [ ] Tests de integración manuales contra DEV usando mailboxes propios.
- [ ] Plan de rotación de la API Key.

---

## 13. Glosario

| Término | Definición |
|---|---|
| **Envelope** | Sobre digital que contiene uno o varios documentos a firmar y la lista de firmantes. |
| **Template** | Plantilla preconfigurada en Firmoyo con el layout del documento, las variables a inyectar y la posición de las firmas. |
| **Affiliate** | Cliente integrador de Firmoyo identificado por su API Key. |
| **Idempotency-Key** | Identificador único de un request lógico, que permite reintentar de forma segura. |
| **Recipient** | Destinatario que debe firmar el envelope. |

---

## 14. Puntos pendientes (TODO)

Esta versión deja confirmados los puntos sobre `externalId`, firma en paralelo y disponibilidad de sandbox. Quedan abiertos los siguientes ítems, necesarios antes de pasar a producción:

1. URL del ambiente de **producción** (y staging si existe).
2. Catálogo completo de enums: `userType`, `deliveryType`, `signType`, `verificationType`.
3. Catálogo de `errorCode` (machine-readable) y `errorMessage`.
4. (Falta)Cantidad máxima de `entries` por request.
5. (Falta)Rate limits oficiales (req/seg, req/min, burst).
6. TTL de `Idempotency-Key` del lado del servidor.
7. Cómo se obtiene/lista el `templateCode`.
8. Variables esperadas por cada template (claves válidas para `data`).
9. Endpoint(s) de consulta de estado del envelope y estados posibles.
10. Cómo descargar el documento firmado una vez completado.
11. Política de expiración de envelopes (TTL del link de firma).
12. Soporte de reenvío / cancelación / reasignación de envelopes.
13. ¿Existe firma secuencial con orden definido o se confirma que solo hay paralela?
14. SLA del servicio y contacto de soporte/escalamiento.

---

## 15. Contacto

> **TODO:** completar con email/canal de soporte del equipo Firmoyo, owner técnico y horarios de atención.
