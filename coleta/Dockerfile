# 1. Usa a base que você acabou de buildar com sucesso
# Base do R (dags/utils/r_duckdb). O Makefile sobrescreve via --build-arg.
ARG BASE_IMAGE=utils-r_duckdb:4.6.1-1
FROM ${BASE_IMAGE}

# 2. Instala os pacotes R necessários para a coleta
# O httr2 detecta o DTB mais recente no FTP do IBGE; readxl lê o arquivo de municípios
RUN R -q -e "pkgs <- c('httr2','readxl','dplyr','readr','glue'); \
             install.packages(pkgs); \
             stopifnot(all(pkgs %in% rownames(installed.packages())))"

# 3. Define onde o trabalho acontece
WORKDIR /app

# 4. Copia os arquivos necessários
# Atenção: O build deve ser rodado da RAIZ do projeto
COPY dags/base_municipios_brasil/coleta/coleta_municipios.R .

# 5. Execução
CMD ["Rscript", "coleta_municipios.R"]
