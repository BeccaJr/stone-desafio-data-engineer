Desafio Stone — Pipeline de Dados (Receita Federal CNPJ)
Este repositório contém a solução do Desafio Stone para ingestão e processamento dos dados abertos de CNPJ disponibilizados pela Receita Federal. O projeto segue o modelo medalhão (bronze → silver → gold), é containerizado com Docker e persiste o resultado final em um banco Postgres para consumo transacional.

Sumário:
- Objetivo
- Arquivos principais
- Pré-requisitos
- Estrutura do projeto
- Descrição das camadas (medalhão)
- Passo a passo para execução
- Saídas esperadas
- Comandos úteis
- Observações e troubleshooting
- Contato

Objetivo:
Ingerir os arquivos "Empresas.ZIP" e "Socios.ZIP" do endpoint público da Receita Federal, tratar os dados e gerar uma tabela final com as colunas solicitadas pelo desafio:
- cnpj (string)
- qtde_socios (int)
- flagsocioestrangeiro (boolean)
- doc_alvo (boolean)

Além disso, salvar os artefatos intermediários nas camadas bronze/silver/gold e carregar o resultado final em uma tabela Postgres (empresas_gold).

Arquivos principais:
- docker-composer.yml — definição dos serviços (Postgres e container da aplicação/JupyterLab)
- Dockerfile — imagem base do app
- requirements.txt — dependências Python
- src/ — código e notebooks (stg_bronze.ipynb, stg_silver.ipynb, stg_gold.ipynb)
- data/ — diretórios mountados para armazenar bronze/, silver/, gold/
- .env (não comitado; .env.example disponível no Reposiório)

Pré-requisitos:
- Docker (recomendado Docker Desktop) instalado
- Docker Compose (ou Docker Compose V2 integrado: docker compose)
- Espaço em disco suficiente (os arquivos da Receita são grandes; a camada bronze pode chegar a dezenas de GB)
- Memória RAM suficiente para processamento (os notebooks usam pandas/polars; polars reduz uso de memória na etapa gold)
- Crie um arquivo .env na raiz do projeto com o conteúdo do .env.example
  - Observação: o docker-composer.yml já cria o banco Postgres com usuário stone e senha stone. As variáveis do .env são usadas pelo notebook stg_gold para conectar ao Postgres.
  
Estrutura do projeto (resumo):
- docker-composer.yml
- Dockerfile
- requirements.txt
- .env.example
- src/
  - stg_bronze.ipynb — baixa os .zip da Receita, salva e extrai na camada data/bronze
  - stg_silver.ipynb — lê os arquivos extraídos, faz tipagem, seleção e sanitização; salva em data/silver/*.parquet
  - stggold.ipynb — usa polars para agregar sócios por CNPJ, aplicar regras de negócio (qtdesocios, flagsocioestrangeiro, doc_alvo), salva data/gold/gold.parquet e carrega a tabela empresas_gold no Postgres
- data/
  - bronze/
  - silver/
  - gold/
  
Descrição das camadas (medalhão) adotadas neste projeto:
 - Bronze: armazenamento raw dos arquivos .zip e dos arquivos extraídos originais. Objetivo: persistir exatamente o que foi baixado, sem transformações. Local: data/bronze.
 - Silver: dados tratados e tipados (colunas renomeadas, conversão de tipos, limpeza básica). Objetivo: dataset pronto para análises e junções. Formato: Parquet. Local: data/silver.
 - Gold: produto final com regras de negócio aplicadas (agregações, flags, colunas calculadas) e pronto para consumo transacional ou aplicações. Salvo em Parquet (data/gold/gold.parquet) e carregado em Postgres na tabela empresas_gold.
 
Passo a passo para execução local com Docker:
1) Clone o repositório:

  git clone GITHUB

  cd

2) Crie o arquivo .env na raiz (use o .env.example).

3) Levante o ambiente com Docker Compose (exemplo com o arquivo presente docker-composer.yml):

  Usando Docker Compose V2 (recomendado):
    docker compose -f docker-composer.yml up --build

  Ou com docker-compose (se disponível):
    docker-compose -f docker-composer.yml up --build

Este comando irá:

- subir um container Postgres (stone_db) com as credenciais definidas
- buildar a imagem do app e subir o container (stone_app) expondo JupyterLab em localhost:8888
- Os volumes montados garantem persistência dos dados em ./data e do Postgres em pgdata.

4) Acesse o JupyterLab:

- Abra no navegador http://localhost:8888. O command no compose já define --NotebookApp.token='', portanto não deve exigir token.

5) Sequência de execução dos notebooks (ordem obrigatória):

- stg_bronze.ipynb — execute todas as células para baixar o .zip mais recente do site da Receita e extrair os arquivos em data/bronze.
- stg_silver.ipynb — execute todas as células para ler os arquivos extraídos, aplicar limpeza e salvar os datasets em Parquet dentro de data/silver.
- stg_gold.ipynb — execute todas as células para carregar os Parquet da silver, agregar/juntar os dados, gerar as colunas de negócio e:
  - salvar data/gold/gold.parquet
  - carregar a tabela empresas_gold no Postgres usando COPY via psycopg2 (os parâmetros de conexão são lidos de ../.env dentro do container).

Observação: Caso queira executar os notebooks em modo automático, pode-se adaptar os scripts para rodar via papermill ou converter para scripts Python; no entanto, a entrega contém os notebooks para avaliação manual.

Saídas esperadas:
- Parquet Silver:
  - data/silver/empresas_silver.parquet — colunas: cnpj, razaosocial, naturezajuridica, qualificacaoresponsavel, capitalsocial, cod_porte
  - data/silver/socios_silver.parquet — colunas: cnpj, tiposocio, nomesocio, documentosocio, codigoqualificacao_socio
- Parquet Gold:
  - data/gold/gold.parquet — tabela final unindo empresas e agregados de sócios com colunas:
    - cnpj (string)
    - razao_social (string)
    - natureza_juridica (int)
    - qualificacao_responsavel (int)
    - capital_social (float/decimal)
    - cod_porte (string)
    - qtde_socios (int)
    - flagsocioestrangeiro (boolean) — true se há ao menos um sócio estrangeiro (documento_socio começando com '999' no tratamento atual)
    - docalvo (boolean) — regra: True quando codporte == "03" e qtde_socios > 1
Postgres:
- Tabela empresas_gold criada e populada via COPY com o conteúdo do Gold.
  - Exemplo do DDL usado no notebook (para referência):

	DROP TABLE IF EXISTS empresas_gold;

	DROP TABLE IF EXISTS empresas_gold;
	    CREATE TABLE empresas_gold (
		cnpj TEXT,
		qtde_socios INT,
		flag_socio_estrangeiro BOOLEAN,
		doc_alvo BOOLEAN
	    );

Comandos úteis:
- Subir ambiente:
  - docker compose -f docker-composer.yml up --build
- Subir em background:
  - docker compose -f docker-composer.yml up -d --build
- Parar e remover containers:
  - docker compose -f docker-composer.yml down
- Entrar no container app (ex.: para abrir um terminal):
  - docker exec -it stone_app bash
- Acessar Postgres via psql no host (ex.: dentro do container db):
  - docker exec -it stone_db psql -U stone -d stone
- Conexão externa (por exemplo, psql local) usando:
  - psql postgresql://stone:stone@localhost:5432/stone
  
Observações importantes / Performance:
- Os arquivos públicos da Receita são grandes (milhões de linhas). Ao processar as tabelas completas, espere tempo de execução considerável e uso de CPU/RAM/disk substanciais.
- A etapa Gold usa Polars para melhorar performance comparado ao pandas em agregações; ainda assim é recomendável ter >= 8-16GB de RAM dependendo do volume.
- Se o avaliador quiser testar rapidamente, é possível modificar os notebooks para processar apenas um subset (usar .head(n) ou nrows na leitura) — adicione um parâmetro de SAMPLE_MODE nos notebooks para facilitar testes.
- Certifique-se de ter espaço em disco suficiente para os arquivos extraídos em data/bronze.

Troubleshooting (erros comuns):
- "FileNotFoundError: Nenhum arquivo terminando com '.EMPRECSV' encontrado": confirme se o stg_bronze.ipynb foi executado e os arquivos foram extraídos em data/bronze.
- Problemas de permissão ao escrever em data/: ajuste permissões ou execute o Docker com o usuário apropriado.
- Ports ocupadas (5432 ou 8888): pare serviços locais que usam essas portas ou altere os mapeamentos no docker-composer.yml.
- Erro de conexão no stg_gold para Postgres: verifique o .env e se o container Postgres está em execução (docker ps).

Boas práticas aplicadas:
- Uso do padrão medalhão (bronze/silver/gold) para separar raw/tratado/produto final.
- Containerização para reprodutibilidade (Docker).
- Persistência dos dados via volumes e diretórios data/.
- Uso de Polars para operações de agregação mais eficientes.
- Carregamento em massa no Postgres via COPY para performance.

O que o avaliador precisa executar (resumo mínimo)
1) Clonar repositório.

2) Criar .env conforme .env.example.

3) Executar docker compose -f docker-composer.yml up --build.

4) Abrir http://localhost:8888 e executar os notebooks na ordem: stgbronze → stgsilver → stg_gold.

5) Validar data/gold/gold.parquet e tabela empresas_gold no Postgres.

Contato:
Se precisar de ajustes (ex.: reduzir volume de teste, adicionar script automático de execução, gerar .env de exemplo no repositório), me avise.
