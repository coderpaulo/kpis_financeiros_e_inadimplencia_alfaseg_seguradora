# KPIs Financeiros e Inadimplência | AlfaSeg Seguros

Projeto de portfólio em **SQL (Databricks)** e **Power BI**, com foco em indicadores financeiros e de receita de uma seguradora fictícia. Cobre o fluxo completo: modelagem, tratamento em camadas, cálculo de KPIs e dashboard.

> **Aviso:** todos os dados são **fictícios** e sintéticos. Nenhuma informação de empresa real foi usada.

## Objetivo

Responder, com dados, perguntas que uma área financeira de seguros acompanha:

- Quanto prêmio foi **emitido** e quanto foi **ganho** (competência)?
- Quanto dinheiro realmente **entrou** (caixa), já descontados os estornos?
- Qual a **taxa de inadimplência** e como ela varia por linha de negócio?

**Ferramentas:** Databricks (SQL, arquitetura medalhão) e Power BI. As bases foram geradas em Python (pandas e numpy).

## Conteúdo do repositório

```
├── README.md
├── SQL/          notebook do Databricks (camadas Silver e Gold)
├── docs/         dicionário de dados (campos, tipos, chaves e regras)
└── dashboards/   prints do Power BI e base consolidada (CSV)
```

## Modelo de dados (Star Schema)

- **`f_apolices` (fato):** 200.400 linhas, sendo 200.000 apólices únicas e 400 duplicatas. Traz prêmio, vigência, cancelamento e renovação.
- **`d_produtos` (dimensão):** 4 produtos (Auto, Residencial, Vida e Empresarial) e suas linhas de negócio.
- **`f_parcelas` (fato):** 1.305.791 linhas com parcelas, pagamentos e estornos.

Todas as datas são `timestamp`, os valores em R$ são `double` e os textos são `string`. O detalhamento de cada campo está em `docs/dicionario_dados_AlfaSeg.xlsx`.

## Situações de negócio simuladas

- **Cancelamento:** cerca de 7,6% das apólices; as parcelas posteriores ficam canceladas.
- **Estorno:** devolução de prêmio, com valor negativo em `f_parcelas`.
- **Vigência aberta:** parte das apólices de Vida sem data de fim.
- **Renovação:** 12% das apólices renovam uma anterior, com mesmo cliente e produto.
- **Inadimplência:** parcelas sem pagamento, além das ainda não vencidas.
- **Duplicidade:** 400 apólices repetidas, tratadas com `DISTINCT`.

## Arquitetura em camadas

```
Bronze (RAW)           Silver (tratada)          Gold (KPIs)
f_apolices      ──►    silver_apolices    ──►    gold_premio_emitido
d_produtos      ──►    silver_parcelas    ──►    gold_premio_ganho
f_parcelas                                       gold_arrecadacao
                                                 gold_inadimplencia
                                                 gold_kpis_mensal ──► Power BI
```

- **Bronze:** tabelas como foram geradas.
- **Silver:** deduplicação, `INNER JOIN` com produtos, situação de vigência e situação da parcela.
- **Gold:** KPIs por mês e linha de negócio, consolidados na `gold_kpis_mensal` para o dashboard.

## Regras e decisões

1. **Data de corte: 04/10/2026.** A base é uma "foto" desse dia. Os cálculos de atraso e vigência usam essa data fixa, para o resultado ser reproduzível.
2. **Produto inválido:** o `INNER JOIN` com `d_produtos` mantém apenas apólices com produto cadastrado.
3. **Prêmio emitido:** valor bruto no mês de emissão, incluindo apólices depois canceladas.
4. **Prêmio ganho:** reconhecido dia a dia (prêmio ÷ 365) do início da vigência até o menor entre o fim da vigência, o cancelamento e a data de corte. Na Vida com vigência aberta, assumiu-se 365 dias, pois o prêmio é anual.
5. **Arrecadação:** pagamentos efetivos por mês de pagamento. Os estornos entram negativos, resultando na arrecadação líquida.
6. **Situação da parcela:** estorno, cancelada, paga, a vencer, **em atraso** (sem pagamento há menos de 30 dias) ou **inadimplente** (sem pagamento há 30 dias ou mais).
7. **Taxa de inadimplência:** valor inadimplente ÷ valor vencido, considerando só parcelas vencidas até 04/09/2026. Canceladas, a vencer, em atraso e estornos ficam fora.

### Por que a regra de 30 dias?

Na primeira versão, toda parcela vencida e sem pagamento era inadimplente. Isso criou um **pico artificial** nos meses finais (8,64% em set/2026 e 61,88% em out/2026), porque os clientes pagam em até 10 dias e os meses recentes ainda não tiveram tempo. Com o critério de 30 dias, a taxa fica estável entre 3,56% e 4,31%.

## Principais resultados (posição em 04/10/2026)

- **Prêmio emitido:** R$ 1.587.030.045,69
- **Prêmio ganho:** R$ 1.099.620.266,06
- **Arrecadação líquida:** R$ 1.289.783.193,14 (bruta de R$ 1.308.707.235,80 menos estornos de R$ 18.924.042,66)
- **Ticket médio:** R$ 7.935,15
- **Taxa de inadimplência:** **4,08%** (R$ 53.200.023,85 sobre R$ 1.304.238.373,93 vencidos)

### Leituras do dashboard

- **Caixa na frente da competência no início:** a 1ª parcela vence na emissão, então o dinheiro entra antes de o prêmio ser reconhecido. As linhas se aproximam depois de cerca de um ano.
- **Salto na emissão a partir de out/2025:** efeito das renovações, que só começam quando as primeiras apólices completam um ano.
- **Linhas de negócio parecidas:** a base tem distribuição uniforme entre produtos, então a taxa varia pouco (4,02% a 4,14%).

## Dashboard (Power BI)

- **Visão Executiva:** emitido, ganho, arrecadação líquida e taxa de inadimplência; competência vs caixa; emitido vs arrecadado por linha de negócio; filtros de período e linha de negócio.
- **Inadimplência:** valor vencido, valor inadimplente, taxa e estornos; taxa por mês e por linha de negócio; matriz de detalhe.

Os prints estão em `dashboards/`. A base consolidada (`gold_kpis_mensal`) fica na mesma pasta: ao carregá-la no Power BI, use a localidade **Inglês (EUA)** para os decimais.

## Próximos passos

- Segmentar a inadimplência por faixas de atraso (30, 60 e 90 dias)
- Análise de renovação (retenção por produto)
- Automatizar o pipeline com Databricks Workflows

## Autor

**Paulo Henrique Reis** | [LinkedIn](https://linkedin.com/in/paulohstreis)
