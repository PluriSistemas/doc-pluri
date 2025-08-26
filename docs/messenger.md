# Integração com Messenger

[Voltar](../README.md)

A integração com o Messenger requer atenção, pois o processo é mais demorado e depende da aprovação da Meta. O aplicativo precisa passar por análise e revisão antes de ser utilizado em produção.


## Por onde começar?

Certifique-se de que o aplicativo esteja corretamente configurado na seção Configurações do App, incluindo todas as informações obrigatórias e o ícone no tamanho adequado, garantindo que tudo esteja em conformidade antes do envio para revisão da Meta.

![configurações do app](image-6.png)

Aplicativo deixado de exemplo chat_pluri em relação ao messenger pois tem a permissao de homologada 

[chat_pluri](https://developers.facebook.com/micro_site/url/?click_from_context_menu=true&country=BR&destination=https%3A%2F%2Fdevelopers.facebook.com%2Fapps%2F499918954710632&event_type=click&last_nav_impression_id=0ueonFCeiRH4CsLRP&max_percent_page_viewed=96&max_viewport_height_px=1362&max_viewport_width_px=2732&orig_http_referrer=https%3A%2F%2Fdevelopers.facebook.com%2Fapps%2F499918954710632%2Fsettings%2Fbasic%2F%3Fbusiness_id%3D3804389239686015&orig_request_uri=https%3A%2F%2Fdevelopers.facebook.com%2Fapps%2F&region=latam&scrolled=false&session_id=1FYrQHjrt14NwOvPU&site=developers)


1. Primeiro vc adiciona o produto  no seu aplicativo 

    ![Adiciona produto](image-1.png)

2. Configure o webhook de acordo com o DNS. No campo de verificação de token, utilize o valor informado na chave dentro de "Canais de Chat" do CRM. Esse processo é necessário para validar o webhook do cliente.

    ![Configuração do webhook](image-2.png)

3. Preencher as informações app name em canais de chat do CRM  

    ![App name](image-4.png)

4. Preencher as informações token em canais de chat do CRM  
    ![Gerar o token](image-3.png)


Para enviar e receber mensagens, solicite a permissão de pages_messaging.
![Faça analise ](image-5.png)


Exemplo de pedido revisao de aplicativo que deu certo

[Link](https://developers.facebook.com/apps/499918954710632/app-review/submissions/feedback/?submission_id=920029582699565&business_id=3804389239686015)