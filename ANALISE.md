# Análise

## Execução

O agente rodou o loop inteiro em 5 iterações até consertar o bug. O system prompt exigia que o modelo respondesse no formato de texto `tool: NAME({...})` (proibindo o tool calling nativo), e foi esse formato que ele seguiu:

```
----- Iteracao 1 | ACTION -----
tool: list_files({"path": "."})

----- Iteracao 1 | OBSERVATION -----
tool_result({"path": ".../atividade-anatomia-agentes", "files": [... "test_inventory.py" ... "inventory.py" ...]})

----- Iteracao 2 | ACTION -----
tool: read_file({"filename": "test_inventory.py"})

----- Iteracao 2 | OBSERVATION -----
tool_result({... "content": "...assert apply_discount(200, 10) == 180\n"})

----- Iteracao 3 | ACTION -----
tool: read_file({"filename": ".../inventory.py"})

----- Iteracao 3 | OBSERVATION -----
tool_result({... "content": "def apply_discount(price, percent):\n ... return price - percent\n"})

----- Iteracao 4 | THOUGHT -----
O bug está em `inventory.py`. A função `apply_discount` está subtraindo o valor
do desconto diretamente do preço (`price - percent`), em vez de calcular o
desconto percentual corretamente.
tool: edit_file({"path": ".../inventory.py", "old_str": "    return price - percent", "new_str": "    return price - (price * percent / 100)"})

----- Iteracao 4 | OBSERVATION -----
tool_result({"path": ".../inventory.py", "action": "edited"})

----- Iteracao 5 | RESPOSTA FINAL (loop encerra) -----
O bug estava em `inventory.py`. ... Agora `apply_discount(200, 10)` retorna `180`,
como o teste espera. ✅
```

>> LOOP: o laço rodou 5 iterações completas, sendo contínuo da iteração 1 até a 4 porque cada uma das respostas trazia uma linha de chamada a tool, e parou na 5 porque o modelo respondeu só com texto.

>> CONTEXTO: cada OBSERVATION foi anexada à conversa do modelo. Na iteração 4 o modelo já tinha o conteúdo de `inventory.py` em contexto e soube exatamente qual `old_str` substituir por conta disso.

>> TOOLS / ACI: o modelo seguiu o formato de texto para as chamadas de tools com os nomes certos. Na iteração 4 a chamada veio no meio do texto, e o parser ainda assim a extraiu, por conta da varredura linha a linha.

>> THOUGHT: ocorre na iteração 4, visto que o modelo explica o bug antes de agir. Esse raciocínio que normalmente fica escondido, nessa execução a instrumentação expõe.

>> GUARDRAIL: na última iteração o modelo declara que a função "retorna 180, como o teste espera" e encerra sem  ter rodado o teste. Ele acertou por conta própria, sem verificação.

---

## Visão geral

### Loop

O `run_coding_agent_loop` tem dois laços: um externo que lê a entrada do usuário, e o interno que é o loop do agente. A cada `iteration` do loop ele chama o LLM, extrai as tool calls, executa elas, devolve as observações e repete. O laço continua caso `extract_tool_invocations` retorne pelo menos uma tool, daí o bloco `for name, args` roda, acrescenta `tool_result(...)` à conversa e o `while` dá outra volta, seguindo assim por diante. Ele para quando o modelo responde sem nenhuma linha `tool:`, ou quando ocorre exceção na chamada.

Na execução dá pra ver isso acontecer. Foram 5 iterações: da 1 à 4 o modelo devolveu uma `tool:`, então o laço seguiu; na iteração 5 ele respondeu só com texto, caiu no `if not tool_invocations: break` e o loop encerrou.

### Contexto

O resultado de uma tool volta para a conversa em `conversation.append({"role": "user", "content": tool_result_msg})`. Essa linha faz a observação virar contexto, porque na chamada seguinte o LLM recebe a conversa inteira e tem acesso ao que já foi feito. A cada iteração da execução o `tool_result` era anexado, então quando o modelo chegou na iteração 4 ele já tinha o conteúdo de `inventory.py` (lido na iteração 3) no contexto, montou o `edit_file` com o `old_str` exato (`return price - percent`).

### Tools / ACI

O agente usa um protocolo de texto onde o modelo deve escrever
`tool: list_files({"path": "."})`, e o parsing manual é feito no código, achando as linhas que começam com `tool:`, separando o nome do que está entre parênteses e rodando `json.loads` nos argumentos. Para o modelo seguir esse formato, o system prompt precisa ser explícito, proibindo o tool calling nativo; com isso ele usou o texto com os nomes certos.

O parsing de texto é frágil porque depende da obediência do modelo a um formato que contraria o seu próprio treinamento, e qualquer desvio no formato escapa ou quebra. A estratégia de JSON estruturado seria melhor, mas ainda tem a dependência do modelo produzir um JSON válido e disso ser validado. O tool calling nativo deixa o modelo emitir a chamada no canal próprio, e a validação da chamada das tools fica por conta do servidor.

### Thought

Na iteração 4 o modelo escreve o raciocínio explicando o bug (`a função está subtraindo o valor do desconto diretamente do preço...`) antes de emitir o `edit_file`, e na iteração 5 a resposta final também é Thought, sem tool. Nas iterações 1 a 3 o bloco THOUGHT registrado é igual ao ACTION, porque o modelo não escreveu nenhum raciocínio, só a chamada; a instrumentação loga o `assistant_response` inteiro como THOUGHT de qualquer jeito, então quando não há prosa o THOUGHT e o ACTION ficam duplicados.

### Guardrail

O agente não tem nenhum. A condição de parada do loop é apenas se o modelo respondeu sem nenhuma linha `tool:`, daí assume-se que terminou a tarefa, sem verificação adicional. A execução mostra esse problema quando o agente edita o `inventory.py` e, na iteração 5, declara que `apply_discount(200, 10)` "retorna 180, como o teste espera ✅", sem rodar o teste, dado que eles nem tem tool pra isso. A correção no código foi certa, mas apenas por raciocínio. Para o agente, "terminar" significa que o modelo parou de pedir tools, e não que o teste passou.

### Falhas de parsing

Na execução o parser de texto rodou e funcionou para todas as chamadas de tool.
