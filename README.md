# Fluxo Web Seguro: Docker, API e MySQL

Este projeto preserva a atividade original: MySQL e phpMyAdmin em Docker, comunicando-se por rede interna, com o phpMyAdmin publicado somente na porta `8080` para uso na LAN. A evolucao acrescenta uma interface web e uma API para demonstrar o fluxo completo de dados sem expor o MySQL ao navegador ou a Internet.

## Arquitetura

```text
Navegador (computador/celular)
        | HTTP na LAN ou HTTPS com dominio publico
        v
Reverse proxy Caddy (porta configuravel: 8081/8443 no desenvolvimento)
        |
        v
Aplicacao Express: interface estatica + API /api/users
        |                 (rede database interna)
        v
MySQL --------------------------------------------------- volume mysql_data

phpMyAdmin (porta 8080, somente LAN/VPN) --- rede database interna --- MySQL
```

Ha duas redes:

- `proxy`: conecta somente `reverse-proxy` e `app`.
- `database` (`internal: true`): conecta `app`, `mysql` e `phpmyadmin`. O MySQL nao tem `ports`, portanto nao recebe conexoes diretamente do host, LAN ou Internet.

## Fluxo demonstrado

1. A pessoa preenche **nome** e **e-mail** na pagina inicial.
2. O JavaScript envia `POST /api/users` com JSON pela mesma origem.
3. A API valida tamanho, caracteres de controle e formato de e-mail.
4. A API usa `INSERT INTO usuarios (nome, email) VALUES (?, ?)` com parâmetros, nunca concatena a entrada em SQL.
5. MySQL grava no volume persistente e a API devolve `201 Created` ou uma mensagem segura de erro.
6. A pagina usa `textContent` para renderizar a resposta, evitando que dados do banco sejam interpretados como HTML.
7. Ao abrir ou atualizar a lista, a pagina envia `GET /api/users`; a API consulta no maximo 100 registros e devolve JSON.

## Arquivos

```text
.
├── docker-compose.yml
├── Caddyfile
├── .env.example
├── .gitignore
├── README.md
├── docker/mysql/init/01-schema-and-user.sh
└── app/
    ├── Dockerfile
    ├── package.json
    ├── server.js
    └── public/
        ├── index.html
        ├── app.js
        └── styles.css
```

## Configuracao local

1. Copie o arquivo de exemplo:

   ```powershell
   Copy-Item .env.example .env
   ```

2. Edite `.env`. Troque principalmente `MYSQL_ROOT_PASSWORD` e `MYSQL_APP_PASSWORD` por senhas longas e unicas. O arquivo `.env` esta no `.gitignore` e nao deve ser enviado ao Git.

3. Suba os servicos:

   ```powershell
   docker compose up -d --build
   docker compose ps
   ```

Em uma instalacao nova, o script de inicializacao cria a tabela `usuarios` e o usuario da aplicacao com apenas `SELECT` e `INSERT` nessa tabela. O script executa apenas na primeira criacao do volume `mysql_data`.

Se voce ja possuia o volume da atividade anterior, mantenha-o: rode o comando abaixo uma unica vez para criar a tabela e o usuario sem apagar dados. Os valores devem corresponder ao seu `.env`:

```powershell
$sql = @'
CREATE TABLE IF NOT EXISTS atividade_db.usuarios (
  id INT UNSIGNED NOT NULL AUTO_INCREMENT,
  nome VARCHAR(100) NOT NULL,
  email VARCHAR(254) NOT NULL,
  criado_em TIMESTAMP NOT NULL DEFAULT CURRENT_TIMESTAMP,
  PRIMARY KEY (id),
  UNIQUE KEY usuarios_email_unique (email)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
CREATE USER IF NOT EXISTS 'app_usuario'@'%' IDENTIFIED BY 'SUBSTITUA_PELA_SENHA_DO_ENV';
GRANT SELECT, INSERT ON atividade_db.usuarios TO 'app_usuario'@'%';
FLUSH PRIVILEGES;
'@
$sql | docker compose exec -T mysql mysql -uroot -pSUA_SENHA_ROOT
```

Nao use `docker compose down -v` se quiser preservar os dados. Esse comando remove tambem o volume.

## Acessos e testes

| Item | Endereco local | Exposto para Internet? |
|---|---|---|
| Aplicacao HTTP de demonstracao | `http://localhost:8081` | Nao, por padrao |
| Aplicacao HTTPS local | `https://localhost:8443` | Nao, por padrao; o certificado local pode gerar aviso em outros dispositivos |
| phpMyAdmin | `http://localhost:8080` | Nao deve ser publicado; mantenha LAN/VPN |
| MySQL | nao possui URL/porta de host | Nao |

Teste a entrada e saida pelo navegador em `http://localhost:8081`:

1. Cadastre `Ana Silva` e `ana@example.com`.
2. Veja a mensagem **Cadastro realizado com sucesso**.
3. Confirme que a linha aparece na tabela.
4. Clique em **Atualizar lista**: isto dispara o `GET /api/users`.

Teste a API sem navegador:

```powershell
Invoke-RestMethod http://localhost:8081/api/users
Invoke-RestMethod http://localhost:8081/api/users -Method Post -ContentType 'application/json' -Body '{"nome":"Ana Silva","email":"ana@example.com"}'
```

Para ver os logs:

```powershell
docker compose logs app
docker compose logs reverse-proxy
docker compose logs mysql
docker compose logs phpmyadmin
```

Para confirmar redes e isolamento:

```powershell
docker network inspect aula02_proxy
docker network inspect aula02_database
docker compose ps
```

O resultado de `docker compose ps` deve mostrar portas somente em `reverse-proxy` e `phpmyadmin`; MySQL deve mostrar somente `3306/tcp`, sem `0.0.0.0:3306->...`.

## Teste pela LAN

No host Windows, descubra o IPv4 associado ao gateway padrao:

```powershell
Get-NetIPConfiguration | Where-Object { $_.IPv4DefaultGateway -ne $null } | Select-Object InterfaceAlias,IPv4Address
```

Se o IP for, por exemplo, `10.1.10.83`, outro computador ou celular na mesma Wi-Fi abre:

```text
http://10.1.10.83:8081
```

O phpMyAdmin da atividade continua em `http://10.1.10.83:8080`. Nao o encaminhe no roteador; use-o apenas na LAN ou via VPN. Caso a rede Windows esteja marcada como Publica, crie como Administrador uma regra de Firewall limitada a rede local para a porta que deseja demonstrar, por exemplo `8081`:

```powershell
New-NetFirewallRule -DisplayName "Docker app atividade" -Direction Inbound -Action Allow -Profile Public -Protocol TCP -LocalPort 8081 -RemoteAddress LocalSubnet
```

## HTTPS e publicacao real na Internet

`localhost` e o IP da LAN **nao** sao acessiveis automaticamente pela Internet. Para acesso publico real:

1. Use um dominio, como `app.seudominio.com`, e configure um registro DNS A/AAAA para o IP publico do servidor.
2. No `.env`, defina `DOMAIN=app.seudominio.com`, `HTTP_PORT=80` e `HTTPS_PORT=443`.
3. Permita TCP 80 e 443 no firewall do host, firewall cloud e/ou roteador; encaminhe essas portas ao host quando ele estiver atras de NAT.
4. Mantenha o host alcançavel pela Internet. Caddy obtera e renovara automaticamente o certificado TLS quando o dominio apontar corretamente e as portas estiverem acessiveis.
5. Prefira um servidor/cloud para publicacao. Nao exponha a porta 8080 nem o phpMyAdmin publicamente. Se administracao remota for necessaria, use VPN ou uma camada adicional de autenticacao forte.

Em desenvolvimento, o HTTPS local usa certificado interno do Caddy. Para dispositivo externo, o navegador nao confiara nele por padrao; para uma demonstracao LAN simples use HTTP na porta `8081`. HTTPS confiavel em qualquer dispositivo requer dominio valido e certificado publico.

## Medidas de seguranca implementadas

- Sem porta publicada para MySQL e rede `database` interna.
- Aplicacao usa usuario MySQL proprio, sem root, com `SELECT` e `INSERT` somente em `usuarios`.
- Segredos ficam no `.env`, ignorado pelo Git; `docker-compose.yml` contem somente referencias a variaveis.
- Validacao no frontend e, de forma obrigatoria, no backend.
- Prepared statements em todas as consultas com entrada do usuario.
- Respostas de erro genericas; detalhes ficam somente em logs do container.
- Sem CORS permissivo: interface e API sao same-origin.
- Limite de corpo de 10 KB e rate limit de 100 requisicoes/15 min para API e 10 cadastros/15 min por IP.
- Helmet na aplicacao e headers de seguranca no Caddy.
- Container da aplicacao sem porta publicada, somente leitura, sem capacidades Linux adicionais e sem novos privilegios.
- Dados inseridos sao apresentados por `textContent`, nao por `innerHTML`.

Nao ha autenticacao de usuarios nesta demonstracao; portanto, nao ha senha de usuario a armazenar. Se ela for adicionada, a senha deve ser hash (Argon2id ou bcrypt), nunca texto puro.

## Evidencias para o professor

1. **Arquitetura e containers:** terminal com `docker compose ps`, mostrando `mysql`, `app`, `reverse-proxy` e `phpmyadmin` como `Up`; destaque que MySQL nao publica porta.
2. **Entrada e processamento:** navegador com o formulario preenchido e, apos enviar, a mensagem de sucesso e a nova linha na lista. Se possivel, abra a aba Network do navegador mostrando `POST /api/users` com status `201`.
3. **Consulta/saida:** aba Network mostrando `GET /api/users` com status `200` e a tabela renderizada.
4. **Banco e isolamento:** phpMyAdmin na LAN mostrando a tabela `usuarios` e o registro; terminal com `docker network inspect aula02_database`.
5. **Outro dispositivo:** foto ou captura de celular/segundo computador em `http://IP-DO-HOST:8081`. Para a atividade original, inclua tambem o phpMyAdmin na porta `8080` pela LAN, se o professor exigir essa prova especifica.
