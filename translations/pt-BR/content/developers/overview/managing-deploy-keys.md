---
title: Gerenciar chaves de implantação
intro: Aprenda maneiras diferentes de gerenciar chaves SSH em seus servidores ao automatizar scripts de implantação e da melhor maneira para você.
redirect_from:
  - /guides/managing-deploy-keys
  - /v3/guides/managing-deploy-keys
versions:
  fpt: '*'
  ghes: '*'
  ghae: '*'
  ghec: '*'
topics:
  - API
---


Você pode gerenciar chaves SSH em seus servidores ao automatizar scripts de implantação usando o encaminhamento do agente SSH, HTTPS com tokens do OAuth, chaves de implantação ou usuários de máquina.

## Encaminhamento de agente SSH

Em muitos casos, especialmente no início de um projeto, o encaminhamento de agentes SSH é o método mais rápido e simples de utilizar. O encaminhamento de agentes usa as mesmas chaves SSH que o seu computador de desenvolvimento local.

#### Prós

* Você não tem que gerar ou monitorar nenhuma chave nova.
* Não há gerenciamento de chaves; os usuários têm as mesmas permissões no servidor e localmente.
* Não há chaves armazenadas no servidor. Portanto, caso o servidor esteja comprometido, você não precisa buscar e remover as chaves comprometidas.

#### Contras

* Os usuários **devem** ingressar com SSH para implantar; os processos de implantação automatizados não podem ser usados.
* Pode ser problemático executar o encaminhamento de agente SSH para usuários do Windows.

#### Configuração

1. Ativar o encaminhamento do agente localmente. Consulte o [nosso guia sobre o encaminhamento de agentes SSH][ssh-agent-forwarding] para obter mais informações.
2. Defina seus scripts de implantação para usar o encaminhamento de agentes. Por exemplo, em um script bash, permitir o encaminhamento de agentes seria algo como isto: `ssh -A serverA 'bash -s' < deploy.sh`

## Clonagem de HTTPS com tokens do OAuth

Se você não quiser usar chaves SSH, você poderá usar HTTPS com tokens OAuth.

#### Prós

* Qualquer pessoa com acesso ao servidor pode implantar o repositório.
* Os usuários não precisam alterar suas configurações SSH locais.
* Não são necessários vários tokens (um para cada usuário); um token por servidor é suficiente.
* Um token pode ser revogado a qualquer momento, transformando-o, basicamente, em uma senha de uso único.
{% ifversion ghes %}
* Gerar novos tokens pode ser facilmente programado usando [a API do OAuth](/rest/reference/oauth-authorizations#create-a-new-authorization).
{% endif %}

#### Contras

* Você deve certificar-se de configurar seu token com os escopos de acesso corretos.
* Os Tokens são, basicamente, senhas e devem ser protegidos da mesma maneira.

#### Configuração

Consulte [nosso guia sobre a criação de um token de acesso pessoal](/authentication/keeping-your-account-and-data-secure/creating-a-personal-access-token).

## Chaves de implantação

{% data reusables.repositories.deploy-keys %}

{% data reusables.repositories.deploy-keys-write-access %}

#### Prós

* Qualquer pessoa com acesso ao repositório e servidor é capaz de implantar o projeto.
* Os usuários não precisam alterar suas configurações SSH locais.
* As chaves de implantação são somente leitura por padrão, mas você pode lhes conferir acesso de gravação ao adicioná-las a um repositório.

#### Contras

* As chaves de implementação só concedem acesso a um único repositório. Projetos mais complexos podem ter muitos repositórios para extrair para o mesmo servidor.
* De modo geral, as chaves de implantação não são protegidas por uma frase secreta, o que a chave facilmente acessível se o servidor estiver comprometido.

#### Configuração

1. [Execute o procedimento `ssh-keygen`][generating-ssh-keys] no seu servidor e lembre-se o local onde você salvou o par de chaves da chave da rsa pública e privada.
2. No canto superior direito de qualquer página do {% data variables.product.product_name %}, clique na sua foto do perfil e, em seguida, clique em **Seu perfil**. ![Navegação para o perfil](/assets/images/profile-page.png)
3. Na sua página de perfil, clique em **Repositórios** e, em seguida, clique no nome do seu repositório. ![Link dos repositórios](/assets/images/repos.png)
4. No seu repositório, clique em **Configurações**. ![Configurações do repositório](/assets/images/repo-settings.png)
5. Na barra lateral, clique em **Implantar Chaves** e, em seguida, clique em **Adicionar chave de implantação**. ![Link para adicionar chaves de implantação](/assets/images/add-deploy-key.png)
6. Forneça um título e cole na sua chave pública.  ![Página da chave implantação](/assets/images/deploy-key.png)
7. Selecione **Permitir acesso de gravação**, se você quiser que esta chave tenha acesso de gravação no repositório. Uma chave de implantação com acesso de gravação permite que uma implantação faça push no repositório.
8. Clique em **Adicionar chave**.

#### Usar vários repositórios em um servidor

Se você usar vários repositórios em um servidor, você deverá gerar um par de chaves dedicado para cada um. Você não pode reutilizar uma chave de implantação para vários repositórios.

No arquivo de configuração do SSH do servidor (geralmente `~/.ssh/config`), adicione uma entrada de pseudônimo para cada repositório. Por exemplo:

```bash
Host {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}-repo-0
        Hostname {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}
        IdentityFile=/home/user/.ssh/repo-0_deploy_key

Host {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}-repo-1
        Hostname {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}
        IdentityFile=/home/user/.ssh/repo-1_deploy_key
```

* `Host {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}-repo-0` - O alias do repositório.
* `Hostname {% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}` - Configura o nome de host a ser usado com o alias.
* `IdentityFile=/home/user/.ssh/repo-0_deploy_key` - Atribui uma chave privada ao pseudônimo.

Em seguida, você pode usar o apelido do host para interagir com o repositório usando SSH, que usará a chave de deploy exclusiva atribuída a esse pseudônimo. Por exemplo:

```bash
$ git clone git@{% ifversion fpt or ghec %}github.com{% else %}my-GHE-hostname.com{% endif %}-repo-1:OWNER/repo-1.git
```

## Tokens do servidor para servidor

Se seu servidor precisar acessar repositórios em uma ou mais organizações, você poderá usar um aplicativo GitHub para definir o acesso que você precisa e, em seguida, gerar tokens de _escopo limitado_, _servidor para servidor_ a partir daquele aplicativo GitHub. Os tokens do servidor para servidor podem ter escopo de repositório único ou múltiplo e podem ter permissões refinadas. Por exemplo, você pode gerar um token com acesso somente leitura para o conteúdo de um repositório.

Uma vez que os aplicativos GitHub são um ator de primeira classe em  {% data variables.product.product_name %}, os tokens do servidor para servidor são dissociados de qualquer usuário do GitHub, o que os torna comparáveis aos "tokens de serviço". Além disso, tokens de servidor para servidor têm limites de taxa dedicados que escalam com o tamanho das organizações sobre as quais eles atuam. For more information, see [Rate limits for {% data variables.product.prodname_github_apps %}](/developers/apps/rate-limits-for-github-apps).

#### Prós

- Tokens com escopo limitado com conjuntos de permissões bem definidos e tempos de expiração (1 hora, ou menos se for revogado manualmente usando a API).
- Limites de taxa dedicados que crescem com a sua organização.
- Separados das identidades de usuários do GitHub para que não consumam nenhuma estação licenciada.
- Nunca concedeu uma senha. Portanto, não pode efetuar o login diretamente.

#### Contras

- É necessária uma configuração adicional para criar o aplicativo GitHub.
- Os tokens de servidor para servidor expiram após 1 hora. Portanto, precisam ser gerados novamente, geralmente sob demanda e usando código.

#### Configuração

1. Determine se seu aplicativo GitHub deve ser público ou privado. Se o seu aplicativo GitHub agir apenas nos repositórios da organização, é provável que você queira que ele seja privado.
1. Determine as permissões que o aplicativo GitHub exige, como acesso somente leitura ao conteúdo do repositório.
1. Crie seu aplicativo GitHub por meio da página de configurações da sua organização. Para obter mais informações, consulte [Criar um aplicativo GitHub](/developers/apps/creating-a-github-app).
1. Observe seu id `id` do aplicativo GitHub.
1. Gere e faça o download da chave privada do seu aplicativo GitHub e armazene-a com segurança. Para obter mais informações, consulte [Gerar uma chave privada](/developers/apps/authenticating-with-github-apps#generating-a-private-key).
1. Instale o aplicativo GitHub nos repositórios nos quais ele precisa agir. Opcionalmente você poderá instalar o aplicativo GitHub em todos os repositórios da sua organização.
1. Identifique o `installation_id` que representa a conexão entre o aplicativo GitHub e os repositórios da organização à qual ele pode acessar.  Cada aplicativo GitHub e par de organização tem, no máximo, um `installation_id` único. Você pode identificar este `installation_id` por meio de [Obter uma instalação da organização para o aplicativo autenticado](/rest/reference/apps#get-an-organization-installation-for-the-authenticated-app). Isto exige autenticação como um aplicativo GitHub usando um JWT. Para obter mais informações, consulte [Efetuar a autenticação como um aplicativo GitHub](/developers/apps/authenticating-with-github-apps#authenticating-as-a-github-app).
1. Gere um token de servidor para servidor usando o ponto de extremidade correspondente da API REST, [Crie um token de acesso de instalação para um aplicativo](/rest/reference/apps#create-an-installation-access-token-for-an-app). Isto exige autenticação como um aplicativo GitHub usando um JWT. Para obter mais informações, consulte [Efetuar a autenticação como um aplicativo GitHub](/developers/apps/authenticating-with-github-apps#authenticating-as-a-github-app) e [Efetuar a autenticação como uma instalação](/developers/apps/authenticating-with-github-apps#authenticating-as-an-installation).
1. Use este token de servidor para servidor para interagir com seus repositórios, seja por meio das APIs REST ou GraphQL, ou por meio de um cliente Git.

## Usuários máquina

Se o servidor tiver de acessar vários repositórios, você poderá criar uma nova conta em {% ifversion ghae %}{% data variables.product.product_name %}{% else %}{% data variables.product.product_location %}{% endif %} e anexar uma chave SSH que será usada exclusivamente para automatização. Como essa conta em {% ifversion ghae %}{% data variables.product.product_name %}{% else %}{% data variables.product.product_location %}{% endif %} não será usada por uma pessoa, ela é chamada denominada _usuário de máquina_. É possível adicionar o usuário máquina como [colaborador][collaborator] em um repositório pessoal (concedendo acesso de leitura e gravação), como [colaborador externo][outside-collaborator] em um repositório da organização (concedendo leitura, acesso gravação, ou administrador) ou como uma [equipe][team], com acesso aos repositórios que precisa automatizar (concedendo as permissões da equipe).

{% ifversion fpt or ghec %}

{% tip %}

**Dica:** Nossos [termos de serviço][tos] afirmam que:

> *Contas registradas por "bots" ou outros métodos automatizados não são permitidas.*

Isto significa que você não pode automatizar a criação de contas. Mas se você desejar criar um único usuário máquina para automatizar tarefas como scripts de implantação em seu projeto ou organização, isso é muito legal.

{% endtip %}

{% endif %}

#### Prós

* Qualquer pessoa com acesso ao repositório e servidor é capaz de implantar o projeto.
* Nenhum usuário (humano) precisa alterar suas configurações de SSH locais.
* Não são necessárias várias chaves; o adequado é uma por servidor.

#### Contras

* Apenas organizações podem restringir os usuários máquina para acesso somente leitura. Os repositórios pessoais sempre concedem aos colaboradores acesso de leitura/gravação.
* Chaves dos usuário máquina, como chaves de implantação, geralmente não são protegidas por senha.

#### Configuração

1. [Execute o procedimento `ssh-keygen`][generating-ssh-keys] no seu servidor e anexe a chave pública à conta do usuário máquina.
2. Dê acesso à conta de usuário máquina aos repositórios que deseja automatizar. Você pode fazer isso adicionando a conta como [colaborador][collaborator], como [colaborador externo][outside-collaborator] ou como uma [equipe][team] em uma organização.

## Leia mais
- [Configurar notificações](/account-and-profile/managing-subscriptions-and-notifications-on-github/setting-up-notifications/configuring-notifications#organization-alerts-notification-options)

[ssh-agent-forwarding]: /guides/using-ssh-agent-forwarding/
[generating-ssh-keys]: /articles/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent/#generating-a-new-ssh-key
[tos]: /free-pro-team@latest/github/site-policy/github-terms-of-service/
[collaborator]: /articles/inviting-collaborators-to-a-personal-repository
[outside-collaborator]: /articles/adding-outside-collaborators-to-repositories-in-your-organization
[team]: /articles/adding-organization-members-to-a-team
