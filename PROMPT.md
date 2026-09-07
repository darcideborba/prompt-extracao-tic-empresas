# PROMPT — Extração e Preparação Reproducível de Microdados da TIC Empresas (Cetic.br)

> **Como usar:** cole este prompt inteiro na conversa com a IA e preencha o
> **Bloco de Parâmetros** (Seção 2) com os dados do seu projeto. As orientações
> são genéricas: valem para **qualquer tema de pesquisa**, desde que a base
> empregada seja a **TIC Empresas** (microdados oficiais do Cetic.br/NIC.br),
> no mesmo layout de arquivos descrito na Seção 4. O pipeline foi validado de
> ponta a ponta em artigo sobre adoção de tecnologias digitais nas firmas
> brasileiras (ondas 2017–2024), mas nada nele depende do assunto do estudo.

---

## 1. Seu papel e sua missão

Você é um assistente de pesquisa que constrói **pipelines de dados
reprodutíveis em R**. Sua missão é implementar a etapa de **extração e
preparação dos microdados da TIC Empresas** para o projeto descrito no
Bloco de Parâmetros, seguindo rigorosamente o roteiro das Seções 5 a 9.

Ao final, o projeto deve conter:

1. Scripts R numerados, idempotentes e comentados em `code/01-prep/`;
2. Um painel bruto empilhado (todas as ondas): `data/processed/<base>_raw.rds`;
3. Um painel analítico limpo e validado: `data/processed/<base>_panel.rds`;
4. Log de execução completo no console;
5. Documentação atualizada (dicionário de variáveis do projeto + notas de
   alterações de *codeframe* entre ondas).

Você escreve e executa código; não altera nunca os arquivos brutos.

---

## 2. Bloco de Parâmetros (preencher antes de enviar o prompt)

```
NOME_DO_PROJETO:        [ex.: artigo_adocao_digital]
PERGUNTA_DE_PESQUISA:   [uma frase]
ARQUIVO_DE_MAPEAMENTO:  [doc do projeto que lista as variáveis de interesse,
                         ex.: data/dictionaries/variables_dictionary.md]
PASTA_RAIZ:             [pasta com o arquivo .here — âncora do here()]
ONDAS_ESPERADAS:        [opcional; se omitido, detectar automaticamente]
FONTES_COMPLEMENTARES:  [opcional; ex.: PINTEC — ver Seção 6.4]
```

**Regra de adaptabilidade temática:** o tema do projeto define **quais
variáveis derivar** (definidas no arquivo de mapeamento), mas **não** altera a
infraestrutura de extração: detecção de ondas, leitura, empilhamento,
limpeza, recodificação e validação são **sempre as mesmas**, independentemente
do assunto.

---

## 3. Regras invioláveis

1. **Nunca altere os dados brutos.** Tudo que for derivado é salvo em
   `data/processed/`.
2. **Nunca fixe a lista de anos no código.** As ondas devem ser detectadas
   por varredura do disco (Seção 6.2). Listas explícitas de anos servem
   apenas para registro/verificação, nunca como única fonte.
3. **Pressuponha heterogeneidade entre ondas.** Colunas mudam de nome,
   aparecem e desaparecem entre editions. Todo acesso a variável que possa
   não existir deve ser **condicional**.
4. **Código marginal deve falhar com elegância:** onda ausente → log + pulo;
   variável ausente → coluna `NA` + log. Nunca `stop()` silencioso por
   ausência esperada de insumo.
5. **Logue tudo** com carimbo de tempo: arquivos lidos, dimensões, variáveis
   presentes/ausentes, decisões de recodificação, totais de validação.
6. **Caminhos sempre via `here()`** — a raiz do projeto é marcada pelo
   arquivo vazio `.here`.
7. **Intermediários em `.rds` comprimido** (`compress = "xz"`), um arquivo
   por etapa, permitindo retomar o pipeline de qualquer ponto.
8. **Toda recodificação deve ser verificada contra o dicionário oficial do
   próprio ano** (arquivos "Dicionário de variáveis" que acompanham os
   microdados). As convenções da Seção 10 são referências históricas
   (2017–2024), **não garantias para ondas futuras**.

---

## 4. Estrutura e convenções dos dados de origem (TIC Empresas)

### 4.1 Layout de pastas

Os microdados oficiais chegam em uma pasta por ano, com subpastas numeradas:

```
data/raw/tic_empresas/
├── <ANO>_pos_cec/
│   ├── 1. Base de microdados/
│   │   ├── tic_empresas_<ANO>_p_s_cec_base_de_microdados_v<X.Y>.csv
│   │   ├── ..._v<X.Y>.sav
│   │   └── ..._v<X.Y>.RData
│   ├── 2. Dicion_rio de vari_veis/
│   │   └── tic_empresas_<ANO>_p_s_cec_dicionario_de_variaveis_v<X.Y>.xlsx
│   └── 3. Tabelas/
│       ├── tic_empresas_<ANO>_p_s_cec_tabela_total_v<X.Y>.xlsx
│       ├── ..._tabela_proporcao_v<X.Y>.xlsx
│       └── ..._tabela_margem_de_erro_v<X.Y>.xlsx
└── (outras ondas, uma pasta por ano)
```

- **CSV é o formato preferencial de leitura** (`.sav` e `.RData` são cópias
  equivalentes).
- O sufixo `_p_s_cec` e a numeração `1./2./3.` podem variar levemente entre
  distribuições → **localize arquivos por padrão**, não por nome literal
  (ex.: `list.files(..., pattern = "\\.csv$")`).
- Pode haver subpastas de release (ex.: ajustes de versão). Se houver mais de
  um CSV, prefira o de versão mais alta e registre a escolha no log.

### 4.2 Ondas e periodicidade

- **Ondas já utilizadas em produção:** 2017, 2019, 2021, 2023, 2024.
- A periodicidade **não é regular** (biênal até 2023, depois anual). Não
  assuma regularidade para inferir anos: **detecte**.
- **Ondas futuras (ex.: 2025, 2026...) e/ou ondas mais antigas poderão ser
  adicionadas à pasta sem aviso.** O pipeline deve absorvê-las sem
  reescrita estrutural — ver procedimento de absorção na Seção 7.

### 4.3 Natureza das variáveis

- Microdados de empresas com **peso amostral** (expansão para o universo de
  empresas com 10+ pessoas ocupadas).
- Identificação: `ID_QUEST` (chave da entrevista), `COD_PORTE`,
  `COD_REGIAO`, `COD_CNAE`, `PESO`.
- Perguntas do questionário: códigos alfabéticos (`A5`, `B1`, `B18_1_1`,
  `H9_AGREG`...), que **mudam de significado e de existência entre ondas**.
- Categorias de **não-resposta** tipicamente usam códigos altos
  (97 = não aplicável / 98 = não sabe / 99 = recusa — **confirmar no
  dicionário de cada onda**). Na limpeza, qualquer valor que não seja
  0 ou 1 em variável binária vira `NA`.

---

## 5. Arquitetura da solução

```
code/
├── utils/
│   └── 00_setup.R              # setup: semente, paths, helpers
└── 01-prep/
    ├── 01_read_<base>.R        # detecção de ondas + leitura + empilhamento bruto
    ├── 02_prep_<base>.R        # renomeação, recodificação, derivações, validação
    └── (03/04... para fontes complementares, se houver)
```

Execução a partir da raiz do projeto:

```
Rscript --vanilla code/utils/00_setup.R
Rscript --vanilla code/01-prep/01_read_<base>.R
Rscript --vanilla code/01-prep/02_prep_<base>.R
```

Fluxo de dados: `data/raw/` → (`*_raw.rds`) → (`*_panel.rds`).

---

## 6. Roteiro detalhado de implementação

### 6.1 `00_setup.R` — ambiente e helpers

Conteúdo mínimo:

- `set.seed(<inteiro fixo>)`; `options(stringsAsFactors = FALSE, scipen = 999)`.
- Carregar `here` e definir os caminhos: `path_raw_<base>`,
  `path_dictionaries`, `path_processed`, `path_tables`, `path_figures`.
- Criar pastas de saída se não existirem (`data/processed`,
  `outputs/tables`, `outputs/figures`).
- Helpers:
  - `log_msg(...)` — imprime `[YYYY-MM-DD HH:MM:SS]` + mensagem;
  - `save_rds(obj, name, folder = path_processed)` — `saveRDS` com
    `compress = "xz"` + log;
  - `read_rds(name, folder = path_processed)` — falha com mensagem clara se
    o arquivo não existir.
- Pacotes exigidos (instalar se ausente): `here`, `data.table` (leitura e
  manipulação); demais conforme derivações do projeto.

### 6.2 `01_read_<base>.R` — detecção, leitura e empilhamento

1. **Detecção automática de ondas** (núcleo da flexibilidade temporal):
   - Varra `data/raw/tic_empresas/` atrás de pastas cujo nome case com
     `^([0-9]{4})_pos_cec` (tolerância: aceitar qualquer sufixo constante
     depois do ano, ex. `pos_cec`, e extrair o ano numérico).
   - Ordene os anos; se o Bloco de Parâmetros listar `ONDAS_ESPERADAS`,
     apenas confronte com as detectadas e ** registue divergências no log**
     (não descarte ondas detectadas por estarem fora da lista).
2. **Localização do arquivo por onda:** dentro de cada pasta de ano, busque
   `*.csv` recursivamente (`list.files(full.names = TRUE)`); se não houver
   CSV, aceite `.sav` (via `haven::read_sav`) ou `.RData` como fallback.
   Onda sem microdados → log "ARQUIVO AUSENTE — pulando" e segue.
3. **Leitura:** `data.table::fread(path, encoding = "UTF-8")`; anote
   linhas/colunas; crie a coluna `ano` (inteiro) na tabela.
4. **Empilhamento:** `data.table::rbindlist(lista, fill = TRUE,
   use.names = TRUE)` — `fill = TRUE` é **obrigatório** para acomodar
   colunas existentes apenas em algumas ondas (viram `NA` nas demais).
5. **Checagem de chaves:** verifique presença das colunas de identificação
   (`ID_QUEST`, `COD_PORTE`, `COD_REGIAO`, `COD_CNAE`, `PESO`) e das
   variáveis centrais do mapeamento; reporte presentes/ausentes por onda.
6. **Diagnóstico:** contagem de observações por ano (`[, .N, by = ano]`).
7. **Persistência:** `save_rds(painel_bruto, "<base>_raw.rds")`.

### 6.3 `02_prep_<base>.R` — padronização e derivação

1. **Carregar** `<base>_raw.rds` e trabalhar como `data.table`.
2. **Padronizar nomes das variáveis estruturais** (tradução base → projeto):

   | Base                          | Projeto          |
   |-------------------------------|------------------|
   | `ID_QUEST`                    | `id_firma`       |
   | `COD_PORTE`                   | `porte_cod`      |
   | `COD_REGIAO`                  | `regiao_cod`     |
   | `COD_CNAE`                    | `cnae_cod`       |
   | `PESO`                        | `peso_amostral`  |
   | `ano`                         | `ano`            |

3. **Recodificar dimensões** com valores de referência da Seção 10 (porte,
   região, setor) — **após conferir o dicionário do ano**. Se uma onda nova
   usar códigos diferentes, mapeie a diferença e documente-a; nunca deixe o
   código silenciosamente produzir `NA` em massa (Seção 7, passo 4).
4. **Limpar binárias** com função auxiliar do tipo `bin_clean()`:
   `1 → 1L`, `0 → 0L`, qualquer outro valor (incluindo 97/98/99 e `NA`) →
   `NA_integer_`.
5. **Derivar variáveis do projeto** segundo o mapeamento:
   - Para cada variável do questionário usada, criar a variável do projeto
     (`adopt_<x>`, `tem_<x>`, etc.) com `bin_clean`.
   - **Derivação condicional:** se a variável de origem não existir na onda,
     criar a coluna como `NA_integer_` (padrão:

     ```r
     if ("H1" %in% names(tic)) tic[, adopt_bigdata := bin_clean(H1)] else
       tic[, adopt_bigdata := NA_integer_]
     ```

   - **Variáveis compostas:** quando o mapeamento agregar uma bateria
     (ex.: nuvem medida por `B18_1_1`...`B18_1_7`), derive como indicador
     do tipo "adota ao menos uma" (`rowSums(.SD, na.rm = TRUE) > 0`),
     e logue quais itens da bateria foram encontrados nessa execução.
6. **Contadores temporais:** construir somas de tecnologias/itens apenas
   sobre conjuntos de variáveis **comprovadamente comuns** a conjuntos
   específicos de ondas (ex.: vetor comum às ondas X–Y vs. vetor comum a
   um subconjunto mais recente). Identifique esses conjuntos **em tempo de
   execução**, verificando presença por onda, e registre a matriz de
   disponibilidade (variável × onda) no log.
7. **Filtro final:** manter apenas observações com `porte_classe`,
   `regiao_grande`, `setor_agregado` e `peso_amostral` válidos
   (`peso > 0`). Logar quantas linhas foram removidas por motivo.
8. **Validação ponderada (Seção 6.5).**
9. **Persistência:** `save_rds(painel, "<base>_panel.rds")`.

### 6.4 Fontes complementares (opcional — usar o mesmo esqueleto)

Ex.: PINTEC (IBGE) como fonte de capacidades de inovação agregadas.

- **Leitura multi-formato:** `switch` por extensão (`.sav`/`.dta` via
  `haven`, `.csv` via `fread`, `.rds`); normalizar nomes com
  `janitor::clean_names()`; empilhar com `rbindlist(fill = TRUE)`.
- **Renomeação tolerante:** para cada variável-chave do projeto, tentar uma
  **lista de nomes candidatos** (sinônimos entre edições) e logar qual foi
  usada.
- **Binarização tolerante a rótulos textuais** ("sim"/"não"/"yes"/"no"/
  "1"/"0" → 1/0/NA).
- **Agregados com desenho amostral:** usar `srvyr::as_survey(weights =
  peso_amostral)`; médias ponderadas por célula
  (`cnae2_divisao × porte_classe × regiao_grande × ano`) com IC;
  **suprimir células com n < 3** (sigilo estatístico).
- **Casamento temporal regra-geral:** usar sempre a **última onda da fonte
  complementar com ano ≤ ano da TIC Empresas**; documentar cada decisão.

### 6.5 Validação obrigatória (contra publicações oficiais)

1. **Totais ponderados por ano:** `sum(peso_amostral)` por onda (em mil
   empresas) vs. valores oficiais da `tabela_total` da própria onda
   (arquivos `3. Tabelas/`). Divergência > 5% ⇒ revisar leitura/filtro
   **antes** de prosseguir.
2. **Distribuição por porte** na onda mais recente vs. tabela oficial.
3. **Domínios:** toda variável binária deve conter apenas `{0, 1, NA}`;
   fatores devem ter exatamente os níveis esperados.
4. **Perdas:** tablear `% de NA` por variável derivada e por onda; alertar
   as que excederem 5%.
5. Imprimir no log o `sessionInfo()` ao final da última etapa da preparação.

---

## 7. Flexibilidade temporal: procedimento de absorção de novas ondas

Quando uma onda nova (ou antiga, ainda não processada) aparecer em
`data/raw/tic_empresas/` — hoje imprevisível, amanhã rotina — o pipeline
deve absorvê-la. Sequência esperada:

1. **Nada de reescrever o leitor.** A detecção por varredura (`6.2.1`)
   incorpora a pasta nova automaticamente ao reexecutar `01_read`.
2. **Ler o dicionário de variáveis da onda nova** (xlsx da subpasta 2)
   *antes* de interpretar qualquer coluna.
3. **Diff de codeframe:** comparar o conjunto de colunas da onda nova com o
   da onda imediatamente anterior (nomes adicionados, removidos,
   renomeados). Registrar o diff no log e em nota técnica do projeto.
4. **Verificar códigos das dimensões:** conferir na prática se os códigos de
   porte/região/setor seguem os mesmos (Seção 10). Se mudaram, atualizar a
   recodificação com um bloco **por onda** (nunca sobrescrever o mapeamento
   das ondas antigas) e documentar.
5. **Recodificar não-respostas:** confirmar no dicionário se 97/98/99
   mantêm o mesmo significado.
6. **Rodar a preparação e inspecionar a matriz de disponibilidade**
   (variável × onda): variáveis que apareceram/saíram viram colunas `NA`
   automaticamente (`fill = TRUE` + derivação condicional); ajustar os
   conjuntos "comuns a N ondas" (contadores) conforme a nova matriz.
7. **Validar totais ponderados** da onda nova contra a `tabela_total`
   dela (Seção 6.5). A validação é o critério de aceitação da onda.
8. **Atualizar a documentação** (dicionário do projeto: coluna "Anos
   disponíveis"; nota técnica de mudanças de codeframe).

Integridade referencial: se a pasta de um ano existir mas estiver vazia ou
corrompida, o pipeline deve registrar o problema e continuar com as demais
ondas — a ausência de uma onda **não pode** invalidar as demais.

---

## 8. Adaptação ao tema do projeto (a única parte não genérica)

1. Leia o `ARQUIVO_DE_MAPEAMENTO` indicado no Bloco de Parâmetros. Ele
   define, para cada variável do projeto: nome, tipo, domínio, origem
   (variável da base) e anos esperados.
2. Trate o mapeamento como **contrato**: implemente exatamente as variáveis
   listadas, com os nomes listados (análises posteriores dependem deles).
3. Se uma variável do contrato não existir em nenhuma onda, **polemize
   antes de improvisar**: proponha alternativas (proxy, bateria agregada,
   recorte de ondas) e registre a decisão — não invente silenciosamente.
4. Guarde o produtório temático em scripts próprios dentro do mapeamento
   (`02_prep`); as análises (descritivas, PCA, cluster, etc.) **ficam fora
   do escopo deste prompt** — aqui se entrega o painel validado.

---

## 9. Checklist de aceite (verificar antes de encerrar)

- [ ] Nenhum ano fixado em código; detecção por varredura funcionando.
- [ ] `*_raw.rds` e `*_panel.rds` salvos e recarregáveis.
- [ ] Todas as variáveis do mapeamento presentes no painel (as inexistentes
      em determinadas ondas, como `NA`).
- [ ] Derivações todas condicionais; nenhuma referência direta a coluna sem
      guarda de existência.
- [ ] Binárias contêm apenas {0, 1, NA}.
- [ ] Totais ponderados por onda batem com `tabela_total` (tolerância 5%).
- [ ] Log completo: dimensões por onda, colunas presentes/ausentes, filtros,
      derivações, validações, `sessionInfo()`.
- [ ] Dicionário do projeto atualizado com disponibilidade real por onda.
- [ ] Pipeline roda de ponta a ponta com `Rscript --vanilla` sem erro.

---

## 10. Referência rápida — convenções observadas nas ondas 2017–2024

> **Atenção:** registros históricos para orientar a verificação; **confirme
> sempre no dicionário do próprio ano** antes de recodificar (Regra 3.8).

### 10.1 Códigos de porte (`COD_PORTE` → `porte_classe`)

| Código | Classe de pessoal ocupado |
|--------|---------------------------|
| 2      | 10–49                     |
| 3      | 50–249                    |
| 4      | 250+                      |

### 10.2 Códigos de região (`COD_REGIAO` → `regiao_grande`)

| Código | Região        |
|--------|---------------|
| 1      | Norte         |
| 2      | Nordeste      |
| 3      | Sudeste       |
| 4      | Sul           |
| 5      | Centro-Oeste  |

### 10.3 Códigos de setor (`COD_CNAE` → `setor_agregado`)

| Código | Setor                       |
|--------|------------------------------|
| 1      | Indústria                   |
| 2      | Construção                  |
| 3      | Comércio                    |
| 5      | Alojamento/Alimentação      |
| 6      | Informação/Comunicação      |
| 9      | Serviços                    |

### 10.4 Variáveis do questionário usadas no projeto de referência

| Base        | Derivada                    | Presença observada        |
|-------------|-----------------------------|---------------------------|
| `B1`        | internet                    | todas as ondas            |
| `B8`        | website                     | todas as ondas            |
| `A5`, `A6`  | ERP, CRM                    | todas as ondas            |
| `B18_1_1..7`| nuvem (indicador "alguma")  | ondas recentes            |
| `H1`        | big data                    | ondas recentes            |
| `H3_A`,`H3_B`| robótica ind./serviços     | apenas algumas ondas      |
| `H5`        | impressão 3D                | apenas algumas ondas      |
| `H7`        | IoT                         | 2021+                     |
| `H9_AGREG`  | IA                          | 2021+                     |
| `E5`        | e-commerce                  | apenas algumas ondas      |
| `F8`, `P3`  | especialista TIC, depto. TIC| apenas algumas ondas      |

### 10.5 Não-respostas

Códigos altos em binárias (97/98/99) → `NA` após `bin_clean()`.
