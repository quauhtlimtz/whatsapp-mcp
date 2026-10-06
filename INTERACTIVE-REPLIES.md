# Simulación de taps reales (respuestas interactivas) en whatsapp-mcp

Modificación local (2026-08-24) para que el MCP pueda **enviar respuestas interactivas reales** — es decir, simular que el usuario *tocó* una opción de una lista o un botón, no que escribió texto.

## Por qué

El MCP original solo enviaba `Conversation` (texto plano). Al probar bots con menús de WhatsApp:

- Un **tap real** entrega `metadata.list_reply.id` (ej. `intent_recargar`) y, como cuerpo visible, `título\ndescripción`.
- Mandar texto plano **no** entrega ese id, así que no se ejercitaba la extracción del id ni el emparejamiento por id — solo el respaldo por título.

## Qué se cambió

### 1. `whatsapp-bridge/main.go`
- `SendMessageRequest` acepta `reply_id`, `reply_type` (`list` | `button`) y `context_id`.
- `sendWhatsAppMessageEx()` construye `ListResponseMessage` o `ButtonsResponseMessage` con `ContextInfo`.
- `sendWhatsAppMessage()` se conserva como wrapper — nada del comportamiento previo cambia.

### 2. `whatsmeow-patched/` (copia parcheada de la librería)
⚠️ **Este es el cambio que hace que funcione.** whatsmeow adjunta un nodo `<biz>` al enviar respuestas de botón/lista; los clientes reales no lo hacen, y el servidor rechaza el stanza con **error 479**.

El parche (equivalente a [whatsmeow PR #1221](https://github.com/tulir/whatsmeow/pull/1221), aún abierto) elimina los casos de *response* en `getButtonTypeFromMessage()` (`send.go:983`) para que no se adjunte ese nodo.

Se enlaza vía `go.mod`:
```
replace go.mau.fi/whatsmeow => ../whatsmeow-patched
```

Si algún día se actualiza whatsmeow y el PR ya está mergeado, se puede quitar el `replace` y borrar la carpeta.

### 3. `whatsapp-mcp-server/{main.py,whatsapp.py}`
`send_message()` acepta `reply_id`, `reply_type` y `context_id` (todos opcionales — las llamadas existentes no cambian).

## Cómo usarlo

**`context_id` es obligatorio** cuando mandas `reply_id`: es el stanza id del mensaje interactivo original. Sin él, WhatsApp responde 479. Se obtiene decodificando el `wamid.` del mensaje saliente y tomando la corrida hexadecimal final.

```bash
curl -X POST http://localhost:8080/api/send -H 'Content-Type: application/json' -d '{
  "recipient": "15553668671",
  "message": "💳 Recargar mi servicio\nRecarga tu paquete de datos en línea",
  "reply_id": "intent_recargar",
  "reply_type": "list",
  "context_id": "57FDFF7F2B4934CEFE"
}'
```

El `message` debe ser la **etiqueta visible** tal como la pinta WhatsApp: `título` + salto de línea + `descripción` (si la fila tiene descripción). El ruteo lo determina `reply_id`, no el texto.

### Atajo: `simulate-tap.py`

Automatiza todo (resuelve el stanza id y arma la etiqueta desde la metadata del último mensaje interactivo):

```bash
python3 simulate-tap.py intent_recargar        # fila de lista
python3 simulate-tap.py si_genera button       # botón
```

Ajustar `ORG` y `PHONE` dentro del script para otro tenant/número.

## Verificado

Fila guardada en la BD tras un tap simulado — idéntica a la de un tap real:

```
17:23:28 | 💳 Recargar mi servicio \n Recarga tu paquete de datos en línea | intent_recargar
```

Probado también encadenando taps a través de varios turnos (menú → intent → opción dentro del flujo).

## Respaldos

`main.go.bak`, `go.mod.bak`, `whatsapp-client.bak`, `main.py.bak`, `whatsapp.py.bak` quedaron junto a cada archivo modificado.
