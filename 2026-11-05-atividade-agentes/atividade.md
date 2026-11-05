# Atividade 10. Agentes spec e coder

Atividade individual de **0,5 ponto**. Entrega em **05/11/2026, até 23h59**, pelo formulário da sua turma.

Crie dois agentes para o pedido definido pelo seu RA. O **spec** escreve critérios e dúvidas. O **coder** implementa a função depois que as dúvidas forem respondidas. Execute os testes e registre o resultado.

## Material de apoio

- [Slides da atividade](slide.pdf), bloco “Criar agentes no harness” e seção de consulta.
- [Kit e exemplos no GitHub](https://github.com/pedrosatin/processo-software-agentes/tree/main/criar-agentes). O README explica a instalação e os arquivos de cada ferramenta.

## O que fazer

1. Na pasta `criar-agentes`, execute `node pedido.mjs SEU-RA`, substituindo `SEU-RA` pelo seu RA completo.
2. Crie os agentes `spec` e `coder` seguindo os slides e o README. Adapte o coder ao arquivo indicado no seu `pedido.md`.
3. Peça ao harness `Execute o processo de desenvolvimento para pedido.md.` Responda às dúvidas em `respostas.md`, usando as respostas da pasta do seu pedido em `pedidos/`, e retome o processo. As instruções completas estão no formulário.
4. Confira a delegação e os seis testes. Execute `node evidencia.mjs`.

## Entrega

[Formulário de entrega da Turma B](https://docs.google.com/forms/d/e/1FAIpQLScX4hDZ3hS5aKjKcc1h3Q1cdEVe1Zqugnt06jFWqQicqqCWCQ/viewform)

No formulário da sua turma, informe nome e RA e anexe:

- `entrega/evidencia-<RA>.json`, sem editar.
- Um print do harness mostrando a chamada ao `spec` ou ao `coder`.

A nota considera os dois agentes, a especificação com as respostas e os testes aprovados. A distribuição dos pontos está no formulário.
