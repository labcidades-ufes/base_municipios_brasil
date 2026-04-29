#!/usr/bin/env Rscript
# ==============================================================================
# PRÉ-PROCESSAMENTO — BASE MUNICÍPIOS (Silver)
# Lê bronze, renomeia colunas, valida cod_ibge_7, detecta duplicatas e UFs.
# Entrada:  bronze/base_municipios/raw_municipios_YYYYMMDD.parquet
# Saída:    silver/base_municipios/silver_municipios_YYYYMMDD.parquet
# ==============================================================================
library(dplyr)
library(stringr)
library(glue)
source("utils.R")
Sys.setlocale("LC_ALL", "C.UTF-8")

# ------------------------------------------------------------------------------
# 1. LEITURA DO BRONZE
# ------------------------------------------------------------------------------
read_from_minio_duckdb <- function() {
  cat("[PRE_PROCESSAMENTO] Lendo bronze...\n")
  tryCatch({
    arquivos <- list_parquet_files_in_minio("bronze/base_municipios/")
    if (length(arquivos) == 0) stop("Nenhum arquivo bronze encontrado.")
    caminho <- sub(sprintf("^s3://%s/", Sys.getenv("MINIO_BUCKET", "airflow")), "",
                   sort(arquivos, decreasing = TRUE)[1])
    cat(glue("[PRE_PROCESSAMENTO] Lendo: {caminho}\n"))
    read_parquet_from_minio(caminho)
  }, error = function(e) {
    cat("[PRE_PROCESSAMENTO] Erro ao ler bronze:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

# ------------------------------------------------------------------------------
# 2. PRÉ-PROCESSAMENTO + VALIDAÇÕES
# ------------------------------------------------------------------------------
process_data <- function(data) {
  tryCatch({
    cat(glue("[PRE_PROCESSAMENTO] {nrow(data)} municípios recebidos do bronze.\n"))

    # Renomeação e normalização
    registros <- as_tibble(data) |>
      rename(
        cod_uf                    = UF,
        sigla_uf                  = Nome_UF,
        cod_municipio             = `Município`,
        cod_ibge_7                = `Código Município Completo`,
        nome_municipio_original   = `Nome_Município`,
        cod_regiao_intermediaria  = `Região Geográfica Intermediária`,
        nome_regiao_intermediaria = `Nome Região Geográfica Intermediária`,
        cod_regiao_imediata       = `Região Geográfica Imediata`,
        nome_regiao_imediata      = `Nome Região Geográfica Imediata`
      ) |>
      filter(!is.na(cod_ibge_7), !is.na(cod_municipio)) |>
      mutate(across(everything(), ~ dplyr::na_if(trimws(as.character(.x)), "")))

    # --- Validação 1: cod_ibge_7 deve ter exatamente 7 dígitos ---
    invalidos_ibge7 <- registros |>
      filter(!str_detect(cod_ibge_7, "^\\d{7}$"))

    if (nrow(invalidos_ibge7) > 0) {
      cat(glue("[PRE_PROCESSAMENTO] AVISO: {nrow(invalidos_ibge7)} municípios com cod_ibge_7 inválido:\n"))
      print(invalidos_ibge7 |> select(cod_ibge_7, nome_municipio_original, sigla_uf) |> head(10))
    } else {
      cat("[PRE_PROCESSAMENTO] Validação cod_ibge_7 (7 dígitos): OK\n")
    }

    # --- Validação 2: Duplicatas por cod_ibge_7 ---
    duplicatas <- registros |>
      group_by(cod_ibge_7) |>
      filter(n() > 1) |>
      ungroup()

    if (nrow(duplicatas) > 0) {
      cat(glue("[PRE_PROCESSAMENTO] AVISO: {nrow(duplicatas)} registros duplicados por cod_ibge_7:\n"))
      print(duplicatas |> select(cod_ibge_7, nome_municipio_original, sigla_uf) |> head(10))
    } else {
      cat("[PRE_PROCESSAMENTO] Validação de duplicatas: OK\n")
    }

    # --- Validação 3: Contagem de UFs ---
    n_ufs <- n_distinct(registros$sigla_uf)
    if (n_ufs < 27) {
      cat(glue("[PRE_PROCESSAMENTO] AVISO: apenas {n_ufs}/27 UFs encontradas.\n"))
      ufs_presentes <- sort(unique(registros$sigla_uf))
      ufs_todas     <- c("AC","AL","AM","AP","BA","CE","DF","ES","GO","MA","MG",
                         "MS","MT","PA","PB","PE","PI","PR","RJ","RN","RO","RR",
                         "RS","SC","SE","SP","TO")
      cat(glue("[PRE_PROCESSAMENTO] UFs ausentes: {paste(setdiff(ufs_todas, ufs_presentes), collapse=', ')}\n"))
    } else {
      cat(glue("[PRE_PROCESSAMENTO] Validação de UFs: {n_ufs}/27 OK\n"))
    }

    # --- Validação 4: Contagem esperada de municípios (~5570) ---
    n_mun <- nrow(registros)
    if (n_mun < 5500) {
      cat(glue("[PRE_PROCESSAMENTO] AVISO: apenas {n_mun} municípios (esperado ~5570).\n"))
    } else {
      cat(glue("[PRE_PROCESSAMENTO] Contagem de municípios: {n_mun} OK\n"))
    }

    registros
  }, error = function(e) {
    cat("[PRE_PROCESSAMENTO] Erro:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

# ------------------------------------------------------------------------------
# 3. SALVAR SILVER
# ------------------------------------------------------------------------------
save_to_minio_duckdb <- function(data) {
  tryCatch({
    filepath <- sprintf("silver/base_municipios/silver_municipios_%s.parquet", format(Sys.time(), "%Y%m%d"))
    cat(glue("[PRE_PROCESSAMENTO] Salvando em: {filepath}\n"))
    write_parquet_to_minio(data, filepath)
    filepath
  }, error = function(e) {
    cat("[PRE_PROCESSAMENTO] Erro ao salvar:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

bronze <- read_from_minio_duckdb()
silver <- process_data(bronze)
saida  <- save_to_minio_duckdb(silver)
cat("[PRE_PROCESSAMENTO] Finalizado:", saida, "\n")
