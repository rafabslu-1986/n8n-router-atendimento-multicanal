# Router de Atendimento com IA - WhatsApp + Instagram

Roteador de atendimento em n8n que recebe mensagens de WhatsApp e Instagram, classifica a intencao com Claude (API da Anthropic) e encaminha cada conversa para o time correto, registrando todos os leads em uma tabela consultavel.

## Problema que resolve

Operacao com dois canais de entrada e triagem manual: alguem le cada mensagem e decide para quem encaminhar. Isso gera atraso na resposta, perda de lead e nenhum historico consultavel. O fluxo elimina a triagem manual e cria o registro.

## Arquitetura

Entrada: dois webhooks POST independentes, router/whatsapp e router/instagram.

Normalizacao: cada canal passa por um no Set que padroniza os campos para canal, contato_id, nome, mensagem e recebido_em. A partir dai o resto do fluxo e agnostico de canal, o que permite plugar um terceiro canal sem alterar a logica de decisao.

Classificacao: no Text Classifier usando Claude Sonnet 4.6, com quatro categorias descritas em portugues (reserva, preco, suporte, humano) e fallback para other.

Roteamento: cada categoria cai em um no Set que define rota, agente e sla_minutos. Mensagens fora das categorias vao para um no de descarte.

Persistencia: todos os caminhos convergem em um no Data Table que grava o lead.

Resposta: no Respond to Webhook devolve JSON com status, canal, rota, agente e SLA, para o canal de origem confirmar o recebimento.

## Politica de falha

O classificador e o modelo tem retentativa automatica: ate 3 tentativas com 2 segundos de intervalo. Se a IA continuar indisponivel, a saida de erro do classificador cai em um no de fallback que marca rota igual a humano_fallback, agente igual a fila_humana e sla_minutos igual a 2, e escreve no campo observacao o numero da execucao do n8n junto com a mensagem retornada pelo provedor.

O principio e que indisponibilidade de IA nunca deve virar lead perdido. Na duvida, o lead vira atendimento humano prioritario e fica rastreavel.

## Chave da API

A credencial da Anthropic e do cliente, cadastrada na instancia do cliente. O consumo de tokens e faturado na conta dele, com controle total de limites e de rotacao da chave. O fluxo nao embute nenhuma chave.

## Esquema da Data Table Leads de Atendimento

O export de workflow do n8n nao inclui o esquema de Data Tables, apenas a referencia ao id. Recrie a tabela com estas colunas antes de importar o fluxo.

| Coluna | Tipo |
| --- | --- |
| recebido_em | string |
| canal | string |
| contato_id | string |
| nome | string |
| mensagem | string |
| rota | string |
| agente | string |
| sla_minutos | number |
| observacao | string |

## Como rodar

Suba o n8n com versao fixa e volume nomeado:

```
docker run -d --restart unless-stopped --name n8n -p 5678:5678 -v n8n_data:/home/node/.n8n docker.n8n.io/n8nio/n8n:2.32.7
```

Depois, na interface em http://localhost:5678:

1. Crie a Data Table Leads de Atendimento com as colunas da tabela acima.
2. Importe o arquivo workflow/router-atendimento.json.
3. Cadastre sua credencial da Anthropic e vincule ao no do modelo.
4. Reaponte o no Registrar Lead para a Data Table recem-criada.
5. Execute em modo de teste e dispare um POST para o webhook de WhatsApp.

Exemplo de payload:

```
{"from": "5551999999999", "name": "Ana Souza", "message": "Oi, voces tem quarto livre para o feriado?"}
```

## Testes executados

| Cenario | Canal | Rota esperada | Resultado |
| --- | --- | --- | --- |
| Pedido de reserva | WhatsApp | reserva | ok |
| Spam de investimento | WhatsApp | descartado | ok |
| Pergunta de preco | Instagram | preco | ok |
| Falha simulada do modelo | WhatsApp | humano_fallback | ok em 5,1s |
| Problema no quarto | WhatsApp | suporte | ok |
| Pedido explicito de atendente | WhatsApp | humano | ok |

Todos os seis desfechos foram validados de ponta a ponta e gravados na Data Table. Os dados usados nos testes sao ficticios.

## Estrutura do repositorio

```
workflow/router-atendimento.json   export do fluxo
docs/canvas.png                    canvas com os 18 nos
docs/arquitetura.png               diagrama da solucao
docs/leads-data-table.png          tabela de leads preenchida
docs/leads-fallback.png            detalhe do registro de fallback
```

## Aviso de seguranca

Nunca versione o arquivo backup-creds.json nem o arquivo config do volume do n8n. Juntos eles expoem credenciais em texto claro. O .gitignore deste repositorio ja cobre esses nomes.
