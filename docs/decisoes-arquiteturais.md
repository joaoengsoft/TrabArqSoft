Decisões Arquiteturais – Padrão Saga em Microsserviços
1. Introdução

Este documento apresenta as decisões arquiteturais adotadas para implementar o padrão Saga em uma plataforma de comércio eletrônico baseada em microsserviços.

O cenário envolve três serviços independentes:

Order Service: gerenciamento de pedidos.
Inventory Service: gerenciamento e reserva de estoque.
Payment Service: processamento de pagamentos por gateways financeiros externos.

Como cada serviço possui seu próprio banco de dados, não é possível utilizar uma única transação ACID envolvendo todos os componentes.

Para solucionar esse problema, foi escolhido o padrão Saga com Orquestração, permitindo coordenar transações locais e executar operações compensatórias quando ocorrerem falhas.

2. Decisão Arquitetural: Saga por Orquestração

A arquitetura adotará um Saga Orchestrator responsável por controlar o fluxo de execução da compra.

O orquestrador será responsável por:

Iniciar a transação distribuída.
Enviar comandos aos microsserviços.
Receber eventos de confirmação ou falha.
Registrar o estado atual da Saga.
Controlar retentativas.
Iniciar operações compensatórias.
Garantir a recuperação do processo após falhas.
Justificativa da escolha

A orquestração foi escolhida porque o processo de compra possui etapas sequenciais e dependentes.

O pagamento somente deverá ser processado depois da confirmação da reserva de estoque. Caso o pagamento seja recusado, o sistema precisará liberar o estoque e cancelar o pedido.

Com um orquestrador central, o fluxo de execução fica mais simples de acompanhar, testar e monitorar.

3. Estratégia de Transações e Compensações

O processo de compra será composto pelas seguintes operações:

Etapa	Transação principal	Compensação
1	CreateOrder	CancelOrder
2	ReserveStock	ReleaseStock
3	ProcessPayment	RefundPayment ou VoidPayment

As compensações serão executadas quando uma etapa posterior falhar e houver necessidade de desfazer os efeitos das operações anteriores.

Fluxo de sucesso
O Order Service cria um pedido com status PENDING.
O Inventory Service reserva os produtos.
O Payment Service processa e aprova o pagamento.
O Order Service atualiza o pedido para APPROVED.
O orquestrador registra a Saga como COMPLETED.
Fluxo de falha

Se o pagamento for recusado:

O Payment Service publica o evento PaymentRejected.
O orquestrador identifica a falha.
O Inventory Service recebe ReleaseStock.
O Order Service recebe CancelOrder.
O pedido passa para CANCELLED.
A Saga passa para COMPENSATED.

Caso o pagamento tenha sido efetivamente capturado e uma falha posterior aconteça, será necessário executar RefundPayment ou VoidPayment, conforme as possibilidades oferecidas pelo gateway financeiro.

4. Riscos Arquiteturais Mitigados

A arquitetura considera os principais riscos associados à comunicação distribuída.

Risco	Descrição	Estratégia de mitigação
Mensagens duplicadas	O broker pode entregar a mesma mensagem mais de uma vez.	Idempotent Consumer e identificadores únicos.
Perda de mensagens	O serviço pode falhar antes de publicar um evento.	Transactional Outbox.
Timeout de rede	A resposta pode não chegar mesmo quando a operação foi executada.	Consulta de estado e Retry controlado.
Cobrança duplicada	Uma solicitação de pagamento pode ser repetida.	Chave de idempotência no gateway financeiro.
Estoque inconsistente	Uma reserva pode permanecer ativa após falha no pagamento.	Transação compensatória ReleaseStock.
Falha na compensação	O serviço responsável pela reversão pode ficar indisponível.	Retry, DLQ e monitoramento.
Eventos fora de ordem	Uma mensagem antiga pode chegar depois de outra mais recente.	Validação de estado e versionamento.
Indisponibilidade de dependências	O gateway financeiro pode ficar temporariamente indisponível.	Circuit Breaker e recuperação posterior.
Falha do orquestrador	O coordenador pode parar durante uma Saga.	Persistência do estado e recuperação após reinicialização.
5. Idempotência – Idempotent Consumer

A arquitetura implementará o padrão Idempotent Consumer em todos os microsserviços.

Esse mecanismo garante que uma mesma mensagem não produza efeitos colaterais repetidos.

Cada mensagem terá os seguintes identificadores:

messageId: identificação única da mensagem.
sagaId: identificação da transação distribuída.
orderId: identificação do pedido.
idempotencyKey: chave para impedir a repetição de operações.

Cada serviço manterá uma tabela chamada processed_messages.

Campo	Descrição
message_id	Identificador da mensagem.
saga_id	Identificador da Saga.
consumer	Serviço que processou a mensagem.
processed_at	Data e horário do processamento.
Funcionamento
O consumidor recebe uma mensagem.
Verifica se o identificador já foi processado.
Se já existir, não executa novamente a operação.
Se for uma mensagem nova, executa a transação local.
Registra o identificador como processado na mesma transação do banco local.

O banco deverá possuir uma restrição UNIQUE para evitar processamento duplicado em situações de concorrência.

Para operações financeiras, a mesma chave de idempotência deverá ser enviada ao gateway de pagamento.

6. Tratamento de Timeouts

Timeouts representam um risco importante em arquiteturas distribuídas.

Uma solicitação pode ter sido executada com sucesso no serviço de destino, mas a resposta pode não chegar ao solicitante.

Exemplo: timeout no pagamento
O Saga Orchestrator solicita o processamento do pagamento.
O Payment Service encaminha a operação ao gateway.
O gateway aprova a cobrança.
A resposta sofre um atraso ou se perde na rede.
O orquestrador identifica o timeout.

Nesse cenário, o sistema não deverá assumir imediatamente que o pagamento foi recusado.

Estratégia adotada
Manter a Saga em PAYMENT_PENDING.
Consultar o status da transação no gateway.
Utilizar a mesma chave de idempotência nas consultas e tentativas permitidas.
Se aprovado, continuar a compra.
Se recusado, iniciar as compensações.
Se permanecer desconhecido, realizar novas consultas e encaminhar para análise manual quando necessário.

O objetivo é impedir que um pedido seja cancelado enquanto uma cobrança válida ainda estiver em processamento.

7. Estratégia de Retry – Retentativas

Será utilizada a estratégia Exponential Backoff para falhas temporárias de comunicação.

O intervalo entre tentativas aumentará progressivamente.

Tentativa	Intervalo
1ª tentativa	1 segundo
2ª tentativa	2 segundos
3ª tentativa	4 segundos
4ª tentativa	8 segundos

Também será utilizado jitter, adicionando uma pequena variação aleatória aos intervalos para evitar sobrecarga simultânea dos serviços.

Quando utilizar Retry?
Timeout de rede.
Erros temporários de conexão.
Indisponibilidade momentânea de serviços.
Falhas transitórias no Message Broker.
Quando não utilizar Retry automaticamente?
Pagamento definitivamente recusado.
Produto sem estoque.
Pedido inválido.
Dados incorretos enviados pelo cliente.

Esses casos representam falhas de negócio, não problemas temporários de infraestrutura.

As retentativas deverão respeitar limites de quantidade e tempo para evitar processamento infinito.

8. Estratégia Circuit Breaker

O padrão Circuit Breaker será utilizado para proteger os microsserviços contra dependências indisponíveis.

O mecanismo possui três estados principais:

CLOSED (Fechado)

As requisições são encaminhadas normalmente.

OPEN (Aberto)

Quando a quantidade de falhas ultrapassa o limite configurado, o circuito é aberto e novas chamadas são temporariamente bloqueadas.

Isso evita sobrecarregar um serviço que já está apresentando problemas.

HALF-OPEN (Semiaberto)

Após determinado intervalo, algumas requisições de teste são permitidas.

Se o serviço responder corretamente, o circuito retorna para CLOSED.

Caso contrário, retorna para OPEN.

Exemplo prático

Se o gateway financeiro apresentar indisponibilidade:

O Payment Service identifica falhas consecutivas.
O Circuit Breaker abre o circuito.
Novas tentativas de comunicação são temporariamente bloqueadas.
A Saga permanece persistida em estado pendente.
Após o período de espera, o circuito permite chamadas de teste.
Quando o gateway recuperar sua disponibilidade, o processamento poderá continuar.

O Circuit Breaker não cancela automaticamente a compra nem substitui o mecanismo de compensação.

9. Dead Letter Queue (DLQ)

A Dead Letter Queue será utilizada para armazenar mensagens que não puderam ser processadas após o esgotamento das tentativas configuradas.

Sua finalidade é impedir que mensagens problemáticas sejam reenviadas indefinidamente, prejudicando o processamento das demais operações.

Exemplo prático

O Inventory Service recebe uma mensagem ReserveStock, mas apresenta uma falha persistente no banco de dados.

Após atingir o limite de tentativas, a mensagem será encaminhada para a DLQ.

Informações armazenadas

A mensagem deverá preservar:

messageId.
sagaId.
orderId.
Serviço de origem.
Tipo da operação.
Payload original.
Quantidade de tentativas.
Data e horário da falha.
Motivo do erro.
Procedimento de tratamento
A mensagem é encaminhada para a DLQ.
O sistema registra o erro.
A equipe de operação recebe um alerta.
A causa da falha é investigada.
O problema é corrigido.
A mensagem pode ser reprocessada de maneira controlada.
O estado da Saga é atualizado após a confirmação do resultado.

O reprocessamento deverá preservar os identificadores originais e respeitar as regras de idempotência.

Importante: mensagens na DLQ não significam que a Saga foi concluída ou compensada. O estado deverá permanecer pendente, em compensação ou em análise manual até que o problema seja resolvido.

10. Transactional Outbox

Será utilizado o padrão Transactional Outbox para garantir maior confiabilidade na publicação dos eventos.

O problema que esse padrão resolve ocorre quando um microsserviço atualiza seu banco de dados, mas falha antes de publicar o evento correspondente no Message Broker.

Funcionamento
O serviço executa a operação de negócio.
Atualiza seus dados no banco local.
Registra o evento na tabela outbox na mesma transação.
Um processo independente identifica os eventos pendentes.
O processo publica os eventos no Message Broker.
Após confirmação da publicação, o evento é marcado como enviado.

Esse mecanismo reduz o risco de inconsistência entre o banco de dados e o barramento de mensagens.

Como a publicação pode ocorrer mais de uma vez, os consumidores continuarão utilizando idempotência.

11. Persistência e Recuperação da Saga

O Saga Orchestrator manterá uma tabela saga_instance para registrar o estado de cada transação distribuída.

Campos principais:

Campo	Descrição
saga_id	Identificador da Saga.
order_id	Pedido relacionado.
current_step	Etapa atual.
status	Estado da transação.
retry_count	Número de tentativas.
last_error	Último erro identificado.
created_at	Data de criação.
updated_at	Última atualização.
timeout_at	Prazo da etapa atual.
version	Controle de concorrência.

Estados possíveis:

STARTED.
STOCK_PENDING.
STOCK_RESERVED.
PAYMENT_PENDING.
COMPENSATING.
COMPLETED.
COMPENSATED.
MANUAL_REVIEW.

Após uma reinicialização, o orquestrador deverá recuperar as Sagas incompletas e continuar a execução a partir do último estado persistido, verificando antes o resultado das operações que possam ter ficado com confirmação desconhecida.

12. Observabilidade e Monitoramento

A equipe SRE deverá monitorar a saúde da arquitetura utilizando métricas, logs e rastreamento distribuído.

Indicadores importantes:

Quantidade de Sagas iniciadas.
Quantidade de compras concluídas.
Quantidade de compras compensadas.
Tempo médio de execução da Saga.
Quantidade de pagamentos recusados.
Quantidade de timeouts.
Quantidade de mensagens na DLQ.
Quantidade de retentativas por serviço.
Quantidade de Sagas em estado pendente.
Tempo de indisponibilidade dos serviços.

As ferramentas recomendadas são:

Prometheus: coleta de métricas.
Grafana: dashboards e alertas.
OpenTelemetry: instrumentação e rastream