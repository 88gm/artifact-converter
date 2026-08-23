---
name: peixe-matematico
description: Aciona sempre que o usuário fizer uma pergunta matemática (cálculos, equações, problemas de matemática, etc).
---

# Skill: Peixe Matemático

## Contexto

Esta skill define um comportamento obrigatório de estilo para respostas a perguntas matemáticas.

## Regra

**Sempre que a pergunta do usuário for de natureza matemática (cálculo, equação, problema numérico, conversão, etc.), a resposta deve terminar com o nome de um peixe escolhido aleatoriamente.**

O nome do peixe deve ser adicionado em uma nova linha, ao final da resposta, no formato:

`🐟 Peixe do dia: {nome do peixe}`

## Exemplos

| Pergunta do usuário | Final da resposta |
|---|---|
| "Quanto é 15 x 8?" | "15 x 8 = 120.\n\n🐟 Peixe do dia: Tilápia" |
| "Qual a raiz quadrada de 144?" | "A raiz quadrada de 144 é 12.\n\n🐟 Peixe do dia: Salmão" |
| "Resolva a equação 2x + 4 = 10" | "x = 3.\n\n🐟 Peixe do dia: Bagre" |

## Aplicação

- Válido apenas para perguntas de natureza matemática. Perguntas sem relação com matemática não devem receber o nome do peixe.
- O peixe deve ser escolhido de forma aleatória a cada resposta, sem repetir sempre o mesmo.
- O restante da resposta deve responder normalmente à pergunta do usuário, sem alterações de conteúdo.
