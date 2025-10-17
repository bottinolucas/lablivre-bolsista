# lablivre-bolsista
Este repositório é dedicado à resolução do teste técnico para a vaga de bolsista em Engenharia/Análise de Dados no LabLivre.

# Executando o projeto

1.  Crie um ambiente virtual Python: `python -m venv venv`
2.  Inicie o ambiente virtual: `source venv/bin/activate`
3.  Suba o container: `docker-compose up --build -d`
4.  Execute o arquivo `etl.ipynb`
5.  Acesse o pgAdmin em `http://localhost:5051/browser/` com login `admin@admin.com` e senha `admin`
6.  Para ter acesso ao banco de dados pelo pgAdmin, insira os dados:
    -  Na aba General:
      -  Name: lablivre
    -  Na aba Connection:
      -  Host name/address: postgres
      -  Port: 5432
      -  Maintenance database: lablivre
      -  Username: admin
      -  Password: admin  


# Documentando a solução para o problema proposto

Para a solução do desafio técnico, pensei em modularizar algumas funções para fazer o processo de ETL (Extract, Transform, Load). Entre a etapa de Extração de dados brutos e Transformação, a análise de dados foi realizada.

# ETL – ObrasGov

## 1. Extract

O objetivo desta etapa é consumir dados de uma API externa (**obrasgov**) e retornar os dados em um **DataFrame pandas**, facilitando a manipulação posterior.

O endpoint `/projeto-investimento`, disponibilizado no [obrasgov](https://api.obrasgov.gestao.gov.br/obrasgov/api/swagger-ui/index.html#/Projeto%20De%20Investimento/buscarPorFiltro), retorna uma **lista paginada** com os dados de identificação das intervenções com base nos filtros utilizados. Esses filtros não são obrigatórios, mas, para a solução do problema proposto, buscamos pelo **UF principal da intervenção igual a DF**.

Por padrão, o endpoint retorna **10 elementos por página**. Para extrair todos os dados, um **DataFrame principal**  foi criado e, a cada iteração, um **DataFrame temporário representando a página** concatena os dados no DataFrame principal. Foram encontrados no total 784 registros.

---

## 2. Transform

Para realizar a transformação dos dados, o primeiro passo é entender a estrutura do DataFrame após a extração.

Os dados brutos são retornados em **JSON**, alguns já com a coluna bem definida e outros em formato de **lista**. Para analisá-los corretamente, é necessário **desempacotar a lista e transformar os elementos em colunas**, garantindo que cada atributo da intervenção se torne uma coluna do DataFrame.

A função `normalize_df` normaliza a base de dados, desempacotando os dados em lista e transformando cada elemento em uma coluna no DataFrame.

A função `data_analysis` explora os dados, mostrando uma visão geral do DataFrame, apresentando a quantidade de linhas, colunas, dimensão, tipos de variáveis, taxa de valores nulos, estatísticas descritivas e valores únicos por coluna.

Para visualizar todas as colunas do DataFrame, o pandas permite que utilizemos o `pd.set_option` para indicar qual tipo de display queremos. Como queremos ver todo o DataFrame, utilizamos `display.max_columns` como o primeiro parâmetro e `None` para o pandas não limitar o número de colunas exibidas.

Para realizar a etapa de Transformação, três funções auxiliares foram criadas. 

A função `standard_columns` padroniza as colunas do DataFrame no formato snake_case, sem caracteres especiais e em letras minúsculas.

A função `type_casting` recebe um DataFrame do pandas e aplica conversões de tipos para todas as colunas do DataFrame de acordo com a sua natureza. A conversão ocorre para os seguintes tipos: int, string, float, datetime64, category, boolean.

A função `nulls_treatment` tem como objetivo tratar valores ausentes do DataFrame a fim de garantir que as análises posteriores não sejam prejudicadas. Como o DataFrame é relativamente pequeno (784 registros), perder dados excluindo valores nulos seria problemático por cada registro representar uma parte significativa do Dataset. Dessa forma, a função preenche os valores ausentes com informações padrão adequadas ao tipo da coluna.

Em seguida, a remoção das duplicatas é feita na linha `df = df.drop_duplicates()`

Por fim, a função `transform` é chamada, recebendo um DataFrame e retornando outro DataFrame tratado, concluindo a etapa de Transformação.

---

## 3. Load

Após a transformação, os dados são tratados para garantir **tipagem adequada**, **normalização** (padronização de colunas, remoção de duplicatas e tratamento de valores ausentes) e consistência.

Os dados finalizados são então armazenados em um **Banco de Dados**. Para este teste, escolhemos o **PostgreSQL**, por ser **open-source** e robusto para manipulação de dados estruturados.

Para realizar a etapa de Carregamento, três funções auxiliares foram criadas. 

A função `sql_to_str` converte o arquivo DDL de criação do Banco de Dados no Postgres para string, permitindo executar a criação da tabela pelo SQLAlchemy. 

A função `create_table` cria a tabela no Banco de Dados, fazendo o casting com o tipo `text` do ORM.

A função `insert_data` insere os dados no Banco de Dados pelo ORM. 

Por fim, a função `load`é chamada, convertendo o script sql para string, criando a tabela no banco de dados e inserindo os dados na tabela.

## 4.  Análise de Dados

Na etapa de análise de dados, a função `data_analysis` foi construída com o intuito de ter uma visão geral do nosso DataFrame. 

O DataFrame possui 784 registros, 40 colunas, todas em primeira mão com os tipos `object` e `float64`.

Colunas que estão praticamente completas (poucos valores nulos):
-   idUnico
-   nome
-   descricao
-   dataCadastro
-   uf
-   fontesDeRecurso_origem
-   fontesDeRecurso_valorInvestimentoPrevisto

As colunas com mais valores nulos são:
-   dataInicialEfetiva: 765 (~97%)
-   dataFinalEfetiva: 779 (~99%)
-   descPopulacaoBeneficiada: 607 (~77%)
-   observacoesPertinentes: 644 (~82%)
-   naturezaOutras: 599 (~76%)
-   qdtEmpregosGerados: 616 (~78%) 

Foram encontrados também 146 registros duplicados (cerca de 19% do dataset), que indica que os dados realmente precisam de um processo de limpeza antes de análises mais profundas.

Após a etapa de transformação, ao verificarmos novamente os dados com a função `data_analysis`, ainda é possível observar alguns valores nulos. Isso ocorre porque remover essas linhas ou preenchê-las automaticamente com valores como a mediana poderia comprometer análises posteriores, alterando distribuições e relações importantes entre as variáveis. Portanto, optou-se por manter esses nulos, preservando a integridade do conjunto de dados para análises futuras.

## 5.   Data Visualization

Na visualização de dados, podemos identificar algumas informações bem interessantes:

-   A fonte de recurso de todos os projetos é o Governo Federal.
-   Quanto maior o investimento, maior a chance do projeto ser cadastrado.
-   As obras CADASTRADAS representam a maior quantidade dentro do dataset.
-   O Fundo Nacional de Desenvolvimento da Educação é o executor com mais projetos.
-   A natureza com mais projetos é Obra.
-   O Departamento Nacional de Infraestrutura de Transportes é o que possui a maior previsão de investimento por executor.
