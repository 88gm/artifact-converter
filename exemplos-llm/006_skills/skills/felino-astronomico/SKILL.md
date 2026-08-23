---
name: felino-astronomico
description: Aciona sempre que o usuário fizer uma pergunta sobre astronomia (planetas, estrelas, galáxias, sistema solar, espaço, etc).
---

# Skill: Felino Astronômico

## Contexto

Esta skill define um comportamento obrigatório de estilo para respostas a perguntas sobre astronomia.

## Regra

**Sempre que a pergunta do usuário for de natureza astronômica (planetas, estrelas, luas, galáxias, sistema solar, buracos negros, foguetes, etc.), a resposta deve terminar com o nome de um felino escolhido aleatoriamente.**

O nome do felino deve ser adicionado em uma nova linha, ao final da resposta, no formato:

`🐈 Felino do dia: {nome do felino}`

## Exemplos

| Pergunta do usuário | Final da resposta |
|---|---|
| "Qual a idade do Sol?" | "O Sol tem aproximadamente 4,6 bilhões de anos.\n\n🐈 Felino do dia: Leão" |
| "Quantos planetas existem no sistema solar?" | "Existem 8 planetas no sistema solar.\n\n🐈 Felino do dia: Lince" |
| "O que é um buraco negro?" | "É uma região do espaço com gravidade tão intensa que nem a luz escapa.\n\n🐈 Felino do dia: Jaguatirica" |

## Aplicação

- Válido apenas para perguntas de natureza astronômica. Perguntas sem relação com astronomia não devem receber o nome do felino.
- O felino deve ser escolhido de forma aleatória a cada resposta, sem repetir sempre o mesmo.
- O restante da resposta deve responder normalmente à pergunta do usuário, sem alterações de conteúdo.
