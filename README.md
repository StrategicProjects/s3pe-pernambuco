# s3pe no s3.pe.gov.br

Exemplos, em R, Python, Rust, Go e linha de comando, de como ler e gravar
dados no **s3.pe.gov.br**, o armazenamento de objetos S3 (um servidor
[Garage](https://garagehq.deuxfleurs.fr)) da plataforma Condepe/Fidem do
Estado de Pernambuco, usando o pacote genérico [s3pe](https://github.com/StrategicProjects/s3pe).

**Site:** <https://strategicprojects.github.io/s3pe-pernambuco/>

## Em três linhas

```
S3PE_ENDPOINT=https://s3.pe.gov.br
AWS_ACCESS_KEY_ID=GK...
AWS_SECRET_ACCESS_KEY=...
```

No R, isso vai no `~/.Renviron`; no shell, num `.env` ou `export`. Peça a
chave do seu bucket a quem administra a plataforma: cada projeto tem um
bucket e uma chave próprios (leitura, ou leitura e escrita), e **não há
leitura anônima**. Nunca deixe a chave em script ou repositório. A região
de assinatura é a padrão, `us-east-1`. Dentro do próprio servidor da
plataforma o endpoint é local: `S3PE_ENDPOINT=http://127.0.0.1:9003`.

## Buckets publicados

| Bucket | O que tem | Chave de leitura |
|---|---|---|
| `iaempe` | o lote do dashboard do IAEM-PE sob `dados/` (JSON e GeoJSON, `metadata.json` por último) e o painel do ETL sob `etl/` (Parquet, `manifest.json` por último) | `iaempe-leitura` |
| `horizonte-fiscal` | séries fiscais sob `series/estado/<codigo>.csv.gz` e `series/municipios/<ibge>/<codigo>.csv.gz`, previsões sob `previsoes/<ibge>/<codigo>/ultima.json`, `manifest.json` por último | `horizonte-fiscal-leitura` |
| `sds` | bucket do projeto SDS, criado em 2026-10-24 para testes com o s3pe; sem layout definido ainda | `sds-leitura` (leitura) e `sds` (leitura e escrita) |

**Regra do manifesto:** o `metadata.json` / `manifest.json` é sempre o
último objeto a subir. Se a data dele é T, tudo o que ele lista já estava
no bucket em T. Leia-o primeiro, use a data como versão do lote e a lista
dele em vez de listar o bucket.

## R

```r
library(s3pe)
use_endpoint("https://s3.pe.gov.br")    # ou S3PE_ENDPOINT no ~/.Renviron
use_bucket("iaempe")

meta       <- read_s3("dados/metadata.json")        # lista; meta$ultimo_mes
series     <- read_s3("dados/series.json")
municipios <- read_s3("dados/pe_municipios.geojson") # sf
painel     <- read_s3("etl/painel_iaempe.parquet")   # tibble (nanoparquet)
list_s3("dados/")

# Horizonte Fiscal: chaves completas quando há mais de um bucket
manifest <- read_s3("s3://horizonte-fiscal/manifest.json")
icms     <- read_s3("s3://horizonte-fiscal/series/estado/icms.csv.gz")
fpm      <- read_s3("s3://horizonte-fiscal/series/municipios/2611606/fpm.csv.gz")
prev     <- read_s3("s3://horizonte-fiscal/previsoes/26/icms/ultima.json")

# escrita (chave com permissão de escrita; use um prefixo seu, não dados/)
write_s3(painel, "nota_tecnica/painel.parquet")
upload_s3("figura1.png", "nota_tecnica/")
```

## Python

```python
import s3pe   # pip install "s3pe[pandas]"

s3pe.configure(endpoint="https://s3.pe.gov.br", bucket="iaempe")
meta   = s3pe.read_s3("dados/metadata.json")          # dict
painel = s3pe.read_s3("etl/painel_iaempe.parquet")    # DataFrame
icms   = s3pe.read_s3("s3://horizonte-fiscal/series/estado/icms.csv.gz")
s3pe.write_s3(painel, "nota_tecnica/painel.csv")
s3pe.download_s3("dados/pe_municipios.geojson", "pe.geojson")
```

## Rust

```rust
let s3 = s3pe::Client::builder()
    .endpoint("https://s3.pe.gov.br")   // ou S3PE_ENDPOINT
    .bucket("iaempe")
    .build()?;                          // credenciais do ambiente
let meta = s3.read("dados/metadata.json")?;
s3.upload("painel.parquet", "etl/painel_iaempe.parquet", None)?;
```

## Go

```go
s3, err := s3pe.New(s3pe.Config{Endpoint: "https://s3.pe.gov.br", Bucket: "iaempe"})
meta, err := s3.Read(ctx, "dados/metadata.json")
objects, err := s3.List(ctx, "dados/")
```

## Publicar um lote: `sync` com manifesto por último

O que os scripts de publicação fazem na mão (subir o que mudou, manifesto no
fim), o `sync` faz por ETag, nas quatro linguagens e na CLI:

```r
sync_up_s3("output/painel", "s3://iaempe/etl/", last = "manifest.json")
```

```bash
s3pe sync output/painel s3://iaempe/etl/ --last manifest.json
s3pe sync s3://iaempe/dados/ ./dados-local          # espelho local do lote
```

Arquivos grandes sobem em partes automaticamente (acima de 64 MiB). Para
compartilhar um objeto com quem não tem credencial:
`presign_s3("etl/painel_iaempe.parquet", expires = 3600)` gera um link
temporário.

## Linha de comando

```bash
cargo install s3pe
export S3PE_ENDPOINT=https://s3.pe.gov.br
s3pe buckets
s3pe ls -l s3://iaempe/dados/
s3pe cat s3://iaempe/dados/metadata.json | jq .ultimo_mes
s3pe get s3://horizonte-fiscal/series/estado/icms.csv.gz
s3pe put painel.parquet s3://iaempe/etl/
```

## Problemas conhecidos

- **403 `SignatureDoesNotMatch`**: credencial errada no ambiente (a chave
  começa com `GK`), ou região diferente de `us-east-1`.
- **403 `AccessDenied`**: a chave não tem permissão para este bucket ou
  para escrita.
- **Timeout**: a máquina não alcança `s3.pe.gov.br:443` (redes internas do
  Estado podem exigir liberação de firewall).
- **`ConfigError: no endpoint`**: falta `S3PE_ENDPOINT` (o s3pe não assume
  nenhum servidor).

Feito em Pernambuco 🌵 · MIT
