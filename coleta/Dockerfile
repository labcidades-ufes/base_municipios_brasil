# 1. Usa a base que você acabou de buildar com sucesso
FROM utils-r_duckdb:latest

# 2. Instala os pacotes R necessários para a coleta
# O httr2 detecta o DTB mais recente no FTP do IBGE; readxl lê o arquivo de municípios
RUN R -e "install.packages(c('httr2','readxl','dplyr','readr','glue'), repos='https://cloud.r-project.org/')"

# 3. Define onde o trabalho acontece
WORKDIR /app

# 4. Copia os arquivos necessários
# Atenção: O build deve ser rodado da RAIZ do projeto
COPY dags/utils/utils.R .
COPY dags/base_municipios_brasil/coleta/coleta_municipios.R .

# 5. Execução
CMD ["Rscript", "coleta_municipios.R"]
