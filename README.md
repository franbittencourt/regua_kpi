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

## Fase 2 — Régua hierárquica + drill em barras divergentes

`specs/regua-kpi-drill.prototype.vg.json` — protótipo com dados fictícios
embutidos. Testável em vega.github.io/editor (o clique funciona lá).

**Régua (nível 1):** a **empresa** aparece no topo, destacada (faixa azul +
fonte maior), e abaixo dela as áreas. **Toda linha é clicável** — a linha
inteira é um único alvo de clique, com realce no hover.

**Drill (ao clicar):** a régua dá lugar a um **gráfico de barras divergente**
dos "filhos" do nó clicado:
- clicar na **empresa** → as **áreas** viram barras;
- clicar numa **área** → os **projetos** daquela área viram barras.

As barras divergem de um **eixo central**: **estouro (gastou mais) à esquerda**,
**economia (gastou menos) à direita**, ordenadas pelo **desvio absoluto** (maior
magnitude no topo, independente do lado). A cor (Nota 1–5) reforça o sinal.
Botão "← Voltar" retorna à régua.

Mecânica (Opção A): signal `currentNode` (inicia `null`); o clique seta o nó,
"Voltar" volta a `null`. Todos os níveis convivem no mesmo dataset — o clique só
filtra no cliente, sem nova query. O drill segue a relação `Pai`.

### Contrato de dados (nomes esperados pelo Deneb)

Uma única tabela "empilhada" (empresa + áreas + projetos), uma linha por nó:

| Campo       | Tipo    | Descrição                                              |
|-------------|---------|--------------------------------------------------------|
| `Nome`      | texto   | Rótulo do nó (empresa, área ou projeto)                |
| `Pai`       | texto   | `Nome` do pai (vazio/nulo na empresa); chave do drill  |
| `Nivel`     | inteiro | 1 = empresa, 2 = área, 3 = projeto                     |
| `Ordem`     | inteiro | Ordem vertical na régua (níveis 1–2)                   |
| `Desvio`    | número  | Desvio % do nó                                          |
| `Nota`      | 1–5     | Zona vencedora (régua) e cor da barra (drill)          |
| `Planejado` | número  | Valor planejado/alocado                                |
| `Realizado` | número  | Valor realizado                                        |

A régua mostra os nós de `Nivel <= 2`. O drill mostra os filhos
(`Pai === nó clicado`). O **saldo** (`Planejado − Realizado`), o lado do eixo e a
ordenação são calculados dentro do spec — não precisa medida nova. Convenção:
gastar mais que o planejado → `saldo` negativo (ofensor, lado esquerdo).

## Versão Deneb (produção)

`specs/regua-kpi-drill.deneb.vg.json` — pronta para colar no Deneb. Diferenças
em relação ao protótipo:

- lê do `dataset` do Power BI (sem `values` embutidos);
- usa `pbiContainerWidth` (injetado pelo Deneb) no lugar de `width`;
- formatação **pt-BR** e valores em **R$ mil** (a tabela `DCAPEX_Drill` divide
  por 1000), inclusive nos tooltips, com a legenda "Valores em R$ mil" no drill.

Alimenta direto a saída da tabela calculada `DCAPEX_Drill`
(`Nome`, `Pai`, `Nivel`, `Ordem`, `Desvio`, `Nota`, `Planejado`, `Realizado`).
O nó raiz (`Nivel = 1`) é o "Total"/empresa; clicar nele mostra as áreas, clicar
numa área mostra os projetos. A relação de drill é `Pai === Nome` do nó clicado.

O `prototype.vg.json` segue com dados fictícios (mesma formatação) para testes em
vega.github.io/editor.

### Nível 2: rolagem, nomes e nulos

- **Rolagem** quando há muitos projetos: as linhas primeiro se **auto-ajustam**
  (altura entre 30 e 44px conforme a quantidade); estourando a área visível,
  entra **scroll** por roda do mouse e uma **barra de rolagem arrastável** à
  direita. O conteúdo fica recortado (`clip`) na viewport, sem cortar torto.
- **Mais espaço para o nome** do projeto (coluna ~`min(240, 33% da largura)`).
- **Rótulo da barra adaptativo:** mostra o valor em R$ e, quando a barra é longa,
  o rótulo entra **dentro** da barra (texto branco) em vez de transbordar sobre o
  nome do projeto. O % do projeto fica no tooltip.
- **Nulos:** projeto sem `Realizado` (ou sem `Planejado`) vira **barra cinza**
  ("sem realização" no tooltip), sem a cor de severidade enganosa. O saldo trata
  o nulo como zero (sem `NaN`).
- **Botão Voltar:** o texto é `interactive: false`, então o clique passa para o
  botão inteiro (antes, passar sobre o texto não clicava).

### Transição animada

Na troca de nível há uma transição de ~350ms: as **barras crescem a partir do
eixo central** (todas partem do desvio zero e se abrem) com fade-in, e a **régua
dá fade-in** na volta. É feita com `now()` + um `timer` dirigindo o signal `tEase`
(easing ease-out), sem `transition` declarativo (que o Vega não tem). O timer só
re-renderiza enquanto `tEase` muda — em repouso (`tEase = 1`) fica ocioso, sem
custo. Para desligar/ajustar a duração, mexa no signal `animMs` (0 = sem animação).
