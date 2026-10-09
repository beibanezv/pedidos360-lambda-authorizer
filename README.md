# Pedidos360 — Validador de token del API Manager

Función **AWS Lambda authorizer** (tipo `REQUEST`, payload `2.0` simple) que valida JWT de
**dos proveedores de identidad al mismo tiempo** y decide si el API Manager deja pasar
una petición.

Es la pieza que permitió sumar el login con **AWS Cognito** sin romper el login con
**Azure AD (Entra ID)** que ya sostenía la evaluación 1.

---

## El problema que resuelve

El API Manager (`13uwepgzy9`, HTTP API) usaba el authorizer JWT **nativo** de API Gateway.
Ese authorizer acepta **un solo emisor y una sola audiencia**. Al llegar una segunda vía de
login (Cognito) con el mismo Gateway, pasaba una de estas dos cosas:

- El front manda un token de Cognito → el authorizer nativo lo rechaza con **401** (el
  checkout de la vía nueva quedaba roto).
- Se cambia el emisor del authorizer nativo al de Cognito → se rompe Azure, que es lo que
  sostiene la evaluación anterior.

Una Lambda propia resuelve las dos cosas: lee el claim `iss` del token, y según el emisor
aplica las reglas de ese proveedor.

**Orden de implementación que se respetó:** la Lambda validada primero, el login de Cognito
en el frontend después. Ese gate existe porque sin Lambda no hay segunda vía posible.

---

## Cómo funciona

```
Petición HTTP
   │  header Authorization: Bearer <token>
   ▼
Lambda (com.lambda.validador.AuthorizerHandler)
   │
   ├─ Lee el claim "iss" SIN verificar firma todavía (MultiIssuerJwtDecoder)
   │     └─ issuer no registrado → rechaza (403)
   │
   ├─ Busca el proveedor por emisor → valida firma contra su JWKS
   │  └─ validadores encadenados: fechas + emisor + audiencia
   │
   └─ ¿Algo falla en cualquier punto? → denied(). Nunca deja pasar por excepción.
```

Puntos de diseño que conviene poder explicar en la defensa:

- **Firma validada contra el JWKS de cada proveedor**, no contra uno fijo. Azure y Cognito
  publican sus llaves en URLs distintas.
- **`aud` o `client_id`**: los tokens de acceso de Cognito ponen la audiencia en `client_id`,
  no en `aud`. El validador acepta el valor en cualquiera de los dos.
- **Respuesta mínima `{ "isAuthorized": bool }`**, sin bloque `context`. El backend
  re-valida el token por su cuenta; copiar identidad al `context` la escribiría en los
  *access logs* del Gateway.
- **No usa `FunctionInvoker` de Spring Cloud Function**: ese envoltorio mete la respuesta en
  `{isBase64Encoded, statusCode, body}`, y API Gateway no encuentra `isAuthorized` al nivel
  superior → responde **500** en vez de autorizar. El handler escribe el JSON directo al
  `OutputStream`. (Está documentado en el javadoc de `AuthorizerHandler`.)

---

## Proveedores configurados

| Proveedor | Emisor (issuer) | JWKS | Audiencia |
|---|---|---|---|
| **Azure AD** | `https://sts.windows.net/e5372bf0-c5e3-4286-887c-79069f209c1f/` | `https://login.microsoftonline.com/e5372bf0-…/discovery/keys` | `api://446c57cb-aba2-4b7a-ab01-6f2e6af7d35c` |
| **AWS Cognito** | `https://cognito-idp.us-east-1.amazonaws.com/us-east-1_AU4jqskhP` | `https://cognito-idp.us-east-1.amazonaws.com/us-east-1_AU4jqskhP/.well-known/jwks.json` | `5ikmrldrcrkejq5d4v6bqcjo8f` |

Los valores viven en `src/main/resources/application.yaml` (bloque `app.security.jwt`) y se
pueden sobreescribir por variables de entorno:
`APP_SECURITY_JWT_PROVIDERS_0_JWKSETURI`, `_0_ISSUERS_0`, `_0_AUDIENCES_0`.

> **Azure en v1.0, no v2.0.** El material de clase traía el JWKS y el issuer en `v2.0`
> (`login.microsoftonline.com/<tid>/v2.0`). Los tokens reales de este tenant son **v1.0**
> (issuer `https://sts.windows.net/<tid>/`), y con la configuración v2.0 **ninguno habría
> validado**. Este fue el principal error de adaptación que había que corregir.

---

## Estado de verificación

> **Leer con honestidad**: esto está probado, esto no.

**Probado (39/39 tests, `mvn test`):**
- Token válido del proveedor correcto → autoriza.
- Token caducado, de otra API, firmado con llave ajena, emisor no registrado → rechaza.
- Audiencia en `aud` o en `client_id` → autoriza.
- Sin header `Authorization`, con prefijo `Bearer`/`bearer`, cabecera en minúsculas → rechaza.
- Los tests generan un par RSA y publican un JWKS real en `127.0.0.1`, así que validan la
  firma de verdad (no comparan cadenas).

**Falta verificar:**
- La **matriz completa en el Gateway con tokens reales de Azure**. Está probada solo con
  tokens reales de **Cognito** (sin token → 401, token inválido → 403, token válido → 200,
  `OPTIONS` → 200). La mitad con token Azure requiere login interactivo en Microsoft.
- El **redespliegue sobre la infra definitiva**: la cuenta de AWS Learner Lab se reseteó y la
  infra de la evaluación 1 (API de negocio, EC2, RDS) quedó destruida. Hay que reconstruirla
  y volver a adjuntar el authorizer.
- **Callback de producción en Cognito**: el cliente solo tiene registrado
  `http://localhost:4200/cognito/callback`. Falta registrar la URL desplegada
  (`https://4zg0frz1qg.execute-api.us-east-1.amazonaws.com/desarrollo/cognito/callback`)
  para que el login funcione fuera de local.

---

## Construir y desplegar

El build produce `target/function.jar` (fat jar con plugin *shade*). `aws-lambda-java-core`
va en scope `provided` porque el runtime ya lo carga: empaquetarlo hacía que el handler se
cargara dos veces y nunca se invocaba.

```bash
# Tests
mvn test

# Fat jar
mvn package -DskipTests

# Desplegar con SAM (usa template.yaml)
sam build
sam deploy --guided
```

Desplegada, se adjunta a las rutas del HTTP API como authorizer `REQUEST` con
`IdentitySource: $request.header.Authorization` y `AuthorizerPayloadFormatVersion: '2.0'`.
Las rutas `OPTIONS` (preflight de CORS) deben ir **sin** authorizer.

---

## Nota sobre `function.jar`

El repositorio base del curso commiteaba un `function.jar` de 26 MB. Aquí **no**: está en
`.gitignore` y se genera con `mvn package`. Comitear un binario de 26 MB en cada cambio de
código no tiene sentido.

**Origen del código:** adaptado sobre el repositorio público del material de clase
`https://github.com/j-camposar/validador-lambda` (guardado aquí por trazabilidad). Los cambios
propios son la configuración de los dos proveedores reales y el paso de Azure a v1.0.
