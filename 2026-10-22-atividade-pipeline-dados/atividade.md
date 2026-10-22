# A9. Fluxo de dados públicos com contrato

Atividade individual de 0,5 ponto, realizada em sala na quinta-feira, 22/10. Entrega pelo formulário da turma até 23:59 do mesmo dia.

Na aula de quarta, a ficha do município reuniu três fluxos publicados no Dify: Município, População e Saúde. Cada fluxo consulta uma fonte pública e entrega um contrato JSON. O orquestrador chama os três pela API e monta a ficha.

Nesta atividade você cria um fluxo novo, com fonte e contrato definidos pelo seu cenário. Depois do contrato, um nó LLM escreve um resumo para quem mora no município, e um `Code` confere se cada número do resumo veio do contrato. O município consultado sai do seu RA.

## Material

- Conta gratuita do Dify, com o mesmo modelo da A8. Cada execução que chega ao resumo gasta 1 crédito do Dify, e a atividade pede duas ou três. Sem créditos, use a chave gratuita do Gemini.
- O fluxo de exemplo [dify/exemplo-cor-raca.yml](dify/exemplo-cor-raca.yml), completo, da consulta ao apisidra até o resumo conferido. Ele não corresponde a nenhum cenário e serve de ponto de partida para o `HTTP Request` e o `Code`. Para importar, no Dify, abrir Studio, clicar em Create e escolher "Import DSL file". Os fluxos da aula de quarta consultam `servicodados.ibge.gov.br`, que o Dify deixou de alcançar, e não servem de base para esta atividade.
- [Documentação do nó LLM do Dify](https://docs.dify.ai/en/cloud/use-dify/nodes/llm).
- A lista [municipios-pr.md](municipios-pr.md), com os 399 municípios do Paraná em ordem de código IBGE.

## Seu município

Use o número do RA antes do hífen. Calcule o resto da divisão desse número por 399 e some 1. O resultado é a posição do seu município na lista.

No RA fictício `12345678-9`, o resto de `12345678` por 399 é `219`. A posição é `220`, que corresponde a Mercedes, código `4115853`.

## Seu cenário

O cenário depende da turma e do último dígito do RA antes do hífen.

### Turma A

| Dígito | Cenário | Fonte |
|---|---|---|
| 0, 4, 8 | A1. Área e densidade | apisidra, tabela 4714 |
| 1, 5, 9 | A2. População por sexo | apisidra, tabela 9514 |
| 2, 6 | A3. Distritos | apisidra, tabela 9923 |
| 3, 7 | A4. Laboratórios e exames | Cópia do CNES de 19/10 |

### Turma B

| Dígito | Cenário | Fonte |
|---|---|---|
| 0, 4, 8 | B1. Domicílios | apisidra, tabela 4712 |
| 1, 5, 9 | B2. PIB municipal | apisidra, tabela 5938 |
| 2, 6 | B3. Região imediata | Mapa de regiões e apisidra, duas consultas |
| 3, 7 | B4. Hospitais gerais | Cópia do CNES de 19/10 |

O detalhamento de cada cenário está no fim deste enunciado. Ele traz a URL, os campos de `dados` e o resultado de conferência.

## Fontes

Em 22/10, o Dify Cloud deixou de alcançar `servicodados.ibge.gov.br`, usado na aula de quarta, e a API de dados abertos do CNES saiu do ar. Os cenários usam outras fontes.

- **apisidra** (`apisidra.ibge.gov.br`). Outra API do IBGE, com as mesmas tabelas do SIDRA. A resposta é uma lista. O primeiro item descreve as colunas, e cada item seguinte traz um valor em `V`. `D1N` é o nome do local, `D2C` o código da variável, `D3N` o período, e `D4N` em diante as categorias. Com código inexistente, responde `400` com uma mensagem em texto.
- **Cópias publicadas no repositório da turma.** O mapa de regiões imediatas do Paraná e uma cópia do CNES de 19/10 com os municípios dos alunos de A4 e B4. Cada arquivo é um objeto JSON cuja chave é o código IBGE de 7 dígitos.

Primeira linha de dados do A1 para Maringá:

```json
{"V": "487.012", "D1N": "Maringá (PR)", "D2C": "6318", "D2N": "Área da unidade territorial", "D3N": "2022"}
```

## Contrato

Todo fluxo entrega um objeto `contrato` no mesmo formato dos fluxos da aula.

```json
{"ok": true, "fonte": "...", "codigo_ibge": "4115200", "dados": {}}
```

Quando a consulta não produz dados válidos, o contrato informa o problema.

```json
{"ok": false, "fonte": "...", "codigo_ibge": "9999999", "status_http": 500, "erro": "..."}
```

Os nomes e os tipos dos campos de `dados` fazem parte do contrato. O orquestrador, o resumo e qualquer outro consumidor dependem deles.

## Montar o fluxo

1. Criar um Workflow chamado `A9 | <nome do cenário>`, com a entrada `codigo_ibge` do tipo texto.
2. Acrescentar o `HTTP Request` com método `GET` e a URL do cenário. Inserir `codigo_ibge` pelo seletor de variáveis.
3. Acrescentar o `Code` "Validar e montar contrato". Ele recebe `codigo_ibge`, `status_code` e `body` e devolve `ok` (número, `1` ou `0`) e `contrato` (objeto). Partir do `Code` do fluxo de exemplo.
4. Acrescentar o `If/Else` "Contrato válido?" com a condição `ok` `=` `1`. Ligar o ramo ELSE a um `Output` com a variável `erro`, que recebe o `contrato` do `Code`. O ramo IF segue para o resumo, descrito na próxima seção.
5. Conferir no Last Run do `Code`. Nos cenários do IBGE, executar com Maringá, `4115200`, e comparar com o detalhamento. Em A4 e B4, executar com o seu código e comparar com o arquivo da cópia.

O cenário B3 faz duas consultas em sequência. O detalhamento mostra a ligação entre elas.

## Resumo conferido

No ramo IF de "Contrato válido?", acrescentar três nós.

1. `LLM` "Escrever resumo", com o modelo da A8. Se os créditos do Dify acabaram, siga a seção "Chave gratuita do Gemini". No prompt de sistema, usar o texto abaixo. Na mensagem do usuário, inserir o `contrato` do `Code` "Validar e montar contrato" pelo seletor de variáveis. No painel do nó, ativar Retry on Failure com 2 tentativas, porque os modelos gratuitos respondem `503` em picos de demanda.

```text
Você escreve para moradores do município. Escreva até três frases em português com os valores do contrato JSON recebido. Use os números como aparecem no contrato, sem arredondar e sem converter unidades. Não acrescente informações que não estejam no contrato.
```

2. `Code` "Conferir resumo", com as entradas `contrato`, do `Code` "Validar e montar contrato", e `texto`, do `LLM`. Ele devolve `ok` (número) e `reprovado` (objeto). Usar o código abaixo.

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

3. `If/Else` "Resumo aprovado?", com a condição `ok` `=` `1`. Ligar o ramo IF a um `Output` com as variáveis `contrato`, do `Code` "Validar e montar contrato", e `texto`, do `LLM`. Ligar o ramo ELSE a um `Output` com a variável `reprovado`, do `Code` "Conferir resumo". O Dify não publica um fluxo com dois `Output` de variável com o mesmo nome.

## Chave gratuita do Gemini

Faça esta seção só se os créditos do Dify acabaram. A chave do Google AI Studio é gratuita e não pede cartão.

1. Abrir [aistudio.google.com/apikey](https://aistudio.google.com/apikey) e entrar com uma conta Google. Se a conta institucional não tiver acesso, use uma conta pessoal.
2. Clicar em "Criar chave de API", depois em "Criar chave". Na janela da chave, clicar em "Copiar chave".
3. No Dify, abrir Integrations, Model Provider. No cartão Gemini, clicar em "Add API Key" e de novo em "Add API Key". Colar a chave em API Key e clicar em Save.
4. No mesmo cartão, em Usage Priority, escolher API Key. Assim o Dify usa a sua chave e não os créditos.
5. No nó `LLM`, escolher o modelo `gemini-3.5-flash-lite`.

A chave dá acesso à sua cota no Google. Não a coloque no formulário nem no DSL exportado. O Dify guarda a chave nas credenciais do workspace, fora do DSL. Ao fim do semestre, apague a chave no AI Studio.

## Executar e registrar

1. Execute com o código do seu município. Copie o `contrato` do Last Run do `Code` "Validar e montar contrato" e a saída do fluxo, com `contrato` e `texto` ou com `reprovado`.
2. Execute com `9999999`. No log do nó HTTP, copie o `status_code` e o `body`. Copie também a saída do fluxo.
3. Acrescente ao fim do prompt de sistema a frase "Inclua também o número de escolas e a renda média do município." Execute de novo com o código do seu município e copie a saída. Nenhum contrato da atividade traz escolas ou renda.

## Entrega

Responder ao [formulário da turma](https://docs.google.com/forms/d/e/1FAIpQLSc7iVgIIVvEtSfimqMxuZ2OqLStU7fRwHU0gRyuVGT0_9Ch9Q/viewform). Exportar o DSL do seu fluxo e conservar uma cópia. Capturas de tela são opcionais.

## Critérios de pontuação

| Critério | Valor |
|---|---:|
| Fluxo do cenário: URL, `Code`, condição e contrato com os campos e tipos pedidos | 0,10 |
| Contrato do seu município, com os valores conferidos pelo professor | 0,10 |
| Execução com `9999999`: status, corpo e saída registrados, com a explicação de como o fluxo identifica a falha | 0,10 |
| Resumo e conferência: nós `LLM`, `Code` e `If/Else` montados e as duas execuções com o seu município registradas | 0,10 |
| Análise do que a conferência garante e do que ela deixa passar | 0,10 |
| Total | 0,50 |

Registros com município ou cenário diferentes dos calculados pelo seu RA não recebem os pontos de contrato, de execução e de resumo.

## Detalhamento dos cenários

Nas URLs, `{{#inicio.codigo_ibge#}}` indica a variável inserida pelo seletor do Dify. Os nós `Code` usam JavaScript. Nas listas de nomes, use `.sort()` sem argumentos.

### A1. Área e densidade

Fonte no contrato: `IBGE SIDRA, Censo 2022, tabela 4714`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/4714/n6/{{#inicio.codigo_ibge#}}/v/6318,614/p/2022
```

| Campo | Tipo | Origem |
|---|---|---|
| `area_km2` | decimal | `V` da linha com `D2C` igual a `6318` |
| `densidade_hab_km2` | decimal | `V` da linha com `D2C` igual a `614` |

Maringá: `{"area_km2": 487.012, "densidade_hab_km2": 841.16}`

### A2. População por sexo

Fonte no contrato: `IBGE SIDRA, Censo 2022, tabela 9514`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/9514/n6/{{#inicio.codigo_ibge#}}/v/93/p/2022/c2/4,5/c287/100362/c286/113635
```

| Campo | Tipo | Origem |
|---|---|---|
| `homens` | inteiro | `V` da linha com `D4N` igual a `Homens` |
| `mulheres` | inteiro | `V` da linha com `D4N` igual a `Mulheres` |
| `homens_por_100_mulheres` | decimal, uma casa | `Math.round((1000 * homens) / mulheres) / 10` |

Maringá: `{"homens": 195447, "mulheres": 214210, "homens_por_100_mulheres": 91.2}`

### A3. Distritos

Fonte no contrato: `IBGE SIDRA, Censo 2022, tabela 9923, distritos`. A consulta pede a população de cada distrito do município e usa só os nomes. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/9923/n10/in%20n6%20{{#inicio.codigo_ibge#}}/v/93/p/2022/c1/6795
```

| Campo | Tipo | Origem |
|---|---|---|
| `quantidade_distritos` | inteiro | número de linhas de dados |
| `distritos` | lista de textos | `D1N` de cada linha até o primeiro ` - `, ordenado. `Floriano - Maringá - PR` vira `Floriano` |

Maringá: `{"quantidade_distritos": 3, "distritos": ["Floriano", "Iguatemi", "Maringá"]}`

### A4. Laboratórios e exames

Fonte no contrato: `DataSUS CNES, tipo de unidade 39, cópia de 19/10`. O tipo de unidade 39 reúne as unidades de apoio à diagnose e terapia, como laboratórios e clínicas de exames. O arquivo traz, para cada município, a lista de unidades com `nome_fantasia`, como a API do CNES devolvia, com no máximo 20 itens. Um código ausente do arquivo corresponde a lista vazia, igual a um município sem unidades.

```text
https://raw.githubusercontent.com/TI-UNICESUMAR/2026-tecnologias-emergentes-esoft8s-a/main/2026-10-22-atividade-pipeline-dados/dados/cnes-tipo-39-pr.json
```

| Campo | Tipo | Origem |
|---|---|---|
| `unidades_diagnose` | inteiro | tamanho da lista do município |
| `pode_haver_mais` | booleano | lista com 20 itens |
| `nomes` | lista de textos | campo `nome_fantasia`, ordenado |

Conferência: abrir o arquivo no navegador e comparar a lista do seu código com `nomes`.

### B1. Domicílios

Fonte no contrato: `IBGE SIDRA, Censo 2022, tabela 4712`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/4712/n6/{{#inicio.codigo_ibge#}}/v/381,5930/p/2022
```

| Campo | Tipo | Origem |
|---|---|---|
| `domicilios_ocupados` | inteiro | `V` da linha com `D2C` igual a `381` |
| `media_moradores` | decimal | `V` da linha com `D2C` igual a `5930` |

Maringá: `{"domicilios_ocupados": 156517, "media_moradores": 2.6}`

### B2. PIB municipal

Fonte no contrato: `IBGE SIDRA, PIB dos Municípios, tabela 5938`. Em código inexistente, a API responde `400`.

```text
https://apisidra.ibge.gov.br/values/t/5938/n6/{{#inicio.codigo_ibge#}}/v/37/p/2021,2023
```

| Campo | Tipo | Origem |
|---|---|---|
| `pib_2021_mil_reais` | inteiro | `V` da linha com `D3N` igual a `2021` |
| `pib_2023_mil_reais` | inteiro | `V` da linha com `D3N` igual a `2023` |
| `variacao_pct` | decimal, uma casa | `Math.round((1000 * (pib_2023 - pib_2021)) / pib_2021) / 10` |

Maringá: `{"pib_2021_mil_reais": 22655738, "pib_2023_mil_reais": 27817346, "variacao_pct": 22.8}`

### B3. Região imediata

Fonte no contrato: `IBGE, regiões imediatas, SIDRA tabela 4714`.

A primeira consulta baixa o mapa de regiões do Paraná. Um `Code` "Extrair região imediata" recebe `codigo_ibge`, `status_code` e `body` e devolve `id_regiao` e `nome_regiao` do município. Quando o código não está no mapa, devolve o identificador `0`. A segunda consulta lista os municípios da região pelo apisidra. Com o identificador `0`, ela responde `400`.

```text
https://raw.githubusercontent.com/TI-UNICESUMAR/2026-tecnologias-emergentes-esoft8s-b/main/2026-10-22-atividade-pipeline-dados/dados/regioes-imediatas-pr.json
https://apisidra.ibge.gov.br/values/t/4714/n6/in%20n25%20{{#regiao.id_regiao#}}/v/6318/p/2022
```

| Campo | Tipo | Origem |
|---|---|---|
| `regiao_imediata` | texto | `nome_regiao` extraído na primeira consulta |
| `quantidade_municipios` | inteiro | número de linhas de dados da segunda consulta |
| `municipios` | lista de textos | `D1N` de cada linha até o primeiro ` - `, ordenado. `Astorga - PR` vira `Astorga` |

Maringá: região `Maringá`, identificador `410014`, com 23 municípios, de `Astorga` a `Ângulo` na ordem do `.sort()`.

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

- Slides da disciplina, bloco "Pipeline de dados no Dify".
- [Nó LLM](https://docs.dify.ai/en/cloud/use-dify/nodes/llm) e [nó If/Else](https://docs.dify.ai/en/cloud/use-dify/nodes/ifelse).
- [Nó HTTP Request](https://docs.dify.ai/en/cloud/use-dify/nodes/http-request) e [Nó Code](https://docs.dify.ai/en/cloud/use-dify/nodes/code).
- [API do SIDRA (apisidra)](https://apisidra.ibge.gov.br/home/ajuda).
