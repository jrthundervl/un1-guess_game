# un1-guess_game

Documentação de Infraestrutura: Guess Game

Este repositório é onde você encontra a orquestração e a conteinerização do projeto Guess Game utilizando Docker Compose. A ideia aqui é fazer com que a aplicação se torne um ecossistema de microsserviços que seja resiliente, escalável, isolado e fácil de manter.

Opções de Design Adotadas

1. Separação de Serviços (Arquitetura)

A aplicação foi dividida em 4 blocos de serviços no Docker Compose:

	- postgres_db: Banco de dados relacional baseado na imagem oficial postgres:16.

	- backend: Cluster Python 3.12 rodando a API Flask, configurado para ler as variáveis de ambiente do repositório.

	- frontend: Container Node 18 que compila os assets estáticos do React e os exporta via volume de produção.

	- nginx_proxy: Servidor Web NGINX que atua como proxy reverso, balanceador de carga e servidor de arquivos estáticos.

2. Rede Isolada (Segurança)

Para evitar ataques e garantir o isolamento de componentes, criamos duas redes do tipo bridge:

	- backend_net: Rede privada de backend. Apenas os containers do backend e o postgres_db se comunicam aqui. O banco de dados fica invisível para o mundo externo.

	- frontend_net: Rede de borda. Permite a comunicação do nginx_proxy com o frontend e com as instâncias do backend para o encaminhamento de requisições de API.

3. Resiliência, Persistência e Healthcheck

	- Persistência com Volumes: Criamos o volume nomeado postgres_data mapeado para /var/lib/postgresql/data. Isso garante que os dados persistam mesmo se o container for destruído ou atualizado.

	- Resiliência: Configuração que garante o reinício imediato de qualquer container caso sofra um crash interno.

	- Sincronismo com Healthcheck: O backend depende do Postgres, mas o banco demora alguns segundos para inicializar. Adicionamos um healthcheck usando pg_isready. O backend só inicia quando o Postgres estiver pronto para receber conexões.

4. Estratégia de Balanceamento de Carga (Load Balancing)

O serviço backend foi configurado com a flag deploy.replicas: 2. O NGINX foi instruído a interceptar as chamadas feitas para o endpoint /api/ e distribuí-las via algoritmo Round-Robin entre as instâncias ativas do Flask. Isso melhora o tempo de resposta e garante que, se uma instância falhar, a outra assumirá o tráfego de forma transparente.

Instruções de Instalação e Execução

Pré-requisitos

	- Docker instalado (versão 20.10+)

	- Docker Compose instalado (versão 2.0+)



Passo a Passo

Garanta que a árvore de diretórios do projeto esteja organizada da seguinte forma:

	- / (Raiz do projeto)
	- ├── guess_game/ # Repositório original clonado (com run.py e Dockerfile do backend)
	- ├── frontend/ # Pasta do frontend original (com src/ e Dockerfile do frontend)
	- ├── nginx/
	- │ └── nginx.conf # Arquivo de configuração do proxy
	- └── docker-compose.yml # Orquestrador principal

Na raiz do projeto, execute o comando abaixo para realizar o build das imagens locais e subir toda a infraestrutura em segundo plano:

	docker compose -f docker-compose.yml up -d --build

Certifique-se de que todos os serviços estão ativos e saudáveis:

	docker compose ps


URL de Acesso à Aplicação

A aplicação é centralizada em uma única porta de produção (Porta 80).

Após a inicialização, acesse o jogo através da URL:

	http://localhost

O Frontend React é servido diretamente na raiz (/).

O NGINX captura as requisições de API (/api/*) e faz o redirecionamento invisível para o cluster Flask na porta interna 5000.



Estratégia de Atualização de Componentes

A arquitetura foi projetada para permitir a atualização contínua de qualquer parte do sistema modificando apenas uma linha ou rodando um comando simples.

1. Atualizar o Banco de Dados (Postgres)

Se for necessário atualizar o motor do banco de dados, abra o arquivo docker-compose.yml e altere a propriedade image:

image: postgres:17

Atualize apenas o container modificado:

	docker compose -f docker-compose.yml up -d postgres_db

2. Atualizar o Backend (Python/Flask)

Se houver uma nova funcionalidade ou atualização de segurança no código Python, atualize os arquivos dentro da pasta guess_game/ e force a reconstrução das réplicas:

	docker compose build backend --no-cache
	docker compose -f docker-compose.yml up -d --no-deps backend

3. Atualizar o Frontend (React)

Se a interface gráfica sofrer alterações visuais, atualize o código dentro da pasta frontend/ e recompile os pacotes estáticos do Node:

	docker compose build frontend --no-cache
	docker compose -f docker-compose.yml up -d --no-deps frontend

O container do Node atualizará os arquivos dentro do volume compartilhado e o NGINX passará a servir o novo layout imediatamente.
