# 1. Usa a base que você acabou de buildar com sucesso
FROM utils-r_duckdb:latest

# 2. Instala os pacotes R necessários para o pré-processamento
# O dplyr e stringr renomeiam colunas e validam cod_ibge_7
RUN R -e "install.packages(c('dplyr','readr','glue','stringr'), repos='https://cloud.r-project.org/')"

# 3. Define onde o trabalho acontece
WORKDIR /app

# 4. Copia os arquivos necessários
# Atenção: O build deve ser rodado da RAIZ do projeto
COPY dags/utils/utils.R .
COPY dags/base_municipios_brasil/pre_processamento/pre_processamento_municipios.R .

# 5. Execução
CMD ["Rscript", "--vanilla", "pre_processamento_municipios.R"]
