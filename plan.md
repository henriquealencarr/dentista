# Alterações — 04/06/2026

## 1. Upload de imagem via webhook `alterar-dados` (FormData)

**Problema:** O botão de imagem no painel Editar estava enviando os dados do cliente em JSON, sem incluir o arquivo de imagem corretamente.

**O que foi feito:**
- Alterado `submitEditarForm()` para sempre usar `multipart/form-data` (via `FormData`) em vez de JSON
- A imagem (se selecionada) é anexada ao `FormData` como `file`
- Todos os campos do cliente são enviados em toda requisição: `clientName`, `phone`, `caseName`, `lawyer`, `status`, `deadline`, `currentImageUrl`, `currentProntuarioUrl`, `sessionId`, `action`
- `clearEditarImage()` movido para depois da resposta bem-sucedida (evitava perda de referência ao arquivo antes do envio)

---

## 2. Modal de upload de foto do chat → webhook `alterar-dados`

**Problema:** O modal flutuante de troca de foto (no chat) usava um webhook diferente (`upload-image_mesmofluxo`) e não enviava os dados existentes do cliente.

**O que foi feito:**
- `uploadImage()` redirecionado para `https://n8n.srv1497968.hstgr.cloud/webhook/alterar-dados`
- Payload completo incluído: todos os campos do cliente buscados de `allClients`, `currentImageUrl`, `currentProntuarioUrl`, `sessionId`, `action: 'editImage'`
- Dados buscados via lookup em `allClients` pelo `selectedClientName`

---

## 3. Supabase real-time — mensagens do chat em tempo real

**Problema:** As respostas do agente só apareciam no chat após recarregar a página.

**O que foi feito:**
- Adicionada dependência via CDN: `@supabase/supabase-js@2`
- Criada subscription em `n8n_chat_histories` via `postgres_changes` (evento `INSERT`)
- Mensagens do tipo `ai` são renderizadas automaticamente ao chegar, sem polling
- `pollForReply()` simplificado para fallback de timeout de 60s apenas

---

## 4. Correção: imagem aparecia como "🔗 View link" no chat após upload

**Problema:** Após atualizar a foto de um paciente, a resposta do agente incluía a URL da nova imagem do Airtable, mas ela aparecia como link em vez de imagem inline.

**Causa raiz:** O novo formato de URL do Airtable (`v5.airtableusercontent.com`) não contém o padrão `/attXXXXXX/` na path. O regex `/\/(att\w+)\//` em `tryFreshImage` não conseguia extrair o ID, caindo imediatamente no fallback de link.

**Iterações:**
1. Tentativa 1: adicionar retry de 3s com flag `_retryScheduled` — falhou porque o regex nunca chegava a bater
2. Tentativa 2: fila de pendentes (`pendingFreshImages`) + `flushPendingImages()` — falhou por race condition (flush rodava antes de `tryFreshImage` ser chamado, deixando elementos na fila sem novo flush agendado)
3. **Solução final:** `tryFreshImage` gerencia seu próprio loop de retry (até 3 tentativas: imediata, +2s, +4s). Em cada tentativa, tenta:
   - Lookup via `freshImageMap[attXXX]` (formato antigo)
   - Lookup via `lastUploadedClient` → `allClients[x].fields['Image'][0].url` (formato v5)
   - Só mostra o link após todas as tentativas falharem

**Variável `lastUploadedClient`:** definida em `uploadImage()` e `submitEditarForm()` após sucesso, limpa após 30s. Permite que `tryFreshImage` saiba qual cliente buscar no `allClients` recém-atualizado.

---

## Stack de referência

| Serviço | Detalhe |
|---|---|
| Airtable | Base `app0NITiJOMkdsr7k`, tabela `tblq2FbjhoVHW96pM` |
| n8n webhook (chat) | `.../webhook/7a0384de-70be-4b01-9141-977ba6f274d9` |
| n8n webhook (editar/imagem) | `.../webhook/alterar-dados` |
| Supabase real-time | tabela `n8n_chat_histories`, evento `INSERT` |
| Branch de desenvolvimento | `claude/admiring-noether-rb1Kb` |
