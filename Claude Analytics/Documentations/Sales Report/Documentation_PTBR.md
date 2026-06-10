# Modelo Semântico Power BI — Sales Report

**Documentação Técnica Completa** | Gerado em: Junho de 2026

---

## 1. Visão Geral do Modelo Semântico

### Propósito e Objetivos

O modelo semântico **Sales Report** foi desenvolvido para suportar a análise de desempenho de vendas por vendedor, produto e período. Ele viabiliza os seguintes cenários de relatório:

- Receita realizada versus metas orçamentárias (análise de atingimento).
- Volume de pedidos e mix de categorias de produto (Drink vs. Food).
- Ranking de vendedores, top-3 performers e participação individual na receita total.
- Comparações entre períodos utilizando uma dimensão de data dedicada.
- Enriquecimento visual com fotos de vendedores e grupos de produtos via colunas de URL.

### Características Principais

| Propriedade | Valor |
|---|---|
| Cultura de exibição | `en-US` |
| Cultura de consulta de origem | `en-GB` |
| Modo de carregamento | Import (todos os dados em memória; DirectQuery não utilizado) |
| Time intelligence | Habilitado (`__PBI_TimeIntelligenceEnabled = 1`) |
| Ferramentas de desenvolvimento | Anotação DevMode presente |

---

## 2. Fontes de Dados

| Arquivo | Tipo | Planilha / Objeto | Alimenta a(s) Tabela(s) | Descrição |
|---|---|---|---|---|
| `SalesData.xlsx` | Pasta de Trabalho Excel | Sheet1 | Sales, Salesperson | Registros transacionais de vendas: datas de pedido, quantidades, preços unitários e hierarquia de vendedores (Salesperson → Supervisor → Manager). |
| `Product.xlsx` | Pasta de Trabalho Excel | Table1 | Product | Cadastro de produtos: ID, nome, grupo e categoria. |
| `Budget.xlsx` | Pasta de Trabalho Excel | Todas as planilhas (expandidas) | Budget | Valores orçamentários mensais por vendedor. Os dados originais estão no formato tabela cruzada (pivot) e são normalizados (unpivot) durante o carregamento. |
| `Photos.xlsx` | Pasta de Trabalho Excel | Group (tabela), Salesperson (tabela) | Product (GroupPhoto), Salesperson (SalespersonPhoto) | URLs de imagens para grupos de produtos e vendedores. Incorporadas às tabelas dimensão por meio de mesclagem no Power Query. |

> **Observação:** Todos os arquivos são carregados a partir de um caminho local na máquina do autor do relatório (`C:\Users\amoreira\Desktop\…\Database\`). As strings de conexão precisarão ser atualizadas ao publicar no Power BI Service ou ao mover os arquivos para um local compartilhado.

---

## 3. Estrutura do Modelo de Dados

### 3.1 Tabelas

| Tabela | Papel | Origem | Colunas |
|---|---|---|---|
| **Sales** *(Fato)* | Tabela fato central — uma linha por linha de pedido. | SalesData.xlsx – Sheet1 | OrderDate, OrderNumber, ProductKey, SalespersonKey, Channel, Quantity, UnitPrice, SalesAmount |
| **Budget** *(Fato)* | Metas orçamentárias mensais por vendedor. | Budget.xlsx | SalespersonID, Amount, BudgetDate |
| **Product** *(Dimensão)* | Catálogo de produtos com agrupamento e categoria. | Product.xlsx + Photos.xlsx | ID, ProductName, ProductGroup, ProductCategory, GroupPhoto |
| **Salesperson** *(Dimensão)* | Hierarquia da equipe de vendas (Salesperson → Supervisor → Manager). | SalesData.xlsx + Photos.xlsx | SalespersonKey, Salesperson, Supervisor, Manager, SalespersonPhoto |
| **Date** *(Calculada)* | Dimensão de data gerada automaticamente a partir do intervalo de datas do modelo. | `CALENDARAUTO()` | Date, Year, Quarter, Month Number, Month, Day |
| **Key Measures** *(Medidas)* | Tabela virtual usada exclusivamente para hospedar todas as medidas DAX do modelo. | Inline (tabela vazia) | Column1 (oculta, placeholder) + 16 medidas |
| **DateTableTemplate…** *(Oculta)* | Template de data gerado automaticamente pelo Power BI (interno). | Sistema | — |
| **LocalDateTable…** *(Oculta)* | Suporta a hierarquia de datas padrão na coluna Date. | Sistema | Date (com Date Hierarchy) |

### 3.2 Relacionamentos

> O modelo segue o padrão **star schema**: duas tabelas fato (*Sales* e *Budget*) conectadas a dimensões compartilhadas (*Salesperson* e *Date*), com *Product* servindo exclusivamente a *Sales*.

| De (lado Muitos) | Para (lado Um) | Tipo | Ativo | Observação |
|---|---|---|---|---|
| Sales[ProductKey] | Product[ID] | Muitos-para-Um | Sim | Associa cada venda a um produto. |
| Sales[SalespersonKey] | Salesperson[SalespersonKey] | Muitos-para-Um | Sim | Associa cada venda ao vendedor responsável. |
| Budget[SalespersonID] | Salesperson[SalespersonKey] | Muitos-para-Um | Sim | Vincula entradas de orçamento ao vendedor correspondente, viabilizando a comparação entre Revenue e Budget. |
| Sales[OrderDate] | Date[Date] | Muitos-para-Um | Sim | Relacionamento de data principal para filtragem das vendas ao longo do tempo. |
| Budget[BudgetDate] | Date[Date] | Muitos-para-Um | Sim | Conecta os valores mensais de orçamento à dimensão de data. |
| Date[Date] | LocalDateTable[Date] | Muitos-para-Um | Sim | Relacionamento interno do Power BI que habilita a hierarquia de data padrão (Year / Quarter / Month / Day). |

### 3.3 Hierarquias

| Tabela | Hierarquia | Níveis (maior → menor granularidade) | Finalidade |
|---|---|---|---|
| Date (LocalDateTable) | Date Hierarchy *(automática)* | Year → Quarter → Month → Day | Permite drill-down padrão em qualquer eixo de data/hora nos visuais. Disponibilizada automaticamente ao usar a coluna Date. |
| Salesperson | Hierarquia organizacional *(implícita)* | Manager → Supervisor → Salesperson | Permite consolidar resultados do vendedor individual até o supervisor e a gerência. Nenhum objeto de hierarquia formal do Power BI foi definido; as colunas podem ser usadas de forma independente em segmentações e linhas. |
| Product | Hierarquia de produto *(implícita)* | ProductCategory → ProductGroup → ProductName | Permite drill-down da categoria de alto nível (Drink / Food) até os produtos individuais. Nenhum objeto de hierarquia formal do Power BI foi definido. |

---

## 4. Colunas Calculadas e Medidas

### 4.1 Colunas Calculadas

| Tabela | Coluna | Expressão / Origem | Tipo de Dado | Significado |
|---|---|---|---|---|
| Sales | **SalesAmount** | `Quantity × UnitPrice` (Power Query) | Decimal | Principal métrica de receita. Calculada linha a linha durante o carregamento dos dados, evitando o overhead de desempenho de uma coluna calculada DAX em uma tabela fato de grande volume. |
| Date | **Year** | `YEAR('Date'[Date])` | Inteiro | Utilizado para filtrar anos específicos (ex.: medida *Revenue 2020*). |
| Date | **Quarter** | `"Qtr " & QUARTER('Date'[Date])` | Texto | Rótulo formatado (Qtr 1 … Qtr 4) para agrupamento trimestral em visuais. |
| Date | **Month Number** | `MONTH('Date'[Date])` | Inteiro | Número do mês (1–12) usado para ordenação cronológica correta da coluna Month (texto). |
| Date | **Month** | `FORMAT('Date'[Date], "mmmm")` | Texto | Nome completo do mês para exibição em eixos e segmentações (ex.: January, February). |
| Date | **Day** | `DAY('Date'[Date])` | Inteiro | Dia do mês, possibilitando granularidade diária em visuais de série temporal. |

### 4.2 Medidas (Tabela Key Measures)

| Medida | Formato | Finalidade |
|---|---|---|
| **Revenue** | $#,0 | Receita total de vendas — base da maioria dos KPIs financeiros. |
| **Orders** | #,0 | Contagem de números de pedido distintos — mede o volume de transações. |
| **Budget** | $#,0 | Soma dos valores orçados por vendedor/período. |
| **ATP** | $#,0.00 | Average Transaction Price: Receita ÷ Pedidos. Indica o valor médio por transação. |
| **Revenue vs Budget** | 0.0% | Razão entre receita realizada e orçamento. Valor > 100% indica superação da meta. |
| **Revenue 2020** | $#,0 | Receita filtrada para o ano-calendário de 2020 — utilizada em cartões de KPI com referência histórica fixa. |
| **Countrows Sales** | 0 | Contagem de linhas da tabela Sales — útil para validação de dados e monitoramento de carga. |
| **Orders by Drink** | 0 | Contagem de pedidos filtrada para ProductCategory = "Drink". |
| **Orders by Drink %** | 0% | Participação dos pedidos de Drink no total de pedidos — indicador de mix de produtos. |
| **Orders by Food** | 0 | Contagem de pedidos filtrada para ProductCategory = "Food". |
| **Orders by Food %** | 0% | Participação dos pedidos de Food no total de pedidos — indicador de mix de produtos. |
| **% Revenue** | #,0.0% | Receita de cada vendedor como percentual da receita total da empresa. |
| **Rank SP** | 0 | Classificação do vendedor por receita (1 = maior receita). Ignora qualquer filtro na tabela Salesperson para que o ranking seja sempre relativo a todos os vendedores. |
| **Revenue Top 3** | $#,0 | Receita atribuível aos 3 vendedores com maior receita. |
| **Orders Top 3** | #,0 | Contagem de pedidos dos 3 vendedores com maior receita. |
| **ATP Top 3** | $#,0.00 | Ticket médio dos 3 vendedores com maior receita. |

---

## 5. Fórmulas DAX

### 5.1 Medidas de Agregação Base

Estes são os blocos atômicos do modelo; todas as demais medidas os referenciam.

**Revenue**
```dax
Revenue = SUM(Sales[SalesAmount])
```
Soma simples da coluna. Como `SalesAmount` é pré-calculada no Power Query, esta medida é extremamente rápida mesmo em grandes volumes de dados.

---

**Orders**
```dax
Orders = DISTINCTCOUNT(Sales[OrderNumber])
```
Utiliza `DISTINCTCOUNT` em vez de `COUNTROWS` para lidar com pedidos que possuem múltiplas linhas (diferentes produtos no mesmo pedido), evitando dupla contagem.

---

**Budget**
```dax
Budget = SUM(Budget[Amount])
```
Agrega a tabela fato Budget. O valor se torna significativo ao ser combinado com filtros de Date ou Salesperson.

---

### 5.2 Medidas de Razão e Variância

**ATP (Average Transaction Price)**
```dax
ATP = DIVIDE([Revenue], [Orders], 0)
```
A função `DIVIDE` é utilizada em todo o modelo no lugar do operador de divisão (`/`) para retornar 0 com segurança — em vez de um erro — quando o denominador é zero ou está em branco. Isso é essencial ao filtrar por um vendedor ou período sem pedidos registrados.

---

**Revenue vs Budget**
```dax
'Revenue vs Budget' = DIVIDE([Revenue], [Budget])
```
Acompanha o atingimento do orçamento. Um valor de 1,0 (100%) indica resultado exatamente na meta. Valores acima de 1 indicam superação; abaixo de 1, resultado abaixo da meta.

---

**% Revenue**
```dax
'% Revenue' =
DIVIDE(
    [Revenue],
    CALCULATE(
        [Revenue],
        ALL(Salesperson)
    )
)
```
`ALL(Salesperson)` remove qualquer filtro sobre a tabela Salesperson, fazendo com que o denominador seja sempre a receita total da empresa — independentemente do vendedor selecionado no relatório. Isso garante que a participação de cada vendedor seja calculada corretamente em relação ao total geral.

---

### 5.3 Medidas de Mix de Categoria

**Orders by Drink**
```dax
'Orders by Drink' =
CALCULATE(
    [Orders],
    'Product'[ProductCategory] == "Drink"
)
```

**Orders by Drink %**
```dax
'Orders by Drink %' = DIVIDE([Orders by Drink], [Orders], 0)
```

**Orders by Food**
```dax
'Orders by Food' =
CALCULATE(
    [Orders],
    'Product'[ProductCategory] == "Food"
)
```

**Orders by Food %**
```dax
'Orders by Food %' = DIVIDE([Orders by Food], [Orders], 0)
```

A função `CALCULATE` substitui o contexto de filtro vigente, aplicando um filtro fixo sobre `ProductCategory`. As variantes percentuais expressam o volume de cada categoria como fração do total de pedidos — úteis em gráficos de pizza/donut para análise de mix.

---

### 5.4 Time Intelligence

**Revenue 2020**
```dax
'Revenue 2020' =
CALCULATE(
    [Revenue],
    'Date'[Year] = 2020
)
```
Define fixamente o ano 2020 como filtro, fornecendo um valor de referência histórico estático. É exibido tipicamente em cartões de KPI quando o relatório abrange um período histórico definido. A coluna calculada `Date[Year]` (criada via `YEAR()`) é o alvo do filtro.

---

### 5.5 Medidas de Ranking e Top-N

**Rank SP**
```dax
'Rank SP' = RANKX(ALL(Salesperson), [Revenue])
```
`RANKX` itera sobre todos os vendedores (independentemente do filtro corrente) e atribui posição 1 ao de maior receita. O argumento `ALL(Salesperson)` garante que o ranking permaneça estável mesmo quando o visual está filtrado para um subconjunto de vendedores.

---

**Revenue Top 3**
```dax
'Revenue Top 3' =
CALCULATE(
    [Revenue],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

**Orders Top 3**
```dax
'Orders Top 3' =
CALCULATE(
    [Orders],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

**ATP Top 3**
```dax
'ATP Top 3' =
CALCULATE(
    [ATP],
    TOPN(3, ALL(Salesperson), [Revenue])
)
```

As três medidas *Top 3* compartilham o mesmo filtro: `TOPN(3, ALL(Salesperson), [Revenue])`. Esta expressão produz uma tabela virtual com os 3 vendedores de maior receita em todo o modelo, e o `CALCULATE` restringe a medida base a esse subconjunto. Assim, as três métricas (receita, pedidos, ATP) são sempre calculadas para os mesmos três vendedores, garantindo consistência entre os cartões de KPI e visuais comparativos.

---

### 5.6 Colunas Calculadas da Tabela Date (DAX)

```dax
-- A tabela Date é criada automaticamente
Date[Date]         = CALENDARAUTO()        -- gera uma linha por dia no intervalo de datas do modelo

Date[Year]         = YEAR('Date'[Date])
Date[Quarter]      = "Qtr " & QUARTER('Date'[Date])
Date[Month Number] = MONTH('Date'[Date])
Date[Month]        = FORMAT('Date'[Date], "mmmm")
Date[Day]          = DAY('Date'[Date])
```

`CALENDARAUTO()` inspeciona todas as colunas de data do modelo e cria um intervalo contínuo de datas desde a mais antiga até a mais recente encontrada — sem necessidade de definir datas de início e fim manualmente. As colunas derivadas estendem a data base para os atributos exigidos pelos visuais e segmentações.

---

*Documentação gerada a partir dos arquivos de definição do Modelo Semântico Power BI (formato TMDL) · PowerBIWeek Report.SemanticModel*
