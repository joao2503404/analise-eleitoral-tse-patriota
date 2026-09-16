# Checklist Crítico: Sanitização de Dados do TSE (2026)

**principais problemas do arquivo bruto do TSE** e as **ações imediatas de prevenção** antes de qualquer análise ou carga SQL.

---

## Inconsistências Críticas & Ações Diretas

| # | Inconsistência | Causa / Ocorrência no TSE | Ação Direta (Pandas / SQL) |
|---|---|---|---|
| **1** | **Espaços no Começo/Fim** | Espaços ocultos (`" SÃO PAULO "`). Quebra agrupamentos (`GROUP BY`). | `df[col] = df[col].str.strip()` <br>`TRIM(col)` |
| **2** | **NULL em Texto** | `#NULO#`, `#NE#`, `NULL`, `""` contam como preenchidos. | `df.replace(["#NULO#", "#NE#", "NULL", ""], np.nan)` <br>`NULLIF()` / `CASE` |
| **3** | **Número em Texto (Zeros à Esquerda)** | `NR_CPF` (11d), `NR_CANDIDATO` (5d), `CD_CARGO` (2d) perdem zeros. | `df["NR_CPF"] = df["NR_CPF"].str.zfill(11)` <br>`LPAD(col, 11, '0')` |
| **4** | **Duplicidade de Registros** | Múltiplas atualizações para a mesma chave `SQ_CANDIDATO`. | `df.sort_values(["DT_GERACAO"]).drop_duplicates("SQ_CANDIDATO", keep="last")` |
| **5** | **Encoding Incorreto** | Arquivo em `ISO-8859-1` ou `Windows-1252` corrompe acentos (`SÃ£o Paulo`). | Ler com `encoding='iso-8859-1'` e exportar em `utf-8`. |
| **6** | **Valores Monetários com Vírgula** | `QT_DESPESA_MAX_CAMPANHA` usa vírgula (`"10000,00"`). | `df[col] = pd.to_numeric(df[col].str.replace(",", "."), errors="coerce")` |
| **7** | **Datas Fora do Padrão ISO** | `DT_NASCIMENTO` em `DD/MM/YYYY` em vez de `YYYY-MM-DD`. | `pd.to_datetime(df["DT_NASCIMENTO"], format="%d/%m/%Y")` |
| **8** | **Inconsistência de Caixa/Acentos** | Textos misturando maiúsculas, minúsculas e variações de acento. | `df[col] = df[col].str.upper()` |

---

## Script de Limpeza em 1 Clique (Python)

```python
import pandas as pd, numpy as np

# 1. Carga mantendo tipo texto e encoding correto
df = pd.read_csv("candidatos_2026.csv", sep=";", encoding="iso-8859-1", dtype=str)

# 2. Trim + Caixa Alta + Tratamento de Nulos
df = df.apply(lambda col: col.str.strip().str.upper() if col.dtype == "object" else col)
df.replace(["#NULO#", "#NE#", "#NULO", "NULL", "NAN", "N/A", ""], np.nan, inplace=True)

# 3. Zeros à Esquerda (CPF, Número do Candidato e Código do Cargo)
if "NR_CPF" in df.columns: df["NR_CPF"] = df["NR_CPF"].str.zfill(11)
if "NR_CANDIDATO" in df.columns: df["NR_CANDIDATO"] = df["NR_CANDIDATO"].str.zfill(5)
if "CD_CARGO" in df.columns: df["CD_CARGO"] = df["CD_CARGO"].str.zfill(2)

# 4. Moeda (Vírgula para Ponto) e Data (ISO)
if "QT_DESPESA_MAX_CAMPANHA" in df.columns:
    df["QT_DESPESA_MAX_CAMPANHA"] = pd.to_numeric(df["QT_DESPESA_MAX_CAMPANHA"].str.replace(",", "."), errors="coerce")
if "DT_NASCIMENTO" in df.columns:
    df["DT_NASCIMENTO"] = pd.to_datetime(df["DT_NASCIMENTO"], format="%d/%m/%Y", errors="coerce")

# 5. Deduplicação mantendo o registro mais recente
if "DT_GERACAO" in df.columns:
    df.sort_values(by=["DT_GERACAO", "HH_GERACAO"], inplace=True)
df.drop_duplicates(subset=["SQ_CANDIDATO"], keep="last", inplace=True)

# 6. Salvar em UTF-8
df.to_csv("candidatos_2026_sanitizado.csv", sep=";", index=False, encoding="utf-8")