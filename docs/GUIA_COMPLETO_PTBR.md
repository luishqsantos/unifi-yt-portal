# Guia Completo do Projeto - Portal Cativo Externo UniFi

## 1. Visao geral

Este projeto implementa um portal cativo externo para redes UniFi.

Na pratica:

1. O cliente conecta ao Wi-Fi e e redirecionado para o portal.
2. O portal mostra formulario (nome, sobrenome, e-mail).
3. O sistema autoriza o MAC do cliente no controlador UniFi por um periodo definido.
4. O cliente e redirecionado para a URL final configurada.

Tambem existe logica para usuario recorrente: se o MAC ja existe no banco, o portal pula o formulario e autoriza direto.

## 2. Estrutura do projeto

```text
unifi-yt-portal/
|-- .env.example
|-- README.md
`-- default/
    |-- composer.json
    |-- header.php
    |-- config.php
    |-- index.php
    |-- welcome.php
    |-- connect.php
    |-- thanks.php
    |-- policy.php
    `-- assets/
```

Observacao importante sobre pasta `default`:

- O UniFi chama o portal externo usando o `site_id`.
- Se seu site no UniFi nao for `default`, renomeie a pasta para o `site_id` real (ex.: `rig5vxr7`).

## 3. Dependencias e funcao de cada uma

Dependencias definidas em `default/composer.json`:

- `art-of-wifi/unifi-api-client`: cliente PHP para autenticar usuario no controlador UniFi.
- `vlucas/phpdotenv`: leitura de variaveis no arquivo `.env`.
- `fortawesome/font-awesome`: icones no frontend.

## 4. Fluxo completo do sistema

### 4.1 Entrada no portal

Arquivo: `default/index.php`

O UniFi envia parametros via query string, principalmente:

- `id`: MAC do cliente.
- `ap`: MAC do access point.

O `index.php`:

1. guarda `id` e `ap` na sessao;
2. marca o tipo de usuario como `new`;
3. consulta o banco por MAC;
4. se ja existir MAC, muda `user_type` para `repeat` e redireciona para `welcome.php`;
5. se nao existir, renderiza formulario.

### 4.2 Usuario recorrente

Arquivo: `default/welcome.php`

- Exibe mensagem "Welcome back!".
- Usa meta refresh para chamar `connect.php` apos 5 segundos.

### 4.3 Autorizacao no UniFi e persistencia

Arquivo: `default/connect.php`

Passos:

1. le dados da sessao (`id`/MAC e `ap`);
2. cria cliente da API UniFi com credenciais do `.env`;
3. executa login no controlador;
4. chama `authorize_guest($mac, $duration, ..., $apmac)`;
5. se usuario for `new`, cria tabela (se nao existir) e grava cadastro no MySQL;
6. redireciona para URL final apos 2 segundos.

### 4.4 Pagina de sucesso

Arquivo: `default/thanks.php`

- Exibe mensagem de sucesso.
- Redireciona para `REDIRECT_URL` apos 5 segundos.

## 5. Papel de cada arquivo principal

- `default/header.php`
  - inicia sessao;
  - bloqueia acesso direto sem parametro `id`;
  - carrega autoload do Composer;
  - carrega variaveis de ambiente com `phpdotenv`;
  - expoe `BUSINESS_NAME` para as views.

- `default/config.php`
  - abre conexao MySQL com variaveis vindas do ambiente.

- `default/index.php`
  - ponto de entrada do hotspot;
  - decide entre usuario novo e recorrente.

- `default/welcome.php`
  - tela intermediaria para usuario recorrente.

- `default/connect.php`
  - ponto central da regra de negocio:
    - autorizacao no UniFi;
    - persistencia no MySQL;
    - redirecionamento final.

- `default/thanks.php`
  - tela de confirmacao final.

- `default/policy.php`
  - texto de termos de uso exibido no link do checkbox.

## 6. Variaveis de ambiente (.env)

Arquivo base: `.env.example`

### Banco de dados

- `HOST_IP`: host/IP do MySQL
- `DB_USER`: usuario MySQL
- `DB_PASS`: senha MySQL
- `DB_NAME`: nome do banco
- `TABLE_NAME`: tabela para salvar usuarios

### UniFi Controller

- `CONTROLLER_USER`: usuario do controller
- `CONTROLLER_PASSWORD`: senha do controller
- `CONTROLLER_URL`: URL do controller (ex.: `https://unifi.seudominio.com:443`)
- `CONTROLLER_VERSION`: versao esperada pela lib (compativel com seu controller)
- `DURATION`: tempo de autorizacao do cliente (em minutos)
- `SITE_ID`: identificador do site UniFi

### Frontend / redirecionamento

- `BUSINESS_NAME`: nome exibido no portal
- `REDIRECT_URL`: destino final apos autorizacao

## 7. Banco de dados

A tabela e criada automaticamente no primeiro acesso de usuario novo.

Estrutura criada por `connect.php`:

- `id` (PK autoincrement)
- `firstname`
- `lastname`
- `email`
- `mac` (UNIQUE)
- `last_updated`

Regra funcional:

- se MAC existe: considera usuario recorrente;
- se MAC nao existe: considera novo e grava cadastro.

## 8. Como configurar e rodar

## 8.1 Clonar projeto

```bash
git clone https://github.com/splash-networks/unifi-yt-portal
```

## 8.2 Ajustar pasta do site

Se o `SITE_ID` for diferente de `default`, renomeie:

```bash
mv default <SEU_SITE_ID>
```

## 8.3 Criar arquivo de ambiente

```bash
cp .env.example .env
```

Preencha todas as variaveis no `.env`.

## 8.4 Instalar dependencias PHP

No diretorio do site (ex.: `default`):

```bash
composer install
```

## 8.5 Publicar via servidor web

Exemplo de conceito:

- DocumentRoot apontando para a raiz do projeto que contem a pasta do `site_id`.
- URL de chamada externa no UniFi configurada para o caminho correto (incluindo o `site_id`).

## 8.6 Configurar no UniFi

No controlador UniFi:

1. habilite guest portal com external captive portal;
2. configure URL externa do portal;
3. valide que o `SITE_ID` da URL e da pasta sao iguais;
4. valide certificado TLS (recomendado HTTPS).

## 9. Checklist de validacao

1. Ao abrir hotspot, `index.php` recebe `id` e `ap` na URL.
2. Usuario novo ve formulario.
3. Ao enviar formulario, `connect.php` autoriza no UniFi.
4. Registro e salvo no MySQL.
5. Usuario e redirecionado para `REDIRECT_URL`.
6. Em novo acesso com mesmo MAC, fluxo vai para `welcome.php` e depois autoriza automaticamente.

## 10. Troubleshooting

### Nao conecta ao banco

- confirme host/porta do MySQL;
- confirme usuario/senha/permissoes;
- confira se `DB_NAME` e `TABLE_NAME` estao corretos.

### Nao autoriza no UniFi

- valide `CONTROLLER_URL`, usuario e senha;
- confira se `SITE_ID` corresponde ao site real no controller;
- valide `CONTROLLER_VERSION` compativel com o controller;
- confirme que o portal consegue alcancar a rede do controller.

### Sessao nao mantida

- confirme configuracao de sessao no PHP;
- verifique se requests permanecem no mesmo dominio/caminho esperado.

### Redirecionamento nao funciona

- valide `REDIRECT_URL` no `.env`;
- confirme se URL e acessivel do lado do cliente.

## 11. Limitacoes e riscos atuais (importante)

Para entender completamente o projeto, vale conhecer os pontos tecnicos abaixo:

1. SQL sem prepared statements:
   - atualmente ha interpolacao direta em queries (`$_POST` e `$_SESSION`).
   - recomendacao: migrar para prepared statements para reduzir risco de SQL Injection.

2. Sem validacao robusta de entrada:
   - ideal validar/sanitizar nome, sobrenome, e-mail e formato de MAC.

3. Sem tratamento estruturado de erros:
   - se login UniFi falhar, fluxo ainda pode seguir sem mensagem adequada.
   - recomendacao: tratar retorno da API e exibir erro amigavel.

4. Tabela criada em runtime:
   - funciona, mas em producao e melhor criar schema via migration/DDL controlada.

5. `thanks.php` nao esta no fluxo principal atual:
   - o `connect.php` redireciona direto para `REDIRECT_URL`.
   - se quiser usar tela de agradecimento, ajuste o redirecionamento para `thanks.php`.

## 12. Melhorias recomendadas (roadmap)

1. Seguranca
   - prepared statements;
   - CSRF token no formulario;
   - validacao server-side completa.

2. Observabilidade
   - logs de sucesso/erro UniFi;
   - logs de falha de banco;
   - correlacao por MAC/session id.

3. Confiabilidade
   - pagina de erro amigavel com opcoes de retry;
   - timeout e retentativa para chamada ao controller.

4. Evolucao funcional
   - painel admin para relatorios de captacao;
   - opt-in de marketing separado dos termos;
   - exportacao CSV.

## 13. Fluxo resumido em uma linha

`Cliente -> index.php -> (novo: formulario / recorrente: welcome.php) -> connect.php -> autoriza UniFi + salva no MySQL -> REDIRECT_URL`

