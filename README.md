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

## Próximas fases (planejado)

- **Fase 2 — Drill-down (Opção A).** Manter a régua como nível 1 e, ao clicar
  numa Área, transicionar para um gráfico de barras de detalhe no mesmo visual
  (signal `currentArea` + grupos de marks alternados + botão voltar), na linha
  da mecânica de hierarchical bar chart. Exige os dois níveis no mesmo dataset.
- **Fase 3 — Medidas de detalhe.** Adicionar `Nivel`, `AreaPai`, `Item` e as
  métricas por item para alimentar o nível 2.
