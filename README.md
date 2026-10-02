# Atividade: anatomia de um agente

Na aula vimos o loop de um agente a partir de um trace pronto: `apply_discount` tinha um teste falhando, e o agente leu o código, rodou o teste, formulou uma hipótese, corrigiu e confirmou. Aqui você vai gerar o seu. O objetivo é instrumentar o código de um agente (já implementado), rodar contra um código com bug e analisar o trace identificando loop, contexto, tools (ACI), thought e guardrail.

Não importa se o agente conserta o bug. O que vale é como você identifica e explica os componentes.

## Sobre o código

O `agent.py` é uma simplificação de um AI Coding Agent. Ele foi implementado pelos responsáveis pela disciplina The Modern Software Developer, de Stanford, e disponibilizado publicamente por eles. Nós apenas o usamos aqui.

## Arquivos

- `agent.py`: o agente, com o loop e as tools `read_file`, `list_files` e `edit_file`.
- `inventory.py`: código com um bug em `apply_discount`.
- `test_inventory.py`: teste que falha por causa do bug.
- `requirements.txt`: dependências.
- `ANALISE-TEMPLATE.md`: template da análise.
- `src/`: código original do agente, para referência.

## Setup

1. Clone o repositório:
   ```bash
   git clone https://github.com/dev-ia-ufcg/atividade-anatomia-agentes.git
   cd atividade-anatomia-agentes
   ```
2. Crie um ambiente virtual e instale as dependências:
   ```bash
   python -m venv .venv
   source .venv/bin/activate
   pip install -r requirements.txt
   ```
3. Crie uma conta gratuita em [console.groq.com](https://console.groq.com) (sem cartão) e gere uma chave em [console.groq.com/keys](https://console.groq.com/keys).
4. Crie um `.env` na raiz com a chave (não faça commit dele):
   ```
   GROQ_API_KEY=sua_chave_aqui
   ```
5. Em `agent.py`, aponte o cliente para o Groq, que tem API compatível com o SDK da OpenAI.

   Antes:
   ```python
   openai_client = OpenAI(api_key=os.environ["OPENAI_API_KEY"])
   ```
   Depois:
   ```python
   openai_client = OpenAI(
       api_key=os.environ["GROQ_API_KEY"],
       base_url="https://api.groq.com/openai/v1",
   )
   ```
   Em `execute_llm_call`, troque o modelo:
   ```python
   model="qwen/qwen3.8-27b"
   ```
   Use exatamente esse modelo, e não outro do Groq, para que as execuções da turma sejam comparáveis. Se der erro em `max_completion_tokens`, troque por `max_tokens`.

## O que fazer

### 1. Instrumentar o trace

Hoje o `agent.py` só imprime o nome da tool e os argumentos (`print(name, args)`). Ele não mostra o raciocínio entre uma observação e a próxima ação, nem organiza a execução em Thought / Action / Observation. Modifique o código para que cada iteração do loop registre, de forma legível:

- Thought: o texto do modelo antes da chamada de tool ou da resposta final. O modelo costuma escrever raciocínio antes da linha `tool: ...`. Não descarte a resposta só porque ela tem uma chamada de tool.
- Action: a tool chamada e os argumentos exatos.
- Observation: o resultado da tool, do jeito que volta para a conversa.

Pode ser print formatado, escrita em arquivo, o que preferir. O importante é o registro ficar completo e legível, porque você vai anexá-lo à análise.

### 2. Executar o agente e experimentar com o código com bug

```bash
python agent.py
```

Dê ao agente a tarefa da aula:

```
encontre e conserte o bug baseado no teste que está falhando em test_inventory.py
```

Deixe rodar até ele parar, ou interrompa se entrar em loop. Guarde o trace completo.

Modelos gratuitos seguem o formato pedido com menos consistência. É esperado que o parser (`extract_tool_invocations`) não reconheça alguma chamada, ou que o modelo responda fora do formato. Se acontecer, não conserte o parser para esconder o problema. Esse tipo de falha é o argumento da aula a favor de tool calling nativo. Relate o que ocorreu na análise.

### 3. Escrever a análise

Copie o `ANALISE-TEMPLATE.md` (por exemplo, para `ANALISE.md`), cole o trace e, ao lado ou logo abaixo de cada trecho relevante, insira uma nota começando com `>>` identificando o componente. Você precisa cobrir:

1. Loop: uma iteração completa e o que faz o laço continuar ou parar ali.
2. Contexto: onde o resultado de uma tool volta para a conversa e passa a fazer parte do que o LLM vê na chamada seguinte.
3. Tools / ACI: uma chamada de tool e o formato usado (parsing de texto sobre `tool: nome({...})`), comparado com JSON estruturado e tool calling nativo.
4. Thought: um trecho de raciocínio do modelo que não gera chamada de tool. Segundo a aula, esse é o passo que normalmente fica escondido dentro das ferramentas prontas.
5. Guardrail: é o item mais importante. O agente não tem nenhum: ele não roda o teste nem verifica se o problema foi resolvido antes de parar. Explique, com base na sua execução, o que isso significou na prática. Ele pode ter parado achando que terminou sem ter terminado?
6. Falhas de parsing: o que o parser errou ou deixou passar. Se não houve falha, diga isso.

## Entrega

Suba em um repositório `agent.py` instrumentado, `inventory.py`, `test_inventory.py` e a `ANALISE.md`, e entregue o link no form do classroom.
