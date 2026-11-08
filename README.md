# Guess Game - Docker Compose
 
Jogo de adivinhação desenvolvido em Flask e React, utilizando PostgreSQL para persistência dos dados.
 
Este repositório utiliza como base a aplicação disponibilizada pelo professor em:
 
https://github.com/fams/guess_game
 
A aplicação original foi mantida e a infraestrutura foi preparada utilizando Docker Compose para executar o frontend, múltiplas instâncias do backend e o banco de dados PostgreSQL.
 
## Arquitetura
 
A solução é composta pelos seguintes serviços:
 
- Frontend React servido pelo NGINX.
- NGINX funcionando também como proxy reverso e balanceador de carga.
- Duas instâncias do backend Python/Flask.
- Banco de dados PostgreSQL.
- Volume Docker persistente para armazenamento dos dados do PostgreSQL.
- Rede Docker exclusiva para comunicação entre os serviços.
 
Fluxo das requisições:
 
```text
Usuário
|
| http://localhost:8080
v
+------------------+
| NGINX |
| |
| React |
| Reverse Proxy |
| Load Balancer |
+--------+---------+
|
/api/*
|
Round-Robin
/ \
v v
+-----------+ +-----------+
| Backend 1 | | Backend 2 |
| Flask | | Flask |
| :5000 | | :5000 |
+-----+-----+ +-----+-----+
| |
+-------+-------+
|
v
+---------------+
| PostgreSQL |
| :5432 |
+-------+-------+
|
v
Volume persistente
postgres_data
Tecnologias utilizadas
Python 3.12
Flask
React
Node.js 18.17.0
NGINX
PostgreSQL 16
Docker
Docker Compose

As versões de Python e Node.js seguem as versões indicadas para a aplicação original.

Estrutura Docker
Backend

O backend utiliza uma imagem baseada em:

Plain Text
python:3.12-slim

As dependências são instaladas a partir do arquivo requirements.txt.

A aplicação é iniciada pelo Flask na porta interna 5000.

Existem duas instâncias do mesmo backend:

Plain Text
backend1
backend2

As duas utilizam a mesma imagem:

Plain Text
guess-game-backend:1.0

Isso permite que o NGINX distribua as requisições entre múltiplas instâncias da aplicação.

Os containers de backend não são expostos diretamente para a máquina host.

Frontend e NGINX

O frontend utiliza Node.js 18.17.0 para realizar o build da aplicação React.

O Dockerfile utiliza multi-stage build:

Node.js executa a instalação das dependências e o build do React.
NGINX recebe os arquivos estáticos gerados pelo build.
NGINX disponibiliza o frontend e atua como proxy reverso.

A variável:

Plain Text
REACT_APP_BACKEND_URL=/api

é configurada durante o build do frontend.

Dessa forma, as chamadas realizadas pelo React para /api são recebidas pelo mesmo NGINX que disponibiliza o frontend.

Proxy reverso e balanceamento de carga

O NGINX possui um upstream com as duas instâncias Flask:

Nginx Config
upstream flask_backend {
server backend1:5000;
server backend2:5000;
}

As chamadas recebidas em:

Plain Text
/api/*

são encaminhadas para o backend.

Por exemplo:

Plain Text
/api/create

é encaminhado para:

Plain Text
/create

e:

Plain Text
/api/guess/<game_id>

é encaminhado para:

Plain Text
/guess/<game_id>

O NGINX utiliza a estratégia padrão de balanceamento Round-Robin entre os servidores configurados no upstream.

Se uma das instâncias do backend estiver indisponível, a outra permanece disponível para atender as requisições.

PostgreSQL

O PostgreSQL utiliza a imagem:

Plain Text
postgres:16

O banco de dados fica disponível somente através da rede interna criada pelo Docker Compose.

Os backends encontram o banco utilizando o nome do serviço:

Plain Text
postgres

Portanto, a conexão ocorre internamente através de:

Plain Text
postgres:5432

A porta 5432 não precisa ser publicada na máquina host.

Persistência dos dados

Os dados do PostgreSQL são armazenados no volume Docker:

Plain Text
postgres_data

O volume é montado em:

Plain Text
/var/lib/postgresql/data

Isso garante que os dados continuem disponíveis mesmo quando os containers forem removidos e recriados.

Por exemplo:

Shell
docker compose down
docker compose up -d

não remove os dados existentes no banco.

ATENÇÃO:

O comando abaixo remove também os volumes:

Shell
docker compose down -v

Portanto, ele não deve ser utilizado caso seja necessário preservar os dados do jogo.

Health Check

O PostgreSQL possui um health check utilizando:

Plain Text
pg_isready

Os containers Flask dependem do estado saudável do PostgreSQL antes de serem iniciados.

Isso evita que o backend tente estabelecer uma conexão enquanto o banco de dados ainda está sendo inicializado.

Resiliência

Os serviços utilizam a política:

YAML
restart: unless-stopped

Essa política configura os containers para serem reiniciados pelo Docker quando necessário, exceto quando forem explicitamente interrompidos.

Além disso, duas instâncias do backend são disponibilizadas atrás do NGINX para reduzir a dependência de uma única instância Flask.

Rede

Os containers utilizam uma rede Docker do tipo bridge:

Plain Text
guess-network

A comunicação entre NGINX, backends e PostgreSQL ocorre nessa rede.

Somente o NGINX é publicado para a máquina host.

Isso evita a exposição desnecessária das portas dos containers de backend e banco de dados.

Como executar
Pré-requisitos

É necessário possuir:

Docker
Docker Compose

No Windows, pode ser utilizado o Docker Desktop.

Iniciar a aplicação

Na raiz do projeto execute:

Shell
docker compose up -d

O Docker Compose irá iniciar:

PostgreSQL
Backend Flask 1
Backend Flask 2
Frontend React / NGINX

Para verificar o estado dos containers:

Shell
docker compose ps

O PostgreSQL deverá aparecer como:

Plain Text
healthy
URL da aplicação

Depois que os containers estiverem ativos, acesse:

http://localhost:8080

Essa é a única URL necessária para utilizar a aplicação.

O frontend é servido pelo NGINX nessa porta e as requisições para o backend são encaminhadas internamente através do proxy reverso.

Como jogar
Criar um jogo

Acesse:

Plain Text
http://localhost:8080

Entre na opção de criação de jogo e informe uma palavra ou frase secreta.

A aplicação retornará um game_id.

Guarde esse identificador.

Tentar adivinhar

Acesse a opção de tentativa, informe:

o game_id;
a palavra ou frase que deseja tentar.

O backend irá retornar o resultado da tentativa.

Testando a API

O endpoint de saúde pode ser consultado através do NGINX:

Plain Text
http://localhost:8080/api/health

Resposta esperada:

JSON
{
"status": "ok"
}

Também é possível testar no PowerShell:

PowerShell
Invoke-RestMethod http://localhost:8080/api/health
Parar a aplicação

Execute:

Shell
docker compose down

Os containers e a rede serão removidos, mas o volume do PostgreSQL será preservado.

Para iniciar novamente:

Shell
docker compose up -d
Atualização dos componentes

A arquitetura foi organizada para permitir a atualização independente dos componentes.

Backend

O backend utiliza:

Plain Text
guess-game-backend:1.0

Uma nova versão pode utilizar, por exemplo:

Plain Text
guess-game-backend:1.1

Após alterar a versão correspondente, a imagem pode ser reconstruída e os containers recriados.

Frontend

O frontend utiliza:

Plain Text
guess-game-frontend:1.0
 

Uma nova versão pode utilizar:

Plain Text
guess-game-frontend:1.1
 

O container pode ser reconstruído sem necessidade de alterar o PostgreSQL.

PostgreSQL

A versão está definida pela imagem:

Plain Text
postgres:16

A versão da imagem pode ser alterada no docker-compose.yml.

Atualizações de versão principal do PostgreSQL, entretanto, exigem os procedimentos apropriados de migração do banco de dados e não devem ser realizadas apenas substituindo a tag sem planejamento.

Reconstrução das imagens

Quando houver alteração nos Dockerfiles ou componentes da aplicação:

Shell
docker compose build

Depois:

Shell
docker compose up -d

Também é possível executar:

Shell
docker compose up -d --build
Logs

Para visualizar os logs de todos os serviços:

Shell
docker compose logs

Para acompanhar os logs em tempo real:

Shell
docker compose logs -f

Para um serviço específico:

Shell
docker compose logs backend1

Exemplo para o PostgreSQL:

Shell
docker compose logs postgres
Decisões de design
Duas instâncias do backend

Foram utilizadas duas instâncias Flask para permitir que o NGINX realize balanceamento de carga e para evitar que todas as requisições dependam de uma única instância.

NGINX como frontend e proxy reverso

O mesmo container NGINX:

serve os arquivos estáticos gerados pelo React;
recebe as chamadas /api;
atua como proxy reverso;
distribui as requisições entre os backends.

Essa solução permite disponibilizar toda a aplicação através de uma única porta.

Banco não exposto externamente

O PostgreSQL não publica a porta 5432 no host.

Somente os containers pertencentes à rede Docker conseguem acessar diretamente o banco de dados.

Backend não exposto externamente

A porta 5000 dos backends também não é publicada para a máquina host.

O acesso ocorre exclusivamente através do NGINX.

Volume dedicado

O banco utiliza um volume Docker separado dos containers, garantindo persistência dos dados quando os containers são recriados.

Health Check do PostgreSQL

O pg_isready garante que os backends aguardem o PostgreSQL ficar disponível antes de serem iniciados.

Código original

O objetivo deste trabalho é a conteinerização e orquestração da aplicação.

Por esse motivo, o código-fonte da aplicação Flask e do frontend React original foi preservado. As alterações realizadas concentram-se na infraestrutura Docker, configuração do NGINX e documentação.

Arquivos principais da solução
Plain Text
guess_game/
|
|-- Dockerfile
|-- docker-compose.yml
|-- requirements.txt
|-- run.py
|
|-- guess/
|-- repository/
|
`-- frontend/
|-- Dockerfile
|-- default.conf
|-- package.json
`-- src/
Dockerfile

Responsável pela construção da imagem do backend Python/Flask.

frontend/Dockerfile

Responsável pelo build do React utilizando Node.js e pela criação da imagem final utilizando NGINX.

frontend/default.conf

Configuração do NGINX responsável por:

servir o frontend;
atuar como proxy reverso;
balancear as requisições entre os backends.
docker-compose.yml

Responsável por definir:

PostgreSQL;
duas instâncias Flask;
frontend/NGINX;
volume persistente;
rede interna;
health check;
dependências dos serviços;
política de reinício.
Aplicação original

A aplicação utilizada neste trabalho foi disponibilizada pelo professor:

https://github.com/fams/guess_game

O projeto original implementa um jogo de adivinhação utilizando Flask e React.


## Aplicação original

A aplicação utilizada neste trabalho foi disponibilizada pelo professor:

https://github.com/fams/guess_game

O projeto original implementa um jogo de adivinhação utilizando Flask e React.