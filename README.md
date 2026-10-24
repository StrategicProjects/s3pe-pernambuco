# s3pe no s3.pe.gov.br

Exemplos, em R, Python, Rust, Go e linha de comando, de como ler e gravar
dados no **s3.pe.gov.br**, o armazenamento de objetos S3 (um servidor
[Garage](https://garagehq.deuxfleurs.fr)) da plataforma Condepe/Fidem do
Estado de Pernambuco, usando o pacote genérico [s3pe](https://github.com/StrategicProjects/s3pe).
Os nomes de bucket e as chaves são de cada projeto e não aparecem aqui:
peça os seus a quem administra a plataforma.

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

## Buckets

Cada projeto tem o seu bucket e a sua chave (de leitura, ou de leitura e
escrita), criados por quem administra a plataforma: peça o nome do bucket e
a chave do seu projeto. Os exemplos abaixo usam `meu-bucket`.

**Regra do manifesto:** nos lotes publicados pela plataforma, o
`manifest.json` (ou `metadata.json`) é sempre o último objeto a subir. Se a
data dele é T, tudo o que ele lista já estava no bucket em T. Leia-o
primeiro, use a data como versão do lote e a lista dele em vez de listar o
bucket.

## R

```r
library(s3pe)
use_endpoint("https://s3.pe.gov.br")    # ou S3PE_ENDPOINT no ~/.Renviron
use_bucket("meu-bucket")

meta       <- read_s3("dados/manifesto.json")       # lista
series     <- read_s3("dados/series.json")
municipios <- read_s3("dados/municipios.geojson")  # sf
painel     <- read_s3("etl/painel.parquet")          # tibble (nanoparquet)
list_s3("dados/")

# chaves completas quando há mais de um bucket
manifest <- read_s3("s3://outro-bucket/manifest.json")
icms     <- read_s3("s3://outro-bucket/series/icms.csv.gz")
prev     <- read_s3("s3://outro-bucket/previsoes/icms/ultima.json")

# escrita (chave com permissão de escrita; use um prefixo seu)
write_s3(painel, "nota_tecnica/painel.parquet")
upload_s3("figura1.png", "nota_tecnica/")
```

## Python

```python
import s3pe   # pip install "s3pe[pandas]"

s3pe.configure(endpoint="https://s3.pe.gov.br", bucket="meu-bucket")
meta   = s3pe.read_s3("dados/manifesto.json")          # dict
painel = s3pe.read_s3("etl/painel.parquet")           # DataFrame
icms   = s3pe.read_s3("s3://outro-bucket/series/icms.csv.gz")
s3pe.write_s3(painel, "nota_tecnica/painel.csv")
s3pe.download_s3("dados/municipios.geojson", "municipios.geojson")
```

## Rust

```rust
let s3 = s3pe::Client::builder()
    .endpoint("https://s3.pe.gov.br")   // ou S3PE_ENDPOINT
    .bucket("meu-bucket")
    .build()?;                          // credenciais do ambiente
let meta = s3.read("dados/manifesto.json")?;
s3.upload("painel.parquet", "etl/painel.parquet", None)?;
```

## Go

```go
s3, err := s3pe.New(s3pe.Config{Endpoint: "https://s3.pe.gov.br", Bucket: "meu-bucket"})
meta, err := s3.Read(ctx, "dados/manifesto.json")
objects, err := s3.List(ctx, "dados/")
```

## Publicar um lote: `sync` com manifesto por último

O que os scripts de publicação fazem na mão (subir o que mudou, manifesto no
fim), o `sync` faz por ETag, nas quatro linguagens e na CLI:

```r
sync_up_s3("output/painel", "s3://meu-bucket/etl/", last = "manifest.json")
```

```bash
s3pe sync output/painel s3://meu-bucket/etl/ --last manifest.json
s3pe sync s3://meu-bucket/dados/ ./dados-local          # espelho local do lote
```

Arquivos grandes sobem em partes automaticamente (acima de 64 MiB). Para
compartilhar um objeto com quem não tem credencial:
`presign_s3("etl/painel.parquet", expires = 3600)` gera um link
temporário.

## Linha de comando

```bash
cargo install s3pe
export S3PE_ENDPOINT=https://s3.pe.gov.br
s3pe buckets
s3pe ls -l s3://meu-bucket/dados/
s3pe cat s3://meu-bucket/dados/manifesto.json | jq .gerado_em
s3pe get s3://outro-bucket/series/icms.csv.gz
s3pe put painel.parquet s3://meu-bucket/etl/
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
