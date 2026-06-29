# Régua KPI — Deneb / Vega

Visual de "régua" (escala 1–5 por Área) para Power BI via Deneb.
Cada linha representa uma Área; a faixa colorida marca a zona em que a Área
caiu (Nota), com o Desvio em % e um ponteiro (triângulo) na zona vencedora.

## Arquivo

- `specs/regua-kpi.vg.json` — spec Vega 5 da régua (Fase 1, refatorada).

## Campos esperados no dataset (Deneb roles)

| Campo        | Tipo    | Descrição                                                        |
|--------------|---------|------------------------------------------------------------------|
| `Area`       | texto   | Nome da área (rótulo da linha)                                   |
| `OrdemFinal` | inteiro | Ordem da linha (0, 1, 2…); define a posição vertical            |
| `Desvio`     | número  | Desvio em % exibido na linha                                     |
| `Nota`       | 1–5     | Zona vencedora; posiciona ponteiro/valor e destaca o segmento    |
| `IsTotal`    | 0/1     | 1 = linha de total (fonte maior, sem régua de separação)        |

`Nota`/`Desvio` ausentes são tratados com `isValid` (mostra `--`, sem ponteiro).

## Mudanças da Fase 1 (refatoração)

- **Zonas em 1 dataset + 1 mark.** As 5 cópias `zonas1..zonas5` + 5 `rect` + 5
  `text` foram colapsadas: dataset `zonas` gera as 5 zonas por Área via
  `flatten` de `[1..5]`, desenhadas por **um** `rect`. ~13 marks → ~5.
- **`colorScale` única.** Fills usam `scale('colorScale', notaZona)` em vez de
  cor hardcoded (`corZona` removido).
- **Zona vencedora destacada.** Zonas não-selecionadas ficam esmaecidas
  (opacity 0.28); a zona da `Nota` fica em destaque (opacity 0.95 + borda branca).
- **Cabeçalho de escala único** (`Crítico · Ruim · Atenção · Bom · Ótimo`) no
  topo, no lugar dos números 1–5 repetidos sob cada linha.
- **Valor do Desvio alinhado à zona vencedora**, em coluna com o ponteiro.
- **Tooltip** por linha (Área, Desvio, Nota).
- **Formatação pt-BR** do Desvio (vírgula decimal via `replace`).
- **Ponteiro com tamanho limitado** (`min(segW*14, 350)`) para não inflar em
  containers largos.

## Fase 2 — Drill para projetos ofensores

`specs/regua-kpi-drill.prototype.vg.json` — protótipo com dados fictícios
embutidos. Clique numa linha de área → a régua dá lugar a barras horizontais
dos projetos daquela área, **ordenados por maior impacto em R$** (o que move o
desvio). Botão "← Voltar" reseta. Testável em vega.github.io/editor.

Mecânica (Opção A): signal `currentArea` (inicia `null`); clique na linha seta a
área, "Voltar" volta a `null`. Os dois níveis convivem no mesmo dataset — o
clique só filtra no cliente, sem nova query.

### Contrato de dados (nomes esperados pelo Deneb)

Tudo numa única tabela "empilhada" alimentando o visual. Campos por nível:

**Nível 1 — régua (uma linha por área):**

| Campo        | Tipo    | Descrição                                          |
|--------------|---------|----------------------------------------------------|
| `Nivel`      | inteiro | **1** para estas linhas                            |
| `Area`       | texto   | Nome da área (rótulo e chave de drill)             |
| `OrdemFinal` | inteiro | Ordem vertical (0, 1, 2…)                          |
| `Desvio`     | número  | Desvio % da área                                   |
| `Nota`       | 1–5     | Zona vencedora                                     |
| `IsTotal`    | 0/1     | 1 = linha de total (não dá drill)                  |

**Nível 2 — projetos (uma linha por projeto):**

| Campo        | Tipo    | Descrição                                          |
|--------------|---------|----------------------------------------------------|
| `Nivel`      | inteiro | **2** para estas linhas                            |
| `Area`       | texto   | Área pai (mesmo valor da área do nível 1)          |
| `Projeto`    | texto   | Nome do projeto (rótulo da barra)                  |
| `Planejado`  | número  | Valor planejado/alocado                            |
| `Realizado`  | número  | Valor realizado                                    |
| `Desvio`     | número  | Desvio % do projeto (rótulo de severidade)         |
| `Nota`       | 1–5     | Cor da barra                                       |

O impacto em R$ (`Realizado − Planejado`) e a ordenação são calculados dentro do
spec — não precisa medida nova para isso. Convenção de sinal usada: gastar mais
que o planejado → `Desvio` negativo (ofensor).

## Fase 3 — pendente

Substituir os `values` embutidos do protótipo pela tabela real (roles do Deneb)
e trocar o signal `width` por `pbiContainerWidth`.
