
# Guia de Configuração: Azure API Management (APIM) e Segurança com JWT

Este guia descreve o processo passo a passo para publicação e proteção de uma API no Azure. O fluxo abrange desde o provisionamento do recurso e isolamento de rede até a implementação de camadas de segurança via Subscription Keys e Azure AD (JWT).
## 🛠️ 1. Provisionamento do API Management

A criação do serviço de APIM funciona como a nossa camada de abstração e gateway de entrada.

    Resource Group: Crie ou selecione um Resource Group dedicado para concentrar os recursos da solução.

    Marketplace: Pesquise por API Management.

    Configuração de Camada: Para ambientes de teste/desenvolvimento, selecione o Pricing Tier: Developer (focado em custo-benefício, sem SLA de disponibilidade).

⏳ Nota de Infraestrutura: O provisionamento inicial do APIM é um processo demorado. Pode levar de 30 a 45 minutos para concluir a ativação completa dos serviços de rede.

##  🔒 2. Isolamento do Backend (Web App via CORS)

Para garantir que o seu Web App só aceite requisições que venham estritamente através do Gateway do APIM, configure a política de CORS:

    No portal do Azure, acesse o seu App Service (Web App) de backend.

    No menu lateral esquerdo, navegue até CORS.

    Restrição de Origem:

        Remova o caractere curinga (*), se houver.

        Adicione apenas a URL de gateway do seu APIM (Ex: [https://seu-apim.azure-api.net](https://seu-apim.azure-api.net)).

💡 Resultado: Qualquer tentativa de requisição direta ao Web App vinda de outras origens será bloqueada pelo navegador/client.

##  🌐 3. Importação e Configuração da API

Com o gateway pronto, é hora de expor os endpoints do seu Web App.

    No recurso do APIM, acesse a aba APIs > APIs.

    Selecione a opção App Service para listar e importar automaticamente as definições do seu Web App.

    Customização do Header de Assinatura:

        Por padrão, o APIM utiliza o header Ocp-Apim-Subscription-Key.

        Para seguir as boas práticas de mercado, altere ou mapeie este nome para x-api-key nas configurações da API.

## 🔑 4. Gestão de Acesso (Subscriptions)

Para liberar o acesso externo à API para parceiros ou front-ends, precisamos gerar chaves de consumo.

    No menu do APIM, vá em Subscriptions e clique em Add Subscription.

    Defina o escopo desejado (pode ser restrito a uma única API ou global para o Produto).

    Após a criação, clique em "Show/Hide keys" e copie a Primary Key.

    A partir deste momento, qualquer requisição externa só será aceita se incluir o header configurado:
    HTTP

    x-api-key: d1a82b3c4e5f6g7h8i9j0k...


---

## 🛡️ 5. Camada de Segurança Avançada (JWT e Microsoft Entra ID)

Para cenários que exigem autenticação baseada em identidade e controle de acesso robusto, adicionamos a validação de tokens JWT.

### 5.1 App Registration
1. No **Microsoft Entra ID** (antigo Azure Active Directory), vá em **App Registrations** > **New Registration**.
2. Registre a sua aplicação e anote imediatamente os seguintes valores da tela de visão geral:
   * **Application (client) ID**
   * **Directory (tenant) ID**
3. Acesse a área de *Endpoints* da aplicação e copie a URL do **OpenID Connect metadata endpoint**.

### 5.2 Client Secret
1. Dentro do App Registration criado, vá no menu **Certificates & Secrets**.
2. Clique em **New Client Secret**, defina a expiração e clique em adicionar.
3. ⚠️ **Atenção:** Copie o valor do campo **Value** imediatamente. Ele será ocultado permanentemente após você sair dessa tela.

### 5.3 Configuração de Políticas no APIM
Para validar o token antes que ele chegue ao seu Web App, insira a política de validação no nó `<inbound>` da sua API no APIM:

```xml
<policies>
    <inbound>
        <base />
        <!-- Validação do Token JWT -->
        <validate-jwt header-name="Authorization" failed-validation-httpcode="401" failed-validation-error-message="Acesso não autorizado. Token inválido ou expirado.">
            <openid-config url="https://login.microsoftonline.com/{tenant-id}/v2.0/.well-known/openid-configuration" />
            <required-claims>
                <claim name="aud">
                    <value>{client-id}</value>
                </claim>
            </required-claims>
        </validate-jwt>
    </bound>
    <backend>
        <base />
    </backend>
    <outbound>
        <base />
    </outbound>
    <on-error>
        <base />
    </on-error>
</policies>
```
##  📋 Checklist de Credenciais

```
Certifique-se de que sua equipe de desenvolvimento possui as seguintes variáveis mapeadas antes de iniciar os testes:
Variável	Descrição do Item	Origem no Portal Azure
Gateway URL	O endereço público de consumo da sua API.	APIM > Overview
Subscription Key	A chave (x-api-key) para controle de cota e auditoria.	APIM > Subscriptions
Client ID	Identificador da aplicação usado como aud (Audience) no JWT.	Entra ID > App Registrations
Client Secret	Chave secreta usada para gerar os tokens OAuth2.	Entra ID > Certificates & Secrets
Tenant ID	Identificador único do diretório Azure da sua empresa.	Entra ID > Overview

```