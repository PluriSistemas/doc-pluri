# Integração com Telegram

A integração com o **Telegram** é simples e rápida, permitindo que mensagens enviadas ao bot sejam recebidas e processadas diretamente pelo sistema.

---

## Como integrar?

1. Utilize o endpoint oficial da API do Telegram para configurar o webhook:

```
https://api.telegram.org/bot<SEU_TOKEN>/setWebhook?url=<URL_DO_WEBHOOK>&allowed_updates=["message","callback_query","inline_query"]
```

2. No caso deste projeto, a URL de exemplo é:

```
https://api.telegram.org/bot6009266991:AAHJLt-h1nP41UP3q-JRluqVhQN1OCCqweqw/setWebhook?url=https://crm.acessocloud.com/chat/app/webhook/telegram.php&allowed_updates=["message","callback_query","inline_query"]
```

3. Substitua:

   * `<SEU_TOKEN>` pelo token do seu bot.
   * `<URL_DO_WEBHOOK>` pela URL pública do seu webhook (url vai ter sempre webhook na url).


4. Abra a URL de configuração no navegador ou utilize uma requisição HTTP para definir o webhook.
---


## Tipos de mensagens suportadas

A integração suporta o recebimento e envio dos seguintes tipos de mensagens:

* **Áudio**
* **Arquivo**
* **Texto**
* **Imagem**
* **Vídeo**
* **Contato**
* **Localização**
* **Link**