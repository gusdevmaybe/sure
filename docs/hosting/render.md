# Implantar o Sure na Render

Este guia sobe o Sure na [Render](https://render.com) com o Blueprint em [`render.yaml`](../../render.yaml), na raiz do repositório. O Blueprint cria quatro recursos na região `oregon` e liga um ao outro pela rede privada:

- um serviço web em Docker, a partir do `Dockerfile` deste repositório
- um worker Sidekiq com a mesma imagem
- um PostgreSQL da Render
- um Key Value da Render (compatível com Redis)

O check de saúde do web é `GET /up`. O schema do banco é aplicado antes do processo atender tráfego: no web, o [`bin/docker-entrypoint`](../../bin/docker-entrypoint) roda `./bin/rails db:prepare` quando o comando é `./bin/rails server`; no worker, o `preDeployCommand` roda o mesmo `db:prepare` antes do Sidekiq. O Rails usa um advisory lock, então as duas execuções podem coincidir sem corromper o schema.

Não foi preciso alterar código da aplicação. O Puma escuta `0.0.0.0` e a porta `PORT` que a Render injeta. O `config/database.yml` lê `DB_HOST`, `DB_PORT`, `POSTGRES_USER`, `POSTGRES_PASSWORD` e `POSTGRES_DB`, e o Rails 8 mescla `DATABASE_URL` por cima desse arquivo: host, porta, usuário, senha e nome do banco vêm da URL. O Blueprint não tem propriedades separadas de host e porta para o Postgres, então a URL interna é o que a aplicação usa. `REDIS_URL` aponta para o Key Value.

## Custo mensal estimado

Valores de compute da Render em outubro de 2026, no workspace Hobby (a assinatura do workspace é US$ 0). O compute é cobrado por segundo, então um recurso apagado no meio do mês custa só o tempo em que ficou ligado.

| Recurso | Plano no Blueprint | Preço | Gratuito ou pago |
| --- | --- | --- | --- |
| Web `sure` | `free` (0,1 CPU, 512 MB) | US$ 0 | Gratuito. Adormece depois de um período sem acesso e não aceita disco persistente. |
| Worker `sure-worker` | `0.5c-512mb` (0,5 CPU, 512 MB) | US$ 7 | Pago. Worker não tem plano gratuito. Este é o menor. |
| Postgres `sure-db` | `free` (256 MB RAM, 1 GB) | US$ 0 | Gratuito. Expira 30 dias depois da criação. |
| Key Value `sure-redis` | `free` (25 MB, sem disco) | US$ 0 | Gratuito. Perde os dados quando reinicia. |
| **Total deste Blueprint** | | **cerca de US$ 7 por mês** | Só o worker entra na fatura. |

O workspace Hobby inclui 5 GB de banda de saída por mês (depois, US$ 0,15/GB) e 500 minutos de build (depois, US$ 5 por 1.000 minutos). A primeira build Docker instala gems e precompila assets, então consome uma parte grande desses minutos. Dois serviços compilam a mesma imagem.

### O que os planos gratuitos não sustentam

- O Postgres gratuito fica inacessível 30 dias depois de criado. Há 14 dias para mudar o plano antes da Render apagar o banco e os dados.
- O Key Value gratuito não grava em disco. Um restart perde a fila do Sidekiq. Os cron jobs voltam quando o worker sobe de novo; um job que estava no meio do caminho não volta.
- O web gratuito adormece. A primeira visita depois disso espera o container acordar, em geral cerca de um minuto.
- Uploads (extratos, imagens) usam o disco local em `/rails/storage`. No plano gratuito esse disco é efêmero: um deploy ou restart novo perde os arquivos, e o banco continua apontando para eles.

### Se quiser manter os dados

Troque os planos no `render.yaml` (ou no painel) e faça um novo deploy. Mudança de plano do worker só vale no próximo deploy, não num restart simples.

| Troca | Plano | Preço a mais | Para quê |
| --- | --- | --- | --- |
| Postgres durável | `0.1c-256mb` | US$ 6/mês | Não expira. Defina `diskSizeGB: 1` para ficar no 1 GB incluído. Cada GB extra custa US$ 0,30/mês. |
| Redis com persistência | `256mb` | US$ 10/mês | A fila sobrevive a restart. |
| Web sempre ligado | `0.5c-512mb` | US$ 7/mês | Não adormece. Aí o `preDeployCommand` passa a ser aceito, e um disco em `/rails/storage` custa US$ 0,25/GB/mês. |

Uma instância que não perde banco, fila nem uploads fica em torno de US$ 7 + US$ 6 + US$ 10 + US$ 7, mais o disco que você anexar. O Blueprint de propósito começa no mínimo que sobe.

O app cabe em 512 MB no uso do dia a dia (o guia de [Docker](docker.md) mediu cerca de 352 MB em regime). Não cabe gerar dados de demonstração nesse limite: o gerador roda no worker e estoura a memória. Suba o plano do worker, faça um deploy novo, carregue os dados e, se quiser, volte o plano.

## Antes de começar

- Uma conta na Render. A Render faz uma autorização de US$ 1 no cartão e estorna em seguida, mesmo quando o compute é gratuito.
- O `render.yaml` no branch que a Render vai buildar. O botão abaixo lê o branch padrão do repositório (`main`). Pelo painel dá para escolher outro branch.

Cotações e preços de mercado saem do Yahoo Finance (`EXCHANGE_RATE_PROVIDER` e `SECURITIES_PROVIDER`), como no `.env.example`, e não pedem chave. IA, e-mail e um domínio próprio são opcionais e ficam de fora do Blueprint.

## Implantar pelo botão

O botão só funciona depois que este `render.yaml` estiver no branch padrão.

[![Deploy to Render](https://render.com/images/deploy-to-render-button.svg)](https://render.com/deploy?repo=https://github.com/gusdevmaybe/sure)

1. Clique no botão e entre na Render.
2. Autorize o acesso ao repositório Git, se a Render pedir.
3. Confira o Blueprint. Devem aparecer `sure`, `sure-worker`, `sure-redis` e `sure-db`, todos em Oregon.
4. A Render pede `ONBOARDING_STATE` duas vezes (web e worker). Digite `open` nas duas. Os valores precisam ser iguais. `closed` e `invite_only` impedem criar a primeira conta pela tela de cadastro.
5. Aplique o Blueprint e espere os deploys. A primeira build Docker demora mais que um restart.

`SECRET_KEY_BASE` é gerado sozinho, uma vez, e os dois processos compartilham o mesmo valor. Não troque esse valor depois: os cookies assinados e a criptografia dos campos sensíveis dependem dele.

## Implantar pelo painel

1. No [Dashboard](https://dashboard.render.com), abra **New** e escolha **Blueprint**.
2. Conecte o repositório que contém este arquivo. Se for um fork, use o fork, não o upstream, a menos que o upstream já tenha o `render.yaml` no branch escolhido.
3. Selecione o branch.
4. Revise os quatro recursos e os planos da tabela de custo.
5. Preencha `ONBOARDING_STATE` com `open` no web e no worker.
6. Confirme. A Render cria o Postgres e o Key Value, builda a imagem duas vezes (web e worker) e sobe os processos.

O Postgres e o Key Value nascem com `ipAllowList` vazio: só os serviços desta rede privada conectam. Para abrir um `psql` do seu computador, inclua o seu IP na allow list do banco, no painel.

## Variáveis de ambiente

Definidas pelo Blueprint, sem você digitar:

| Variável | Origem |
| --- | --- |
| `SELF_HOSTED` | `true` |
| `RAILS_ENV` | `production` |
| `SECRET_KEY_BASE` | gerado (`generateValue: true`), compartilhado |
| `RAILS_ASSUME_SSL`, `RAILS_FORCE_SSL` | `true` (a Render termina o TLS) |
| `WEB_CONCURRENCY`, `RAILS_MAX_THREADS` | `1` e `3`, para caber em 512 MB |
| `EXCHANGE_RATE_PROVIDER`, `SECURITIES_PROVIDER` | `yahoo_finance` |
| `DATABASE_URL` | URL interna do Postgres |
| `POSTGRES_USER`, `POSTGRES_PASSWORD`, `POSTGRES_DB` | o mesmo banco |
| `REDIS_URL` | URL interna do Key Value |
| `APP_DOMAIN` | hostname `onrender.com` do serviço web |

### O que você precisa preencher

Só `ONBOARDING_STATE`, duas vezes, com o valor `open`.

Não há outras variáveis obrigatórias do `.env.example` para a instância subir. O Blueprint não declara as opcionais com `sync: false`: isso faria a Render exigir um valor para cada uma na criação, inclusive as que você não usa.

Opcionais, se quiser acrescentar depois em **Environment** (nos dois serviços, com o mesmo valor):

- `OPENAI_ACCESS_TOKEN`, e se precisar `OPENAI_MODEL` e `OPENAI_URI_BASE`. IA em 512 MB fica apertada; o guia de Docker recomenda pelo menos 1 GB.
- `SMTP_ADDRESS`, `SMTP_PORT`, `SMTP_USERNAME`, `SMTP_PASSWORD`, `SMTP_TLS_ENABLED` e `EMAIL_SENDER`, para redefinir senha e enviar relatórios. O cadastro da primeira conta não precisa de SMTP.
- `TWELVE_DATA_API_KEY`, se trocar os provedores de `yahoo_finance` para `twelve_data`.
- `WEBAUTHN_RP_ID` e `WEBAUTHN_ALLOWED_ORIGINS`, se for usar passkeys. Veja [WebAuthn](webauthn.md). Com domínio próprio, ajuste também `APP_DOMAIN`.
- Variáveis de Active Storage (`ACTIVE_STORAGE_SERVICE` e as chaves S3, R2 ou GCS do `.env.example`) se não quiser depender do disco local.

Não defina `PORT`. A Render escolhe a porta e o Puma já lê essa variável. `3000` faria o proxy perder o processo.

## Criar o primeiro usuário

1. No serviço `sure`, copie a URL `*.onrender.com`.
2. Abra no navegador. Se o web gratuito estiver dormindo, espere ele acordar. Um 502 logo no começo costuma ser isso, ou a build ainda em andamento. O check `/up` precisa responder 200.
3. Na tela de login, use o link para criar a conta.
4. Informe o e-mail e uma senha com pelo menos 8 caracteres, letra maiúscula, letra minúscula, um número e um caractere especial.
5. A primeira pessoa vira super admin da instância. O painel do Sidekiq em `/sidekiq` só existe para esse usuário.
6. Se não quiser outros cadastros abertos, entre em **Configurações → Self-Hosting → Onboarding** e mude para **Closed**. Dá para fazer o mesmo trocando `ONBOARDING_STATE` para `closed` nos dois serviços e fazendo um novo deploy. Feche só depois que a sua conta existir.

Sem SMTP configurado, "esqueci a senha" não envia e-mail. Guarde a senha dessa primeira conta.

## Operação

- Cada push no branch ligado dispara deploy dos dois serviços. Os dois precisam subir juntos quando mudar uma variável compartilhada, em especial `SECRET_KEY_BASE`.
- Logs do web e do worker ficam no painel de cada serviço. Falha de migration aparece no pre-deploy do worker ou no boot do web, antes do Puma escutar.
- O gerador de dados de demonstração não completa em 512 MB. Suba o plano do `sure-worker`, faça deploy, carregue os dados e reduza de novo se quiser.
- Para apontar um domínio seu, adicione o domínio no serviço web. A Render emite o certificado. Atualize `APP_DOMAIN` para esse hostname nos dois serviços.

## Quando algo não sobe

- **Health check em `/up` falha.** Veja o log do web. `db:prepare` roda dentro do entrypoint e, se o Postgres ainda não aceita conexão, o processo sai e a Render tenta de novo. Confira se `sure` e `sure-db` estão em Oregon.
- **Worker sobe e cai em seguida.** O `preDeployCommand` precisa conseguir falar com o Postgres e o Redis. Plano gratuito de Key Value cabe numa fila pequena; se encher, `noeviction` faz o Redis recusar escrita em vez de apagar job. O log mostra `OOM command not allowed`.
- **Login volta, mas sincronização e importação não andam.** O worker não está de pé, ou o `REDIS_URL` dos dois serviços divergiu. Os dois têm que usar o mesmo Key Value.
- **Cadastro não aparece.** `ONBOARDING_STATE` não está `open`, ou os dois serviços receberam valores diferentes e o que gravou primeiro no banco não foi `open`. Ajuste em **Self-Hosting**, ou alinhe a variável e faça deploy de novo.
