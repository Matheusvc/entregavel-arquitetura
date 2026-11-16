# Diagrama de Componentes UML — iFood

Projeto da disciplina de **Arquitetura de Software** do **Oxygeni Hub**.

Sistema escolhido: **iFood**
Escopo: **fluxo de pedido de comida** (do clique em "Fazer Pedido" até a entrega).

---

## Parte 1 — Definição do Escopo e Componentes

### Escopo

O diagrama cobre o fluxo principal de um pedido no iFood: o consumidor escolhe um prato, realiza o pagamento, o restaurante prepara, e o entregador leva até o endereço. Componentes como sistema de avaliação, cupons e programa de fidelidade estão fora do escopo.

### Componentes Identificados

| Componente | Responsabilidade |
|---|---|
| **App Cliente** | Aplicativo mobile do consumidor. Permite navegar pelo cardápio, montar o carrinho, fazer o pedido e acompanhar a entrega em tempo real. |
| **App Restaurante** | Painel do estabelecimento. Recebe os pedidos, permite aceitar/recusar e atualizar o status de preparo. |
| **App Entregador** | Aplicativo mobile do motoboy. Exibe pedidos disponíveis para coleta, GPS de navegação e confirmação de entrega. |
| **Load Balancer** | Distribui as requisições entre múltiplas instâncias do API Gateway usando round-robin, evitando sobrecarga em um único servidor. |
| **API Gateway** | Ponto de entrada único para todos os clientes. Roteia requisições para o microsserviço correto, valida tokens JWT e aplica rate limiting. |
| **Serviço de Autenticação** | Gerencia login, registro, geração de tokens JWT e controle de permissões (consumidor, restaurante, entregador). |
| **Serviço de Pedidos** | Núcleo do sistema. Cria, atualiza e cancela pedidos. Orquestra a comunicação entre pagamento, restaurante e entregador. |
| **Serviço de Pagamentos** | Valida e processa pagamentos via cartão de crédito, Pix e boleto. Comunica-se com gateways de pagamento externos (Stripe, PagSeguro). |
| **Serviço de Notificações** | Envia notificações push, SMS e e-mail para consumidores ("Seu pedido saiu para entrega"), restaurantes ("Novo pedido!") e entregadores ("Nova coleta disponível"). |
| **Serviço de Rastreamento** | Recebe a localização GPS do entregador em tempo real e transmite para o App Cliente via WebSocket. |
| **Cache (Redis)** | Armazena dados de acesso frequente: cardápios, sessões de usuário e status de pedidos. Reduz a carga no banco de dados principal. |
| **Fila de Mensagens (RabbitMQ)** | Desacopla os microsserviços. Quando um pedido é criado, um evento é publicado na fila; o Serviço de Notificações e o Serviço de Rastreamento consomem o evento de forma assíncrona. |
| **Banco de Dados (PostgreSQL)** | Armazena dados transacionais: usuários, pedidos, pagamentos, endereços. Garante integridade com ACID. |
| **Réplica de Leitura** | Cópia do banco principal usada para consultas de relatórios e analytics, sem impactar a performance de escrita. |
| **Object Storage** | Armazena imagens de cardápio, fotos de perfil e comprovantes de entrega (ex: Amazon S3). |

---

## Parte 2 — Diagrama de Componentes UML

O diagrama está no arquivo [`diagrama-componentes.svg`](diagrama-componentes.svg).

### Estrutura em camadas

O diagrama está organizado em 4 camadas:

1. **Camada de Apresentação** — os 3 aplicativos cliente (consumidor, restaurante, entregador).
2. **Camada de Infraestrutura** — Load Balancer e API Gateway, que recebem e distribuem as requisições.
3. **Camada de Serviços** — os microsserviços que implementam a lógica de negócio, além do cache e da fila de mensagens.
4. **Camada de Dados** — banco de dados principal, réplica de leitura e armazenamento de objetos.

### Tipos de comunicação

- **Síncrona (seta contínua):** requisição HTTP/REST com resposta imediata. Exemplo: App Cliente → API Gateway → Serviço de Pedidos.
- **Assíncrona (seta tracejada):** evento publicado na fila, consumido quando o serviço estiver disponível. Exemplo: Serviço de Pedidos publica "pedido criado" → Fila → Serviço de Notificações consome e envia push.

---

## Parte 3 — Infraestrutura, Fluxo e SPOF

### Componentes de infraestrutura adicionados

- **Load Balancer** na frente do API Gateway, distribuindo requisições via round-robin.
- **Cache (Redis)** para consultas rápidas de cardápio e status de pedidos.
- **Fila de Mensagens (RabbitMQ)** para desacoplar os serviços de Notificações e Rastreamento.
- **Réplica de Leitura** do banco de dados para relatórios sem impactar a escrita.

### Fluxo: "O usuário clica em Fazer Pedido"

1. O **App Cliente** envia uma requisição HTTPS para o **Load Balancer** com os dados do pedido (itens, endereço, forma de pagamento).
2. O **Load Balancer** distribui a requisição para uma instância do **API Gateway**.
3. O **API Gateway** valida o token JWT do usuário chamando o **Serviço de Autenticação** e, se autorizado, roteia a requisição para o **Serviço de Pedidos**.
4. O **Serviço de Pedidos** cria o registro do pedido no **Banco de Dados** e publica um evento "pedido_criado" na **Fila de Mensagens**.
5. O **Serviço de Pagamentos** processa a cobrança no cartão/Pix do usuário e atualiza o status para "pago".
6. O **Serviço de Notificações** consome o evento da fila e envia um push para o **App Restaurante** ("Novo pedido!").
7. O restaurante aceita o pedido pelo **App Restaurante**. O status é atualizado para "em preparo".
8. Quando o prato está pronto, um novo evento é publicado. O **Serviço de Rastreamento** começa a monitorar entregadores disponíveis próximos ao restaurante.
9. O **App Entregador** recebe a notificação de coleta. Ao aceitar, a localização GPS é transmitida em tempo real.
10. O **Serviço de Rastreamento** envia as coordenadas via WebSocket para o **App Cliente**, que exibe o mapa com a posição do entregador.
11. Ao chegar, o entregador confirma a entrega no app. O status final "entregue" é salvo no banco.

### SPOF identificado: Banco de Dados principal

Se o banco de dados principal cair, todos os serviços que dependem de escrita param: não é possível criar pedidos, processar pagamentos ou atualizar status.

**Solução proposta (marcada no diagrama com borda verde):** adicionar uma **réplica de leitura** que assume consultas SELECT, e configurar **failover automático** (ex: PostgreSQL com Patroni ou AWS RDS Multi-AZ) para que, se o banco primário cair, a réplica seja promovida a primário em segundos.

Outro SPOF potencial é o **Load Balancer** — se houver apenas uma instância, ele se torna um gargalo. Em produção, serviços como AWS ELB ou Nginx em modo ativo-passivo resolvem isso com redundância automática.

---

## Parte 4 — Reflexão

### Qual a importância de desenhar a arquitetura de um sistema antes de começar a codificar?

Desenhar a arquitetura antes de codificar funciona como a **planta de uma casa**: ninguém começa a erguer paredes sem saber onde ficam os cômodos, a tubulação e a fiação elétrica. Da mesma forma, um diagrama de componentes define quais partes existem, quais são suas responsabilidades e como elas se comunicam — antes de escrever uma única linha de código.

**Benefícios práticos:**

- **Evita retrabalho caro.** Descobrir na metade do desenvolvimento que o Serviço de Pagamentos precisa se comunicar com o Serviço de Notificações (e que não existe um canal para isso) significa refatorar código já escrito, reescrever testes e atrasar entregas. No diagrama, essa conexão é uma seta que leva 5 segundos para desenhar.

- **Identifica gargalos antes que eles existam.** Olhando o diagrama, percebemos que o Banco de Dados principal é um ponto único de falha. Essa percepção em fase de projeto custa uma linha tracejada no desenho ("adicionar réplica"). Em produção, com 100 mil pedidos por hora, descobrir esse gargalo custaria horas de downtime e milhões em pedidos perdidos.

- **Permite dividir o trabalho.** Com os componentes definidos e suas interfaces claras, cada desenvolvedor (ou equipe) pode trabalhar em um microsserviço de forma independente. O time de pagamentos não precisa esperar o time de notificações terminar — basta que ambos respeitem o contrato da API definido no diagrama.

### Como o diagrama de componentes ajuda na comunicação com stakeholders?

O diagrama é a linguagem franca entre quem programa e quem toma decisões de negócio:

- **Para o gerente de projeto:** mostra quantos componentes precisam ser desenvolvidos, ajudando a estimar prazo e alocar equipes.
- **Para o investidor:** visualiza o sistema como um todo, transmitindo confiança de que a arquitetura foi pensada para escalar (Load Balancer, cache, filas).
- **Para o time de QA:** identifica quais integrações precisam de testes end-to-end (ex: o fluxo completo do pedido passa por 6 componentes).
- **Para um novo desenvolvedor:** serve como mapa de onboarding — em 5 minutos olhando o diagrama, entende onde cada funcionalidade mora.

A arquitetura desenhada não é um documento estático que vai para uma gaveta: é uma **ferramenta viva de comunicação** que alinha todos os envolvidos em torno da mesma visão do sistema.

---

## Arquivos

| Arquivo | Descrição |
|---|---|
| [`diagrama-componentes.svg`](diagrama-componentes.svg) | Diagrama de componentes UML do iFood (exportado em SVG) |
| [`README.md`](README.md) | Documentação completa: escopo, componentes, fluxo, SPOF e reflexão |
