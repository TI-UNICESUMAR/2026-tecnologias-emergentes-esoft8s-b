# Atividade 9. Fluxo de dados públicos com contrato

Atividade individual de 0,5 ponto, realizada em sala na quinta-feira, 22/10. Entrega pelo formulário da turma até 23:59 do mesmo dia.

Na aula de quarta, a ficha do município reuniu três fluxos publicados no Dify: Município, População e Saúde. Cada fluxo consulta uma fonte pública e entrega um contrato JSON. O orquestrador chama os três pela API e monta a ficha.

Você vai criar um fluxo no Dify que consulta dados públicos do município definido pelo seu RA. O fluxo organiza a resposta em um contrato JSON e usa um modelo de linguagem para escrever um resumo. Depois, um nó `Code` verifica se os números do resumo aparecem no contrato.

Siga esta ordem:

1. Descubra seu município e seu cenário pelo RA.
2. Leia o [detalhamento do seu cenário](#detalhamento-dos-cenários), no fim do enunciado.
3. Monte a consulta e o contrato JSON.
4. Acrescente o resumo e a conferência dos números.
5. Faça as três execuções, registre os resultados e envie o DSL pelo formulário.

## Antes de começar

- Conta gratuita do Dify, com o mesmo modelo da [Atividade 8](../2026-10-15-atividade-agents/atividade.md), o `gpt-4o-mini`. Cada execução que chega ao resumo gasta 1 crédito do Dify, e a atividade pede duas ou três. Sem créditos, use a chave gratuita do Gemini.
- O plano gratuito do Dify permite 5 apps. Se a sua conta estiver cheia, exporte o DSL de um fluxo da aula de quarta e apague o app antes de criar o da atividade.
- Os slides e os fluxos da aula de quarta, na pasta `2026-10-21-workflows-dify` do repositório da turma. Os fluxos consultam `servicodados.ibge.gov.br`, que o Dify deixou de alcançar. Eles servem de referência para a montagem dos nós, mas não para a leitura dos dados. O formato das fontes desta atividade está na seção "Fontes".
- [Documentação do nó LLM do Dify](https://docs.dify.ai/en/cloud/use-dify/nodes/llm).
- A lista [municipios-pr.md](municipios-pr.md), com os 399 municípios do Paraná em ordem de código IBGE.

## Seu município

Use o número do RA antes do hífen. Calcule o resto da divisão desse número por 399 e some 1. O resultado é a posição do seu município na lista.

No RA fictício `12345678-9`, o resto de `12345678` por 399 é `219`. A posição é `220`, que corresponde a Mercedes, código `4115853`. O último dígito antes do hífen é `8`, então o cenário é B1. O `9` depois do hífen não entra no cálculo.

## Seu cenário

O cenário depende do último dígito do RA antes do hífen.

| Dígito | Cenário | Fonte |
|---|---|---|
| 0, 4, 8 | B1. Domicílios | apisidra, tabela 4712 |
| 1, 5, 9 | B2. PIB municipal | apisidra, tabela 5938 |
| 2, 6 | B3. Região imediata | Mapa de regiões e apisidra, duas consultas |
| 3, 7 | B4. Hospitais gerais | Cópia do CNES de 19/10 |

Leia apenas o detalhamento do seu cenário no fim deste enunciado. Nele estão a URL da consulta, os campos de `dados` e os valores para conferir o resultado.

## Fontes

Em 22/10, o Dify Cloud deixou de alcançar `servicodados.ibge.gov.br`, usado na aula de quarta, e a API de dados abertos do CNES saiu do ar. Os cenários usam outras fontes.

- **apisidra** (`apisidra.ibge.gov.br`). Outra API do IBGE, com as mesmas tabelas do SIDRA. A resposta é uma lista. O primeiro item descreve as colunas, e cada item seguinte traz um valor em `V`. `D1N` é o nome do local, `D2C` o código da variável, `D3N` o período, e `D4N` em diante as categorias. Com código inexistente, responde `400` com uma mensagem em texto.
- **Cópias publicadas no repositório da turma.** O mapa de regiões imediatas do Paraná e uma cópia do CNES de 19/10 com os municípios dos alunos de B4. Cada arquivo é um objeto JSON cuja chave é o código IBGE de 7 dígitos.

A tabela 4709, que não é de nenhum cenário, serve de exemplo. A consulta abaixo pede a população de Maringá.

```text
https://apisidra.ibge.gov.br/values/t/4709/n6/4115200/v/93/p/2022
```

A primeira linha de dados da resposta é esta:

```json
{"V": "409657", "D1N": "Maringá (PR)", "D2C": "93", "D2N": "População residente", "D3N": "2022"}
```

No `Code`, o `body` chega como texto, e `V` também, por isso precisa de `Number` ou `parseInt`. O trecho abaixo lê a população dessa resposta.

```javascript
if (status_code !== 200) {
  return { ok: 0, contrato: { ok: false, ...base, status_http: status_code, erro: "Consulta HTTP sem sucesso" } }
}
const linhas = JSON.parse(body).slice(1)
const linha = linhas.find((l) => l.D2C === "93")
const populacao = linha ? parseInt(linha.V) : null
```

O teste de `status_code` vem antes do `JSON.parse`, porque com código inexistente o corpo é texto e não JSON. `base` reúne os campos comuns do contrato, como `const base = { fonte: "...", codigo_ibge }`.

Nos arquivos do repositório, o valor do município é lido pela chave, como `JSON.parse(body)[codigo_ibge]`.

## Contrato

Todo fluxo entrega um objeto `contrato` no mesmo formato dos fluxos da aula.

```json
{"ok": true, "fonte": "...", "codigo_ibge": "4115200", "dados": {}}
```

Quando a consulta não produz dados válidos, o contrato informa o problema.

```json
{"ok": false, "fonte": "...", "codigo_ibge": "9999999", "status_http": 400, "erro": "..."}
```

Os nomes e os tipos dos campos de `dados` fazem parte do contrato. O orquestrador, o resumo e qualquer outro consumidor dependem deles.

## Monte a consulta e o contrato

1. Criar um Workflow chamado `A9 | <nome do cenário>`, com a entrada `codigo_ibge` do tipo Short Text.
2. Acrescentar o `HTTP Request` com método `GET` e a URL do cenário. No lugar de `<codigo_ibge>`, inserir a variável `codigo_ibge` digitando `/` no campo da URL.
3. Acrescente o `Code` "Validar e montar contrato". Substitua as variáveis padrão `arg1`, `arg2` e `result` pelas seguintes configurações.

   - Entradas: `codigo_ibge`, `status_code` e `body`.
   - Saídas em Output Variables: `ok` como Number e `contrato` como Object.
   - O código deve devolver `ok` igual a `1` para dados válidos ou `0` para falha, junto do objeto `contrato`.

   A seção "Fontes" mostra como ler a resposta da consulta.
4. Acrescentar o `If/Else` "Contrato válido?" com a condição `ok` `=` `1`. Ligar o ramo ELSE a um `Output`, o nó que os slides chamam de `End`, com a variável `erro`, que recebe o `contrato` do `Code`. O ramo IF segue para o resumo, descrito na próxima seção.
5. Conferir no Last Run do `Code`. Nos cenários do IBGE, executar com Maringá, `4115200`, e comparar com o detalhamento. Em B4, executar com o seu código e comparar com o arquivo da cópia.

O cenário B3 faz duas consultas em sequência. O detalhamento mostra a ligação entre elas.

## Acrescente o resumo e a conferência

No ramo IF de "Contrato válido?", acrescentar três nós.

1. Acrescente o `LLM` "Escrever resumo", com o modelo da [Atividade 8](../2026-10-15-atividade-agents/atividade.md). Se os créditos acabaram, siga a seção "Chave gratuita do Gemini".

   - Na mensagem do usuário, insira o `contrato` do `Code` "Validar e montar contrato" pelo seletor de variáveis.
   - Ative Retry on Failure com 2 tentativas para lidar com respostas `503` em picos de demanda.
   - No prompt de sistema, cole o texto abaixo.

   ```text
   Você escreve para moradores do município. Escreva até três frases em português com os valores do contrato JSON recebido. Use os números como aparecem no contrato, sem arredondar e sem converter unidades. Não acrescente informações que não estejam no contrato.
   ```

2. `Code` "Conferir resumo", com as entradas `contrato`, do `Code` "Validar e montar contrato", e `texto`, que recebe a saída `text` do `LLM`. Ele devolve `ok` (número) e `reprovado` (objeto). Usar o código abaixo.

   ```javascript
   function main({ contrato, texto }) {
     const leituras = (t) => {
       const semPonto = t.replace(/\./g, "")
       if (t.includes(",")) return [Number(semPonto.replace(",", "."))]
       return [Number(t), Number(semPonto)]
     }
     const doContrato = (JSON.stringify(contrato).match(/\d+(\.\d+)?/g) || []).map(Number)
     const doTexto = texto.match(/\d+(?:[.,]\d+)*/g) || []
     const sem_origem = doTexto.filter((t) => !leituras(t).some((n) => doContrato.includes(n)))
     if (sem_origem.length > 0) {
       return { ok: 0, reprovado: { motivo: "Números do texto sem origem no contrato", sem_origem, texto } }
     }
     return { ok: 1, reprovado: {} }
   }
   ```

   O código reúne os números que aparecem em qualquer parte do contrato, inclusive nos nomes dos campos e na fonte. Depois procura nessa lista cada número do resumo. Um número com vírgula, como `91,2`, é lido como decimal. Um número só com pontos tem duas leituras, porque `487.012` pode ser a área `487.012` do contrato ou o milhar `487012`, e o código aceita o número quando uma das leituras está no contrato. Em `sem_origem`, os números aparecem como foram escritos no resumo.

3. `If/Else` "Resumo aprovado?", com a condição `ok` `=` `1`. Ligar o ramo IF a um `Output` com as variáveis `contrato`, do `Code` "Validar e montar contrato", e `texto`, que recebe a saída `text` do `LLM`. Ligar o ramo ELSE a um `Output` com a variável `reprovado`, do `Code` "Conferir resumo". O Dify não publica um fluxo com dois `Output` de variável com o mesmo nome.

## Chave gratuita do Gemini

Faça esta seção só se os créditos do Dify acabaram. A chave do Google AI Studio é gratuita e não pede cartão.

1. Abrir [aistudio.google.com/apikey](https://aistudio.google.com/apikey) e entrar com uma conta Google. Se a conta institucional não tiver acesso, use uma conta pessoal.
2. Clicar em "Criar chave de API", depois em "Criar chave". Na janela da chave, clicar em "Copiar chave".
3. No Dify, abrir Integrations, Model Provider. No cartão Gemini, clicar em "Add API Key" e de novo em "Add API Key". Colar a chave em API Key e clicar em Save.
4. No mesmo cartão, em Usage Priority, escolher API Key. Assim o Dify usa a sua chave e não os créditos.
5. No nó `LLM`, escolher o modelo `gemini-3.5-flash-lite`.

A chave dá acesso à sua cota no Google. Não a coloque no formulário nem no DSL exportado. O Dify guarda a chave nas credenciais do workspace, fora do DSL. Ao fim do semestre, apague a chave no AI Studio.

## Execute e registre os resultados

1. Execute com o código do seu município. Copie o `contrato` do Last Run do `Code` "Validar e montar contrato" e a saída do fluxo, com `contrato` e `texto` ou com `reprovado`.
2. Execute com `9999999`. No log do nó HTTP, copie o `status_code` e o `body`. No B3, use o segundo nó HTTP. Copie também a saída do fluxo. Em B4, registre o que acontecer, mesmo que o fluxo chegue ao resumo.
3. Acrescente ao fim do prompt de sistema a frase "Inclua também o número de escolas e a renda média do município." Execute de novo com o código do seu município e copie a saída. Nenhum contrato da atividade traz escolas ou renda. Exporte o DSL depois deste passo, com a frase acrescentada.

## Envie sua entrega

Responder ao [formulário da turma](https://docs.google.com/forms/d/e/1FAIpQLSc7iVgIIVvEtSfimqMxuZ2OqLStU7fRwHU0gRyuVGT0_9Ch9Q/viewform) e anexar o DSL do seu fluxo. Para exportar, no Studio, abrir "More actions" no card do app e escolher "Export DSL". O arquivo `.yml` não leva a chave do Gemini, que fica nas credenciais do workspace. Capturas de tela são opcionais.

## Detalhamento dos cenários

Nas URLs, `<codigo_ibge>` marca o lugar da variável. Cole a URL no nó HTTP, apague `<codigo_ibge>` e, no mesmo lugar, digite `/` e escolha a variável `codigo_ibge` da Entrada. Não escreva o nome da variável à mão: o Dify só troca pelo valor a variável inserida pelo seletor. Os nós `Code` usam JavaScript. Nas listas de nomes, use `.sort()` sem argumentos.

### B1. Domicílios

Fonte no contrato: `IBGE SIDRA, Censo 2022, tabela 4712`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/4712/n6/<codigo_ibge>/v/381,5930/p/2022
```

| Campo | Tipo | Origem |
|---|---|---|
| `domicilios_ocupados` | inteiro | `V` da linha com `D2C` igual a `381` |
| `media_moradores` | decimal | `V` da linha com `D2C` igual a `5930` |

Maringá: `{"domicilios_ocupados": 156517, "media_moradores": 2.6}`

### B2. PIB municipal

Fonte no contrato: `IBGE SIDRA, PIB dos Municípios, tabela 5938`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/5938/n6/<codigo_ibge>/v/37/p/2021,2023
```

| Campo | Tipo | Origem |
|---|---|---|
| `pib_2021_mil_reais` | inteiro | `V` da linha com `D3N` igual a `2021` |
| `pib_2023_mil_reais` | inteiro | `V` da linha com `D3N` igual a `2023` |
| `variacao_pct` | decimal, uma casa | `Math.round((1000 * (pib_2023 - pib_2021)) / pib_2021) / 10` |

Maringá: `{"pib_2021_mil_reais": 22655738, "pib_2023_mil_reais": 27817346, "variacao_pct": 22.8}`

### B3. Região imediata

Fonte no contrato: `IBGE, regiões imediatas, SIDRA tabela 4714`.

A primeira consulta baixa o mapa de regiões do Paraná. A chave é o código IBGE, e o valor traz o nome do município, o identificador e o nome da região:

```json
"4115200": {"municipio": "Maringá", "id": 410014, "nome": "Maringá"}
```

Um `Code` "Extrair região imediata" recebe `codigo_ibge` e o `status_code` e o `body` da primeira consulta, e devolve `id_regiao` (texto) e `nome_regiao` (texto). Quando o código não está no mapa, devolve `id_regiao` igual a `"0"` e `nome_regiao` vazio.

A segunda consulta lista os municípios da região pelo apisidra. Na URL, apague `<id_regiao>` e insira a variável `id_regiao` do `Code` "Extrair região imediata". Com o identificador `0`, a API responde `400`.

O `Code` "Validar e montar contrato" recebe `codigo_ibge`, o `status_code` e o `body` da segunda consulta e também `regiao_nome`, ligado a `nome_regiao` do primeiro `Code`.

```text
https://raw.githubusercontent.com/TI-UNICESUMAR/2026-tecnologias-emergentes-esoft8s-b/main/2026-10-22-atividade-pipeline-dados/dados/regioes-imediatas-pr.json
https://apisidra.ibge.gov.br/values/t/4714/n6/in%20n25%20<id_regiao>/v/6318/p/2022
```

| Campo | Tipo | Origem |
|---|---|---|
| `regiao_imediata` | texto | `nome_regiao` extraído na primeira consulta |
| `quantidade_municipios` | inteiro | número de linhas de dados da segunda consulta |
| `municipios` | lista de textos | `D1N` de cada linha até o primeiro ` - `, ordenado. `Astorga - PR` vira `Astorga` |

Maringá: `{"regiao_imediata": "Maringá", "quantidade_municipios": 23, "municipios": ["Astorga", "Atalaia", "Doutor Camargo", ..., "Uniflor", "Ângulo"]}`, com os 23 nomes na ordem do `.sort()`.

### B4. Hospitais gerais

Fonte no contrato: `DataSUS CNES, tipo de unidade 5, cópia de 19/10`. O arquivo traz, para cada município, a lista de hospitais gerais com `nome_fantasia`, com no máximo 20 itens. Um código ausente do arquivo corresponde a lista vazia, igual a um município sem hospitais.

```text
https://raw.githubusercontent.com/TI-UNICESUMAR/2026-tecnologias-emergentes-esoft8s-b/main/2026-10-22-atividade-pipeline-dados/dados/cnes-tipo-5-pr.json
```

| Campo | Tipo | Origem |
|---|---|---|
| `hospitais_gerais` | inteiro | tamanho da lista do município |
| `pode_haver_mais` | booleano | lista com 20 itens |
| `nomes` | lista de textos | campo `nome_fantasia`, ordenado |

Conferência: abrir o arquivo no navegador e comparar a lista do seu código com `nomes`.

## Referências

- Slides da aula de quarta, "Workflows no Dify", na pasta `2026-10-21-workflows-dify` do repositório da turma.
- [Nó LLM](https://docs.dify.ai/en/cloud/use-dify/nodes/llm) e [nó If/Else](https://docs.dify.ai/en/cloud/use-dify/nodes/ifelse).
- [Nó HTTP Request](https://docs.dify.ai/en/cloud/use-dify/nodes/http-request) e [Nó Code](https://docs.dify.ai/en/cloud/use-dify/nodes/code).
- [API do SIDRA (apisidra)](https://apisidra.ibge.gov.br/home/ajuda).
