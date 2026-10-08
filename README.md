# Quanto tempo falta para um precatório sair?

[![Open In Colab](https://colab.research.google.com/assets/colab-badge.svg)](https://colab.research.google.com/github/PatrickAgs/precatorios-tjrj-ml/blob/main/notebooks/precatorios_modelo_18_meses.ipynb)

Modelo de machine learning que estima a **chance de um precatório alimentício em aberto no TJRJ ser pago nos próximos 18 meses**, usando dados públicos do Tribunal de Justiça do Rio de Janeiro.

## O problema

Quem compra um precatório (uma financeira, um escritório de advocacia) paga hoje, com desconto, e só recebe quando o ente público pagar. Se o pagamento sai em um ano, o negócio é bom. Se demora dez, o dinheiro fica parado.

A pergunta que o projeto tenta responder é: **entre os precatórios que estão na fila hoje, quais têm mais chance de sair logo?**

## Os dados

- Fonte: listas públicas do TJRJ (precatórios pagos e fila de precatórios em aberto)
- 61.718 precatórios e 27 colunas na base original
- Depois dos filtros: **46.042 precatórios alimentícios**, sem suspensos e sem INSS

Filtros aplicados e por quê:

| Filtro | Motivo |
|---|---|
| Só alimentícios | São os únicos com casos já pagos na base. Os comuns estão todos em aberto, o que ensinaria ao modelo que "comum nunca sai". |
| Sem suspensos | Um precatório suspenso fica parado, e o modelo não tem como enxergar isso. |
| Sem INSS | Segue uma lógica de pagamento diferente (benefícios previdenciários). |

## Como o alvo foi construído

O ponto central do projeto é perguntar **quanto tempo ainda falta**, e não quanto tempo já passou.

Para isso, o modelo "volta no tempo": escolhe uma data de corte 18 meses antes do último pagamento registrado (02/04/2025), pega todos os precatórios que estavam em aberto naquele dia e verifica quais foram pagos nos 18 meses seguintes.

- **1** = saiu dentro da janela
- **0** = continuou esperando

Resultado: 30.060 precatórios, dos quais só **5,6%** saíram na janela. Por isso a acurácia não foi usada como métrica (um modelo que dissesse "não sai" para todos acertaria mais de 94%).

## Cuidados contra vazamento de dados

Algumas colunas foram retiradas de propósito:

- `possui_cessao`, `possui_herdeiro`, `ultima_fase` e `prioridade` só estavam preenchidas em um dos grupos (pagos ou em aberto). O modelo aprenderia que "coluna preenchida = pago".
- `situacao`, `lista`, `quitado`, `saldo` e `ordem_pagamento` praticamente entregam a resposta.
- Ano de protocolo e de orçamento: a base de pagos só tem pagamentos recentes, então o modelo aprenderia que "precatório antigo nunca sai", o que é um defeito da base e não da realidade.

## Features usadas

| Feature | Descrição |
|---|---|
| `idade_dias` | Há quanto tempo o precatório está na fila |
| `posicao_fila` | Posição relativa na fila do ente e ano de orçamento (0 = início, 1 = fim) |
| `dias_ate_prazo` | Dias até o prazo legal de pagamento (31/12 do ano do orçamento, regime comum) |
| `log_valor` | Valor em escala logarítmica |
| `mes_protocolo` | Mês de entrada |
| `regime_especial` | Se o ente está no regime especial ou no ordinário |
| `ent_*` | Ente devedor (30 maiores + "OUTRAS") |

### Revisão: o que eu deixei passar na primeira versão

Conferindo na [página de perguntas frequentes do Portal de Precatórios do TJRJ](https://www.tjrj.jus.br/web/precatorios/perguntas-frequentes), corrigi dois pontos:

- **A `ordem_cronologica` recomeça em cada ente e ano de orçamento.** Na base, o mesmo número se repete entre entes e entre anos, mas nunca dentro do mesmo ente e ano. Na primeira versão ela entrava crua; agora entra como posição relativa na fila.
- **O prazo legal de pagamento não era usado.** No regime comum, o ente paga até 31/12 do ano do orçamento; no especial, até 31/12/2029 (EC 109/21).

A coluna **não vazava informação do futuro**: ela é definida na autuação e não muda (correlação com a data de protocolo próxima de 1 dentro de cada ente e ano). Os resultados abaixo são da primeira versão e continuam válidos; a comparação com a versão revisada será adicionada aqui.

## Modelos e resultados

Divisão estratificada: 60% treino, 20% validação (para escolher o corte de probabilidade) e 20% teste (usado só no fim).

Resultados da **primeira versão** no conjunto de teste:

| Modelo | AUC | PR-AUC | Precision | Recall | F1 |
|---|---|---|---|---|---|
| **XGBoost** | **0,921** | **0,660** | 0,763 | 0,493 | **0,599** |
| LightGBM | 0,915 | 0,651 | 0,852 | 0,442 | 0,583 |
| Random Forest | 0,906 | 0,625 | 0,660 | 0,504 | 0,572 |
| Regressão Logística | 0,684 | 0,182 | 0,221 | 0,277 | 0,246 |

A linha de base de um chute aleatório no PR-AUC é 0,056 (a proporção de positivos). O XGBoost chega a 0,66.

Na prática, com o corte de 85%: **de cada 4 precatórios que o modelo aponta como "vai sair", cerca de 3 saem de fato**, e ele encontra metade dos que saem.

A grande diferença entre a regressão logística e os modelos de árvore indica que a relação entre as variáveis e o pagamento não é linear (cada ente paga num ritmo próprio, e a interação entre ente, idade e posição na fila importa).

![Curva precision x recall](images/curva_precision_recall.png)

![Matriz de confusão do XGBoost](images/matriz_confusao.png)

![Importância das features](images/importancia_features.png)

## Saída

O notebook aplica o XGBoost a todos os 38.742 precatórios em aberto e gera [`outputs/ranking_precatorios_18_meses.csv`](outputs/ranking_precatorios_18_meses.csv), ordenado pela chance estimada de pagamento em 18 meses.

## Limitações

- **A porcentagem serve para ordenar, não é uma probabilidade calibrada.** Um precatório com 95% é um candidato melhor que um com 60%, mas 95% não significa certeza.
- **A base de pagos só cobre pagamentos recentes.** Precatórios pagos há muitos anos não aparecem.
- **A data de fim é aproximada**: vem da última movimentação registrada, que pode ser posterior ao pagamento.
- **A validação usa uma única data de corte.** O teste mede o desempenho dentro do mesmo período, não em um período futuro.
- **O ritmo de pagamento depende da situação fiscal de cada ente**, que pode mudar.
- **A ordem cronológica não é exatamente a ordem de saída.** Precatórios com prioridade constitucional (idosos, doentes graves, pessoas com deficiência) vão para uma lista separada, e a ordem real de pagamento junta as filas de todos os entes de uma mesma origem.
- **O ranking parece otimista.** Na primeira versão, 8.041 precatórios em aberto passam do corte, enquanto o histórico sugere que bem menos saem em 18 meses. Uma causa provável é que os precatórios de hoje estão mais velhos do que os vistos no treino, e modelos de árvore não extrapolam.

## Próximos passos

Em ordem de prioridade:

1. **Comparar com uma regra simples.** O mercado já precifica olhando a fila. Antes de qualquer melhoria, o modelo precisa ganhar de um baseline como "ordenar pela posição na fila dentro do ente". Se não ganhar, a conclusão é que a fila explica quase tudo.
2. **Validação temporal (backtest).** Treinar com várias datas de corte no passado (por exemplo, a cada 6 meses) e testar sempre na janela seguinte, para medir como o modelo se sai prevendo o futuro de verdade.
3. **Calibrar as probabilidades.** Com calibração (`CalibratedClassifierCV`) e curva de calibração, para que "80%" signifique cerca de 80%. Reportar também a precision@k ("dos 100 primeiros do ranking, quantos saíram").
4. **Prever o tempo até o pagamento, não só "sai ou não sai".** Com análise de sobrevivência, o modelo diria quando o dinheiro deve chegar, e não apenas se chega antes de 18 meses.
5. **Traduzir para dinheiro.** Com o tempo esperado de pagamento, calcular o valor presente e o retorno de comprar cada precatório com um dado desconto. É a métrica que interessa a quem compra.
6. **Dados melhores.** Guardar fotos mensais da fila do TJRJ para reconstruir a posição real de cada precatório no passado, e incluir o ritmo de repasses de cada ente.

## Como rodar

**No Google Colab (mais fácil):** clique no botão "Open in Colab" no topo e execute as células em ordem. Os dados são baixados automaticamente.

**Localmente:**

```bash
git clone https://github.com/PatrickAgs/precatorios-tjrj-ml.git
cd precatorios-tjrj-ml
pip install -r requirements.txt
jupyter notebook notebooks/precatorios_modelo_18_meses.ipynb
```

A última célula usa `google.colab` para baixar os arquivos; fora do Colab, ela pode ser ignorada.

## Estrutura do repositório

```
precatorios-tjrj-ml/
├── README.md
├── requirements.txt
├── notebooks/
│   └── precatorios_modelo_18_meses.ipynb
├── data/
│   └── README.md        ← fonte e descrição dos dados
├── images/              ← gráficos usados neste README
└── outputs/
    └── ranking_precatorios_18_meses.csv
```

## Autor

Patrick Anderson · [GitHub](https://github.com/PatrickAgs)
