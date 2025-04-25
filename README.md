# Documentação: Cristal - Agente de Vendas Automatizado (v2.1)

## Visão Geral
O fluxo "Cristal | Nova Versão 2.1" é um agente automatizado de vendas e atendimento ao cliente para o Guia de Tinturas de Matheus Colombo. Este sistema utiliza IA para realizar atendimentos via WhatsApp, gerenciar leads, processar mensagens de clientes, e fornecer respostas contextualizadas utilizando RAG (Retrieval Augmented Generation).

## Arquitetura do Sistema

O fluxo é composto por várias seções principais:

1. **Recebimento de Mensagem**: Recebe mensagens via webhook da Evolution API.
2. **Tratamento de Mensagem**: Processa e classifica a mensagem recebida.
3. **Verificação de Processamento em Andamento**: Verifica se já existe um processamento para o usuário.
4. **Concatenação de Mensagens**: Agrupa mensagens recebidas em um curto período.
5. **Armazenamento e Gestão de Leads**: Verifica se o cliente já existe ou cria um novo registro.
6. **Processamento com IA**: Utiliza modelo GPT para gerar respostas personalizadas.
7. **Envio de Respostas**: Divide e envia as respostas por blocos.

## Componentes Principais

### 1. Webhook (Entrada de Dados)
- **Node**: `Webhook`
- **Função**: Ponto de entrada do fluxo via Evolution API, recebendo mensagens do WhatsApp.
- **Endpoint**: `/agente-cristalv2`

### 2. Verificação e Tratamento de Mensagens
- **Nodes**: `Msg recebida?`, `Switch`, `If1`, `If2`
- **Função**: Filtra mensagens de grupos, verifica se o lead existe e se o atendimento está ativo.

### 3. Processamento de Diferentes Tipos de Mídia
- **Nodes**: `Switch1`
- **Função**: Separa mensagens de texto, áudio e imagem para processamento adequado.
- **Caminho de Áudio**: Converte áudio para texto via transcrição da OpenAI.

### 4. Gerenciamento de Concorrência
- **Nodes**: `GET`, `RUN em andamento?`, `INSERT`, `15seg.`, `GET1`, `Filter`
- **Função**: Evita processamento duplicado de mensagens recebidas em sequência.
- **Mecânica**: Armazena mensagens em tabela temporária, aguarda 15 segundos, compara mensagens para evitar duplicação.

### 5. Gestão de Leads e Clientes
- **Nodes**: `Buscar Cliente`, `Cliente Existe?`, `Criar Cliente`, `Supabase`
- **Função**: Verifica existência do cliente no banco de dados e atualiza seus dados.
- **Tabela Principal**: `v3_leads_cristal`

### 6. Geração de Respostas com IA
- **Nodes**: `AI Agent`, `OpenAI Chat Model`, `Postgres Chat Memory`
- **Função**: Processa mensagens do cliente utilizando GPT e histórico de conversas.
- **Modelo**: GPT-4o (principal) e GPT-4o-mini (suporte)
- **Memória**: Histórico de conversa armazenado em PostgreSQL

### 7. Sistema RAG (Retrieval Augmented Generation)
- **Nodes**: `Supabase Vector Store`, `Embeddings OpenAI`
- **Função**: Fornece contexto específico sobre o Guia de Tinturas, técnicas de vendas e comunicação.
- **Tabela de Vetores**: `cristal_v3_rag`

### 8. Tools Especializadas
- **Nodes**: `cristal_v3_rag`, `toll acesso_cademi`, `email_usuario`, `nome_usuario`, `leads_ticto`, `transferencia`
- **Função**: Ferramentas específicas que o agente IA pode invocar para tarefas especializadas.
- **Capacidades**: Acesso à área de membros, atualização de dados de leads, transferência para atendimento humano.

### 9. Envio de Respostas
- **Nodes**: `Set1`, `Item Lists`, `Split In Batches`, `Enviar Texto1`
- **Função**: Divide as respostas longas em blocos e envia para o WhatsApp do usuário.
- **Método**: Integração com API DataCrazy para envio de mensagens.

## Fluxo de Dados

1. **Recebimento**: Mensagem recebida via webhook.
2. **Validação**: Filtro de grupos e verificação de atendimento ativo.
3. **Processamento**: 
   - Para texto: Processamento direto.
   - Para áudio: Transcrição via OpenAI.
   - Para imagem: Processamento específico com descrição.
4. **Verificação de Concorrência**: Sistema de prevenção de processamento duplicado.
5. **Gestão de Leads**: Atualização ou criação de registro do cliente.
6. **Geração de Resposta**: Processamento da mensagem pelo agente de IA.
7. **Envio**: Divisão da resposta em blocos e envio sequencial.

## Sistema de Prompts

O agente Cristal opera com um sistema de prompt elaborado que inclui:

1. **Identidade**: Definida como consultora sênior de vendas de Matheus Colombo.
2. **Metodologia**: Utiliza o método SPIN Selling para qualificação de leads.
3. **Fluxo de Vendas**: Segue etapas definidas de identificação, exploração, educação estratégica, apresentação e fechamento.
4. **Base de Conhecimento**: Acesso a informações sobre o Guia de Tinturas e técnicas.
5. **Gatilhos**: Respostas pré-definidas para situações específicas como objeções de preço, dúvidas sobre eficácia, etc.

## Banco de Dados

### Tabelas Principais
1. **v3_leads_cristal**: Armazena informações dos leads.
2. **v3_msgTEMP_Cristal**: Tabela temporária para controle de concorrência de mensagens.
3. **cv2_n8n_chat_histories**: Armazena histórico de conversas para memória do chatbot.
4. **cristal_v3_rag**: Armazena vetores para o sistema RAG.
5. **compras_status_guia**: Informações de compras e pagamentos.

### Campos Relevantes em v3_leads_cristal
- `lead_name`: Nome do cliente
- `number`: Número do WhatsApp
- `email`: Email do cliente
- `atendimento_ativo`: Status do atendimento
- `hora_ultima_mensagem`: Timestamp da última interação
- `ultima_msg_by`: Origem da última mensagem (usuário ou assistente)

## Ferramentas Especializadas (Tools)

1. **cristal_v3_rag**: Sistema de busca em base de conhecimento para informações sobre produtos e técnicas de venda.
2. **acesso_cademi**: Verificação e liberação de acesso à área de membros.
3. **email_usuario**: Salva o email informado pelo cliente.
4. **nome_usuario**: Salva o nome informado pelo cliente.
5. **leads_ticto**: Recupera códigos de boleto ou PIX para pagamentos.
6. **transferencia**: Transfere atendimento para um humano em casos específicos.

## Fluxo de Gerenciamento de Concorrência

Um aspecto importante deste sistema é o gerenciamento de concorrência que evita processamento duplicado de mensagens:

1. Ao receber uma mensagem, verifica se há um processamento em andamento (`RUN`).
2. Se não há, insere a mensagem na tabela temporária.
3. Aguarda 15 segundos para agrupar mensagens próximas.
4. Compara mensagens para verificar duplicação.
5. Registra início de processamento.
6. Processa a mensagem com o agente IA.
7. Finaliza o processamento removendo registro temporário.

## Comportamentos Específicos do Agente

O agente Cristal é programado para:

1. **Sempre coletar nome e email** do cliente quando não disponíveis.
2. **Nunca revelar** que é uma IA, identificando-se como parte da equipe de Matheus Colombo.
3. **Evitar conteúdo técnico** do guia, preservando o valor comercial.
4. **Reconhecer sinais de compra** e fornecer o link de venda no momento apropriado.
5. **Transferir atendimento** para humanos em casos específicos.
6. **Salvar dados do usuário** como nome e email quando informados.

## Links e Integrações

1. **Link de Compra**: `https://payment.ticto.app/OA0FC6C5C?utm_source=agente&utm_medium=ia&utm_campaign=rec&utm_term=direto&sck=agente`
2. **Grupo de Alunos**: `https://chat.whatsapp.com/JiUn2dd1TFlLCfmL8UmUhu`
3. **Tutorial de Acesso**: `https://youtube.com/shorts/0Y3g7_Q4c2Y`

## Manutenção e Operação

Para manter o sistema funcionando corretamente:

1. **Monitorar o banco de dados** para crescimento excessivo da tabela temporária.
2. **Atualizar a base de conhecimento RAG** com novas informações sobre produtos ou técnicas.
3. **Verificar regularmente logs** de transferências para identificar padrões de falha.
4. **Ajustar prompts do sistema** conforme necessidades de venda evoluem.

## Resolução de Problemas

### Problemas Comuns:
1. **Mensagens não processadas**: Verificar webhook e tabela temporária.
2. **Respostas duplicadas**: Verificar sistema de concorrência.
3. **Falhas na transcrição de áudio**: Verificar integração com OpenAI.
4. **Transferências frequentes**: Melhorar base de conhecimento ou ajustar prompt.

### Ferramentas de Diagnóstico:
1. **Logs de execução**: Disponíveis no n8n e no node `Execution Data`.
2. **Tabela temporária**: Verificar registros presos com `processada = true`.
3. **Histórico de chat**: Verificar tabela `cv2_n8n_chat_histories`.

## Melhorias Futuras

1. **Incorporação de mais modelos de linguagem** para redundância.
2. **Expansão do RAG** com mais conteúdo sobre produtos e técnicas.
3. **Implementação de análise de sentimento** para detectar clientes insatisfeitos.
4. **Sistema de alerta** para notificar sobre transferências ou vendas realizadas.
5. **Dashboard de métricas** para acompanhamento de desempenho de vendas.

---

## Anexo: Estrutura de Prompt do Sistema

O prompt de sistema que guia o comportamento do agente Cristal inclui:

1. **Identidade**: Cristal, consultora sênior de vendas do Matheus Colombo.
2. **Tom**: Conversacional, acolhedor, confiante; "humano" e empático.
3. **Estilo**: 2-4 frases curtas, uso moderado de emojis (🌿 🍃 ✨ 💚).
4. **KPIs internos**: Resposta imediata, conversão ≥ 73%, satisfação ≥ 90%.
5. **Fluxo de vendas**: Identificação → Exploração (SPIN) → Educação Estratégica → Apresentação → Fechamento → Follow-up.
6. **Regras específicas**: Nunca revelar que é IA, nunca fornecer receitas, usar apenas 1 emoji por mensagem, evitar mensagens longas.

Esta documentação fornece uma visão abrangente do fluxo "Cristal | Nova Versão 2.1", suas funcionalidades, componentes e operação, permitindo compreensão completa para desenvolvimento, manutenção e expansão do sistema.