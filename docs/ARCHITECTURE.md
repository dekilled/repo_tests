# Arquitetura — Controle Remoto Android → Notebook

## Visão geral

```
┌──────────────┐  WebSocket + JSON (LAN)  ┌──────────────────────────────┐
│ App Android  │ ───────────────────────▶ │ Agente (core multiplataforma)│
│ Kotlin +     │ ◀─────────────────────── │  ├─ backend Windows          │
│ Compose      │   estado / respostas     │  └─ backend Linux (DE)       │
└──────────────┘                          └──────────────────────────────┘
```

O **protocolo** (`docs/protocol.md`) é o contrato. O app Android não sabe qual
SO está do outro lado; qualquer implementação que fale o protocolo serve —
inclusive uma implementação nativa dentro da DE Linux.

## Componentes

### 1. App Android (`/android`)
- Kotlin + Jetpack Compose.
- Telas: lista de dispositivos, controle (botões de macro, D-pad, mídia),
  touchpad, editor de macros.
- Descoberta via mDNS (`NsdManager`, serviço `_remotectl._tcp`) ou QR code.

### 2. Agente (`/agent`)
Core independente de SO + backends plugáveis.

| Responsabilidade | Core | Backend |
|---|---|---|
| Servidor WebSocket, pareamento, tokens | ✅ | |
| Anúncio mDNS | ✅ | |
| Execução de macros (sequência de ações) | ✅ | |
| Abrir URL / app | | ✅ |
| Controle de mídia (play/pause, next…) | | ✅ |
| Teclas / mouse simulados | | ✅ |
| Focar janela | | ✅ |

Linguagem do MVP: **Python** (rápido de iterar). O core pode ser reescrito
depois sem afetar o app, porque o protocolo é estável.

### 3. Extensão do navegador (`/extension`, opcional, fase 4)
Recebe comandos do agente (native messaging ou WebSocket local) e atua
no DOM da Netflix (selecionar título, pular abertura etc.).

## Backends

### Windows (primeiro alvo)
- **Mídia:** `GlobalSystemMediaTransportControlsSessionManager` (SMTC, via
  pacote `winrt`). Chrome/Edge expõem a sessão da Netflix ali, então
  play/pause funciona **sem foco de janela e sem simular tecla**.
- **Teclado/mouse:** `SendInput` (via `pynput`) — para `F` (fullscreen),
  setas, Enter, touchpad.
- **Foco de janela:** `pywin32` (`SetForegroundWindow`, com o truque do
  `AllowSetForegroundWindow`/Alt quando necessário).
- **Abrir:** `os.startfile("https://www.netflix.com")` ou `netflix:` para o
  app da Store.
- **Autostart:** atalho em `shell:startup` ou tarefa agendada; ícone na
  bandeja (`pystray`).

### Linux (integrado à DE)
- **Mídia:** MPRIS via D-Bus — o equivalente direto do SMTC; navegadores
  publicam a sessão da Netflix como player MPRIS.
- **Integração com a DE:** o agente roda como serviço D-Bus de sessão
  (ex.: `org.<de>.RemoteControl`) para a DE mostrar dispositivos pareados,
  notificações de conexão e um painel de configuração. Alternativa: a DE
  implementa o protocolo nativamente e dispensa o agente.
- **Entrada simulada no Wayland:** como a DE controla o compositor, ela pode
  expor um protocolo privilegiado de injeção de entrada só para o agente
  (ou usar `libei` / portal `RemoteDesktop`). Fallback X11: `XTest`.
- **Foco de janela:** API da própria DE / compositor.

## Modelo de interação

1. **Botões de macro** — cada botão é uma lista de ações (ver protocolo).
   Ex.: "Netflix fullscreen" = `open_url` → `wait 3s` → `focus` → `key f`.
2. **Mídia** — play/pause, próximo, volume (sempre via SMTC/MPRIS quando
   disponível; tecla como fallback).
3. **D-pad** — setas + Enter + Esc para navegar no catálogo.
4. **Touchpad** — movimento relativo do mouse, clique, scroll.
5. **Favoritos** — links diretos `netflix.com/watch/<id>` para "selecionar
   filme" de forma confiável.

## Segurança
- Só LAN; o agente escuta na interface local.
- Pareamento: agente mostra QR com `host`, `porta` e código de uso único;
  app troca o código por um token persistente.
- Toda mensagem após o pareamento carrega o token; tokens podem ser
  revogados no agente.

## Estrutura do repositório
```
/android          app controle (Kotlin)
/agent
  /core           servidor, pareamento, macros
  /backends
    windows.py
    linux.py
/extension        (fase 4)
/docs
  ARCHITECTURE.md
  protocol.md
```

## Roadmap
1. **MVP Windows:** conexão por IP manual; play/pause (SMTC), fullscreen,
   abrir Netflix.
2. Descoberta mDNS + pareamento por QR + token.
3. Editor de macros, D-pad, touchpad, favoritos.
4. Backend Linux + integração D-Bus com a DE.
5. Extensão do navegador; outros serviços (YouTube, Spotify).
