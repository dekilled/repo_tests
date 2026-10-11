# Protocolo (v0, rascunho)

Transporte: WebSocket em `ws://<host>:8765/`. Mensagens JSON, uma por frame.

## Envelope
```json
{ "v": 0, "id": "uuid", "type": "<tipo>", "token": "<token>", "body": {} }
```
- `id`: gerado pelo cliente; a resposta usa o mesmo `id`.
- `token`: obrigatório em tudo, exceto `pair`.

## Pareamento
```json
→ { "type": "pair", "body": { "code": "482193", "device": "Pixel 8" } }
← { "type": "pair_ok", "body": { "token": "…", "agent": "meu-notebook", "os": "windows" } }
```

## Executar macro
```json
→ { "type": "run", "body": { "actions": [
      { "do": "open_url", "url": "https://www.netflix.com" },
      { "do": "wait", "ms": 3000 },
      { "do": "focus", "match": "Netflix" },
      { "do": "key", "keys": ["f"] }
] } }
← { "type": "ok" }   |   { "type": "error", "body": { "code": "…", "msg": "…" } }
```

## Ações
| `do` | Campos | Descrição |
|---|---|---|
| `open_url` | `url` | Abre URL/protocolo no handler padrão |
| `media` | `cmd`: `play_pause`/`next`/`prev`/`stop` | SMTC/MPRIS, fallback tecla de mídia |
| `volume` | `delta` (-100..100) ou `mute` | Volume do sistema |
| `key` | `keys` (ex. `["ctrl","w"]`) | Combinação de teclas |
| `text` | `text` | Digita texto |
| `focus` | `match` (substring do título) | Traz janela pra frente |
| `wait` | `ms` | Pausa |
| `mouse_move` | `dx`, `dy` | Movimento relativo (touchpad) |
| `mouse_click` | `button`: `left`/`right`/`middle` | Clique |
| `scroll` | `dy` | Rolagem |

## Atalhos
`media`, `key` e `mouse_*` também podem ser enviados como `type` direto
(sem `run`), para baixa latência no touchpad/D-pad:
```json
→ { "type": "mouse_move", "body": { "dx": 12, "dy": -4 } }
```

## Estado (agente → app, opcional)
```json
← { "type": "media_state", "body": { "app": "chrome", "title": "…", "playing": true } }
```
