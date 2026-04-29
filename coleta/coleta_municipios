#!/usr/bin/env Rscript
# ==============================================================================
# COLETA — BASE MUNICÍPIOS
# Detecta o DTB mais recente no FTP do IBGE, baixa, lê e salva bronze Parquet.
# Saída: bronze/base_municipios/raw_municipios_YYYYMMDD.parquet
# ==============================================================================
library(httr2)
library(readxl)
library(dplyr)
library(glue)
source("utils.R")
Sys.setlocale("LC_ALL", "C.UTF-8")

BASE_DTB <- "https://geoftp.ibge.gov.br/organizacao_do_territorio/estrutura_territorial/divisao_territorial/"

# ------------------------------------------------------------------------------
# 1. DESCOBERTA DINÂMICA DO ANO MAIS RECENTE
# ------------------------------------------------------------------------------
descobrir_ano_dtb <- function(base_url, ano_max = as.integer(format(Sys.Date(), "%Y")), janela = 6) {
  cat("[COLETA] Procurando DTB mais recente...\n")
  for (ano in seq(ano_max, ano_max - janela)) {
    url_zip <- glue("{base_url}{ano}/DTB_{ano}.zip")
    ok <- tryCatch({
      resp <- req_perform(request(url_zip) |> req_method("HEAD") |> req_error(is_error = \(r) FALSE))
      resp_status(resp) == 200
    }, error = function(e) FALSE)
    if (ok) {
      cat(glue("[COLETA] DTB {ano} encontrado.\n"))
      return(list(ano = ano, url = url_zip))
    }
  }
  stop("Nenhuma versão do DTB encontrada.")
}

# ------------------------------------------------------------------------------
# 2. COLETA
# ------------------------------------------------------------------------------
collect_data <- function() {
  tryCatch({
    dtb_info <- descobrir_ano_dtb(BASE_DTB)
    ano_dtb  <- dtb_info$ano
    url_dtb  <- dtb_info$url

    zip_tmp <- tempfile(fileext = ".zip")
    extr_tmp <- tempdir()

    cat(glue("[COLETA] Baixando DTB {ano_dtb}...\n"))
    download.file(url_dtb, zip_tmp, mode = "wb", quiet = TRUE)
    unzip(zip_tmp, exdir = extr_tmp)

    xls_files <- list.files(extr_tmp, pattern = "MUNICIPIOS\\.(xls|xlsx)$",
                             recursive = TRUE, full.names = TRUE, ignore.case = TRUE)
    if (length(xls_files) == 0) {
      xls_files <- list.files(extr_tmp, pattern = "MUNICIPIO.*\\.(xls|xlsx)$",
                               recursive = TRUE, full.names = TRUE, ignore.case = TRUE)
    }
    if (length(xls_files) == 0) stop("Arquivo de municípios não encontrado no ZIP.")

    xls_path <- xls_files[1]
    cat(glue("[COLETA] Lendo: {basename(xls_path)}\n"))

    raw <- tryCatch(
      read_excel(xls_path, sheet = 1, col_types = "text", skip = 6),
      error = function(e) read_excel(xls_path, sheet = 1, col_types = "text", skip = 5)
    )

    raw$ano_dtb     <- as.character(ano_dtb)
    raw$data_coleta <- format(Sys.time(), "%Y%m%d")

    cat(glue("[COLETA] {nrow(raw)} municípios lidos.\n"))
    raw
  }, error = function(e) {
    cat("[COLETA] Erro:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

# ------------------------------------------------------------------------------
# 3. SALVAR BRONZE
# ------------------------------------------------------------------------------
save_to_minio_duckdb <- function(data) {
  tryCatch({
    timestamp <- format(Sys.time(), "%Y%m%d")
    filepath  <- sprintf("bronze/base_municipios/raw_municipios_%s.parquet", timestamp)
    write_parquet_to_minio(data, filepath)
    filepath
  }, error = function(e) {
    cat("[COLETA] Erro ao salvar:", conditionMessage(e), "\n")
    quit(status = 1)
  })
}

dados   <- collect_data()
arquivo <- save_to_minio_duckdb(dados)
cat("[COLETA] Finalizado:", arquivo, "\n")
