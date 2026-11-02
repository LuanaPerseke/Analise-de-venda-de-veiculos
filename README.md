[Documentação Projeto veiculos.md](https://github.com/user-attachments/files/34254288/Documentacao.Projeto.veiculos.md)
# Análise do Mercado de Veículos (1992--2020)

Uma solução de Business Intelligence e Engenharia de Dados desenvolvida
para analisar o mercado de veículos usados, com foco em precificação,
características da frota, perfil dos vendedores e comportamento das
vendas ao longo dos anos. O projeto utiliza SQL Server para transformar
os dados originais, separar marca e modelo, traduzir categorias e criar
tabelas dimensão para organizar a análise no Power BI.

## Sumário

-   [Visão Geral do Projeto](#visão-geral-do-projeto)
-   [Arquitetura de Dados](#arquitetura-de-dados)
-   [Tecnologias Utilizadas](#tecnologias-utilizadas)
-   [Tratamento de Dados e Modelagem
    SQL](#tratamento-de-dados-e-modelagem-sql)
    1.  [Separação de marca e modelo e tradução dos
        campos](#1-separação-de-marca-e-modelo-e-tradução-dos-campos)
    2.  [Criação da tabela fato](#2-criação-da-tabela-fato)
    3.  [Criação das tabelas dimensão](#3-criação-das-tabelas-dimensão)
-   [Dashboard Power BI](#dashboard-power-bi)
-   [Como Executar o Projeto](#como-executar-o-projeto)
-   [Desenvolvedora](#desenvolvedora)

------------------------------------------------------------------------

## Visão Geral do Projeto

O objetivo do projeto é transformar os registros de veículos usados em
informações que permitam explorar o mercado de vendas entre **1992 e
2020**.

A análise contempla:

-   **Precificação:** média de venda e comparação do preço médio entre
    marcas.
-   **Perfil da frota:** quantidade de veículos e quilometragem média.
-   **Características dos veículos:** ano de fabricação, marca, modelo,
    tipo de combustível e câmbio.
-   **Perfil de vendedores:** comparação entre vendas particulares,
    concessionárias e concessionárias certificadas.
-   **Perfil dos proprietários:** distribuição dos veículos de acordo
    com o número de proprietários anteriores.
-   **Análise temporal:** evolução do preço médio de venda por ano de
    fabricação.

### Indicadores apresentados

  Indicador                  Valor exibido
  ------------------------ ---------------
  Média de venda                504,13 mil
  Média de quilometragem         66.215,78
  Total de veículos                  4.340
  Período apresentado           1992--2020

Os valores e rótulos desta seção foram transcritos da captura do
dashboard. A moeda e a unidade monetária não estão identificadas
explicitamente na imagem, portanto não foram especificadas.

------------------------------------------------------------------------

## Arquitetura de Dados

``` text
[TB_CARROS_DEKHO]  -- tabela de origem
        |
        | SELECT, CASE e CHARINDEX
        | Separação de MARCA e MODELO
        | Tradução de categorias
        v
[TB_VEICULOS]  -- dados tratados
        |
        | Seleção dos campos analíticos
        v
[FATO_VENDAS_VEICULOS]
        |
        +--> [DIM_CAMBIO]
        |
        +--> [DIM_COMBUSTIVEL]
        |
        +--> [DIM_MARCA]
        |
        +--> [DIM_MODELO]
        |
        +--> [DIM_PROPRIETARIO]
        |
        v
[Dashboard Power BI]
```

A tabela `TB_VEICULOS` recebe os campos tratados a partir de
`TB_CARROS_DEKHO`. Em seguida, `FATO_VENDAS_VEICULOS` concentra os
atributos selecionados para a análise, enquanto as tabelas dimensão
armazenam os valores distintos de câmbio, combustível, marca, modelo e
proprietário.

------------------------------------------------------------------------

## Tecnologias Utilizadas

-   **SQL Server:** transformação e tradução dos dados com `CASE`,
    separação de marca e modelo com `CHARINDEX` e `LEFT`, criação de
    tabelas com `SELECT INTO` e extração de valores únicos com
    `SELECT DISTINCT`.
-   **Power BI:** construção do dashboard com cartões de KPI, filtros,
    gráficos de barras, gráfico de linha, gráfico de pizza e tabela
    detalhada.
-   **GitHub:** versionamento dos scripts, documentação e arquivos do
    projeto.

------------------------------------------------------------------------

## Tratamento de Dados e Modelagem SQL

O tratamento foi realizado no SQL Server a partir da tabela
`TB_CARROS_DEKHO`. O script separa a primeira palavra do campo `name`
como marca e mantém o nome completo no campo modelo. Também traduz
categorias de combustível, vendedor, câmbio e proprietário para
português.

### 1. Separação de marca e modelo e tradução dos campos

A primeira consulta cria a tabela `TB_VEICULOS`. A marca é extraída com
`CHARINDEX` e `LEFT`: quando há espaço no nome, considera-se como marca
o texto anterior ao primeiro espaço; caso contrário, o nome completo é
mantido. O campo original `name` é preservado como `MODELO`.

As categorias em inglês são traduzidas com `CASE WHEN`, facilitando a
leitura dos filtros e visuais no Power BI.

``` sql
SELECT
    name AS MODELO,
    CASE
        WHEN CHARINDEX(' ', name) > 0
            THEN LEFT(name, CHARINDEX(' ', name) - 1)
        ELSE name
    END AS MARCA,
    year AS ANO_FABRICACAO,
    selling_price AS PRECO_VENDA,
    km_driven AS QUILOMETRAGEM,
    fuel,
    CASE
        WHEN fuel = 'Petrol' THEN 'Gasolina'
        WHEN fuel = 'CNG' THEN 'GNV'
        WHEN fuel = 'LPG' THEN 'GLP'
        WHEN fuel = 'Electric' THEN 'Elétrico'
        ELSE fuel
    END AS TIPO_COMBUSTIVEL,
    seller_type,
    CASE
        WHEN seller_type = 'Individual' THEN 'Particular'
        WHEN seller_type = 'Dealer' THEN 'Concessionária'
        WHEN seller_type = 'Trustmark Dealer' THEN 'Concessionária Certificada'
    END AS TIPO_VENDEDOR,
    transmission,
    CASE
        WHEN transmission = 'Automatic' THEN 'Automático'
        ELSE transmission
    END AS CAMBIO,
    owner,
    CASE
        WHEN owner = 'First Owner' THEN '1º Dono'
        WHEN owner = 'Second Owner' THEN '2º Dono'
        WHEN owner = 'Third Owner' THEN '3º Dono'
        WHEN owner = 'Fourth & Above Owner' THEN '4º Dono ou mais'
        WHEN owner = 'Test Drive Car' THEN 'Carro de Test Drive'
    END AS PROPRIETARIO
INTO TB_VEICULOS
FROM TB_CARROS_DEKHO;
```

### 2. Criação da tabela fato

Após o tratamento, uma segunda consulta seleciona os campos utilizados
na análise e cria a tabela `FATO_VENDAS_VEICULOS`.

``` sql
SELECT
    MODELO,
    MARCA,
    ANO_FABRICACAO,
    PRECO_VENDA,
    QUILOMETRAGEM,
    TIPO_COMBUSTIVEL,
    TIPO_VENDEDOR,
    CAMBIO,
    PROPRIETARIO
INTO FATO_VENDAS_VEICULOS
FROM TB_VEICULOS;
```

### 3. Criação das tabelas dimensão

As dimensões são criadas a partir dos valores distintos existentes na
tabela fato. Esse procedimento evita repetir a mesma categoria diversas
vezes na respectiva tabela dimensão.

``` sql
SELECT DISTINCT CAMBIO
INTO DIM_CAMBIO
FROM FATO_VENDAS_VEICULOS;

SELECT DISTINCT TIPO_COMBUSTIVEL
INTO DIM_COMBUSTIVEL
FROM FATO_VENDAS_VEICULOS;

SELECT DISTINCT MARCA
INTO DIM_MARCA
FROM FATO_VENDAS_VEICULOS;

SELECT DISTINCT MODELO
INTO DIM_MODELO
FROM FATO_VENDAS_VEICULOS;

SELECT DISTINCT PROPRIETARIO
INTO DIM_PROPRIETARIO
FROM FATO_VENDAS_VEICULOS;
```

**Observação sobre o escopo do script:** o arquivo SQL fornecido cria as
dimensões com `SELECT DISTINCT`, mas não apresenta a criação de chaves
substitutas (`ID`), a inclusão de chaves estrangeiras na fato ou os
comandos de relacionamento. Por isso, esta documentação descreve apenas
a estrutura efetivamente presente no script, sem presumir etapas
adicionais da modelagem.

------------------------------------------------------------------------

## Dashboard Power BI

<img width="1322" height="737" alt="Dashboard_Veiculos" src="https://github.com/user-attachments/assets/9179ae40-2e50-4dd8-861b-4cc54d55049b" />


O dashboard apresenta uma interface de tema escuro, com contrastes em
vermelho, e filtros laterais para explorar os dados. O cabeçalho destaca
o título, o tema da análise e o período de 1992 a 2020.

### Indicadores e filtros

-   **Cartões de KPI:** média de venda (504,13 mil), média de
    quilometragem (66.215,78) e total de veículos (4.340).
-   **Filtro de câmbio:** permite selecionar o tipo de transmissão.
-   **Filtro de vendedores:** permite segmentar entre Particular,
    Concessionária e Concessionária Certificada.
-   **Filtro de ano:** permite restringir a análise por ano.

### Visuais do dashboard

-   **Preço médio por ano de fabricação:** gráfico de linha que
    apresenta a evolução do preço médio ao longo do período.
-   **Preço médio por marca:** gráfico de barras horizontais que compara
    as marcas com maiores preços médios.
-   **Distribuição por combustível:** gráfico de pizza com a
    participação dos tipos de combustível.
-   **Vendas por proprietário:** tabela detalhada que reúne tipo de
    proprietário, marca, ano de fabricação, média de quilometragem e
    valor de venda.

------------------------------------------------------------------------

## Como Executar o Projeto

1.  **SQL Server**
    -   Importe a base de dados original para o SQL Server, criando a
        tabela `TB_CARROS_DEKHO`.
    -   Execute o script de tratamento para criar `TB_VEICULOS`.
    -   Execute a consulta de seleção para criar `FATO_VENDAS_VEICULOS`.
    -   Execute os comandos de criação das cinco dimensões.
2.  **Power BI**
    -   Abra o arquivo `.pbix` do projeto no Power BI Desktop.
    -   Configure a conexão com o banco de dados SQL Server.
    -   Confira os relacionamentos entre a tabela fato e as dimensões no
        modelo.
    -   Atualize os dados e explore os filtros e gráficos do dashboard.

------------------------------------------------------------------------

## Desenvolvedora

**Desenvolvido por Luana Perseke**

Conecte-se comigo no [LinkedIn](https://www.linkedin.com/in/luana-perseke-84211b179/) \|
[GitHub](https://github.com/LuanaPerseke)
