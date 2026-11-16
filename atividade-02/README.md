# Estilos Arquiteturais — Estudo de Caso

Análise comparativa de estilos arquiteturais para 3 sistemas fictícios: **E-commerce**, **App Bancário** e **Jogo Multiplayer**.

Disciplina de **Arquitetura de Software** — Oxygeni Hub.

---

## Parte 1 — Análise dos Sistemas

### 1. E-commerce (loja online de roupas e eletrônicos)

Plataforma de vendas online com catálogo de milhares de produtos, carrinho de compras, checkout com múltiplas formas de pagamento e logística de entrega. Precisa suportar picos extremos de tráfego (Black Friday, promoções relâmpago) sem sair do ar.

**Requisitos não-funcionais críticos:**

| Requisito | Justificativa |
|---|---|
| Alta escalabilidade | Na Black Friday, o tráfego pode saltar de 10 mil para 500 mil acessos simultâneos em minutos. |
| Tolerância a falhas | Se o serviço de recomendação cair, o catálogo e o checkout devem continuar funcionando. |
| Consistência eventual | O estoque pode levar alguns segundos para sincronizar entre regiões, desde que o pagamento seja consistente. |
| Deploy independente | A equipe de catálogo precisa atualizar filtros sem afetar o checkout. |

---

### 2. App Bancário (banco digital com Pix, transferências e investimentos)

Aplicativo de banco digital que permite transferências via Pix, pagamento de boletos, investimentos em renda fixa e consulta de extratos. Cada transação envolve dinheiro real e é regulada pelo Banco Central.

**Requisitos não-funcionais críticos:**

| Requisito | Justificativa |
|---|---|
| Segurança extrema | Dados financeiros e pessoais são alvos constantes de ataques. Criptografia, autenticação multifator e proteção contra fraude são obrigatórios. |
| Consistência forte (ACID) | Uma transferência de R$ 1.000 deve debitar da conta A **e** creditar na conta B de forma atômica. Se uma etapa falhar, nenhuma deve ser executada. |
| Auditoria completa | Toda transação deve ser rastreável: quem fez, quando, de onde, por quê. Exigência regulatória. |
| Disponibilidade | O app precisa estar disponível 24/7, especialmente para Pix (que funciona fora de horário comercial). |

---

### 3. Jogo Multiplayer (battle royale com 100 jogadores por partida)

Jogo online estilo battle royale onde 100 jogadores entram em um mapa, coletam itens e se enfrentam até restar um vencedor. Cada partida dura de 15 a 25 minutos. O estado do jogo (posição, vida, inventário de cada jogador) precisa ser sincronizado entre todos os clientes em tempo real.

**Requisitos não-funcionais críticos:**

| Requisito | Justificativa |
|---|---|
| Latência ultrabaixa | O jogador precisa ver o resultado de sua ação em menos de 50ms. Acima disso, o jogo "trava" e a experiência é ruim. |
| Tempo real | 100 jogadores simultâneos precisam ver o mesmo estado do mundo ao mesmo tempo. O servidor atualiza o estado 60 vezes por segundo (tick rate 60Hz). |
| Sincronização de estado | Se o jogador A atira no jogador B, ambos os clientes devem concordar sobre o resultado. O servidor é a autoridade (server-authoritative). |
| Escala por sala | Cada partida é isolada. O sistema precisa criar e destruir servidores de jogo dinamicamente conforme a demanda. |

---

## Parte 2 — Decisão e Justificativa

### E-commerce → Microsserviços

**Estilo escolhido:** Arquitetura de Microsserviços

**Justificativa técnica:**

O e-commerce tem domínios claramente separáveis: catálogo, carrinho, pagamento, logística, notificações e recomendação. Cada um desses domínios tem ciclos de deploy diferentes (o catálogo muda diariamente, o pagamento muda a cada semana) e requisitos de escala diferentes (na Black Friday, o catálogo precisa de 50 instâncias enquanto o serviço de devolução precisa de 2).

Com microsserviços:
- **Escala independente:** o serviço de catálogo escala horizontalmente sem gastar recursos com o serviço de devolução.
- **Tolerância a falhas:** se o serviço de recomendação cair, o checkout continua funcionando. O pattern de Circuit Breaker isola as falhas.
- **Deploy independente:** a equipe de pagamentos faz deploy sem afetar o catálogo. Isso acelera o time-to-market.

**Justificativa de negócio:**

Um e-commerce que sai do ar na Black Friday perde milhões. A arquitetura de microsserviços permite que, mesmo que um serviço tenha problemas, os demais continuem gerando receita. Além disso, equipes diferentes podem trabalhar em paralelo, reduzindo o tempo de entrega de features.

**Trade-off aceito:** a complexidade operacional aumenta significativamente. São necessários: API Gateway, service discovery, monitoramento distribuído (Grafana, Jaeger), orquestração de containers (Kubernetes). A equipe precisa estar preparada para essa complexidade.

---

### App Bancário → Monolito Modular

**Estilo escolhido:** Monolito Modular

**Justificativa técnica:**

O app bancário exige **consistência forte acima de tudo**. Quando um usuário faz um Pix de R$ 500, o débito e o crédito devem acontecer na mesma transação ACID. Em microsserviços, essa transação precisaria de padrões como Saga ou Two-Phase Commit, que adicionam complexidade e pontos de falha — inaceitável quando se trata de dinheiro real.

Com monolito modular:
- **Transações ACID nativas:** todas as operações financeiras acontecem dentro do mesmo processo, usando transações do banco de dados diretamente. `BEGIN → DEBIT → CREDIT → COMMIT` — simples e confiável.
- **Superfície de ataque menor:** um único deploy, uma única porta de rede, um único perímetro de segurança. Menos superfície = menos vetores de ataque.
- **Auditoria centralizada:** todos os logs e traces estão no mesmo lugar, facilitando a rastreabilidade exigida pelo Banco Central.

O "modular" é fundamental: o código é organizado em módulos internos com fronteiras claras (módulo de transferências, módulo de investimentos, módulo de cartões), mas todos rodam no mesmo processo. Isso dá a organização interna dos microsserviços sem a complexidade de rede.

**Justificativa de negócio:**

Bancos são regulados. A complexidade operacional dos microsserviços aumenta o risco de falhas distribuídas — e no setor financeiro, uma falha que causa inconsistência de saldo pode gerar processo judicial e multa regulatória. O monolito modular minimiza esse risco com uma arquitetura mais simples de operar, auditar e certificar.

**Trade-off aceito:** a escala horizontal é limitada. Para atender picos de demanda, o monolito escala verticalmente (servidor maior) ou usa réplicas de leitura. Isso é aceitável porque o volume de transações bancárias, mesmo alto, é previsível — não tem "Black Friday" de Pix.

---

### Jogo Multiplayer → Cliente-Servidor com WebSockets + Orientado a Eventos

**Estilo escolhido:** Cliente-Servidor com WebSockets (tempo real) + Arquitetura Orientada a Eventos (backend)

**Justificativa técnica:**

O jogo multiplayer tem um requisito que domina todos os outros: **latência abaixo de 50ms**. Isso elimina HTTP/REST (overhead de handshake) e microsserviços tradicionais (chamadas entre serviços adicionam latência). A solução é uma conexão WebSocket persistente entre cada cliente e o servidor de jogo, com comunicação bidirecional a 60 ticks por segundo.

Com cliente-servidor + WebSockets:
- **Servidor autoritativo:** o game server mantém o estado verdadeiro do mundo. Os clientes enviam inputs (teclas pressionadas), e o servidor calcula o resultado e distribui o novo estado para todos. Isso previne trapaças (cheat).
- **Latência mínima:** WebSocket é uma conexão TCP persistente sem overhead de HTTP. O pacote vai direto, sem handshake a cada mensagem.
- **Escala por sala:** cada partida roda em um game server dedicado. Quando uma partida termina, o servidor é liberado. Com Kubernetes, novos servidores sobem sob demanda.

A parte orientada a eventos cuida do backend que não precisa de tempo real: quando uma partida termina, um evento "partida_finalizada" é publicado no Event Bus (Kafka). Os serviços de Ranking, Perfil e Analytics consomem esse evento de forma assíncrona, sem afetar a próxima partida.

**Justificativa de negócio:**

Para jogadores, lag = frustração = desinstalação. Um jogo com latência ruim perde a base de usuários em semanas. A arquitetura cliente-servidor com WebSockets resolve o problema de latência na origem. O custo de manter game servers dedicados é compensado pelo modelo de negócio (battle pass, skins, moedas virtuais).

**Trade-off aceito:** o game server é stateful (mantém o estado da partida em memória). Se ele cair, a partida é perdida. Isso é aceitável porque uma partida dura ~20 minutos — o jogador entra em outra. Para dados persistentes (ranking, inventário), o backend orientado a eventos garante durabilidade.

---

## Parte 3 — Diagrama do Jogo Multiplayer

O diagrama de blocos do sistema de jogo multiplayer está em [`diagramas/jogo-multiplayer.svg`](diagramas/jogo-multiplayer.svg).

### Resumo da comunicação

```
Clientes ──WebSocket (bidirecional, 60Hz)──► Game Server (sala)
                                                 │
                                          eventos assíncronos
                                                 │
                                                 ▼
                                            Event Bus (Kafka)
                                           ┌─────┼─────┐
                                           ▼     ▼     ▼
                                       Ranking Perfil Analytics
                                           │     │     │
                                           ▼     ▼     ▼
                                     PostgreSQL  Redis  S3
```

### Fluxo: "O jogador entra em uma partida"

1. O **Cliente** abre o jogo e faz login. O **Serviço de Autenticação** valida as credenciais e retorna um token JWT.
2. O jogador clica em "Jogar". O **Cliente** envia uma requisição REST para o **Matchmaking**, que coloca o jogador em uma fila baseada no seu ranking (MMR).
3. Quando 100 jogadores compatíveis estão na fila, o **Matchmaking** cria uma nova sala e inicia um **Game Server** dedicado.
4. Cada cliente recebe o endereço do game server e abre uma **conexão WebSocket** persistente.
5. O **Game Server** roda o game loop a 60Hz: a cada tick (~16ms), recebe os inputs de todos os jogadores, calcula o novo estado do mundo (posições, colisões, dano) e envia o snapshot atualizado para todos os clientes.
6. O cliente recebe o estado e renderiza a cena na tela. Técnicas de **client-side prediction** e **interpolação** suavizam o movimento entre ticks.
7. Quando um jogador é eliminado, o evento é registrado no **Event Bus**. O **Serviço de Ranking** consome o evento e atualiza o MMR.
8. Ao final da partida, o **Game Server** publica "partida_finalizada" no Event Bus com o resultado completo. Os serviços de **Perfil** (XP, moedas) e **Analytics** (métricas de jogo) consomem o evento de forma assíncrona.
9. O game server é destruído. Os clientes voltam ao lobby.

---

## Parte 4 — Reflexão

### Por que não existe uma "bala de prata" na engenharia de software?

Porque cada estilo arquitetural resolve um conjunto de problemas **criando outros**. Não existe uma arquitetura que seja simultaneamente a mais simples, a mais escalável, a mais consistente e a mais barata. Toda escolha é um trade-off:

| Estilo | Resolve | Cria |
|---|---|---|
| **Microsserviços** | Escala independente, deploy isolado, tolerância a falhas | Complexidade operacional (rede, monitoramento, orquestração), consistência distribuída difícil |
| **Monolito Modular** | Simplicidade operacional, transações ACID nativas, menor superfície de ataque | Escala horizontal limitada, deploy é tudo-ou-nada, risco de acoplamento interno |
| **Cliente-Servidor + Eventos** | Latência mínima, tempo real, desacoplamento do backend | Game server stateful (perde dados se cair), complexidade de sincronização de estado |

Se microsserviços fossem a resposta para tudo, o app bancário teria transações distribuídas frágeis e o jogo multiplayer teria latência inaceitável entre chamadas HTTP. Se o monolito fosse a resposta, o e-commerce não conseguiria escalar na Black Friday. Cada contexto demanda uma resposta diferente.

### Como o contexto de negócio influencia a escolha?

A arquitetura não é uma decisão puramente técnica. Três fatores de negócio pesam tanto quanto os requisitos técnicos:

**Orçamento:** microsserviços exigem Kubernetes, observabilidade (Datadog/Grafana), API Gateway, CI/CD por serviço. Uma startup com 3 desenvolvedores não tem orçamento nem equipe para operar isso. O monolito modular permite começar rápido e migrar quando (e se) a escala justificar.

**Tamanho da equipe:** a Lei de Conway diz que a arquitetura do sistema reflete a estrutura de comunicação da equipe. Se há 3 equipes independentes (catálogo, pagamento, logística), microsserviços fazem sentido — cada equipe é dona do seu serviço. Se há 1 equipe de 5 pessoas, microsserviços adicionam overhead de coordenação sem benefício real.

**Tempo de mercado (time-to-market):** se o negócio precisa lançar em 3 meses para captar investimento, um monolito bem feito vai ao ar mais rápido. Refatorar para microsserviços depois, com receita entrando, é uma decisão muito mais segura do que começar com microsserviços e não lançar a tempo.

A pior decisão arquitetural é aquela baseada em hype ("todo mundo está usando microsserviços") em vez de evidências ("nosso sistema precisa de escala independente por domínio?"). A melhor arquitetura é aquela que resolve o problema **de agora** com o menor custo de complexidade, deixando a porta aberta para evoluir quando o contexto mudar.

---

## Arquivos

| Pasta | Conteúdo |
|---|---|
| [`README.md`](README.md) | Documento completo: análise, decisões, justificativas e reflexão |
| [`diagramas/jogo-multiplayer.svg`](diagramas/jogo-multiplayer.svg) | Diagrama de blocos (C4 nível 2) do jogo multiplayer |
| [`slides/apresentacao.pdf`](slides/apresentacao.pdf) | Slides da apresentação oral |
