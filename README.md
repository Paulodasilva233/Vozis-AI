Manual de implementação:

Arquitetura de Integração (Visão Geral)
Plaintext
+-----------------------+           +-----------------------+
|  PLATAFORMA DE VOZ    |           |   MOTOR DE IA (LLM)   |
|  (Ex: Vapi, Twilio)   | <=======> |  (OpenAI / Anthropic) |
|  - Conexão SIP        |   Áudio   |  - Prompt de Sistema  |
|  - Voz Sintética      |           |  - Tool JSON          |
+-----------+-----------+           +-----------------------+
            |
            | Gatilho: Encerramento da Chamada (Function Call)
            | Dispara Payload JSON via Webhook
            v
+-----------------------------------------------------------+
|                      ORQUESTRADOR n8n                     |
|                                                           |
| [ Webhook ] ---> [ Switch / Router ]                      |
|                     |                                     |
|                     +--> [ D1 ] --> CRM + Google Calendar |
|                     |                                     |
|                     +--> [ D2 ] --> CRM (Tarefa Follow-up)|
|                     |                                     |
|                     +--> [ D3 ] --> CRM (Card Perdido)    |
|                                                           |
| [ Google Sheets Node (Todos os caminhos finalizam aqui) ] |
+-----------------------------------------------------------+
Passo a Passo da Implementação:
1. Configuração do Cérebro (Plataforma de IA)

Cole o Prompt Definitivo (Parte 1) no campo de System Instructions do seu modelo.

Configure a Temperature para 0.3 ou 0.4 (garante precisão no roteiro e evita alucinações nas respostas).

Adicione a Ferramenta/Tool colando o JSON Schema (Parte 2). No campo "Endpoint URL" ou "Webhook", cole a URL de produção gerada pelo n8n.

2. Orquestração no n8n

Trigger: Crie um workflow começando com um nó Webhook (Método POST).

Roteamento (Switch Node): Adicione um nó Switch conectado ao Webhook. Crie 3 regras apontando para o valor recebido no JSON: {{ $json.body.status_desfecho }}.

Regra 1: Igual a D1.

Regra 2: Igual a D2.

Regra 3: Igual a D3.

Ações do D1: Conecte a saída D1 a um nó do Google Calendar (para agendar o horário recebido em data_hora_agendamento) e a um nó de CRM para criar/atualizar o negócio.

Gravação Central: Conecte todas as saídas finais a um único nó do Google Sheets (usando a função Append Row). Mapeie as variáveis do JSON (dor, tamanho da equipe, motivo_descarte) para as respectivas colunas da planilha.

3. Teste e Homologação

Ligue para o número da sua IA. Force uma situação de "D3" (diga que não tem interesse e pergunte o preço).

Observações sobre implementação no N8N

Arquitetura do Fluxo no n8n (Atualizada: Omnichannel)
Quando o LLM acionar a ferramenta, ele enviará um payload JSON estruturado para o n8n. No n8n, você montará o fluxo com os seguintes nós:

Webhook Node (Trigger): Recebe o JSON gerado pelo MCP/LLM no final do contato (seja ao desligar a chamada de voz ou ao encerrar a qualificação no chat do WhatsApp).

Switch / Router Node Principal: Uma encruzilhada lógica baseada no campo status_desfecho.

Caminho 1 (Se status_desfecho == "D1" - Agendado):

Nó do Google Calendar: Cria o evento na agenda do executivo humano.

Nó do CRM (HubSpot/Pipedrive): Move o negócio para "Reunião Agendada".

Nó If/Condicional (canal_utilizado):

Se veio da LIGAÇÃO: Nó do WhatsApp dispara: "Olá! Passando para confirmar nossa reunião agendada agora pouco por telefone..."

Se veio do WHATSAPP: Nó do WhatsApp dispara apenas o resumo: "Perfeito! O convite do calendário já está no seu e-mail. Até lá!"

Caminho 2 (Se status_desfecho == "D2" - Follow-up):

Nó do CRM: Cria uma tarefa de follow-up (Task) para a data informada e atualiza o campo de anotações com o resumo da conversa.

Caminho 3 (Se status_desfecho == "D3" - Descarte):

Nó do CRM: Marca o negócio como "Perdido" e insere a variável motivo_descarte gerada pela IA.

Google Sheets Node (Para todos os caminhos convergindo no final):

Grava uma nova linha na sua Planilha de Controle Geral (Dashboard) contendo todas as variáveis recebidas (nome, status, dores, concorrente E o canal_utilizado) para gerar métricas de conversão e comparar a performance do telefone versus WhatsApp.

Ao encerrar a chamada, verifique o histórico de execuções no n8n. Você deve ver o JSON chegando perfeitamente preenchido e caindo na aba correta do Switch Node.
