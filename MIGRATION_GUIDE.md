# Chatplay Assistant — Guia de Migração: Userscript → Extensão

## Status atual: Fase A ✅

O script `Chatplay_Assistant_v9.1.0.user.js` está **totalmente preparado** para a migração.
Todas as abstrações necessárias já estão no código.

---

## Fase A (concluída) — Preparação no userscript

| O que foi feito | Onde |
|---|---|
| `StorageAdapter` com `GM_adapter` e `chrome_adapter` | M1/Storage |
| `_writeQueue` mutex FIFO | M1/Storage |
| `storageUpdate()` para operações RMW | M1/Storage |
| `storageGetMany()` para leitura em batch | M1/Storage |
| `carregarAppState()` async | M1/Storage |
| `AppState` declarado vazio, populado via `carregarAppState()` | Global |
| `inicializar()` async com await | M0/Bootstrap |
| `_inicializarOnChanged()` pronto (no-op em ENV=gm) | M1/Storage |
| 5 funções RMW protegidas pela fila | M6/M7/M8 |

---

## Fase B — Migrar para extensão Chrome/Edge

### Passo 1: Alterar STORAGE_ENV

```js
// Em Chatplay_Assistant_v9.1.0.user.js, linha ~145:
const STORAGE_ENV = "gm";   // ← alterar para:
const STORAGE_ENV = "chrome";
```

Isso ativa automaticamente o `chrome_adapter` em vez do `GM_adapter`.
**Nenhuma outra alteração nos módulos M2–M9 é necessária.**

### Passo 2: Substituir GM_xmlhttpRequest

Localizar a função `gerarRespostasIA` (M5/SuggestionEngine) e substituir:

```js
// ANTES (GM_*)
GM_xmlhttpRequest({
    method: "POST",
    url: "https://api.openai.com/v1/chat/completions",
    // ...
});

// DEPOIS (extensão — via background.js)
const response = await chrome.runtime.sendMessage({
    type: 'OPENAI_REQUEST',
    payload: { apiKey: CONFIG.OPENAI_KEY, messages, model: 'gpt-4o-mini' }
});
if (!response.ok) throw new Error(response.error);
return response.data;
```

### Passo 3: Remover cabeçalho UserScript

Remover todo o bloco `// ==UserScript== ... // ==/UserScript==`.
A extensão usa `manifest.json` + `content_scripts` em vez disso.

### Passo 4: Copiar como src/chatplay_core.js

```bash
cp Chatplay_Assistant_v9.1.0.user.js chatplay-extension/src/chatplay_core.js
```

Envolver o conteúdo em um módulo ES:
```js
// Adicionar no topo:
'use strict';
// Remover o IIFE externo (function() { ... })();
// Exportar inicializar:
export { inicializar };
```

### Passo 5: Atualizar content_script.js

```js
// Em src/content_script.js, linha de bootstrap:
import('./chatplay_core.js').then(mod => mod.inicializar());
```

---

## Fase C — Multi-aba e Sincronização

O listener `_inicializarOnChanged()` já está implementado em M1/Storage.
Ele é ativado automaticamente quando `STORAGE_ENV = "chrome"`.

**Comportamento:**
- `STORAGE_KEYS.SCORES`: merge por `Math.max` (Last-Write-Wins por campo)
- `STORAGE_KEYS.LOG_SUGESTOES`: merge por categoria com deduplicação por `id`
- `STORAGE_KEYS.STATS`: merge por `Math.max` nos contadores

---

## Estrutura da extensão

```
chatplay-extension/
├── manifest.json              ← MV3, permissões, content_scripts
├── popup.html + popup.js      ← UI do badge da extensão
├── _locales/pt_BR/            ← Internacionalização
└── src/
    ├── background.js          ← Service Worker: proxy OpenAI, broadcasts
    ├── content_script.js      ← Injeção na página, bridge de mensagens
    ├── storage_adapter.js     ← chrome_adapter + _writeQueue (Fase B)
    └── modules/               ← (Fase B+) módulos separados por arquivo
        ├── storage.js         ← M1
        ├── text_analysis.js   ← M2
        ├── chat_capture.js    ← M3
        ├── knowledge_base.js  ← M4
        ├── suggestion_engine.js ← M5
        ├── learning_engine.js ← M6
        ├── history_manager.js ← M7
        ├── chat_engine.js     ← M8
        └── ui.js              ← M9
```

---

## Riscos e mitigações

| Risco | Mitigação implementada |
|---|---|
| Lost Update (escrita concorrente) | `_writeQueue` + `storageUpdate()` |
| AppState vazio na UI | `carregarAppState()` antes de `criarBotaoPrincipal()` |
| Race condition multi-aba | `_inicializarOnChanged()` com merge LWW |
| CORS na chamada OpenAI | `background.js` como proxy |
| Service Worker efêmero (MV3) | Todo estado em `chrome.storage`, não em memória |
| Falha de escrita travar a fila | `.catch(console.error)` no `_writeQueue` |
