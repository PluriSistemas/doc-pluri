# Integração com WhatsApp (Meta)

A integração com o WhatsApp exige atenção, pois envolve várias etapas para ser concluída corretamente.
É essencial que o aplicativo tenha as permissões adequadas no ambiente **Meta** do cliente; caso contrário, a integração não funcionará.

---

## Por onde começar?

### 1. Adicionar o produto

No ambiente Meta, adicione o produto **WhatsApp** ao aplicativo.

![Adicionar](image-9.png)

---

### 2. Configuração da API e verificação do número

Após adicionar o produto, acesse a seção de **Configuração da API**.

* Adicione um número de telefone e realize o processo de verificação.
* **Atenção**: Caso o número seja adicionado sem cartão de crédito, não será possível realizar envios ativos, nem mesmo para testes.

![alt text](image-10.png)
![alt text](image-11.png)

---

### 3. Configuração do Webhook

Na aba **Configuração** (logo abaixo de Configuração da API), configure o Webhook.

* Utilize o **DNS do cliente** para definir a URL de callback.
* No campo de verificação de token, insira o valor presente na chave em **Canais de Chat** da plataforma CRM.

![alt text](image-12.png)

Ative as assinaturas para receber os eventos necessários via Webhook:

![alt text](image-13.png)

---

### 4. Criação de Token Definitivo

Para gerar o token permanente da **WhatsApp Business API**:

1. Acesse: **Gerenciar WhatsApp → Configurações → Usuários do Sistema**
2. Crie um novo usuário com privilégios de **Admin**.

![Configuração](image-15.png)
![Como criar o admin](image-16.png)

3. Após criar o usuário, adicione-o às funções do aplicativo utilizando o **ID**:

![Depois de criar usuário você precisa adicionar ele nas funções do aplicativo através do id ](image-17.png)
![Adicione em funções do app com privilégios de admin](image-18.png)

4. Em seguida, volte para **Gerenciar WhatsApp** e clique em **Gerar Token**.

   * Esse token deve ser inserido na plataforma, em **Canais de Chat → Campo Token**.
   * Caso solicitado, habilite as permissões:

     * `whatsapp_business_messaging`
     * `whatsapp_business_management`
     * `business_management`

5. Selecione os ativos necessários para que o WhatsApp funcione corretamente.

![Conta relacionada vai aparecer listada em relação ao usuario](image-19.png)

---

### 5. Finalização da Integração

Após concluir os passos acima, a seção de **Canais de Chat** para o WhatsApp Oficial ficará configurada conforme exemplo abaixo:

![alt text](image-14.png)
