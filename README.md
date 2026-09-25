# HelpDesk Platform - Desafio Técnico

## Descrição do Sistema
A Plataforma Helpdesk é uma aplicação web Full Stack orientada a microsserviços para gerenciamento de chamados de suporte técnico. Ela permite a abertura de chamados, atribuição de técnicos, acompanhamento de status/prioridade e o consumo e exibição de notificações em tempo real.

## Arquitetura e Microsserviços
O projeto foi estruturado seguindo arquitetura baseada em microsserviços. A comunicação externa e do frontend se dá de maneira centralizada via API Gateway, enquanto os serviços utilizam RabbitMQ para comunicação assíncrona de eventos, e OpenFeign para comunicação síncrona.

* **API Gateway:** Ponto de entrada central. Roteia as requisições do Frontend (React) de forma transparente para os microsserviços corretos do Backend, mascarando as portas reais e locais de rede.
* **User Service:** Responsável pelo gerenciamento de clientes, técnicos e administradores, bem como pela emissão, validação e verificação de perfil nos tokens JWT.
* **Ticket Service:**  Responsável por todo o ciclo de vida dos chamados (criação, atualização de status e prioridades, categorias, além da vinculação de cliente e técnico).
* **Notification Service:** Serviço reativo que assina filas do RabbitMQ. Ao receber eventos como "novo chamado" ou "técnico atribuído", o serviço cria e armazena notificações direcionadas aos usuários.

## Tecnologias Utilizadas
**Backend:**
- Java 21
- Spring Boot (Spring Web, Spring Data JPA, Spring Validation, Spring Security, Java Jwt)
- Spring Cloud Gateway
- Swagger / OpenAPI
- JUnit e Mockito (Testes unitário automatizados)
- Open Feign

**Frontend:**
- React 19 (criado com Vite)
- Tailwind CSS
- Axios

**Infraestrutura e Mensageria:**
- Bancos de dados relacionais: PostgreSQL
- Mensageria: RabbitMQ
- Containerização: Docker e Docker Compose

## Pré-requisitos
Para executar este projeto sem precisar configurar Java ou Node.js localmente, você necessitará apenas de:
- [Docker](https://docs.docker.com/get-docker/)
- [Docker Compose](https://docs.docker.com/compose/install/)

## Instruções de Execução

1. Certifique-se de que não existem outros processos (como bancos postgres) utilizando as portas `5432`, `8080`, `5672` ou `80` em sua máquina.
2. Navegue pelo terminal até o diretório `infra/` dentro do projeto:
```bash
cd infra
```
3. Suba todo o ambiente usando o Docker Compose:
```bash
docker compose up -d --build
```
*(Na primeira vez que for executado, o Docker fará o download das imagens do PostgreSQL e RabbitMQ, além de realizar o build (compilação) de cada aplicação Spring e do React, o que pode levar alguns minutos).*

4. Acessos aos serviços em execução:
- **Frontend (Interface do Usuário):** http://localhost (ou http://localhost:80 caso tenha adicionado ao compose)
- **API Gateway (Rotas REST base):** http://localhost:8080
- **RabbitMQ Dashboard:** http://localhost:15672 (Usuário: `guest` / Senha: `guest`)
- **Documentação Swagger:** http://localhost:8080/swagger-ui.html

## Principais Endpoints (via API Gateway - Porta 8080)

**Usuários e Autenticação (`/api/users`)**
- `POST /api/users/auth/login` - Autentica usuário e retorna JWT.
- `POST /api/users` - Criação de novo usuário.
- `GET /api/users` - Lista de usuários filtrada por parâmetros.

**Chamados (`/api/tickets`)**
- `POST /api/tickets` - Cria um novo chamado no status `OPEN`.
- `GET /api/tickets` - Busca e listagem de chamados com filtros (categoria, status, cliente, prioridade).
- `GET /api/tickets/{id}` - Visualização dos detalhes do chamado.
- `PUT /api/tickets/{id}/status` - Atualização do status do chamado (ex: para `IN_PROGRESS` ou `RESOLVED`).
- `PATCH /api/tickets/{id}/assign` - Atribui um técnico a um chamado específico.

**Notificações (`/api/notifications`)**
- `GET /api/notifications` - Traz a lista de notificações pertinentes ao usuário logado.

## Eventos RabbitMQ (Mensageria)
O sistema aplica estratégias baseadas a eventos para processos de notificação:
- **Eventos Principais:** `TicketCreated` e `TicketAssigned`.
- **Como funciona:** Imediatamente após confirmar a transação do banco de dados na criação ou delegação de um chamado, o `ticket-service` dispara uma notificação (mensagem) para a fila do RabbitMQ. 
- O `notification-service`, que atua como um `listener`, consome a mensagem de forma reativa e persiste as notificações no banco dele para visualização futura do cliente ou técnico, sem atrasar a resposta da API (Non-blocking).

## Estratégia de Persistência
Cada microsserviço é autônomo e isolado em relação aos seus dados, garantindo a ausência de acoplamento rígido de bancos de dados.
- Utilizou-se um container central do **PostgreSQL**.
- Um script automatizado (`init-multiple-databases.sql`) no entrypoint do compose divide os esquemas lógicos: `user_db`, `ticket_db` e `notification_db`.
- O `ticket-service` não faz *JOIN* no `user-service`. Referências como `customerId` e `technicianId` ficam armazenadas, e os relacionamentos dependem de chamadas via protocolo HTTP ou composição do payload no frontend.
- Para fazer a verificação do cliente e do técnico associados ao ticket é feita uma requisição ao `user-service` que busca os usuários em banco. Essa chamada é feita de maneira síncrona chamando o endpoint `GET /users/{id}`; e a resposta então é validada de acordo com o tipo de usuário buscado.

## Principais decisões arquiteturais
- Autenticação realizada no `user-service` realizada com o objetivo de diminuir o overhead de rede e manter uma alta coesão entre os dados de usuário e sua identidade; além de diminuir a complexidade da minha arquitetura. Apesar disso podemos considerar que o principio de Single Responsibility foi quebrado, visto que o serviço passou a ter duas responsabilidades (Gestão do CRUD e Autenticação), no entanto para o momento e estágio da aplicação é uma abordagem completamente válida. Caso essa aplicação evolua para realizar integrações externas para autenticação, torna-se válido refatorar a autenticação para um microsserviço especifico.
- No `ticket-service` ao deletar um ticket realizando uma exclusão lógica, para impedir que o estado daquele ticket não possa ser mudado, foi atribuido também a ele o status de *CLOSED*; e com isso tornasse inviável realizar alterações nesse ticket.
- Foi adotada uma abordagem hibrida de comunicação entre os microsserviços. Utilizado comunicação sincrona entre `ticket-service` e `user-service` para realizar o pré-processamento da requisição, validando a existência e a confirmidade de papel com a atribuição de um usuário; com isso consigo evitar que tenha tickets salvos com id de usuário inválidos. Como consequência se o microsserviço de usuários falhar, não será possível cadastrar um usuário, mas para o cenário dado isso valia apena pois consigo manter a integridade dos meus dados.
Já para a comunicação entre o `ticket-service` e `notification-service` foi adotada uma comunicação assíncrona, pois ao concluir o processamento de uma operação de mudança de estado de um ticket, não faz sentido segurar esse serviço mais, então encaminho um evento e armazeno numa fila do rabbitMQ. Dessa forma eu diminuo o acoplamento do meu serviço com possíveis interessados no evento publicado, que apenas assinam o interesse e passam a escutar os eventos.
Além disso, foi implementado o padrão *Command Query Responsibility Segregation* (CQRS) no `notification-service`, onde processos de escrita são realizados de forma assíncrona enquanto processos de leitura são realizados de maneira síncrona.
- Foi optado por fazer a validação do token em cada microsserviço de maneira independente pois isso permite que eles trabalhem de maneira desacoplada, enquanto o token for válido.
Ademais, há um ganho de performance, visto que não há necessidade de se utilizar de um Identity Provider para garantir a validade daquele token.
Outro ponto relevante nessa estratégia é que os microsserviços não confia em qualquer requisição do gateway, o que torna o serviço muito mais resiliente a tentativas de invasão, visto que para que um ofensor consiga fazer requisições ao microsserviço ele precisará do token JWT, que ele não é capaz de gerar sem a chave secreta.
Como consequencia dessa decisão o token fica atrelado ao tempo de vida dele, então mesmo que um usuário seja desativado, ele ainda poderá fazer requisições.
- Utilização do WebFlux se deu pois Spring Gateway não tem compatibilidade com o SprignWebMvc. Isso se dá pois o Sprign WebMvc ele utiliza um modelo de requisição bloqueante, onde para cada requisição ele espera o processamento para então liberar a thread e aceitar uma nova requisição. Já o WebFlux ele funciona por um modelo reativo, o que combina com o funcionamento de um gateway, visto que ele passa a operar baseado em eventos. Dessa forma, o gateay apenas repassa a requisição para o endpoint correto e libera a sua thread para uma nova requisição.
Com essa escolha o gateway ele não se torna um gargalo, além de oferece alta escalabilidade, resiliencia e baixo consumo de memória, mesmo que com várias requisições simultaneas.
- A centralização de CORS e Swagger no gateway se deu pois dessa forma não é necessário expor as portas dos microsserviços para que o frontend consuma e nem para consulta da documentação da api.