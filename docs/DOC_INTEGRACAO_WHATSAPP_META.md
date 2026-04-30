# Documentacao - Integracao WhatsApp Business API (Meta Oficial)

## Visao Geral

Integracao do sistema SisCRM com a API oficial do WhatsApp Business (Meta) para disparo de campanhas em lote utilizando templates aprovados com variaveis nomeadas.

O sistema permite:
- Listar templates aprovados diretamente da API do Meta
- Selecionar um template na tela de Campanhas
- Visualizar preview da mensagem com dados reais do mailing
- Disparar mensagens em lote substituindo variaveis do template pelos dados do mailing importado

---

## Arquitetura

```
[CRM - Formulario Campanha]
        |
        |  (1) Select2 AJAX busca templates aprovados
        v
[modulos/search.php] ---> [API Meta: GET /message_templates]
        |
        |  (2) Retorna templates com nome, idioma, corpo
        v
[Campo Template + Campo Mensagem (readonly) + Campo Preview]
        |
        |  (3) Ao salvar e ativar campanha
        v
[engine_whatsapp_meta.php] ---> [API Meta: POST /messages]
        |                            |
        |  (4) Para cada registro    |  (5) Envia template com
        |      do mailing            |      variaveis nomeadas
        v                            v
[sispbx.campanha_numero]    [WhatsApp do destinatario]
   status = 5 (sucesso)
   status = 4 (falha)
```

---

## Banco de Dados

### Colunas adicionadas em `sispbx.campanha`

```sql
ALTER TABLE sispbx.campanha ADD COLUMN id_canal4 INTEGER;
ALTER TABLE sispbx.campanha ADD COLUMN template VARCHAR(255);
ALTER TABLE sispbx.campanha ADD COLUMN prev_mensagem TEXT;
ALTER TABLE sispbx.campanha ADD COLUMN media_id VARCHAR(255);
ALTER TABLE sispbx.campanha ADD COLUMN dt_media_upload TIMESTAMP;
```

- `id_canal4`: ID do canal WhatsApp Meta (referencia `sischat.canal.id_canal`)
- `template`: nome do template aprovado no Meta (ex: "desenvolvimento")
- `prev_mensagem`: preview da mensagem com variaveis substituidas pelo primeiro registro do mailing
- `media_id`: cache do media_id retornado pelo Meta apos upload de arquivo (templates com header de midia)
- `dt_media_upload`: data/hora do upload do arquivo no Meta (usado para validar cache de 25 dias)

### Tabelas utilizadas (ja existentes)

| Tabela | Descricao |
|--------|-----------|
| `sischat.canal` | Credenciais do canal Meta (`app_nome` = WABA ID, `chave` = Phone Number ID, `token` = Bearer Token) |
| `sispbx.campanha` | Dados da campanha (tipo, status, datas, template, mensagem) |
| `sispbx.campanha_mailing` | Vinculo campanha <-> mailing importado (`id_importacao`) |
| `sispbx.campanha_numero` | Registros individuais para envio (telefone, status, tentativa) |
| `sispbx.campanha_horario` | Janelas de horario permitidas para disparo |
| `siscrm.importacao_reg` | Dados do mailing (nome, tel1, cpf, email, v1, etc.) |
| `sischat.mensagem_direta` | Log de mensagens enviadas |

---

## Arquivos Modificados

### 1. `siscrm/modulos/search.php`

Tres cases adicionados no switch:

#### Case `template`
- Recebe `id_canal` via GET/POST
- Chama `fncGetTemplatesMeta($id_canal)` que consulta a API do Meta: `GET https://graph.facebook.com/v22.0/{WABA_ID}/message_templates`
- Filtra templates com `status = 'APPROVED'`
- Retorna JSON no formato Select2: `{ results: [{ id, text, message }] }`
- O campo `message` contem o texto do body do template (usado para preview)

#### Case `template_preview`
Parametros aceitos (todos via POST):
- `mensagem` (opcional): texto do template com variaveis
- `template_name` (opcional): nome do template no Meta
- `id_canal` (opcional): ID do canal para buscar o template na API
- `id_campanha` (opcional): ID da campanha para buscar mailing vinculado

Fluxo:
1. Se `mensagem` estiver vazia mas `template_name` + `id_canal` forem passados, busca o texto do template na API do Meta via `fncGetTemplatesMeta()` (fallback usado quando a option do Select2 foi criada manualmente e nao tem o `message` no cache)
2. Se `id_campanha` estiver preenchido, busca o mailing vinculado em `sispbx.campanha_mailing`
3. Se encontrar mailing, pega o primeiro registro de `siscrm.importacao_reg` e substitui as variaveis via `fncSubstituirVariaveisTemplate()`
4. Se nao houver mailing, retorna a mensagem original com as variaveis cruas

Retorna JSON: `{ message: "texto bruto do template", preview: "texto com variaveis substituidas" }`

O endpoint devolve tanto `message` quanto `preview` para o frontend poder preencher os dois campos (Mensagem e Preview Mensagem) em uma unica chamada.

#### Case `template_saved`
- Recebe `id_campanha` via GET
- Busca o valor salvo de `template` em `sispbx.campanha` 
- Retorna JSON: `{ template: "nome_do_template" }`
- Usado para restaurar a selecao do Select2 ao reabrir uma campanha existente

### 2. `siscrm/lib/fnc.php`

Tres funcoes adicionadas apos `fncGetTemplatesMeta()`:

#### `fncMapaVariaveisMailing()`
- Retorna array associativo mapeando variavel do template para coluna da tabela `importacao_reg`
- Exemplo: `'v_nome' => 'nome'`, `'v_tel1' => 'tel1'`, `'v_cpf' => 'cpf'`
- Cobre todas as 246 variaveis do sistema (nome, telefones, documentos, enderecos, financeiro, campos customizados v1-v40, n1-n10, i1-i10, dt1-dt10, etc.)
- Centralizado para reuso em qualquer lugar do sistema

#### `fncSubstituirVariaveisTemplate($mensagem, $registro)`
- Recebe o texto do template e um array associativo com os dados do registro
- Substitui cada `{{v_XXX}}` pelo valor correspondente da coluna
- Usa o mapa de `fncMapaVariaveisMailing()`
- Retorna a mensagem final com variaveis substituidas
- Usado no preview e na gravacao da mensagem enviada

#### `fncUploadMidiaMeta($caminho_arquivo, $phone_id, $token, $tipo_mime = '')`
- Faz upload de arquivo de midia para o Meta via `POST /v22.0/{phone_id}/media`
- Usa `multipart/form-data` com `curl_file_create`
- Detecta automaticamente o `mime_type` se nao for passado (`mime_content_type()`)
- Retorna `['media_id' => 'xxx']` em sucesso ou `['error' => '...']` em falha
- O `media_id` retornado pode ser usado em ate 30 dias antes de expirar (cache de 25 dias na tabela `sispbx.campanha` por seguranca)
- Endpoints suportam:
  - **Imagem**: JPG, PNG (max 5MB)
  - **Documento**: PDF, DOC, DOCX, XLS, XLSX, PPT, PPTX, TXT (max 100MB)
  - **Video**: MP4, 3GP (max 16MB)
  - **Audio**: AAC, M4A, AMR, MP3, OGG (max 16MB)

#### `fncMontarBodyTemplateMeta($mensagem, $registro, $numero_destino, $nome_template, $language, $media_id, $header_type)`
- Monta o JSON body para envio via API do Meta
- Extrai variaveis do template via regex: `/\{\{(v_[a-zA-Z0-9_]+)\}\}/`
- Para cada variavel encontrada, busca o valor no registro usando o mapa
- Se `$media_id` for passado e `$header_type` for IMAGE/VIDEO/DOCUMENT, adiciona automaticamente o componente `header` com a midia
- Retorna array PHP pronto para `json_encode()` e envio via cURL
- Formato do body com header de midia:

```json
{
  "messaging_product": "whatsapp",
  "to": "5532988889999",
  "type": "template",
  "template": {
    "name": "desenvolvimento",
    "language": { "code": "pt_BR" },
    "components": [
      {
        "type": "header",
        "parameters": [
          { "type": "image", "image": { "id": "media_id_retornado_pelo_upload" } }
        ]
      },
      {
        "type": "body",
        "parameters": [
          { "type": "text", "parameter_name": "v_nome", "text": "Fulano" }
        ]
      }
    ]
  }
}
```

- Formato do body sem midia (somente body):

```json
{
  "messaging_product": "whatsapp",
  "to": "5532988889999",
  "type": "template",
  "template": {
    "name": "desenvolvimento",
    "language": { "code": "pt_BR" },
    "components": [
      {
        "type": "body",
        "parameters": [
          { "type": "text", "parameter_name": "v_nome", "text": "Fulano" },
          { "type": "text", "parameter_name": "v_tel1", "text": "5532988889999" },
          { "type": "text", "parameter_name": "v_cpf", "text": "111.111.111-00" },
          { "type": "text", "parameter_name": "v_email", "text": "email@teste.com" },
          { "type": "text", "parameter_name": "v_v1", "text": "valor customizado" }
        ]
      }
    ]
  }
}
```

### 3. `code_opt/engine_whatsapp_meta.php` (novo arquivo)

Engine de disparo em lote, localizado em `/opt/pluri/engine_whatsapp_meta.php`.

#### Fluxo de execucao

1. Carrega configuracoes do banco (`siscrm.config`)
2. Busca canais do tipo `whatsapp_meta` em `sischat.canal`
3. Para cada canal, busca campanhas ativas:
   - `tipo IN ('Whatsapp', 'Whatsapp Oficial')`
   - `status = '3'` (Em andamento)
   - `id_canal4` = ID do canal
   - `template IS NOT NULL`
   - Dentro da janela de data (dt_inicio/dt_fim)
   - Dentro do horario permitido (campanha_horario)
   - Com mailing vinculado e ativo (campanha_mailing.status = '3')
4. Para cada campanha, busca o proximo numero pendente:
   - `campanha_numero.status = '1'` (pendente)
   - `tentativa < campanha.tentativa` (maximo de tentativas)
   - Exclui registros com tabulacao "nao discar" ativa
   - Exclui registros com solicitacao em periodo de carencia
   - LIMIT 1 por execucao (processa um por vez em loop)
5. **Detecta o tipo de header do template** via `fncGetTemplatesMeta()` (retorna `header_type`: NONE, TEXT, IMAGE, VIDEO ou DOCUMENT)
6. **Gerencia cache de midia (se template tem header de midia + campanha tem arquivo):**
   - Verifica se `c.media_id` esta preenchido E `c.dt_media_upload` e menor que 25 dias atras
   - Se cache valido: reutiliza o `media_id` existente (evita upload desnecessario)
   - Se cache invalido/expirado: faz upload via `fncUploadMidiaMeta()`, recebe novo `media_id` e atualiza `sispbx.campanha`
7. Normaliza o telefone (`formatTel`) mantendo o digito 9 para celulares brasileiros
8. Monta o body via `fncMontarBodyTemplateMeta()` passando `media_id` e `header_type` (componente header com midia e adicionado automaticamente quando aplicavel)
9. Envia via POST para `https://graph.facebook.com/v22.0/{phone_number_id}/messages`
10. Se sucesso (HTTP 2xx + messages[0].id):
    - Grava em `sischat.mensagem_direta` com `scr = 'Campanha'`
    - Atualiza `campanha_numero.status = '5'` (sucesso)
11. Se falha:
    - Grava erro em `/opt/pluri/data_enginewhatsmeta_error.log`
    - Atualiza `campanha_numero.status = '4'` (falha)
12. Grava log em `/opt/pluri/data_enginewhatsmeta.log`
13. Aguarda `NODEJS_TIMEWHATSAPP` segundos e repete

#### Diferenca do engine Z-API (engine_whatsapp.php)

| Aspecto | Z-API (nao oficial) | Meta (oficial) |
|---------|---------------------|----------------|
| Endpoint | `api.z-api.io/instances/{chave}/token/{token}/send-text` | `graph.facebook.com/v22.0/{phone_id}/messages` |
| Autenticacao | Header `client-token` | Header `Authorization: Bearer` |
| Formato telefone | Sem o 9 (celular BR) | Com o 9 (celular BR) |
| Tipo mensagem | Texto livre | Template aprovado com variaveis nomeadas |
| Verificacao numero | Endpoint `/phone-exists` | Nao existe (envia direto) |
| Arquivos/midia | Base64 inline | Via components (nao implementado) |
| Variaveis | `%nome%`, `%cpf%` (posicional) | `{{v_nome}}`, `{{v_cpf}}` (nomeadas) |
| ID resposta | `response.messageId` | `response.messages[0].id` |

### 4. `code_opt/lib/fnc.php`

Adicionadas as mesmas 3 funcoes do `siscrm/lib/fnc.php`:
- `fncMapaVariaveisMailing()`
- `fncSubstituirVariaveisTemplate()`
- `fncMontarBodyTemplateMeta()`

Necessario porque o engine roda no contexto do `code_opt`, nao do `siscrm`.

### 5. `/var/www/app_server/index.js`

Duas alteracoes para integrar o engine ao loop do PM2:

#### Adicao do require (junto dos outros engines)
```javascript
var exec_enginewhatsappmeta = require('child_process').exec;
```

#### Adicao do loop de execucao (junto dos outros engines)
```javascript
var child_enginewhatsappmeta;
function execEngineWhatsappMeta(iEngineWhatsappMeta) {
  setTimeout(() => {
    child_enginewhatsappmeta = exec_enginewhatsappmeta(
      "cd /opt/pluri && /usr/bin/php -f engine_whatsapp_meta.php",
      function (error, stdout, stderr) {
        execEngineWhatsappMeta(0);
      }
    );
  }, parseInt(200));
}
setTimeout(() => {
  execEngineWhatsappMeta(0);
}, 2000);
```

Importante: o comando usa `cd /opt/pluri &&` antes do PHP porque o engine depende de `require_once 'lib/config.inc.php'` com caminho relativo.

---

## Configuracao do Modulo Campanhas (interface CRM)

### SQL Data
Adicionadas as colunas no SELECT principal:
```sql
r.id_canal4, r.template, r.prev_mensagem
```

### Campo Canal 4 (`id_canal4`)

O OnChange concentra toda a logica do fluxo. Responsabilidades:

**Inicializacao do Select2 do Template**
- Destroi o Select2 anterior (caso exista)
- Monta o `<select>` com option vazia + option da selecao salva (se houver)
- Inicializa Select2 AJAX apontando para `search.php?field=XXtemplate&id_canal=VALUE`
- Usa `processResults` para preservar o campo `message` no objeto Select2

**Restauracao da selecao salva**
- Se campanha ja existe (`id_campanha` preenchido), faz AJAX para `template_saved` e recebe o nome do template salvo
- Chama `aplicarTemplate(savedTemplate)` para restaurar a selecao

**Funcao `aplicarTemplate(savedTemplate)`**
- Adiciona option com `selected` ao `<select>` se ainda nao existir
- Aplica `.val(savedTemplate).trigger('change.select2')` para o Select2 reconhecer
- Aplica readonly + fundo cinza em `mensagem` e `prev_mensagem`
- Faz AJAX para `template_preview` passando `template_name`, `id_canal` e `id_campanha`
- Preenche Mensagem com `resp.message` e Preview com `resp.preview` retornados pelo backend
- E chamada 3 vezes (imediata, 1s, 2.5s) para garantir que o sistema de modulos nao sobrescreva a selecao/preview

**Re-inicializacao automatica (reinitInterval)**
- Um `setInterval` monitora se o Select2 do Template ainda esta configurado corretamente (possui AJAX url com `id_canal=`)
- Se detectar que o sistema recriou o Select2 com config default (ex: ao voltar de submodulo), dispara `trigger('change')` no Canal para re-inicializar
- O interval e limpo apos 15 segundos para nao rodar infinito

**Listener `click.refreshTemplate` no botao `#d-btn-close`**
- Intercepta cliques no botao "Fechar" de submodulos (Mailings, Horarios, Agendamentos)
- Usa namespaced event `click.refreshTemplate` com `.off()` antes do `.on()` para evitar listeners duplicados
- Apos 1s (tempo para o submodulo fechar), faz AJAX para `template_preview` atualizando Mensagem e Preview
- Resolve o cenario: usuario seleciona template, entra em Mailings, vincula mailing, clica Fechar, e o Preview atualiza automaticamente com os dados do mailing recem-vinculado

**Readonly do campo Mensagem**
- Aplica readonly quando `this.value == '4'` (valor especifico)
- Remove readonly quando o Canal e limpo

### Campo Template (`template`)

O OnChange do Template fica simples, ja que a logica principal (AJAX template_preview) esta no `aplicarTemplate()` do OnChange do Canal. Principais ações:

- Le dados via `$('#template').select2('data')`
- Faz AJAX para `template_preview` passando `template_name`, `id_canal`, `id_campanha` e `mensagem` (se disponivel via Select2)
- Preenche `mensagem` e `prev_mensagem` com a resposta do backend
- Aplica readonly em ambos os campos

Esse OnChange dispara sempre que o usuario troca o template, garantindo que a mensagem correta seja carregada mesmo que a option tenha sido criada manualmente (sem `message` no Select2).

### Campo Mensagem (`mensagem`)
- Componente: Textarea
- Readonly quando template selecionado (via JS)
- Contem o texto do template com variaveis (`{{v_nome}}`, `{{v_tel1}}`, etc.)

### Campo Preview Mensagem (`prev_mensagem`)
- Componente: Textarea
- Sempre readonly quando template selecionado
- Contem o texto do template com variaveis substituidas pelos dados do primeiro registro do mailing

---

## Formato das Variaveis

As variaveis do template seguem o padrao `{{v_CAMPO}}` onde `CAMPO` corresponde a coluna da tabela `siscrm.importacao_reg`.

### Exemplos de mapeamento

| Variavel no template | Coluna no banco | Descricao |
|---------------------|-----------------|-----------|
| `{{v_nome}}` | `nome` | Nome do contato |
| `{{v_nome_fantasia}}` | `nome_fantasia` | Nome fantasia |
| `{{v_cpf}}` | `cpf` | CPF |
| `{{v_cnpj}}` | `cnpj` | CNPJ |
| `{{v_email}}` | `email` | E-mail |
| `{{v_tel1}}` | `tel1` | Telefone 1 |
| `{{v_tel2}}` a `{{v_tel20}}` | `tel2` a `tel20` | Telefones 2 a 20 |
| `{{v_cep}}` | `cep` | CEP |
| `{{v_endereco}}` | `endereco` | Endereco |
| `{{v_cidade}}` | `cidade` | Cidade |
| `{{v_uf}}` | `uf` | UF |
| `{{v_v1}}` a `{{v_v40}}` | `v1` a `v40` | Campos texto customizados |
| `{{v_n1}}` a `{{v_n10}}` | `n1` a `n10` | Campos numericos |
| `{{v_dt1}}` a `{{v_dt10}}` | `dt1` a `dt10` | Campos data |
| `{{v_i1}}` a `{{v_i10}}` | `i1` a `i10` | Campos inteiros |

Total: 246 variaveis mapeadas.

---

## API Meta WhatsApp - Endpoints Utilizados

### Listar templates aprovados
```
GET https://graph.facebook.com/v22.0/{WABA_ID}/message_templates
Authorization: Bearer {TOKEN}
```
- `WABA_ID` = `sischat.canal.app_nome`
- Retorna todos os templates da conta
- Filtrados no PHP por `status === 'APPROVED'`

### Enviar mensagem de template
```
POST https://graph.facebook.com/v22.0/{PHONE_NUMBER_ID}/messages
Authorization: Bearer {TOKEN}
Content-Type: application/json
```
- `PHONE_NUMBER_ID` = `sischat.canal.chave`
- Body gerado por `fncMontarBodyTemplateMeta()`
- Usa `parameter_name` (variaveis nomeadas, nao posicionais)

### Upload de midia
```
POST https://graph.facebook.com/v22.0/{PHONE_NUMBER_ID}/media
Authorization: Bearer {TOKEN}
Content-Type: multipart/form-data
```
- `PHONE_NUMBER_ID` = `sischat.canal.chave`
- Body multipart com campos: `messaging_product=whatsapp`, `file=<arquivo>`, `type=<mime_type>`
- Retorna `{ "id": "media_id" }` em sucesso
- O `media_id` retornado e valido por aproximadamente 30 dias no Meta
- Usado por `fncUploadMidiaMeta()` para upload de imagem/video/documento que sera referenciado no header do template

---

## Envio de Arquivos (Templates com Header de Midia)

### Como funciona

O Meta permite que templates aprovados tenham um componente de **header com midia** (imagem, video ou documento). Para enviar um template com midia, o sistema usa o seguinte fluxo:

1. **Detectar tipo de header**: ao chamar `fncGetTemplatesMeta()`, retorna o `header_type` do template (NONE, TEXT, IMAGE, VIDEO ou DOCUMENT)
2. **Upload do arquivo**: usar `fncUploadMidiaMeta()` para enviar o arquivo binario via `POST /v22.0/{phone_id}/media` (multipart/form-data)
3. **Receber media_id**: o Meta retorna um identificador unico do arquivo
4. **Referenciar no template**: passar o `media_id` para `fncMontarBodyTemplateMeta()`, que monta automaticamente o componente `header` no body

### Cache de media_id

Para evitar fazer upload do mesmo arquivo a cada disparo, o sistema usa as colunas `media_id` e `dt_media_upload` na tabela `sispbx.campanha`:

- **Antes de enviar**: o engine verifica se ja existe `media_id` valido (menor que 25 dias)
- **Cache valido**: reutiliza o `media_id` existente
- **Cache invalido/inexistente**: faz upload novo e atualiza `media_id` + `dt_media_upload`

> **Por que 25 dias?** O Meta garante validade de 30 dias para o `media_id`. Usamos 25 dias como margem de seguranca para evitar erros de mensagem com midia expirada.

### Fluxo no engine_whatsapp_meta.php

```php
// 1. Detecta header_type via fncGetTemplatesMeta
$rtnTpl = fncGetTemplatesMeta($_c_id_canal4);
foreach ($rtnTpl['templates'] as $tplInfo) {
    if ($tplInfo['text'] === $_c_template) {
        $_c_header_type = $tplInfo['header_type'] ?? 'NONE';
        break;
    }
}

// 2. Se header tem midia + campanha tem arquivo, garante media_id valido
if (in_array($_c_header_type, ['IMAGE', 'VIDEO', 'DOCUMENT']) && !empty($_c_id_arquivo)) {

    // Cache valido?
    $diff_days = (strtotime('now') - strtotime($_c_dt_media_upload)) / 86400;
    if (!empty($_c_media_id) && $diff_days < 25) {
        $_c_media_id_uso = $_c_media_id;
    } else {
        // Faz upload novo
        $upload = fncUploadMidiaMeta($caminho, $phone_id, $token);
        if (!empty($upload['media_id'])) {
            $_c_media_id_uso = $upload['media_id'];
            // Atualiza cache no banco
            $dbsys->Execute("UPDATE sispbx.campanha SET media_id = ?, dt_media_upload = NOW() WHERE id_campanha = ?",
                [$_c_media_id_uso, $_c_id_campanha]);
        }
    }
}

// 3. Monta body com media_id e header_type
$body = fncMontarBodyTemplateMeta(
    $_c_mensagem, $qcn->fields, $numero_destino,
    $_c_template, $_c_language,
    $_c_media_id_uso, $_c_header_type
);
```

### Tipos de midia suportados

| Tipo de header | Formatos | Tamanho maximo |
|----------------|----------|----------------|
| IMAGE          | JPG, PNG | 5 MB |
| VIDEO          | MP4, 3GP | 16 MB |
| DOCUMENT       | PDF, DOC, DOCX, XLS, XLSX, PPT, PPTX, TXT | 100 MB |
| AUDIO          | AAC, M4A, AMR, MP3, OGG | 16 MB |

### Tabela de fluxo (com vs sem midia)

| Cenario | Header Type | Media ID | Componente header no body? |
|---------|-------------|----------|---------------------------|
| Template texto puro | NONE ou TEXT | qualquer | Nao |
| Template com imagem + campanha sem arquivo | IMAGE | NULL | Nao (envio falha por faltar midia) |
| Template com imagem + campanha com arquivo + cache valido | IMAGE | reutilizado | Sim |
| Template com imagem + campanha com arquivo + cache expirado | IMAGE | regenerado via upload | Sim |

---

## Logs

| Arquivo | Descricao |
|---------|-----------|
| `/opt/pluri/data_enginewhatsmeta.log` | Log de execucao do engine (campanhas encontradas, numeros processados) |
| `/opt/pluri/data_enginewhatsmeta_error.log` | Erros de envio (cURL errors, respostas HTTP nao-2xx) |

---

## Fluxo Operacional (Passo a Passo)

### Configuracao inicial (unica vez)
1. Cadastrar canal do tipo `whatsapp_meta` em `sischat.canal` com `app_nome` (WABA ID), `chave` (Phone Number ID) e `token` (Bearer Token)
2. Criar templates no Meta Business Manager com variaveis nomeadas (`{{v_nome}}`, `{{v_tel1}}`, etc.)
3. Aguardar aprovacao dos templates pelo Meta

### Criacao de campanha
1. No CRM, abrir modulo Campanhas
2. Selecionar Tipo: "Whatsapp Oficial"
3. Selecionar Canal 4 (canal Meta configurado)
4. Selecionar Template (lista carregada da API do Meta em tempo real)
5. O campo Mensagem sera preenchido automaticamente com o corpo do template (readonly)
6. Preencher demais campos (descricao, datas, equipe, etc.)
7. Salvar

### Vinculacao do mailing
1. Reabrir a campanha salva
2. Clicar no botao "Mailings"
3. Clicar em "Novo"
4. Selecionar a importacao desejada (mailing previamente importado em Importacoes)
5. Salvar

### Ativacao
1. Alterar status da campanha para "Em andamento" (status = 3)
2. Alterar status do mailing vinculado para "Em andamento" (status = 3)
3. Garantir que `status_exportacao = '2'` (exportacao completa)

### Disparo automatico
1. O engine (`engine_whatsapp_meta.php`) roda em loop continuo via PM2
2. A cada execucao, busca campanhas ativas dentro da janela de horario
3. Para cada campanha, pega 1 numero pendente
4. Envia o template com as variaveis preenchidas pelos dados do registro
5. Atualiza o status do numero (5 = sucesso, 4 = falha)
6. Repete ate nao haver mais numeros pendentes

---

## Observacoes Importantes

### Formato do telefone
- A API Meta exige o numero COM o digito 9 para celulares brasileiros (ex: `5532988037879`)
- Diferente da Z-API que exige SEM o 9 (ex: `553288037879`)
- O engine Meta usa `formatTel()` mas NAO remove o 9 (diferente do engine Z-API)

### Variaveis nomeadas vs posicionais
- A API Meta suporta dois formatos de variaveis: posicionais (`{{1}}`, `{{2}}`) e nomeadas (`{{v_nome}}`, `{{v_tel1}}`)
- Este sistema usa variaveis **nomeadas** (`parameter_name`) por ser mais robusto e legivel
- Ao criar templates no Meta Business Manager, usar nomes de variavel que correspondam ao mapa (`v_nome`, `v_tel1`, etc.)

### Token de acesso
- O token Bearer da API Meta tem validade limitada
- Quando expirar, atualizar em `sischat.canal.token`
- Recomendado usar System User Token (permanente) em vez de User Token (temporario)

### Idioma do template
- O sistema envia `language.code = 'pt_BR'` fixo
- Se precisar de multiplos idiomas, adicionar campo `language` na tabela `sispbx.campanha`

### Navegacao via submodulos (Mailings, Horarios, Agendamentos)
- O sistema de modulos do CRM trabalha com navegacao SPA (sem mudar a URL) ao abrir submodulos
- Ao clicar em "Mailings" dentro da campanha, a tela da campanha e ocultada e a tela do submodulo aparece
- Ao clicar em "Fechar" (`#d-btn-close`) no submodulo, a tela da campanha volta a aparecer — MAS nao dispara eventos como `onload` ou trocar URL
- Por isso o OnChange do Canal instala um listener em `#d-btn-close` para atualizar o preview automaticamente quando o usuario volta do submodulo (ex: apos vincular um mailing)
- O sistema tambem tem um `reinitInterval` que detecta quando o sistema recria os Select2 com config default apos voltar do submodulo

### Comportamento do Select2 AJAX ao reabrir campanha existente
- Quando a tela carrega com uma campanha salva, o sistema faz `.val('nome_do_template')` no Select2
- Como o Select2 AJAX nao tem a option carregada, o valor nao e exibido (mostra "selecione" vazio)
- Solucao: o `aplicarTemplate()` adiciona a option manualmente com `selected` antes de aplicar o valor
- Como a option e criada manualmente, ela nao tem o campo `message` que normalmente vem do AJAX do Select2
- Por isso o `template_preview` no backend busca o `message` via API Meta quando recebe `template_name` sem `mensagem`
