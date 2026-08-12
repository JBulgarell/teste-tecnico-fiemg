# Teste Técnico FIEMG — Engenharia de Dados

## Contexto

Este repositório contém a resolução do teste técnico de Engenharia de Dados
aplicado pela **FIEMG (Federação das Indústrias do Estado de Minas Gerais)**,
para o processo seletivo.

O teste propõe um cenário fictício: um marketplace B2B que conecta
distribuidoras de alimentos a pequenos supermercados e mercearias em todo o
Brasil, cujo time de negócios identificou inconsistências nos relatórios de
faturamento — números diferentes dependendo do sistema consultado. A partir
desse cenário, quatro desafios de SQL foram propostos para avaliar as
habilidades essenciais de um(a) Engenheiro(a) de Dados.

Este repositório contém a resolução dos 4 desafios, com consultas SQL,
validações e justificativas de negócio para cada decisão tomada.

## Sumário

- [Estrutura do projeto](#estrutura-do-projeto)
- [Tecnologias utilizadas](#tecnologias-utilizadas)
- [Como executar](#como-executar)
- [Estrutura do banco de dados](#estrutura-do-banco-de-dados)
- [Organização do notebook](#organização-do-notebook)
- [Desafios e resultados](#desafios-e-resultados)
- [Tratamentos e validações aplicados](#tratamentos-e-validações-aplicados)
- [Premissas assumidas](#premissas-assumidas)

## Estrutura do projeto

```
.
├── modelo_teste.ipynb   # Notebook com a resolução dos 4 desafios
└── README.md            # Este arquivo
```

## Tecnologias utilizadas

| Ferramenta | Uso no projeto |
|---|---|
| Python 3 | Linguagem base do notebook |
| Pandas | Leitura e manipulação dos CSVs de entrada |
| pandasql | Execução de consultas SQL sobre DataFrames |
| SQLite | Engine SQL utilizada pelo pandasql |
| Google Colab | Ambiente de execução recomendado |

## Como executar

1. Abra o `modelo_teste.ipynb` no **Google Colab** (ou Jupyter local).
2. Faça upload dos 6 arquivos CSV para `/content/` no ambiente:
   `buyers.csv`, `order_items.csv`, `orders.csv`, `payments.csv`,
   `products.csv`, `sellers.csv`.
3. Execute as células **em ordem, do início ao fim**:
   - Instalação de dependências (`pip install pandasql`);
   - Import de bibliotecas e leitura dos CSVs;
   - Análise exploratória e validações;
   - Resolução dos 4 desafios, cada um com query SQL + justificativa + resultado.
4. Todas as consultas SQL rodam via SQLite, através de `pysqldf(query)`.

## Estrutura do banco de dados

| Tabela | Descrição |
|---|---|
| `orders` | Pedidos realizados |
| `order_items` | Itens de cada pedido |
| `products` | Catálogo de produtos |
| `sellers` | Distribuidoras vendedoras |
| `buyers` | Supermercados compradores |
| `payments` | Registro de pagamentos |

## Organização do notebook

O notebook está dividido em duas partes:

1. **Resolução dos Desafios 1 a 4** — consultas que respondem diretamente
   às perguntas de negócio propostas, cada uma acompanhada de uma célula
   markdown explicando o raciocínio e as premissas adotadas.
2. **Análise Exploratória** — realizada para entendimento dos dados que
   estou trabalhando: consultas de apoio (volume de registros, período
   disponível, status dos pedidos, composição dos valores, etc.) usadas
   para validar hipóteses e sustentar as decisões tomadas nos desafios.
   **Não fazem parte das respostas em si.**

## Desafios e resultados

### Desafio 1 — Faturamento Mensal

**Problema de negócio:** o faturamento mensal variava entre relatórios
porque pedidos `cancelled` e `refunded` estavam sendo somados em alguns
deles.

**Solução:** faturamento bruto mensal (últimos 12 meses), quantidade de
pedidos e ticket médio, considerando **apenas** `status IN ('completed',
'delivered')`, ordenado do mês mais recente ao mais antigo.

- Faturamento calculado como `SUM(qty × unit_price)`, antes de descontos.
- Ticket médio protegido contra divisão por zero com `NULLIF`.
- Janela de 12 meses ancorada na última data disponível na base
  (29/11/2024), e não em `CURRENT_DATE`, já que os dados são históricos.

### Desafio 2 — Crescimento de GMV por Seller

**Problema de negócio:** ranking dos 10 sellers com maior crescimento de
GMV entre dois trimestres, considerando apenas sellers com **≥ 50 pedidos
em ambos os trimestres**, para evitar distorção de sellers novos ou
inativos.

**Solução:** a consulta consolida o GMV e a quantidade de pedidos por seller
e trimestre. Em seguida, compara os dois períodos, aplica o critério mínimo
de 50 pedidos em ambos os trimestres, calcula o percentual de crescimento e
retorna os 10 maiores resultados.

O resultado apresenta o **nome do seller, estado, GMV do trimestre anterior,
GMV do trimestre atual e percentual de crescimento**, ordenados do maior para
o menor crescimento.

- Para o cálculo do GMV, foram considerados apenas pedidos com status
  `completed` e `delivered`, seguindo a definição de pedidos válidos adotada
  no Desafio 1.
- A definição de "trimestre atual" e "trimestre anterior" utilizada nesta
  query está detalhada na seção [Premissas assumidas](#premissas-assumidas).
  
### Desafio 3 — Descontos Abusivos

**Problema de negócio:** suspeita de sellers aplicando descontos abusivos
para inflar volume artificialmente.

**Solução:** identificação dos pedidos cujo desconto total representa mais
de 40% do valor bruto do pedido, excluindo pedidos com status `cancelled`
e apresentando o seller responsável e a data do pedido.

- Valor bruto e desconto total são calculados por pedido na CTE, e o filtro
  de desconto superior a 40% é aplicado posteriormente na consulta principal.
- `NULLIF` é utilizado para evitar divisão por zero no cálculo percentual.
  
### Desafio 4 — Produtos com Comportamento Estranho

**Problema de negócio:** identificar produtos com alto volume de vendas
(> 1.000 unidades) que nunca foram o item de maior valor unitário dentro
de um pedido.

**Solução:** utilização de uma função de janela, particionada por pedido e
ordenada por `unit_price DESC`, para identificar o maior valor unitário em
cada pedido. Essa informação foi posteriormente cruzada com o total de
unidades vendidas por produto.

**Resultado:** foram identificados 800 produtos com mais de 1.000 unidades
vendidas. Na validação do segundo critério, verificou-se que esses 800
produtos também já haviam sido o item de maior valor unitário em pelo menos
um pedido. Portanto, **nenhum produto atende simultaneamente aos dois
critérios definidos no enunciado**.

**Validação da conclusão:**

- Produtos com mais de 1.000 unidades vendidas: **800**
- Produtos que já foram o item de maior valor unitário em pelo menos um
  pedido: **800**
- Produtos que atendem simultaneamente aos dois critérios: **0**

**Questionamento analítico:** o resultado obtido não confirma a premissa
apresentada no enunciado de que existe um produto com esse comportamento.

Como validação adicional, os dois critérios foram analisados separadamente
e, posteriormente, cruzados para confirmar que não havia produtos que
atendessem simultaneamente às duas condições. Também foram considerados
empates no maior `unit_price` durante a validação da conclusão.

Como encaminhamento de negócio, seria válido alinhar com a área responsável
se o critério pretendido é realmente o **maior valor unitário** ou se deveria
ser considerado outro critério, como o **maior valor total do item dentro do
pedido**.

Essa constatação foi mantida como resultado da análise, sem forçar a identificação de um produto cuja ocorrência não é sustentada pelos dados disponíveis.

## Tratamentos e validações aplicados

Ao longo do notebook, foram realizados tratamentos e validações para
garantir a consistência das análises e apoiar as decisões tomadas:

- **Divisão por zero:** utilização de `NULLIF` nos cálculos de ticket
  médio e percentual de crescimento.

- **Status dos pedidos:** análise e contagem dos status disponíveis antes
  da definição dos filtros utilizados em cada desafio.

- **Período disponível:** verificação de `MIN` e `MAX` de `created_at`
  para compreender o período disponível e definir as janelas de análise.

- **Consistência de valores:** comparação entre `orders.total_value` e o
  valor calculado a partir de `order_items`
  (`qty × unit_price − discount`), para verificar a consistência dos
  valores registrados.

- **Elegibilidade de sellers:** verificação da quantidade de pedidos por
  seller e trimestre antes da aplicação do filtro de pelo menos 50 pedidos
  no Desafio 2.

- **Identificação do maior valor unitário:** utilização de uma função de
  janela, particionada por pedido e ordenada por `unit_price DESC`, para
  identificar o maior valor unitário em cada pedido. Empates no maior valor
  unitário foram considerados nas validações realizadas para confirmar a
  conclusão.

- **Resultado vazio tratado como achado:** no Desafio 4, a ausência de
  produtos que atendam simultaneamente aos dois critérios foi tratada como
  um resultado da análise, e não como erro. A conclusão foi validada por
  consultas independentes antes de ser reportada.
  
## Premissas assumidas

| Ponto em aberto no enunciado | Premissa adotada | Justificativa |
|---|---|---|
| Desafio 1: qual data usar como referência para "últimos 12 meses" | Última data disponível na base (29/11/2024), não `CURRENT_DATE` | A base é histórica; a janela foi definida a partir da última data disponível e considera os 12 meses-calendário de dezembro/2023 a novembro/2024. |
| Desafio 2: quais trimestres são "atual" e "anterior" | T4/2024 (atual, parcial) vs. T3/2024 (anterior) | O enunciado não especifica a data de referência. Como a última data disponível na base é 29/11/2024, foi adotado o T4/2024 como trimestre atual, mesmo sendo parcial, e o T3/2024 como trimestre anterior. **Essa diferença de cobertura temporal deve ser considerada na interpretação do crescimento, pois o T4/2024 possui dados parciais até 29/11/2024.** |
| Desafio 1: faturamento "bruto" | `qty × unit_price`, sem subtrair descontos | Descontos são tratados separadamente no Desafio 3; faturamento bruto é tratado como o valor anterior à aplicação dos descontos. |
