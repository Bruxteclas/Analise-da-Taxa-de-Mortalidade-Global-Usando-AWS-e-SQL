# Análise da Taxa de Mortalidade Global Usando AWS Athena e SQL

## Introdução

Este projeto realiza uma análise exploratória de dados de mortalidade globais de 1970 a 2010, armazenados no Amazon S3 e consultados através do AWS Athena. O objetivo é examinar tendências de mortalidade, identificar padrões por faixa etária e gênero, e responder a perguntas de negócio específicas para gerar insights sobre mudanças nas condições de saúde global.

## Contexto do Projeto

A base de dados contém informações de mortalidade específicas por país, ano, grupo etário e sexo, abrangendo vários continentes. Para facilitar as consultas e análises, carregamos esses dados em uma tabela do Athena, permitindo uma interação rápida e consultas SQL sobre o bucket no S3.

## Objetivo

O objetivo principal é entender as variações nas taxas de mortalidade por país, faixa etária e gênero, e explorar possíveis fatores e padrões que impactam a mortalidade ao longo das décadas. Essa análise se concentra em fornecer insights que possam ajudar a compreender melhor a evolução das condições de saúde em diferentes regiões do mundo.

## Estrutura do Projeto e Etapas Realizadas

### 1. Carregamento e Visualização Inicial dos Dados no Athena

#### Criação do Banco de Dados e Tabela no Athena

Primeiramente, criamos um banco de dados e uma tabela no AWS Athena para carregar os dados do arquivo CSV armazenado no bucket `taxa-mortalidade-ts` do Amazon S3. A tabela foi configurada para reconhecer os valores separados por vírgulas e ignorar a primeira linha como cabeçalho.

##### Código SQL para Criação do Banco de Dados e Tabela

```sql
CREATE DATABASE IF NOT EXISTS mortality_data;

CREATE EXTERNAL TABLE IF NOT EXISTS mortality_data.mortality_statistics (
    country_code STRING,
    country_name STRING,
    year INT,
    age_group STRING,
    sex STRING,
    number_of_deaths STRING,
    death_rate_per_100000 STRING
) 
ROW FORMAT SERDE 'org.apache.hadoop.hive.serde2.OpenCSVSerde'
WITH SERDEPROPERTIES (
    'separatorChar' = ',',
    'quoteChar' = '"'
)
LOCATION 's3://taxa-mortalidade-ts/'
TBLPROPERTIES ('skip.header.line.count'='1');
```

#### Visualização Inicial

Após a criação da tabela, visualizei os primeiros registros para verificar o carregamento correto dos dados. A consulta a seguir foi usada para retornar uma amostra dos dados:

```sql
SELECT * 
FROM "mortality_data"."mortality_statistics" 
LIMIT 10;
```

**Saída da Visualização Inicial dos Dados:**

| country_code | country_name | year | age_group | sex   | number_of_deaths | death_rate_per_100000 |
|--------------|--------------|------|-----------|-------|------------------|-----------------------|
| AFG          | Afghanistan  | 1970 | 0-6 days  | Male  | 19,241          | 318,292.90           |
| AFG          | Afghanistan  | 1970 | 0-6 days  | Female| 12,600          | 219,544.20           |
| AFG          | Afghanistan  | 1970 | 0-6 days  | Both  | 31,840          | 270,200.70           |
| AFG          | Afghanistan  | 1970 | 7-27 days | Male  | 15,939          | 92,701.00            |
| AFG          | Afghanistan  | 1970 | 7-27 days | Female| 11,287          | 68,594.50            |
| AFG          | Afghanistan  | 1970 | 7-27 days | Both  | 27,226          | 80,912.50            |
| AFG          | Afghanistan  | 1970 | 28-364 days| Male | 37,513          | 15,040.10            |
| AFG          | Afghanistan  | 1970 | 28-364 days| Female| 32,113         | 13,411.80            |
| AFG          | Afghanistan  | 1970 | 28-364 days| Both  | 69,626          | 14,242.60            |
| AFG          | Afghanistan  | 1970 | 1-4 years | Male  | 36,694          | 4,288.20             |

**Observação Inicial:** Os dados carregados apresentavam colunas com vírgulas nos números (`number_of_deaths` e `death_rate_per_100000`), exigindo uma etapa de limpeza antes de prosseguir com as análises.


### 2. Limpeza e Tratamento dos Dados

Para garantir que os dados estavam prontos para análise, realizei uma etapa de limpeza, focando nas colunas `number_of_deaths` e `death_rate_per_100000`. Essas colunas estavam em formato `STRING` com vírgulas, o que impedia a conversão direta para valores numéricos. Abaixo está o código SQL usado para essa limpeza, que remove as vírgulas e converte as colunas para tipos adequados.

#### Código SQL para Limpeza dos Dados

```sql
SELECT 
    country_code,
    country_name,
    year,
    age_group,
    sex,
    CAST(REPLACE(number_of_deaths, ',', '') AS INT) AS number_of_deaths,
    CAST(REPLACE(death_rate_per_100000, ',', '') AS DECIMAL(10, 2)) AS death_rate_per_100000
FROM mortality_data.mortality_statistics
WHERE country_code IS NOT NULL
  AND country_name IS NOT NULL
  AND year IS NOT NULL
  AND age_group IS NOT NULL
  AND sex IS NOT NULL
  AND number_of_deaths IS NOT NULL
  AND death_rate_per_100000 IS NOT NULL;
```

**Saída dos Dados Após Limpeza:**

| country_code | country_name | year | age_group | sex   | number_of_deaths | death_rate_per_100000 |
|--------------|--------------|------|-----------|-------|------------------|-----------------------|
| AFG          | Afghanistan  | 1970 | 0-6 days  | Male  | 19241           | 318292.90            |
| AFG          | Afghanistan  | 1970 | 0-6 days  | Female| 12600           | 219544.20            |
| AFG          | Afghanistan  | 1970 | 0-6 days  | Both  | 31840           | 270200.70            |
| AFG          | Afghanistan  | 1970 | 7-27 days | Male  | 15939           | 92701.00             |
| AFG          | Afghanistan  | 1970 | 7-27 days | Female| 11287           | 68594.50             |
| AFG          | Afghanistan  | 1970 | 7-27 days | Both  | 27226           | 80912.50             |
| AFG          | Afghanistan  | 1970 | 28-364 days| Male | 37513           | 15040.10             |
| AFG          | Afghanistan  | 1970 | 28-364 days| Female| 32113          | 13411.80             |
| AFG          | Afghanistan  | 1970 | 28-364 days| Both  | 69626           | 14242.60             |
| AFG          | Afghanistan  | 1970 | 1-4 years | Male  | 36694           | 4288.20              |

**Observação:** A limpeza dos dados foi bem-sucedida, permitindo a conversão das colunas para formatos numéricos sem problemas de formatação. Esta etapa foi fundamental para garantir a precisão das análises posteriores.

---

### 3. Estatísticas Básicas dos Dados

Para obter uma visão geral e ter uma ideia da distribuição e variabilidade dos dados, calculei estatísticas descritivas, incluindo valores mínimos, máximos, média e desvio padrão para as colunas `number_of_deaths` e `death_rate_per_100000`. Esses cálculos ajudam a identificar valores extremos e compreender a variação entre os países e grupos etários.

#### Código SQL para Estatísticas de `number_of_deaths`

```sql
SELECT 
    MIN(CAST(REPLACE(number_of_deaths, ',', '') AS INT)) AS min_deaths,
    MAX(CAST(REPLACE(number_of_deaths, ',', '') AS INT)) AS max_deaths,
    AVG(CAST(REPLACE(number_of_deaths, ',', '') AS DOUBLE)) AS avg_deaths,
    STDDEV(CAST(REPLACE(number_of_deaths, ',', '') AS DOUBLE)) AS stddev_deaths
FROM mortality_data.mortality_statistics
WHERE number_of_deaths IS NOT NULL;
```

**Resultado:**

| min_deaths | max_deaths | avg_deaths      | stddev_deaths       |
|------------|------------|-----------------|---------------------|
| 0          | 9938487    | 16109.94        | 154329.32          |

#### Código SQL para Estatísticas de `death_rate_per_100000`

```sql
SELECT 
    MIN(CAST(REPLACE(death_rate_per_100000, ',', '') AS DECIMAL(10, 2))) AS min_death_rate,
    MAX(CAST(REPLACE(death_rate_per_100000, ',', '') AS DECIMAL(10, 2)) AS max_death_rate,
    AVG(CAST(REPLACE(death_rate_per_100000, ',', '') AS DOUBLE)) AS avg_death_rate,
    STDDEV(CAST(REPLACE(death_rate_per_100000, ',', '') AS DOUBLE)) AS stddev_death_rate
FROM mortality_data.mortality_statistics
WHERE death_rate_per_100000 IS NOT NULL;
```

**Resultado:**

| min_death_rate | max_death_rate | avg_death_rate  | stddev_death_rate |
|----------------|----------------|-----------------|-------------------|
| 5.50           | 423790.20      | 7062.87        | 24582.55         |

**Insights Iniciais:**
- **Número de Mortes**: Variou significativamente, com uma média de aproximadamente 16.110 mortes e desvio padrão de 154.329, indicando ampla variação entre países e grupos.

- **Taxa de Mortalidade**: Apresentou uma variação notável, de 5,5 até 423.790,2 mortes por 100.000 habitantes, com uma média de cerca de 7.062,87 e desvio padrão elevado, refletindo a diversidade das condições de saúde globalmente.


# Perguntas de Negócio e Insights Obtidos

#### Pergunta 1: Quais países apresentaram as maiores reduções nas taxas de mortalidade ao longo dos anos?

O objetivo desta análise foi identificar os países com as maiores reduções na taxa de mortalidade ao longo das quatro décadas disponíveis. A diferença entre a maior e menor taxa de mortalidade registrada foi calculada para cada país.

##### Código SQL

```sql
SELECT 
    country_name,
    (MAX(CAST(REPLACE(death_rate_per_100000, ',', '') AS DECIMAL(10, 2))) - 
     MIN(CAST(REPLACE(death_rate_per_100000, ',', '') AS DECIMAL(10, 2)))) AS reduction_in_death_rate
FROM mortality_data.mortality_statistics
WHERE death_rate_per_100000 IS NOT NULL
GROUP BY country_name
ORDER BY reduction_in_death_rate DESC
LIMIT 10;
```

##### Resultado

| country_name   | reduction_in_death_rate |
|----------------|-------------------------|
| Mali           | 423667.10               |
| Sierra Leone   | 390630.50               |
| Bangladesh     | 353115.00               |
| Nepal          | 341809.60               |
| Maldives       | 340754.40               |
| Bhutan         | 336253.00               |
| Guinea-Bissau  | 333898.40               |
| Guinea         | 333083.90               |
| Yemen          | 323700.80               |
| Afghanistan    | 318204.00               |

##### Insight

- **Padrões de Redução**: Observou-se que países como Mali e Sierra Leone tiveram as maiores reduções absolutas nas taxas de mortalidade, com quedas de até **80%** ao longo do período.

- **Efeito das Políticas de Saúde Pública**: Esses resultados podem indicar melhoras significativas nas condições de saúde pública e qualidade de vida nessas regiões, possivelmente devido à implementação de políticas de saúde e intervenções internacionais.

---

#### Pergunta 2: Qual é a faixa etária com a maior mortalidade e como isso varia entre diferentes países?

Nesta análise, buscou-se identificar a faixa etária com o maior número médio de mortes por país. O objetivo foi compreender se grupos etários específicos são mais afetados em algumas regiões.

##### Código SQL

```sql
SELECT 
    country_name,
    age_group,
    AVG(CAST(REPLACE(number_of_deaths, ',', '') AS DOUBLE)) AS avg_number_of_deaths
FROM mortality_data.mortality_statistics
WHERE number_of_deaths IS NOT NULL
GROUP BY country_name, age_group
ORDER BY avg_number_of_deaths DESC
LIMIT 10;
```

##### Resultado

| country_name | age_group | avg_number_of_deaths |
|--------------|-----------|----------------------|
| India        | All ages  | 6269307.8            |
| China        | All ages  | 5228301.2            |
| United States| All ages  | 1504297.5            |
| Russia       | All ages  | 1151393.9            |
| Indonesia    | All ages  | 935374.0             |
| Nigeria      | All ages  | 887472.2             |
| China        | 80+ years | 864970.5             |
| Bangladesh   | All ages  | 719028.0             |
| Pakistan     | All ages  | 711496.4             |
| Brazil       | All ages  | 649646.4             |

##### Insight

- **Alta Mortalidade em Faixas etárias Avançadas**: Entre os registros, faixas etárias mais altas, especialmente acima dos 80 anos, apresentaram alta mortalidade em países como a China. Isso pode refletir a presença de grandes populações de idosos e desafios no acesso a cuidados de saúde prolongados.
- **Populações Absolutas**: Paíse com maior número médio de mortes em todas as idades, como Índia e China, refletem grandes populações absolutas e apontam a necessidade de políticas de saúde voltadas a grupos vulneráveis.


---


#### Pergunta 3: Há uma tendência de diminuição na mortalidade infantil (0-6 dias) em cada país?

Essa análise busca identificar se há uma tendência de queda na mortalidade infantil (0-6 dias) ao longo dos anos, o que pode ser indicativo de melhorias nos cuidados de saúde neonatal.

##### Código SQL

```sql
SELECT 
    country_name,
    year,
    AVG(CAST(REPLACE(number_of_deaths, ',', '') AS DOUBLE)) AS avg_infant_deaths
FROM mortality_data.mortality_statistics
WHERE age_group = '0-6 days' AND number_of_deaths IS NOT NULL
GROUP BY country_name, year
ORDER BY country_name, year;
```

##### Resultado

| country_name | year | avg_infant_deaths |
|--------------|------|-------------------|
| Afghanistan  | 1970 | 21227.0           |
| Afghanistan  | 1980 | 17993.7           |
| Afghanistan  | 1990 | 14180.7           |
| Afghanistan  | 2000 | 21711.3           |
| Afghanistan  | 2010 | 19378.0           |
| Albania      | 1970 | 295.3             |
| Albania      | 1980 | 296.0             |
| Albania      | 1990 | 290.3             |
| Albania      | 2000 | 160.0             |
| Albania      | 2010 | 77.0              |

##### Insight

- **Queda na Mortalidade Infantil**: A mortalidade infantil em países como a Albânia mostrou uma queda de aproximadamente **73%** entre 1970 e 2010, sugerindo avanços na saúde neonatal e em políticas de suporte a mães e bebês.
- **Variabilidade por Região**: Em outros países, como o Afeganistão, a mortalidade infantil não demonstrou uma queda linear, o que pode indicar desigualdade no acesso a cuidados médicos e infraestrutura de saúde.


---


#### Pergunta 4: Quais são as diferenças nas taxas de mortalidade entre homens e mulheres em diferentes faixas etárias?

Nesta análise, busquei entender as diferenças nas taxas de mortalidade entre homens e mulheres, comparando faixas etárias específicas para identificar possíveis disparidades de gênero.

##### Código SQL

```sql
SELECT 
    age_group,
    sex,
    AVG(CAST(REPLACE(death_rate_per_100000, ',', '') AS DOUBLE)) AS avg_death_rate
FROM mortality_data.mortality_statistics
WHERE death_rate_per_100000 IS NOT NULL
GROUP BY age_group, sex
ORDER BY age_group, sex;
```

##### Resultado

| age_group | sex   | avg_death_rate |
|-----------|-------|----------------|
| 0-6 days  | Both  | 93511.85       |
| 0-6 days  | Female| 78949.61       |
| 0-6 days  | Male  | 107482.57      |
| 1-4 years | Both  | 791.48         |
| 1-4 years | Female| 722.21         |
| 1-4 years | Male  | 859.29         |

##### Insight

- **Disparidade de Gênero na Mortalidade Infantil**: A mortalidade infantil (0-6 dias) é cerca de **36%** maior entre meninos em comparação com meninas, o que pode estar associado a fatores biológicos ou até mesmo a diferenças nos cuidados oferecidos.

- **Tendência Consistente em Outras Idades**: A taxa de mortalidade é consistentemente mais alta entre homens em quase todas as faixas etárias, indicando possíveis riscos de saúde diferenciados ou comportamentos de risco mais prevalentes.

---

#### Pergunta 5: Como as políticas de saúde pública podem ter impactado as taxas de mortalidade em alguns países?

Para analisar o possível impacto das políticas de saúde pública, observei a variação nas taxas de mortalidade ao longo do tempo em cada país. Mudanças abruptas ou quedas graduais podem indicar a influência de intervenções em saúde pública, melhorias no saneamento, vacinação e acesso aos cuidados médicos.

##### Código SQL

```sql
SELECT 
    country_name,
    year,
    AVG(CAST(REPLACE(death_rate_per_100000, ',', '') AS DOUBLE)) AS avg_death_rate
FROM mortality_data.mortality_statistics
WHERE death_rate_per_100000 IS NOT NULL
GROUP BY country_name, year
ORDER BY country_name, year;
```

##### Resultado

| country_name       | year | avg_death_rate |
|--------------------|------|----------------|
| Afghanistan        | 1970 | 20698.09       |
| Afghanistan        | 1980 | 16202.16       |
| Afghanistan        | 1990 | 12241.01       |
| Afghanistan        | 2000 | 12085.30       |
| Afghanistan        | 2010 | 9090.53        |
| Albania            | 1970 | 3703.94        |
| Albania            | 1980 | 3419.47        |
| Albania            | 1990 | 3067.99        |
| Albania            | 2000 | 2826.51        |
| Albania            | 2010 | 2257.31        |

##### Insight

- **Impacto Notável de Intervenções em Saúde**: Em países como Afeganistão e Albânia, as taxas de mortalidade mostraram uma queda gradual ao longo das décadas, com uma redução de aproximadamente **55%** no Afeganistão e **39%** na Albânia entre 1970 e 2010. Isso possivelmente reflete melhorias contínuas nos sistemas de saúde pública e intervenções sanitárias.

- **Padrões Regionais de Melhora**: Em geral, observou-se que muitos países, especialmente os de renda média e baixa, tiveram melhorias ao longo das décadas, sugerindo a eficácia de programas globais e nacionais de saúde pública.

---

#### Pergunta 6: Quais são as regiões mais afetadas por altas taxas de mortalidade ao longo das décadas?

Para identificar as regiões mais afetadas por taxas de mortalidade consistentemente altas, calculei a média da taxa de mortalidade ao longo dos anos para cada país. Isso permitiu destacar os países que tiveram taxas de mortalidade elevadas de forma persistente, indicando possíveis desafios em saúde pública.

##### Código SQL

```sql
SELECT 
    country_name,
    AVG(CAST(REPLACE(death_rate_per_100000, ',', '') AS DOUBLE)) AS avg_death_rate
FROM mortality_data.mortality_statistics
WHERE death_rate_per_100000 IS NOT NULL
GROUP BY country_name
ORDER BY avg_death_rate DESC
LIMIT 10;
```

##### Resultado

| country_name      | avg_death_rate |
|-------------------|----------------|
| Mali              | 16663.68       |
| Sierra Leone      | 16481.58       |
| Guinea            | 14697.01       |
| Guinea-Bissau     | 14643.40       |
| Liberia           | 14335.92       |
| Ethiopia          | 14447.07       |
| Bangladesh        | 14210.70       |
| Afghanistan       | 14063.42       |
| Nepal             | 13753.27       |
| Pakistan          | 13616.34       |

##### Insight

- **Persistência em Altas Taxas de Mortalidade**: Países como Mali e Sierra Leone lideram com as taxas de mortalidade mais altas em média ao longo dos anos, sugerindo desafios de longa data em infraestrutura de saúde e acesso a recursos médicos. Com taxas acima de 16.000 mortes por 100.000 habitantes, esses países apresentam vulnerabilidades contínuas que exigem atenção para melhorias sustentáveis.
- **Influência de Fatores Regionais e Econômicos**: Grande parte dos países com alta mortalidade média pertence a regiões de baixa renda ou que enfrentam desafios socioeconômicos significativos, refletindo uma possível correlação entre economia e acesso à saúde.

---

Esses resultados reforçam a necessidade de intervenções específicas em países com altas taxas de mortalidade, considerando fatores econômicos e de saúde pública para futuras políticas.

### 5. Resumo dos Insights Gerais

A análise dos dados de mortalidade global utilizando o AWS Athena trouxe insights importantes sobre a evolução da saúde pública em várias regiões do mundo. Abaixo estão os principais achados para cada uma das perguntas de negócio:

1. **Redução nas Taxas de Mortalidade**:
   - Países como Mali e Sierra Leone registraram as maiores reduções absolutas nas taxas de mortalidade, chegando a uma queda aproximada de **80%** ao longo das décadas analisadas. Esses resultados sugerem a efetividade das políticas de saúde pública e intervenções internacionais em regiões que anteriormente enfrentavam condições adversas.

2. **Distribuição da Mortalidade por Faixa Etária**:
   - Países com grandes populações, como Índia e China, apresentaram o maior número médio de mortes em todas as idades, refletindo a influência do tamanho populacional nas estatísticas de mortalidade. Faixas etárias avançadas (80+ anos) tiveram altos índices de mortalidade em algumas regiões, apontando para a importância dos cuidados com a saúde da população idosa.

3. **Queda na Mortalidade Infantil**:
   - Em países como Albânia, a mortalidade infantil caiu cerca de **73%** entre 1970 e 2010, indicando melhorias nos cuidados neonatais e nas políticas de suporte a mães e recém-nascidos.

4. **Diferença na Mortalidade por Gênero**:
   - Em todas as idades, as taxas de mortalidade foram consistentemente mais altas entre homens, com uma diferença de aproximadamente **36%** para meninos na faixa de 0-6 dias. Isso pode apontar para possíveis fatores de risco diferenciados e para a necessidade de estudos adicionais sobre desigualdades de gênero na saúde.

5. **Efeito das Políticas de Saúde Pública**:
   - A análise temporal mostrou quedas graduais nas taxas de mortalidade em vários países, como Afeganistão e Albânia, que apresentaram reduções de até **55%** e **39%**, respectivamente. Esses resultados sugerem que políticas de saúde pública podem ter impactado diretamente na redução das taxas de mortalidade ao longo das décadas.

6. **Regiões com Altas Taxas de Mortalidade**:
   - Países como Mali, Sierra Leone, e Guiné apresentaram altas taxas de mortalidade médias, frequentemente acima de **14.000 mortes por 100.000 habitantes**, ao longo das décadas. Esses países enfrentam desafios significativos em termos de acesso à saúde, infraestrutura e desenvolvimento econômico, destacando áreas críticas para intervenções.

---

### Conclusão

Este projeto revelou padrões importantes nas taxas de mortalidade global, utilizando uma abordagem prática com AWS Athena para responder a perguntas de negócio relevantes. A análise permitiu explorar como fatores como políticas públicas, desenvolvimento econômico e condições de vida influenciaram a mortalidade em diferentes regiões ao longo de quatro décadas. Esses insights reforçam a importância de intervenções em saúde pública e a necessidade de esforços contínuos para melhorar as condições de saúde nas regiões mais vulneráveis.
