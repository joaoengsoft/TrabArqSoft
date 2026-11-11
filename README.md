# TrabArqSoft
Trabalho de Arquitetura de Software - Desenho e modelagem padrão saga
Arquitetura de Microsserviços – Padrão Saga
1. Visão geral

Este projeto apresenta uma proposta de arquitetura distribuída para um sistema de comércio eletrônico que está migrando de um monólito para microsserviços independentes.

O objetivo é implementar o padrão Saga com Orquestração, garantindo consistência eventual entre os serviços de Pedidos, Estoque e Pagamentos, sem depender de transações distribuídas ACID.

2. Serviços envolvidos
Order Service: responsável pela criação, gerenciamento e cancelamento dos pedidos.
Inventory Service: responsável pela disponibilidade, reserva e liberação dos produtos.
Payment Service: responsável pelo processamento dos pagamentos por meio de gateways financeiros externos.
Saga Orchestrator: responsável pela coordenação das transações, controle de estado e execução das compensações.

Cada microsserviço possui seu próprio banco de dados, seguindo o padrão Database per Service.

3. Estratégia arquitetural escolhida

Foi adotado o padrão Saga por Orquestração.

A escolha ocorre porque o processo de compra possui etapas sequenciais e dependentes, exigindo controle centralizado do fluxo de execução e das operações compensatórias.

O orquestrador coordena os comandos enviados aos microsserviços, recebe os eventos de resposta e determina a próxima etapa da transação.

Essa abordagem facilita a rastreabilidade, o tratamento de falhas e a manutenção do processo de compra.

4. Fluxo de sucesso
O cliente solicita um pedido.
O Order Service registra o pedido como PENDING.
O Saga Orchestrator inicia a transação.
O Inventory Service reserva os produtos.
O Payment Service processa e aprova o pagamento.
O Order Service atualiza o pedido para APPROVED.
O orquestrador registra a Saga como COMPLETED.
5. Fluxo de falha e compensação

Quando o Payment Service recusa o pagamento após a reserva do estoque:

O Payment Service publica PaymentRejected.
O Saga Orchestrator identifica a falha.
O Inventory Service recebe ReleaseStock e libera os produtos reservados.
O Order Service recebe CancelOrder e atualiza o pedido para CANCELLED.
O orquestrador registra a Saga como COMPENSATED.
6. Resiliência e confiabilidade

A solução utiliza:

Idempotent Consumer para evitar processamento duplicado.
Retry com Exponential Backoff para falhas temporárias.
Circuit Breaker para proteger serviços indisponíveis.
Dead Letter Queue para mensagens que excedem o limite de tentativas.
Transactional Outbox para garantir a publicação confiável dos eventos.
Controle persistente de estado da Saga para recuperação após falhas.
7. Consistência eventual

Como cada serviço possui seu próprio banco de dados, as atualizações não acontecem simultaneamente.

A Saga garante que o processo continue até a conclusão das etapas ou até a execução das operações compensatórias.

8. Diagramas

Os diagramas estão disponíveis na pasta /diagrams:

saga-architecture-overview.png: visão geral da arquitetura.
sequence-success-flow.png: sequência de execução bem-sucedida.
sequence-compensation-flow.png: sequência de falha e compensação.

Os arquivos .puml também estão disponíveis para edição.

9. Documentação técnica

As decisões arquiteturais, políticas de idempotência, tratamento de timeouts, estratégias de Retry, Circuit Breaker e Dead Letter Queue estão documentadas em:

/docs/decisoes-arquiteturais.md

10. Conclusão

A utilização do padrão Saga com Orquestração permite coordenar transações distribuídas entre microsserviços independentes, mantendo a autonomia dos serviços e garantindo consistência eventual.

A solução também proporciona maior resiliência diante de falhas de comunicação, indisponibilidade de serviços e recusas de pagamento, evitando que pedidos e reservas de estoque permaneçam inconsistentes sem tratamento.
