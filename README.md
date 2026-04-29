## DAG Template

Pipeline de coleta e padronização da Divisão Territorial Brasileira (DTB) mais recente, publicada pelo IBGE. Produz a base oficial de municípios brasileiros com código IBGE de 7 dígitos e hierarquia regional completa.

## Estrutura do projeto

- `coleta/`: detecta o DTB mais recente disponível no FTP do IBGE, baixa o arquivo ZIP, extrai o XLS interno e salva na camada bronze.
- `pre_processamento/`: renomeia colunas para schema padrão, valida `cod_ibge_7`, detecta duplicatas e verifica contagem de UFs para a camada silver.
- `processamento/`: aplica limpeza de texto e produz o schema final para a camada gold.
- `utils.R`: funções compartilhadas para leitura/escrita de dados no MinIO via DuckDB.
- `base_municipios_brasil.py`: DAG do Airflow orquestrando as três etapas via DockerOperator.

## Fluxo base do pipeline

1. Coleta: FTP IBGE (DTB) → bronze
2. Pré-processamento: bronze → silver
3. Processamento: silver → gold

## Convenção de camadas

- Bronze: planilha XLS do DTB bruta, com todas as colunas originais e metadados de coleta (`ano_dtb`, `data_coleta`)
- Silver: colunas renomeadas para schema padrão, tipos normalizados e validações aplicadas
- Gold: schema final com texto padronizado em maiúsculas sem acento e grafia original preservada

## Fonte oficial

| Recurso | URL |
|---|---|
| FTP DTB | https://geoftp.ibge.gov.br/organizacao_do_territorio/estrutura_territorial/divisao_territorial/ |

## Detecção automática de versão

O script de coleta testa anos em ordem decrescente via HEAD request no FTP do IBGE e baixa automaticamente o DTB mais recente encontrado. Não é necessário atualizar URLs manualmente quando o IBGE publicar uma nova divisão territorial.

## Validações aplicadas no pré-processamento

- `cod_ibge_7` com exatamente 7 dígitos numéricos
- Ausência de municípios duplicados por `cod_ibge_7`
- Contagem de UFs (esperado: 27)
- Contagem de municípios (esperado: ~5.570)
- Log detalhado de qualquer divergência sem interrupção do pipeline

## Schema gold

```
cod_uf                    — código numérico da UF (ex: "32")
sigla_uf                  — sigla da UF (ex: "ES")
cod_municipio             — código do município sem UF (ex: "05200")
cod_ibge_7                — código IBGE completo de 7 dígitos (ex: "3205200")
nome_municipio            — nome em maiúsculas sem acento (uso em joins)
nome_municipio_original   — nome exato conforme publicado pelo IBGE
cod_regiao_intermediaria  — código da região geográfica intermediária
nome_regiao_intermediaria — nome da região intermediária padronizado
cod_regiao_imediata       — código da região geográfica imediata
nome_regiao_imediata      — nome da região imediata padronizado
ano_dtb                   — ano da divisão territorial utilizada
```
