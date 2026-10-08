# Dados

Os dados brutos não ficam no repositório. O notebook baixa a base automaticamente do Google Drive na primeira célula.

## Fonte

Listas públicas de precatórios do **Tribunal de Justiça do Estado do Rio de Janeiro (TJRJ)**, disponíveis no [Portal de Precatórios](https://www.tjrj.jus.br/web/precatorios):

- precatórios já pagos
- fila de precatórios em aberto (ordem cronológica por ente devedor)

## Base usada no notebook

- Arquivo no Google Drive: [base de precatórios](https://drive.google.com/file/d/1vWVNaCcEnrN7GxnLQYh4KI5VdDE8vA2m/view)
- Data de referência da extração: 03/10/2026
- Tamanho: 61.718 precatórios e 27 colunas

## Principais colunas

| Coluna | Descrição |
|---|---|
| `lista` | Se o precatório vem da lista de pagos ou da fila em aberto |
| `entidade_devedora` | Ente público que deve o valor |
| `regime` | `E` = regime especial, `O` = regime comum (ordinário) |
| `natureza` | Alimentícia ou comum |
| `data_protocolo` | Data de autuação do precatório |
| `ano_orcamento` | Ano do orçamento em que o precatório foi incluído |
| `valor_historico` | Valor original |
| `ordem_cronologica` | Posição na fila do ente devedor naquele ano de orçamento (recomeça do 1 em cada ente e ano) |
| `ordem_pagamento` | Posição na ordem geral de pagamento (só preenchida para os precatórios em aberto) |
| `data_fim` | Data aproximada de finalização (última movimentação registrada) |
| `finalizado` | 1 = já pago, 0 = em aberto |

Para rodar localmente, salve o CSV nesta pasta e troque a URL da primeira célula do notebook pelo caminho do arquivo.
