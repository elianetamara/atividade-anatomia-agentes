# Análise

## Erros na execução

### `output_parse_failed` 1

Depois de fazer as alterações de código pedidas no README, a execução teve um erro relacionado a saída (`output_parse_failed`). Foi bem no início da configuração, não estava registrando nada no arquivo de log ainda, era só um teste de funcionamento geral, e o código encerrava a execução. A solução para esse erro foi trocar `max_tokens=2000` por `max_completion_tokens=8000` + `reasoning_effort="low"`, já que o erro pareceu ser relacionado a um truncamento da saída, dado o número de tokens máximo na chamada do modelo.

```
  File "/home/eliane/atividade-anatomia-agentes/.venv/lib/python3.10/site-packages/openai/_base_client.py", line 1213, in request
    raise self._make_status_error_from_response(err.response) from None
openai.BadRequestError: Error code: 400 - {'error': {'message': "Parsing failed. The model generated output that could not be parsed. Please adjust your prompt. See 'failed_generation' for more details.", 'type': 'invalid_request_error', 'code': 'output_parse_failed', 'failed_generation': "We need to run tests. Let's list files."}}
```

Explicação da IA:

> Essa string `failed_generation` consiste puramente em conteúdo de raciocínio — o canal de análise — sem um canal final subsequente. O analisador do Groq detectou o fim da geração enquanto o modelo ainda estava "pensando", não havendo nada para processar como a resposta propriamente dita. A causa raiz habitual é o truncamento: o limite `max_tokens=2000` é consumido pelo raciocínio antes que o modelo chegue a produzir sua mensagem final; assim, o envelope "harmony" fica incompleto, resultando em um erro 400.

Ajustando isso, o agente rodou, mas nenhuma das 15 execuções passou da iteração 1. Todas quebraram na chamada ao modelo, com HTTP 400, antes de qualquer tool ser executada.

### 'tool_use_failed'

Das 15 execuções, 11 delas apresentaram esse erro

```
===== USER =====
encontre e conserte o bug baseado no teste que está falhando em test_inventory.py

----- Iteracao 1 | ERRO (loop encerra) -----
A chamada ao modelo falhou: Error code: 400 - {'error': {'message': 'Tool choice
is none, but model called a tool', 'type': 'invalid_request_error',
'code': 'tool_use_failed',
'failed_generation': '{"name": "tool.list_files", "arguments": {"path": ""}}'}}
```

>> TOOLS / ACI : O system prompt manda responder em texto no formato `tool: TOOL_NAME({...})`. O modelo ignorou isso e emitiu uma chamada de função nativa no canal de tool do harmony format: um JSON estruturado `{"name": ..., "arguments": ...}`
>> Ele leu o `tool:` do prompt como namespace e gerou nomes diferentes em cada uma das execuções (`tool.list_files`, `tool:list_files`, `tool.listdir`, `tool.exec`, etc). O modelo acabou inventando um contrato ao invés de seguir o determinado

>> GUARDRAIL: Como a requisição não declara `tools=[...]`, o Groq vê `tool_choice` como `none` e rejeita a chamada nativa com HTTP 400 (`tool_use_failed`)

>> LOOP: A execução entra no `while True` interno, `iteration` vira 1, `execute_llm_call` lança `BadRequestError`, o `except` registra o erro e faz `break`. O laço encerra na primeira volta, sem segunda iteração.

### 'output_parse_failed' 2

4 execuções obtiveram esse erro.

```
===== USER =====
encontre e conserte o bug baseado no teste que está falhando em test_inventory.py

----- Iteracao 1 | ERRO (loop encerra) -----
A chamada ao modelo falhou: Error code: 400 - {'error': {'message': "Parsing
failed. The model generated output that could not be parsed...",
'type': 'invalid_request_error', 'code': 'output_parse_failed',
'failed_generation': 'We need to run tests. List files.'}}
```

>> THOUGHT: `"We need to run tests. List files."` é o raciocínio cru do modelo, que normalmente fica escondido nas ferramentas prontas. Aqui, ele ficou visível porque o parse  falhou e o Groq devolveu o conteúdo bruto. O modelo pensou  mas não produziu texto/chamada nativa as ferramentas..
---

## Visão geral

### Loop

O `run_coding_agent_loop` tem dois laços: um externo que lê a entrada do usuário, e o interno que é o loop do agente. A cada `iteration` do loop ele chama o LLM, extrai as tool calls, executa elas, devolve as observações e repete. O laço continua caso `extract_tool_invocations` retorne pelo menos uma tool,  daí o bloco `for name, args` rodaria, acrescentaria `tool_result(...)` à conversa e o `while` daria outra voltae seguia assim por diante. Ele para quando o modelo responde sem nenhuma linha `tool:`, ou quando ocorre exceção na chamada.

Nas minhas execuções o laço sempre parou na iteração 1 por exceção sem que nenhuma iteração completa ocorresse.

### Contexto

O resultado de uma tool volta para a conversa em `conversation.append({"role": "user", "content": tool_result_msg})`. Essa linha faz a observação virar contexto, porque na chamada seguinte o LLM recebe a conversa inteira e tem acesso ao que já foi feito. Como nenhuma tool executou nas chamadas feitas, o append nunca foi chamado e o contexto não cresceu em nenhuma das execuções.

### Tools / ACI

O agente usa um protocolo de texto onde o modelo deveria escrever
`tool: list_files({"path": "."})`, e o parsing manual é feito no código, achando as linhas que começam com `tool:`, separando o nome do que está entre parênteses e rodando `json.loads` nos argumentos. Todas as execuções mostram que o modelo não usou esse formato em nenhuma das vezes. Foi emitido tanto tool calling nativo quanto apenas raciocínio.

Vendo as execuções, é possível notar que o parsing de texto é frágil porque depende da obediência do modelo a um formato que contraria o seu próprio treinamento, e qualquer desvio no formato escapa ou quebra. A estratégia de JSON estruturado seria melhor, mas ainda tem a dependência do modelo produzir um JSON válido e disso ser validado. O tool calling nativo deixa o modelo emitir a chamada no canal próprio, e a validação da chamada das tools fica por conta do servidor..

### Thought

O Thought acabou aparecendo devido ao lançamento de erro nas execuções, em `failed_generation: 'We need to run tests.'` / `'We need to run tests. List
files.'`. Na exceção. o modelo raciocinou sobre rodar testes, listar arquivos, mas sem gerar uma ação válida, dado o erro nas ferramentas.

### Guardrail

O agente não tem nenhum. A condição de parada do loop é apenas se o modelo respondeu sem nenhuma linha `tool:`, daí assume-se que terminou a tarefa, sem verificação adicional.
Nesta execução, o agente parou na iteração 1 por erro, sem ler arquivo, sem rodar teste e sem ver o bug. Caso a execução tivesse tido continuidade sem erros, ele poderia ter parado a execução sem realmente ter terminado. Aqui, o loop encerraria com uma resposta sem`tool:` do modelo sem nenhuma confirmação lógica e factível, sem um guardrail que rode testes e exija sucesso antes de parar. Para esse agente, a finalização quer dizer apenas que o modelo parou de pedir tools, e não que o problema foi resolvido.

### Falhas de parsing

O parser de texto nunca chegou a rodar pois as falhas aconteceram antes, na camada da API (HTTP 400), no parser do harmony format do próprio Groq, e não no código do agente. As falhas observadas foram o `tool_use_failed`, em que o modelo emitiu uma tool call nativa que o servidor recusou por não ter `tools` declaradas; e o `output_parse_failed`, em que a saída do modelo não fechou o envelope do harmony format (só veio raciocínio) e o servidor não conseguiu parsear. Caso tivesse recebido texto, provavelmente também teria problema no parser, já que nomes como `tool.list_files`/`tool:list_files` presentes nos logs não batem com as chaves de `TOOL_REGISTRY` (`list_files`) e `tool = TOOL_REGISTRY[name]`, provavelmente daria um `KeyError`.

---
