#!/usr/bin/env Rscript
# ==============================================================================
# PROCESSAMENTO — BASE MUNICÍPIOS (Gold)
# Lê silver, aplica limpeza de texto e produz schema final.
# Entrada:  silver/base_municipios/silver_municipios_YYYYMMDD.parquet
# Saída:    gold/base_municipios/gold_municipios_YYYYMMDD.parquet
# Schema:   cod_uf | sigla_uf | cod_municipio | cod_ibge_7 | nome_municipio |
#           nome_municipio_original | cod/nome_regiao_intermediaria |
#           cod/nome_regiao_imediata | ano_dtb
# ==============================================================================
library(dplyr)
library(stringi)
library(stringr)
library(glue)
source("utils.R")
Sys.setlocale("LC_ALL", "C.UTF-8")

limpar_texto <- function(x) {
  x |> stri_trans_general("Latin-ASCII") |> str_to_upper() |> str_squish()
}

read_silver <- function() {
  cat("[GOLD] Lendo silver...\n")
  tryCatch({
    arquivos <- list_parquet_files_in_minio("silver/base_municipios/")
    if (length(arquivos) == 0) stop("Nenhum arquivo silver encontrado.")
    caminho <- sub(sprintf("^s3://%s/", Sys.getenv("MINIO_BUCKET", "airflow")), "",
                   sort(arquivos, decreasing = TRUE)[1])
    cat(glue("[GOLD] Lendo: {caminho}\n"))
    read_parquet_from_minio(caminho)
  }, error = function(e) {
    cat("[GOLD] Erro ao ler silver:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

generate_products <- function(silver) {
  tryCatch({
    cat(glue("[GOLD] Gerando gold com {nrow(silver)} municípios...\n"))

    silver |>
      transmute(
        cod_uf                    = cod_uf,
        sigla_uf                  = sigla_uf,
        cod_municipio             = cod_municipio,
        cod_ibge_7                = cod_ibge_7,
        nome_municipio            = limpar_texto(nome_municipio_original),
        nome_municipio_original   = nome_municipio_original,
        cod_regiao_intermediaria  = cod_regiao_intermediaria,
        nome_regiao_intermediaria = limpar_texto(nome_regiao_intermediaria),
        cod_regiao_imediata       = cod_regiao_imediata,
        nome_regiao_imediata      = limpar_texto(nome_regiao_imediata),
        ano_dtb                   = ano_dtb
      )
  }, error = function(e) {
    cat("[GOLD] Erro na transformação:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

save_gold <- function(data) {
  tryCatch({
    filepath <- sprintf("gold/base_municipios/gold_municipios_%s.parquet", format(Sys.time(), "%Y%m%d"))
    write_parquet_to_minio(data, filepath)
    cat("[GOLD] Salvo em:", filepath, "\n")
    filepath
  }, error = function(e) {
    cat("[GOLD] Erro ao salvar:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

silver <- read_silver()
gold   <- generate_products(silver)
saida  <- save_gold(gold)
cat(glue("[GOLD] Finalizado: {nrow(gold)} municípios | {n_distinct(gold$sigla_uf)} UFs\n"))
