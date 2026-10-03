# Traefik com Cloudflare e Docker Compose

Proxy reverso para centralizar o acesso às aplicações Docker, redirecionar HTTP para HTTPS e emitir certificados TLS automaticamente pelo Let's Encrypt usando a validação DNS-01 da Cloudflare.

Este projeto inicia o Traefik. Para publicar uma aplicação, conecte o contêiner à rede compartilhada e configure suas regras de acesso, conforme o exemplo abaixo.

## Sumário

- [Como funciona](#como-funciona)
- [Pré-requisitos](#pré-requisitos)
- [Instalação e configuração](#instalação-e-configuração)
- [Publicar uma aplicação](#publicar-uma-aplicação)
- [Painel do Traefik](#painel-do-traefik)
- [Operação e manutenção](#operação-e-manutenção)
- [Solução de problemas](#solução-de-problemas)

## Como funciona

O navegador acessa o domínio da aplicação nas portas `80` ou `443` do servidor. O Traefik recebe a requisição, aplica o redirecionamento para HTTPS quando necessário e encaminha o tráfego para a porta interna do contêiner na rede Docker compartilhada.

| Recurso | Configuração neste projeto |
| --- | --- |
| Imagem | `traefik:v3.7.13` |
| HTTP | Ponto de entrada `web`, na porta `80`, com redirecionamento permanente para HTTPS. |
| HTTPS | Ponto de entrada `websecure`, na porta `443`, com TLS habilitado por padrão. |
| Certificados | Resolvedor `cloudflare`, com emissão e renovação automáticas pelo Let's Encrypt. |
| Descoberta de aplicações | Provedor Docker; apenas contêineres com `traefik.enable=true` são habilitados. |
| Rede | Rede Docker externa, definida em `TRAEFIK_NET`. |
| Armazenamento | Pasta local `./letsencrypt`, montada em `/letsencrypt` no contêiner. |
| Painel | Habilitado internamente, sem rota de acesso configurada. |
| Logs | Logs da aplicação e de acesso em JSON; rotação do Docker em até três arquivos de 10 MB cada. |
| Verificação de saúde | Verificação interna a cada 30 segundos, com até três tentativas antes de indicar falha. |
| Reinício | Política `unless-stopped`. |

### Arquivos do projeto

```text
.
├── docker-compose.yml       # Configuração do Traefik
├── env.example              # Modelo das variáveis de ambiente
├── .env                     # Configuração local, criada durante a instalação
├── .gitignore               # Arquivos que não devem ser versionados
├── letsencrypt/
│   └── acme.json            # Conta ACME, certificados e chaves privadas
└── README.md
```

O `.env` e a pasta `letsencrypt/` são ignorados pelo Git. O protocolo ACME é usado pelo Traefik para solicitar e renovar certificados.

## Pré-requisitos

- Docker Engine e Docker Compose disponíveis no servidor pelo comando `docker compose`.
- Acesso ao Docker e ao socket `/var/run/docker.sock`, usado pelo Traefik para descobrir os contêineres.
- Um domínio com a zona DNS ativa na Cloudflare.
- Um token de API da Cloudflare com as permissões indicadas na próxima seção.
- Portas TCP `80` e `443` disponíveis no servidor e acessíveis aos usuários das aplicações. Se houver roteador ou NAT, configure o encaminhamento dessas portas.
- Acesso de saída por HTTPS à API da Cloudflare e ao Let's Encrypt, além de consultas DNS. O Compose utiliza os resolvedores `1.1.1.1` e `8.8.8.8`.

Confira a instalação das ferramentas:

```bash
docker --version
docker compose version
```

### Token da Cloudflare

Crie um token de API na Cloudflare com as permissões:

- `Zone / Zone / Read`: permite identificar a zona DNS.
- `Zone / DNS / Edit`: permite criar e remover os registros TXT da validação.

Restrinja o token às zonas dos domínios que serão atendidos pelo Traefik. Este Compose utiliza um único token, informado em `CF_DNS_API_TOKEN`, com ambas as permissões. Consulte a [documentação do provedor Cloudflare](https://go-acme.github.io/lego/dns/cloudflare/).

### DNS das aplicações

Crie um registro `A` para cada domínio de aplicação, apontando para o IPv4 do servidor. Configure um registro `AAAA` somente se o servidor também estiver acessível por IPv6.

Os exemplos deste README usam `app.seudominio.com.br`. Substitua esse endereço por um domínio sob seu controle.

A validação DNS-01 comprova o controle do domínio por meio de registros TXT. A emissão do certificado não depende de requisições HTTP ao servidor; as portas publicadas atendem ao acesso dos usuários. Veja a [documentação de certificados ACME](https://doc.traefik.io/traefik/reference/install-configuration/tls/certificate-resolvers/acme/).

## Instalação e configuração

Execute os comandos a seguir na pasta deste projeto.

### 1. Preencher o arquivo de ambiente

Em uma instalação nova, copie o modelo e restrinja a leitura do arquivo ao seu usuário:

```bash
cp env.example .env
chmod 600 .env
```

Se já existir um `.env`, edite o arquivo existente para preservar sua configuração.

Abra o `.env` no seu editor e substitua os valores de exemplo:

```dotenv
CF_DNS_API_TOKEN=seu_token_cloudflare
ADMIN_EMAIL=admin@seudominio.com.br
TRAEFIK_NET=traefik-net
```

| Variável | Obrigatória | Descrição |
| --- | --- | --- |
| `CF_DNS_API_TOKEN` | Sim | Token da Cloudflare usado na validação DNS-01. |
| `ADMIN_EMAIL` | Sim | E-mail para o cadastro da conta ACME no Let's Encrypt. |
| `TRAEFIK_NET` | Sim | Nome da rede externa compartilhada com as aplicações. |

As três variáveis não possuem valores padrão no Compose. A configuração será interrompida se alguma estiver ausente ou vazia. O Docker Compose carrega o `.env` automaticamente.

### 2. Criar a rede compartilhada

O nome da rede deve ser igual ao valor de `TRAEFIK_NET`. Para o valor usado no exemplo:

```bash
docker network create traefik-net
```

Se a rede já existir, reutilize-a. Você pode conferir sua configuração com:

```bash
docker network inspect traefik-net
```

O Compose declara essa rede como externa, por isso ela deve existir antes de iniciar o Traefik.

### 3. Preparar o armazenamento dos certificados

```bash
mkdir -p letsencrypt
touch letsencrypt/acme.json
chmod 600 letsencrypt/acme.json
```

O comando `touch` cria o arquivo caso ainda não exista, preservando o conteúdo de um arquivo existente. A pasta precisa permitir a escrita pelo contêiner.

O `acme.json` armazena a conta ACME, os certificados e suas chaves privadas. Preserve esse arquivo entre reinicializações e mantenha um backup protegido.

### 4. Validar e iniciar o Traefik

```bash
docker compose config --quiet
docker compose up -d
docker compose ps
```

O primeiro comando valida a configuração sem exibir o token. Depois da inicialização, o serviço deve aparecer em execução e, após as verificações internas, com o estado `healthy`.

Para acompanhar a inicialização:

```bash
docker compose logs --tail=100 -f traefik
```

Use `Ctrl+C` para sair da visualização dos logs; o contêiner continuará em execução.

Neste momento, o proxy está pronto para receber configurações de aplicações. O Traefik solicita certificados ao descobrir rotas HTTPS para os domínios configurados, sem depender do primeiro acesso de um usuário.

## Publicar uma aplicação

Crie um `compose.yml` em uma pasta separada para a aplicação. O exemplo abaixo publica a página padrão do Nginx em `https://app.seudominio.com.br`:

```yaml
services:
  site:
    image: nginx:stable-alpine
    restart: unless-stopped
    networks:
      - proxy
    labels:
      traefik.enable: "true"
      traefik.docker.network: "traefik-net"
      traefik.http.routers.site.rule: "Host(`app.seudominio.com.br`)"
      traefik.http.routers.site.entrypoints: "websecure"
      traefik.http.routers.site.service: "site"
      traefik.http.services.site.loadbalancer.server.port: "80"

networks:
  proxy:
    external: true
    name: traefik-net
```

Antes de iniciar, ajuste:

- **Domínio:** substitua `app.seudominio.com.br` pelo domínio cujo registro DNS você configurou.
- **Rede:** use o valor de `TRAEFIK_NET` tanto em `networks.proxy.name` quanto na etiqueta `traefik.docker.network`.
- **Porta interna:** troque `80` pela porta em que sua aplicação escuta dentro do contêiner.
- **Nomes:** use nomes exclusivos de roteador e serviço para cada aplicação nas etiquetas `traefik.http.routers` e `traefik.http.services`.

As etiquetas (`labels`) definem a rota: a regra `Host` identifica o domínio, o roteador seleciona as requisições e o serviço indica a porta interna de destino. Veja a [referência de roteamento Docker](https://doc.traefik.io/traefik/reference/routing-configuration/other-providers/docker/).

O exemplo herda TLS e o resolvedor `cloudflare` do ponto de entrada `websecure`, sem precisar repetir essas opções nas etiquetas. Consulte a [referência de pontos de entrada](https://doc.traefik.io/traefik/reference/install-configuration/entrypoints/).

Não é necessário adicionar `ports` à aplicação: o Traefik acessa o contêiner pela rede compartilhada.

Na pasta da aplicação, execute:

```bash
docker compose config --quiet
docker compose up -d
```

A emissão inicial do certificado pode levar alguns minutos devido à propagação DNS. Acompanhe os logs na pasta do projeto do Traefik.

### Conferir o acesso

Abra `https://app.seudominio.com.br` no navegador. Se você tiver o `curl` instalado, também poderá conferir as respostas:

```bash
curl -I http://app.seudominio.com.br
curl -I https://app.seudominio.com.br
```

A resposta HTTP deve redirecionar para HTTPS. A requisição HTTPS deve retornar uma resposta de sucesso da aplicação, com um certificado válido. A opção `-I` exibe apenas os cabeçalhos da resposta.

## Painel do Traefik

O painel de administração está habilitado com `api.dashboard=true`, mas o acesso direto está desabilitado por `api.insecure=false`.

O Compose atual publica apenas as portas `80` e `443` e não configura uma rota para o painel. Portanto, não há um endereço de acesso pronto, nem uma porta de painel publicada em `localhost`.

Para disponibilizar o painel, configure no serviço Traefik:

1. A etiqueta `traefik.enable=true` no próprio contêiner Traefik, caso a configuração seja feita por etiquetas Docker.
2. Uma rota HTTPS para um domínio de administração.
3. Uma regra que cubra os caminhos `/api` e `/dashboard`.
4. O serviço de destino `api@internal`.
5. Um mecanismo de autenticação, como o middleware `BasicAuth`.

O endereço termina em `/dashboard/`, com a barra final. Consulte os [exemplos oficiais de configuração do painel](https://doc.traefik.io/traefik/reference/install-configuration/api-dashboard/).

## Operação e manutenção

Execute os comandos de administração do proxy na pasta deste projeto.

| Ação | Comando |
| --- | --- |
| Validar a configuração | `docker compose config --quiet` |
| Iniciar ou aplicar alterações | `docker compose up -d` |
| Verificar estado e saúde | `docker compose ps` |
| Acompanhar os últimos logs | `docker compose logs --tail=100 -f traefik` |
| Reiniciar o serviço | `docker compose restart traefik` |
| Parar sem remover o contêiner | `docker compose stop` |
| Parar e remover o contêiner | `docker compose down` |

Após alterar o `.env` ou o `docker-compose.yml`, use `docker compose up -d` para aplicar a configuração. O comando `restart` apenas reinicia o contêiner existente e não atualiza suas variáveis de ambiente.

O comando `down` preserva a rede externa e a pasta local `letsencrypt/`. As aplicações iniciadas em outros projetos Compose continuam em execução, mas o acesso por este proxy fica indisponível enquanto ele estiver parado.

### Atualizar o Traefik

Escolha a versão desejada na propriedade `image` do `docker-compose.yml`. Em seguida:

```bash
docker compose config --quiet
docker compose pull traefik
docker compose up -d
docker compose ps
```

Confira os logs após a atualização. O comando `pull` baixa a versão definida no Compose; para mudar de versão, é necessário editar a etiqueta da imagem.

### Preservar a configuração e os certificados

Guarde cópias protegidas do `.env` e de `letsencrypt/acme.json`. Para obter uma cópia consistente do arquivo ACME, faça o backup durante uma parada breve do Traefik e inicie-o novamente em seguida.

Não publique o token ou as chaves privadas no repositório. O socket Docker é montado com `:ro`, mas o acesso à API do Docker continua sendo privilegiado; restrinja o acesso administrativo ao servidor.

## Solução de problemas

Comece pela validação da configuração e pelos logs:

```bash
docker compose config --quiet
docker compose ps
docker compose logs --tail=100 traefik
```

| Sintoma | O que verificar |
| --- | --- |
| Erro pedindo `ADMIN_EMAIL`, `CF_DNS_API_TOKEN` ou `TRAEFIK_NET` | Confirme que o `.env` existe na pasta do projeto e que as três variáveis estão preenchidas. |
| Rede externa não encontrada | Crie a rede com o nome definido em `TRAEFIK_NET`. Confira também a rede configurada no Compose da aplicação. |
| Porta já em uso | Verifique se outro serviço utiliza as portas `80` ou `443` do servidor. |
| Falha na emissão de certificado | Confira as permissões e o escopo do token, a zona DNS, a propagação dos registros TXT e o acesso de saída. Consulte a mensagem ACME nos logs. |
| Erro de permissão no `acme.json` | Confira a permissão `600` no arquivo e o acesso de escrita à pasta montada. |
| Certificado padrão ou inválido | Confira o domínio da rota e aguarde a emissão inicial. Se o problema persistir, consulte os erros de emissão nos logs. |
| Resposta `404` | Confira a regra `Host`, o ponto de entrada `websecure`, a etiqueta `traefik.enable=true` e se o contêiner está em execução. |
| Resposta `502` ou `504` | Confira a porta interna, a rede compartilhada e se a aplicação aceita conexões pela interface de rede do contêiner. |
| Tempo limite ao acessar o domínio | Confira os registros DNS, o firewall e o encaminhamento das portas, quando houver NAT. |
| Serviço com estado `unhealthy` | Consulte os logs do Traefik para identificar falhas na inicialização ou na verificação de saúde. |
| Painel inacessível | O projeto não configura uma rota de acesso ao painel. Siga a seção de configuração do painel para disponibilizá-lo. |
