# MVP — Padrões de Incidentes de Segurança de Rede

**Disciplina:** Engenharia de Produção / UnB — Prof. André Luiz Marques Serrano
**Produto:** Pipeline de dados em nuvem (Databricks), documentado neste repositório
**Autor:** Isadora Messenberg Guimarães Macêdo Rebello## 1. Objetivo

**Problema de negócio.** Times de segurança da informação recebem, todos os
dias, um grande volume de eventos vindos de firewalls, IDS/IPS e servidores.
Sem uma visão consolidada, é difícil decidir onde concentrar esforço de
monitoramento e resposta a incidentes. Este MVP constrói um pipeline de
dados em nuvem que organiza esses eventos em um modelo dimensional (Esquema
Estrela), permitindo responder perguntas que apoiam essa priorização.

**Perguntas de negócio:**
1. Quais tipos de ataque são mais frequentes e como evoluíram mês a mês entre 2020 e 2023?
2. Existe associação entre protocolo de rede e tipo de ataque predominante?
3. A severidade está associada ao segmento de rede ou ao tipo de ataque?
4. A ação tomada (bloqueado/registrado/ignorado) varia conforme o tipo de ataque?
5. O Anomaly Score é maior para incidentes de severidade alta?
6. Existe concentração geográfica de algum tipo de ataque?

## 2. Base de dados

| Campo | Valor |
|---|---|
| Fonte | [Cyber Security Attacks](https://www.kaggle.com/datasets/teamincribo/cyber-security-attacks) — Incribo, via Kaggle |
| Natureza | **Sintética** (declarado pelo próprio fornecedor) — 40.000 linhas, 25 colunas |
| Licença | MIT (confirmar na página do Kaggle antes da entrega) |
| Dados pessoais | `User Information` (nomes) — anonimizado via SHA-256 antes da carga (LGPD) |

Detalhes completos, domínios e linhagem de cada atributo: [`/catalogo/catalogo_de_dados.md`](catalogo/catalogo_de_dados.md).

## 3. Arquitetura e plataforma

Databricks Community Edition, PySpark + Spark SQL, tabelas **Delta Lake**
em duas camadas (*bronze* → dado bruto; *gold* → modelo dimensional).
Diagrama do Esquema Estrela (8 dimensões + 1 fato): [`/catalogo/modelo_estrela.md`](catalogo/modelo_estrela.md).

## 4. Estrutura do repositório

```
├── README.md                          # este documento
├── LICENSE                            # licença do código (MIT) + nota sobre a licença dos dados
├── notebooks/
│   ├── 01_busca_e_coleta.py           # objetivo, perguntas, busca, coleta, carga da camada bronze
│   ├── 02_modelagem_e_carga.py        # ETL: transformação, esquema estrela, carga na camada gold
│   ├── 03_qualidade_de_dados.py       # completude, unicidade, consistência, conformidade, acurácia, atualidade
│   └── 04_analise_de_negocio.py       # resposta às 6 perguntas de negócio (Spark SQL)
├── scripts/
│   └── etl_utils.py                   # funções PySpark reutilizáveis (parsing, anonimização, qualidade)
├── catalogo/
│   ├── catalogo_de_dados.md           # domínios e linhagem de cada atributo (fato + 8 dimensões)
│   └── modelo_estrela.md              # diagrama do modelo dimensional (Mermaid)
└── evidencias/
    ├── LEIA-ME.md                     # o que capturar e onde colocar (screenshots/vídeos da execução na nuvem)
    └── graficos_exploratorios/        # 5 gráficos gerados localmente a partir da base real (apoio à análise)
```

## 5. Como reproduzir

1. Crie uma conta gratuita no [Databricks Community Edition](https://community.cloud.databricks.com/) e suba um cluster single-node.
2. Baixe `cybersecurity_attacks.csv` do Kaggle e faça upload para o DBFS (`Catalog > Add Data > Upload File`), ajustando `CAMINHO_ORIGEM` no Notebook 1 se o caminho gerado for diferente.
3. Importe os 4 arquivos de `/notebooks` como notebooks Databricks (`Workspace > Import`, formato "Databricks source" — o cabeçalho `# Databricks notebook source` já está em cada arquivo).
4. Execute-os em ordem (01 → 04). Cada um indica, em comentário, o screenshot esperado.
5. Salve as evidências em `/evidencias` conforme [`LEIA-ME.md`](evidencias/LEIA-ME.md).

## 6. Modelagem de dados (resumo)

Esquema Estrela com grão "um evento de rede registrado": `fact_evento_seguranca`
+ 8 dimensões (`dim_tempo`, `dim_protocolo_trafego`, `dim_ataque`,
`dim_resposta`, `dim_geografia`, `dim_dispositivo`, `dim_segmento_rede`,
`dim_log_source`). Detalhes em [`/catalogo`](catalogo/).

## 7. Principais resultados

### 7.1 Qualidade dos dados (Notebook 3)
- **Completude:** 20/25 atributos com 0% de nulos; 5 atributos (`Malware Indicators`, `Alerts/Warnings`, `Proxy Information`, `Firewall Logs`, `IDS/IPS Alerts`) com ~50% de nulos cada — confirmado como indicador binário implícito (nulo = ausente), não erro de coleta.
- **Unicidade:** 0 duplicatas em 40.000 linhas.
- **Conformidade/acurácia:** 0 valores fora do domínio declarado em portas (0–65535), `Anomaly Score` (0–100) e todos os atributos categóricos.
- **Atualidade:** eventos de 01/01/2020 a 11/10/2023 (46 meses); 2023 incompleto (8.139 eventos vs. ~10.500–10.750/ano nos anos anteriores).

![Completude por atributo](evidencias/graficos_exploratorios/05_completude_atributos.png)

### 7.2 Análise de negócio (Notebook 4)
Em todas as seis perguntas, os testes de associação (qui-quadrado e ANOVA)
resultaram em **p > 0,25**, sem significância estatística — Protocolo × Ataque
(p = 0,988), Severidade × Segmento (p = 0,257), Severidade × Ataque (p = 0,773),
Ação × Ataque (p = 0,480), Anomaly Score × Severidade (p = 0,317).

![Evolução mensal por tipo de ataque](evidencias/graficos_exploratorios/01_evolucao_mensal_tipo_ataque.png)
![Anomaly Score por severidade](evidencias/graficos_exploratorios/03_anomaly_score_x_severidade.png)

**Isso não é uma falha da análise.** O fornecedor da base (Incribo) descreve
o conjunto como sintético — um "playground" para prática de pipeline, não
uma amostra de incidentes reais. Um gerador sintético sem correlação
propositada entre colunas produz exatamente esse padrão de uniformidade. O
valor deste MVP está na construção do pipeline e do modelo dimensional,
reaplicáveis sem alteração estrutural a uma base real de eventos de
segurança, onde as mesmas perguntas tenderiam a produzir sinal estatístico
genuíno.

## 8. Autoavaliação

> ✏️ **Personalize esta seção** com sua experiência real ao rodar o pipeline
> no Databricks (erros encontrados, tempo de execução, ajustes que precisou
> fazer). O rascunho abaixo parte das descobertas já confirmadas na análise.

1. **Quais perguntas foram respondidas e quais não?** As seis perguntas de
   negócio formuladas no objetivo foram respondidas tecnicamente — mas cinco
   delas com resultado negativo (sem associação estatística relevante). A
   pergunta 1 (frequência e evolução temporal) foi a única com resposta
   descritiva positiva (distribuição praticamente igualitária entre os três
   tipos de ataque, estável ao longo do tempo).
2. **Que limitações dos dados condicionaram os resultados?** A principal
   limitação é a natureza sintética da base: por ser gerada artificialmente
   sem correlações intencionais entre colunas, ela não permite validar
   hipóteses de negócio reais (ex.: "ataques de severidade alta são mais
   bloqueados") — apenas testar se o pipeline e a modelagem funcionam
   corretamente.
3. **Que decisões técnicas você tomaria de outra forma?** *(preencher com
   sua experiência — ex.: se usaria uma chave substituta diferente de
   `dense_rank()`/`monotonically_increasing_id()` em produção, se manteria
   `Payload Data` de outra forma, etc.)*
4. **Que extensões seriam necessárias para uma solução de uso contínuo?**
   Substituir a carga única por um pipeline incremental (CDC ou append
   diário), conectar a uma fonte real (SIEM/firewall da organização),
   adicionar um job agendado (Databricks Jobs) para atualização automática
   da camada gold, e criar um dashboard (Databricks SQL) sobre as 6 perguntas
   de negócio para consumo contínuo pelo time de segurança.

## 9. Checklist de entrega

- [ ] Rodei os 4 notebooks no meu cluster Databricks sem erros
- [ ] Capturei as evidências indicadas em cada notebook (`/evidencias`)
- [ ] Personalizei a Autoavaliação (Seção 8, itens 3 e 4) com minha experiência real
- [ ] Confirmei a licença do dataset na página oficial do Kaggle
- [ ] Publiquei o repositório como **público** no GitHub e testei o link em janela anônima
- [ ] Postei o link no fórum de entrega da disciplina

## 10. Licença

Código deste repositório sob licença MIT ([`LICENSE`](LICENSE)). Dados
originais sob licença MIT do fornecedor (Incribo/Kaggle) — ver Seção 2.
