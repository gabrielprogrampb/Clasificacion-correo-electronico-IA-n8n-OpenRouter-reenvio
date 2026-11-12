# Flujo B [LISTO v8] - IA + email + log errores

Este flujo de trabajo de n8n está diseñado para procesar y gestionar contactos entrantes a través de un webhook. Incluye validación de datos, clasificación de mensajes por IA, almacenamiento en base de datos, envío de correos electrónicos y registro de ejecuciones y errores.

## Componentes del Flujo

El flujo se compone de los siguientes nodos:

1.  **Webhook (b1)**:
    *   **Tipo**: `n8n-nodes-base.webhook`
    *   **Método HTTP**: `POST`
    *   **Ruta**: `contacto-ia`
    *   **Descripción**: Recibe las solicitudes entrantes a través de un webhook.

2.  **Validar (b2)**:
    *   **Tipo**: `n8n-nodes-base.code`
    *   **Descripción**: Valida los datos recibidos del webhook. Verifica que `event_id` sea obligatorio, que el `nombre` tenga entre 2 y 80 caracteres, que el `email` tenga un formato válido, que el `mensaje` tenga al menos 10 caracteres y lo trunca si supera los 2000 caracteres. También añade un campo `origen` por defecto ('web') si no se proporciona y registra la fecha de recepción. El resultado de la validación se almacena en un campo `valido` (booleano) y los errores en un array `errores`.

3.  **Valido? (b3)**:
    *   **Tipo**: `n8n-nodes-base.if`
    *   **Descripción**: Bifurca el flujo basándose en el campo `valido` generado en el nodo "Validar". Si `valido` es verdadero, el flujo continúa por la rama "True" (hacia "Guardar contacto"); de lo contrario, va por la rama "False" (hacia "Log error validacion").

4.  **Guardar contacto (b4)**:
    *   **Tipo**: `n8n-nodes-base.httpRequest`
    *   **URL**: `https://TU_PROYECTO.supabase.co/rest/v1/contactos`
    *   **Método**: `POST`
    *   **Headers**: `apikey`, `Authorization` (Bearer Token), `Content-Type`, `Prefer: return=representation`. **¡Reemplazar `TU_SUPABASE_API_KEY` y `TU_PROYECTO`!**
    *   **Body**: Envía los datos del contacto (event_id, nombre, email, mensaje, origen, truncado) en formato JSON a la tabla `contactos` de Supabase.
    *   **Configuración**: Se reintenta en caso de fallo (`retryOnFail: true`, `maxTries: 3`, `waitBetweenTries: 2000`).

5.  **Clasificar con IA (b5)**:
    *   **Tipo**: `n8n-nodes-base.httpRequest`
    *   **URL**: `https://openrouter.ai/api/v1/chat/completions`
    *   **Método**: `POST`
    *   **Headers**: `Authorization` (Bearer Token), `Content-Type`. **¡Reemplazar `TU_OPENROUTER_API_KEY`!**
    *   **Body**: Envía una solicitud a la API de OpenRouter para clasificar el mensaje. Utiliza modelos gratuitos y tiene un sistema de fallback. El prompt instruye a la IA para que devuelva un JSON con `clasificacion` (urgente|normal|informativo) y `respuesta` (texto breve en español).
    *   **Configuración**: Se reintenta en caso de fallo (`retryOnFail: true`, `maxTries: 2`, `waitBetweenTries: 3000`). Si falla, continúa el flujo (`onError: "continueRegularOutput"`).

6.  **Validar salida IA (b6)**:
    *   **Tipo**: `n8n-nodes-base.code`
    *   **Descripción**: Procesa la respuesta de la IA. Parsea el JSON devuelto, valida que la `clasificacion` sea una de las permitidas ('urgente', 'normal', 'informativo') y extrae la `respuesta`. Si la IA falla o devuelve un formato incorrecto, se marca como `ia_degradada`. También registra información sobre el modelo utilizado y el uso de tokens. Combina esta información con los datos validados del nodo "Validar".

7.  **Enviar email (b7)**:
    *   **Tipo**: `n8n-nodes-base.httpRequest`
    *   **URL**: `https://api.resend.com/emails`
    *   **Método**: `POST`
    *   **Headers**: `Authorization` (Bearer Token), `Content-Type`. **¡Reemplazar `TU_RESEND_API_KEY` y `tu_correo@ejemplo.com`!**
    *   **Body**: Envía un correo electrónico de confirmación al usuario utilizando la API de Resend. El asunto incluye la clasificación del mensaje y el cuerpo contiene una respuesta breve generada por la IA.
    *   **Configuración**: Se reintenta en caso de fallo (`retryOnFail: true`, `maxTries: 2`, `waitBetweenTries: 2000`). Si falla, continúa el flujo (`onError: "continueRegularOutput"`).

8.  **Log ejecucion (b8)**:
    *   **Tipo**: `n8n-nodes-base.httpRequest`
    *   **URL**: `https://TU_PROYECTO.supabase.co/rest/v1/logs_ejecucion`
    *   **Método**: `POST`
    *   **Headers**: `apikey`, `Authorization` (Bearer Token), `Content-Type`. **¡Reemplazar `TU_SUPABASE_API_KEY` y `TU_PROYECTO`!**
    *   **Body**: Guarda un registro en la tabla `logs_ejecucion` de Supabase indicando el éxito de la ejecución y la información de la IA (modelo, tokens). Se marca como `ok_sin_ia` si la IA falló.

9.  **Responder 200 (b9)**:
    *   **Tipo**: `n8n-nodes-base.respondToWebhook`
    *   **Descripción**: Envía una respuesta `200 OK` al cliente que llamó al webhook, incluyendo información sobre el resultado de la IA.

10. **Responder 400 (b10)**:
    *   **Tipo**: `n8n-nodes-base.respondToWebhook`
    *   **Descripción**: Envía una respuesta `400 Bad Request` al cliente si la validación de datos falló, incluyendo los errores encontrados.

11. **Log error validacion (b11)**:
    *   **Tipo**: `n8n-nodes-base.httpRequest`
    *   **URL**: `https://TU_PROYECTO.supabase.co/rest/v1/logs_ejecucion`
    *   **Método**: `POST`
    *   **Headers**: `apikey`, `Authorization` (Bearer Token), `Content-Type`. **¡Reemplazar `TU_SUPABASE_API_KEY` y `TU_PROYECTO`!**
    *   **Body**: Guarda un registro de error en la tabla `logs_ejecucion` de Supabase cuando la validación falla, especificando el paso fallido y los mensajes de error.

## Configuración de Claves y URLs

Es **fundamental** reemplazar los siguientes valores de marcador de posición en los nodos correspondientes:

*   `TU_PROYECTO.supabase.co` (URL de Supabase)
*   `TU_SUPABASE_API_KEY` (API Key de Supabase)
*   `TU_OPENROUTER_API_KEY` (API Key de OpenRouter)
*   `TU_RESEND_API_KEY` (API Key de Resend)
*   `tu_correo@ejemplo.com` (Dirección de correo electrónico de destino para los correos)

## Conexiones

Las conexiones entre los nodos definen el flujo de datos:

*   `Webhook` -> `Validar`
*   `Validar` -> `Valido?`
*   `Valido?` (True) -> `Guardar contacto`
*   `Valido?` (False) -> `Log error validacion`
*   `Guardar contacto` -> `Clasificar con IA`
*   `Clasificar con IA` -> `Validar salida IA`
*   `Validar salida IA` -> `Enviar email`
*   `Enviar email` -> `Log ejecucion`
*   `Log ejecucion` -> `Responder 200`
*   `Log error validacion` -> `Responder 400`

Este README proporciona una visión general del funcionamiento y la configuración del flujo de trabajo. Asegúrate de configurar correctamente todas las credenciales y URLs antes de desplegarlo.
