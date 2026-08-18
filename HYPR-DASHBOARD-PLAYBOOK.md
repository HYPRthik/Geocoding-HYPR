# HYPR · Dashboard de Consolidado de Campanhas — Arquitetura e Playbook

> **Para quem é este documento:** sessões do Claude (ou devs) que precisem (a) manter o dashboard da Claro ou (b) construir um dashboard equivalente para outro cliente da HYPR.
>
> **Referência de implementação:** `HYPRthik/hypr-claro` → `index.html` (arquivo único, ~20 MB) + `vercel.json` + `videos/`.
>
> **Como usar:** leia as Partes 1–3 para entender o modelo; a Parte 4 antes de editar qualquer coisa; a Parte 6 quando algo quebrar; a Parte 7 para começar um cliente novo do zero; a Parte 9 para saber o que perguntar ao time.
>
> **Documentos relacionados no repositório:**
> - `CLAUDE_Claro_Dashboard.md` — briefing **histórico** da construção do dashboard da Claro (identidade visual, decisões de UI). Atenção: cita `Dashboard_Claro_v2.html`; o arquivo vivo hoje é **`index.html`**. Use-o para entender *intenção de design*, não como referência de estrutura atual.
> - `templates/HYPR-TEMPLATE-DASHBOARD-CLIENTE.xlsx` — template de coleta de dados (genérico, multicliente) que acompanha este playbook.

---

## Índice

1. [Visão geral da arquitetura](#1-visão-geral-da-arquitetura)
2. [Modelo de dados (`DATA`)](#2-modelo-de-dados-data)
3. [Metodologia de cálculo dos consolidados](#3-metodologia-de-cálculo-dos-consolidados)
4. [Como editar o arquivo com segurança](#4-como-editar-o-arquivo-com-segurança)
5. [Criativos e assets](#5-criativos-e-assets)
6. [Problemas já resolvidos (troubleshooting)](#6-problemas-já-resolvidos-troubleshooting)
7. [Playbook: novo dashboard para outro cliente](#7-playbook-novo-dashboard-para-outro-cliente)
8. [De onde tirar os dados (resumo + pós-venda → `DATA`)](#8-de-onde-tirar-os-dados-resumo--pós-venda--data)
9. [Perguntas a fazer por cliente](#9-perguntas-a-fazer-por-cliente)
10. [Convenções de trabalho (git, PR, validação)](#10-convenções-de-trabalho-git-pr-validação)
11. [Glossário de métricas](#11-glossário-de-métricas)

---

## 1. Visão geral da arquitetura

### 1.1 Princípios de design

| Princípio | O que significa na prática |
|---|---|
| **Single-file** | Todo o dashboard (HTML + CSS + JS + dados + criativos) vive em **um único `index.html`**. Não há build, bundler, framework nem backend. |
| **Self-contained** | Imagens são embutidas em `data:` URI base64. O dashboard funciona abrindo o arquivo local, sem servidor. (Exceção deliberada: vídeos — ver §5.2.) |
| **Dados como constante** | Existe **um objeto `DATA`** com tudo. A UI é 100% derivada dele: filtros, contadores e agrupamentos são gerados em runtime. |
| **SPA client-side** | Navegação por `data-go="<seção>"`, sem rotas de servidor. O `vercel.json` manda qualquer URL para o `index.html`. |
| **Zero dependência de build** | Editar = abrir o arquivo, mexer no `DATA`, salvar, commitar. |

> **Por que single-file:** o dashboard é entregue a clientes/agências como um link único, precisa abrir rápido, sobreviver a downloads e não depender de infraestrutura. O custo é um arquivo grande — mitigado em §5.2.

### 1.2 Stack e dependências externas

O arquivo depende de apenas **três** recursos externos (tudo o mais é inline):

| Recurso | Uso | Falha se offline? |
|---|---|---|
| `fonts.googleapis.com` / `fonts.gstatic.com` | Tipografia (DM Sans, Urbanist, Space Mono) | Degrada para fallback |
| `cdnjs.cloudflare.com/.../Chart.js/4.4.1` | Gráficos de brand-lift | Gráficos não renderizam |
| `script.google.com/macros/.../exec` | Log de acesso (Apps Script → Google Sheets) | Silencioso (`.catch(()=>{})`) |

### 1.3 Camadas

```
┌───────────────────────────────────────────────────────────┐
│ 1. GATE DE ACESSO   (#email-gate)                         │
│    e-mail + allowlist de domínios → sessionStorage        │
│    → logAccess() dispara POST ao Apps Script              │
├───────────────────────────────────────────────────────────┤
│ 2. DADOS            const DATA = {...}   (1 linha, ~20MB) │
│    creatives · flights · totals · features · brandlift    │
├───────────────────────────────────────────────────────────┤
│ 3. RENDERIZAÇÃO     funções build*() → template strings   │
│    roteamento: go(secao) ← [data-go]                      │
├───────────────────────────────────────────────────────────┤
│ 4. ANALYTICS (admin) painel só para @hypr.mobi            │
│    lê a planilha via gviz JSONP                           │
└───────────────────────────────────────────────────────────┘
```

### 1.4 Seções e funções de renderização

| Rota (`data-go`) | Função | O que mostra |
|---|---|---|
| `hub` | (HTML estático) | Capa: 6 "big numbers" + cards de navegação |
| `consolidado` | `buildConsolidado()` | KPIs display/vídeo, timeline por flight, tabela geral, cards de features |
| `campanhas` | `buildCampanhasHub()` → `buildCampBox()` | Grade de campanhas com filtros (vertical, mês) |
| *(detalhe)* | `buildFlightDetail(code)` | KPIs do flight, features do flight, criativos do flight |
| `criativos` | `buildCriativos()` | Central de criativos com filtros (campanha, tipo, tamanho) |
| `brandlift` | `buildBrandlift()` | Ondas de survey, gráficos exposto × controle (Chart.js) |
| `materiais` | `buildMateriais()` | Pós-vendas, estudos, audience discovery (cards → Google Slides) |
| — | `buildInconsistencias()` | Notas de transparência sobre a base |
| — | `buildTimeline()`, `buildFeatCards()` | Blocos reutilizados pelo consolidado |

### 1.5 Gate de acesso e analytics

**Gate** (`initGate` / `submitGate`):
- Valida formato de e-mail e **allowlist de domínios** — na Claro: `@hypr.mobi`, `@omc.com`, `@claro.com.br`, `@talent.com.br`.
- Persiste em `sessionStorage['hypr_user']` (dura a sessão do navegador).
- **Não é segurança real** — é identificação/telemetria. O HTML inteiro (dados e criativos) é público para quem baixar o arquivo. **Nunca coloque no `DATA` algo que não possa vazar.**

**Analytics**:
- `logAccess(email)` → `fetch(ANALYTICS_ENDPOINT, {mode:'no-cors'})` com `{email, ua}`.
- Um Apps Script grava numa planilha (`SHEET_ID` / `SHEET_GID`).
- `ADMIN_DOMAIN = '@hypr.mobi'` libera um painel que lê a planilha via **gviz JSONP**.
- **Por cliente, estes 4 valores mudam:** `ANALYTICS_ENDPOINT`, `SHEET_ID`, `SHEET_GID`, allowlist de domínios.

### 1.6 Deploy (Vercel)

```json
{
  "version": 2,
  "builds": [
    { "src": "index.html", "use": "@vercel/static" },
    { "src": "videos/**",  "use": "@vercel/static" }
  ],
  "routes": [
    { "src": "/videos/(.*)", "dest": "/videos/$1" },
    { "src": "/(.*)",        "dest": "/index.html" }
  ]
}
```

⚠️ **Ordem importa.** O catch-all `/(.*) → /index.html` engole qualquer asset. Toda pasta de assets precisa de uma rota **antes** dele. Foi exatamente esse o motivo de `videos/**` ter precisado de rota própria.

---

## 2. Modelo de dados (`DATA`)

`DATA` é um objeto JSON com **6 chaves de topo**, escrito em **uma única linha** do arquivo (na Claro, a linha 526).

```js
const DATA = { creatives:[…], flights:[…], totals:{…}, features:[…], brandlift:[…], big:{…} };
```

### 2.1 `flights[]` — a unidade central

Um **flight** = uma entrega de uma campanha num período. Uma campanha pode ter vários flights (ex.: "Motorola" tem Junho e Julho).

| Campo | Tipo | Descrição |
|---|---|---|
| `vert` | string | Vertical/agrupamento (Claro: `PME`, `Cooperados`, `Institucional`) |
| `camp` | string | Nome exibido da campanha |
| `mes` | string | Mês ("Junho") — usado no filtro |
| `periodo` | string | "19–30/06/2025" (texto livre) |
| `code` | string | **Chave única do flight** (código do report). Liga features e navegação |
| `budget` | number | Budget contratado (R$) |
| `custo_over` | number | Custo efetivo + over-delivery (R$) |
| `cpm_neg` / `cpm_ef` | number | CPM negociado / efetivo |
| `rentab` | number | Decimal (0.264 = 26,4%) |
| `impr` / `clicks` | number | Impressões visíveis / cliques |
| `ctr` / `cpc` | number | Decimal / R$ |
| `video` | bool | Liga o bloco de vídeo |
| `creativeCamp` | string | **Rótulo que liga aos criativos** (`creatives[].camp`) |

**Campos de vídeo** (só quando `video: true`): `v_budget`, `v_cpcv_neg`, `v_cpcv_ef`, `v_views` (completos), `v_startviews`, `v_vtr`, `v_clicks`, `v_exposure`.

**Opcionais:** `prelim: true` (mostra chip "preliminar" e nota de campanha em curso), `custo_ef`.

### 2.2 `features[]` — recorte da entrega

Uma linha por feature ativada **em cada flight**.

| Campo | Descrição |
|---|---|
| `feat` | **Nome do tipo** — é a chave de agrupamento nos cards consolidados |
| `vert`, `camp` | Rótulos exibidos |
| `code` | Código do flight. Pode listar vários separados por `" · "` |
| `vi` | Impressões visíveis da feature |
| `clicks`, `ctr` | Cliques / CTR (decimal). `null` quando não se aplica |
| `plays` | Só PDOOH (nº de plays) |
| `obs` | Texto opcional no rodapé do card |

**Regras críticas:**
- `feat` **idêntico** a um existente → agrega no mesmo card (soma `vi`/`clicks`, **média simples** dos CTRs). Nome novo → card novo **e +1 no contador de Features**.
- O detalhe do flight casa por `ft.code.includes(f.code)` — por isso um `code` composto ("VERCNN · UXTFZT") faz a feature aparecer em ambos.
- Existe `FEAT_DISPLAY` para renomear só a exibição (ex.: `'Tap-to-Map / Tap-to-Go'` → `'Tap-to-Go'`).
- Existe `ICONS` **chaveado pelo nome da feature**. **Renomeou a feature? Renomeie a chave do `ICONS`** ou o ícone quebra.
- Features **não alteram** os totais do portfólio (são recorte da entrega, não entrega adicional).

### 2.3 `creatives[]` — peças veiculadas

| Campo | Descrição |
|---|---|
| `id` | `cr#` (imagem), `vid#` (vídeo), `ifr#` (interativo) |
| `camp` | **Deve bater com `flights[].creativeCamp`** |
| `vertical`, `code` | Rótulos/vínculo |
| `type` | `image` \| `video` \| `iframe` |
| `size` / `size_label` | `"970x250"` / `"970×250"` (o label aceita sufixos: `"1280×720 · 15s"`) |
| `w`, `h` | Números (usados p/ layout; `h/w > 2.2` vira card "tall") |
| `fmt` | Rótulo de formato ("Display", "Vídeo", "Tap-to-Map"…) |
| `name` | Nome exibido (vídeo/iframe usam como título) |
| `src` | `data:` URI (imagem), caminho `/videos/x.mp4` ou URL (vídeo), URL de embed (iframe) |
| `feature` | `true` em interativos (entra no filtro "Features") |

**Cover da campanha** (`firstCreativeFor`): pega só `type === "image"`, com preferência **970×250 → 728×90 → 300×250 → 1200×628 → primeira**. ⚠️ **Flight sem nenhuma imagem fica sem cover** — foi o que aconteceu com a Motorola de Junho (só tinha vídeos + iframe) e se resolveu compartilhando imagens via mesmo `camp`.

### 2.4 `brandlift[]` — survey

```jsonc
{
  "wave": "Mar/25", "type": "Awareness",       // Awareness | Intenção | Favoritismo
  "vert": "…", "camp": "…", "code": "K7F9RL",
  "nExp": 70, "nCtrl": 227,                     // tamanho das amostras
  "rows": [ { "r": "Sim", "exp": 0.40, "ctrl": 0.172, "lift_pp": 22.8 } ]
}
```
`exp`/`ctrl` em decimal; `lift_pp` em pontos percentuais.

### 2.5 `totals{}` — consolidado do portfólio

`budget`, `custo_over`, `cpm_ef`, `rentab`, `impr`, `clicks`, `ctr`, `cpc`, `v_budget`, `v_views`, `v_clicks`, `v_vtr`, `investido_cliente`. Como recalcular: **§3**.

### 2.6 `big{}` — ⚠️ vestigial

`DATA.big` **não é renderizado**: os "big numbers" da capa são **HTML fixo**. Mantemos `big` coerente por higiene, mas **quem aparece na tela é o HTML** — ver checklist em §4.3.

---

## 3. Metodologia de cálculo dos consolidados

> Esta é a parte que mais gera erro. **Não recalcule os totais do zero somando os flights** — os totais armazenados carregam ajustes históricos (bonificadas, reconciliações) que a soma crua não reproduz.

### 3.1 Regra de ouro: aditivo sobre a base

Ao adicionar um flight novo, **some ao total existente**:

```python
totals['impr']       += novo['impr']
totals['custo_over'] += novo['custo_over']
totals['budget']     += novo['budget']       # bate com a soma direta
totals['clicks']     += novo['clicks']       # bate com a soma direta
```

Na Claro, `budget` e `clicks` fecham na soma direta dos flights, mas `impr` e `custo_over` **não** (diferença de ~112 mil impressões vinda de bonificadas). Por isso o modelo aditivo.

### 3.2 Custo efetivo ancorado (o truque das taxas)

As taxas (CPM efetivo, CTR, CPC, rentabilidade) **não são médias de médias**: derivam de um "custo efetivo" ponderado por entrega.

```python
# 1) ancore no total ATUAL (preserva o histórico)
E = totals['cpm_ef'] * totals['impr'] / 1000
# 2) acrescente o custo efetivo do(s) flight(s) novo(s)
E += novo['cpm_ef'] * novo['impr'] / 1000
# 3) atualize impr/clicks e derive as taxas
totals['cpm_ef'] = round(E / totals['impr'] * 1000, 2)
totals['rentab'] = round((CPM_NEG_PADRAO - E / totals['impr'] * 1000) / CPM_NEG_PADRAO, 4)
totals['ctr']    = round(totals['clicks'] / totals['impr'], 4)
totals['cpc']    = round(E / totals['clicks'], 3)
```
`CPM_NEG_PADRAO` na Claro = **14,40**. Confirme por cliente (§9).

### 3.3 Vídeo

```python
vids = [f for f in flights if f.get('video')]
totals['v_views']  = sum(f['v_views']  for f in vids)
totals['v_clicks'] = sum(f['v_clicks'] for f in vids)
totals['v_vtr']    = sum(f['v_views'] for f in vids) / sum(f['v_startviews'] for f in vids)
# CPCV consolidado = ponderado por views completos
cpcv_ef     = sum(f['v_cpcv_ef'] * f['v_views'] for f in vids) / totals['v_views']
rentab_cpcv = (CPCV_NEG_PADRAO - cpcv_ef) / CPCV_NEG_PADRAO      # Claro: 0,36
```

### 3.4 Investido pelo cliente

```python
totals['investido_cliente'] = totals['budget'] + totals['v_budget']
```
⚠️ Só some `v_budget` se o budget de vídeo for **separado** do display. Se o vídeo já está dentro do budget geral (caso da Motorola), **não** conte duas vezes.

### 3.5 O que **não** muda

- **Features** não entram nos totais (recorte da entrega).
- **Criativos** não entram nos totais (só no contador de peças).
- **Brand-lift** não entra nos totais.

---

## 4. Como editar o arquivo com segurança

### 4.1 O arquivo é grande demais para as ferramentas normais

- `Read` **falha** (~650k tokens) — não tente ler o `index.html` inteiro.
- `Edit` exige leitura prévia e a linha do `DATA` tem ~20 MB — inviável.
- `grep` sem cuidado devolve megabytes de base64.

**Sempre edite via Python**, tratando o `DATA` como JSON:

```python
import re, json
L = open('index.html').readlines()
i = 525                                  # índice 0-based da linha do DATA
data = json.loads(re.search(r'^const DATA = (\{.*\});\s*$', L[i].strip()).group(1))
# … mutações …
L[i] = "const DATA = " + json.dumps(data, ensure_ascii=False) + ";\n"
open('index.html','w').writelines(L)
```

### 4.2 Teste de round-trip antes da primeira edição

Garante que reserializar não corrompe os base64:

```python
redump = "const DATA = " + json.dumps(data, ensure_ascii=False) + ";"
assert redump == L[i].rstrip("\n"), "round-trip divergiu — investigar antes de escrever"
```
No repo da Claro o round-trip é **idêntico** (`json.dumps` com `ensure_ascii=False` reproduz o original).

Para inspecionar sem poluir o contexto, **nunca imprima `src`**:
```python
print({k: (v[:40]+'…' if k=='src' else v) for k,v in creative.items()})
```

### 4.3 Checklist de textos/contadores fixos

O `DATA` não cobre tudo. **Ao adicionar um flight, atualize também** (busque pelo valor antigo e substitua com asserção de contagem):

- [ ] Capa: os **6 big numbers** (campanhas/flights, features, impressões, investido, cliques, criativos) e o subtítulo dos vídeos.
- [ ] Capa: parágrafo `hub-sub` ("Oito campanhas, doze flights e quase 80 milhões…").
- [ ] Campanhas: `sec-sub` ("Doze flights ao longo de oito campanhas…").
- [ ] Consolidado: `TOTAL — Portfólio (N flights)`.
- [ ] Consolidado: card **"Rentabilidade do CPCV"** (valor **e** subtítulo "CPCV efetivo R$ X vs. R$ Y contratado") — é HTML fixo.
- [ ] Consolidado: rótulos dos KPIs de vídeo que citam campanhas ("iPhone 17 e Motorola").
- [ ] `buildInconsistencias()`: notas que citam campanhas/flights.
- [ ] `DATA.big` (vestigial, mas mantenha coerente).
- [ ] `ICONS`: só se criou/renomeou tipo de feature.

> **Dica:** faça as substituições com `assert` de quantidade — se o texto mudou, você descobre na hora em vez de publicar um número errado.

### 4.4 Validação em navegador headless

Chromium está em `/opt/pw-browsers/chromium-1194/chrome-linux/chrome`; Playwright é global (`$(npm root -g)/playwright`, CommonJS — use `require`).

```js
const { chromium } = require(process.env.GP + "/playwright");
const b = await chromium.launch({ executablePath: "/opt/pw-browsers/chromium-1194/chrome-linux/chrome" });
const p = await b.newPage();
await p.goto("file:///caminho/index.html", { waitUntil: "load" });
// chame as funções puras diretamente — mais estável que navegar pela UI:
await p.evaluate(() => buildConsolidado());
await p.evaluate(() => buildFlightDetail("UXTFZT"));
```

**O que sempre verificar:** big numbers da capa · pills de filtro e contagens · `hasCover` de cada campanha nova · nº de criativos no detalhe · "N flights" no consolidado · decodificação real de uma imagem nova (`naturalWidth`) · `pageerror` novos.

**Ruído esperado no ambiente (não é bug):**
- `ERR_TUNNEL_CONNECTION_FAILED` — CDN/fonts bloqueados pelo proxy.
- `Cannot read properties of null (reading 'addEventListener')` — **pré-existente**, ocorre também na versão original (Chart.js não carrega). Compare sempre contra `git show HEAD:index.html` antes de culpar sua mudança.
- `DEMUXER_ERROR_NO_SUPPORTED_STREAMS` em `<video>` — o Chromium do Playwright **não traz codec H.264**. O arquivo está OK; valide que a requisição do `.mp4` retorna **200** (não 404) e confie no navegador real.

---

## 5. Criativos e assets

### 5.1 Matriz de decisão

| Tipo | Como entra | Por quê |
|---|---|---|
| Imagem (JPG/PNG/GIF) | `data:` URI base64 no `DATA` | Mantém o self-contained; peso aceitável |
| Vídeo (MP4) | Arquivo em `videos/` + caminho `/videos/x.mp4` | Base64 de vídeo inflaria o carregamento inicial |
| Interativo (Tap-to-Map) | URL de **embed** em `<iframe>` | Conteúdo dinâmico hospedado pela HYPR |

**Regra prática:** se o asset passa de ~1 MB, **hospede** em vez de embutir. Foi a decisão que levou o `index.html` de **24 MB → 19 MB** quando os 2 vídeos da Motorola saíram do base64 (o vídeo só baixa quando o usuário clica em "tocar prévia").

### 5.2 Vídeo hospedado — o que precisa acontecer junto

1. Arquivo em `videos/` (nome limpo: `motorola-familia70-15s.mp4`).
2. `creatives[].src = "/videos/motorola-familia70-15s.mp4"`.
3. **Rota no `vercel.json` antes do catch-all** (§1.6).
4. Handler de preview: renderiza `<video>` para `.mp4`/`data:video`, e `<iframe>` para o resto —
   ```js
   lz.outerHTML = (/\.mp4(\?|$)/i.test(src) || src.startsWith("data:video"))
     ? `<video class="media-frame" src="${src}" controls autoplay playsinline …></video>`
     : `<iframe class="media-frame" src="${src}" …></iframe>`;
   ```
5. Prefira **H.264/AAC** — é o que toca em todos os navegadores.

### 5.3 Links de preview: o que embuta e o que não

O card de preview carrega a URL **dentro de um `<iframe>`**. Portanto **a URL precisa permitir framing**:

| Origem | Embute? |
|---|---|
| `maps.hypr.mobi/embed/…` | ✅ endpoint feito para embed |
| `platform.hypr.mobi/share/adbolt/…` | ⚠️ é link de *share*; testar no ar |
| `displayvideo.google.com/doubleclick/preview/…` (DV360) | ❌ `X-Frame-Options` bloqueia — **peça o `.mp4`** |
| `creative-preview-an.com/direct/creative/…` | ✅ usado nos vídeos do iPhone 17 |

> Ao receber "links de preview" no lugar de arquivos, **avise antes de implementar**: DV360 não embute. O caminho que funcionou foi pedir os `.mp4` originais.

### 5.4 Compartilhar criativos entre flights

Criativos ligam-se por **rótulo**, não por flight. Vários flights com o mesmo `creativeCamp` (ex.: `"Motorola"`) exibem **o mesmo conjunto** de peças. Isso resolve dois casos:
- campanha nova reaproveita as peças da anterior;
- flight sem imagem própria (só vídeo/iframe) **ganha cover**.

Se precisar separar por mês, use rótulos distintos (`"Motorola · Jun"`, `"Motorola · Jul"`).

### 5.5 Fluxo de recebimento (o mais rápido)

1. **`.zip` anexado no chat** ← melhor. Descompacta local, sem rede.
2. `.zip` no Drive ← 1 download em vez de N.
3. Pasta do Drive com arquivos soltos ← mais lento (um download por arquivo).

**Sempre peça as dimensões no nome** (`cliente_produto_970x250.gif`) e **valide contra o header** do arquivo:

```python
# GIF: bytes 6..10 → (w,h) little-endian | PNG: bytes 16..24 → (w,h) big-endian | JPEG: varrer marcadores SOF
assert (w_header, h_header) == (w_nome, h_nome), "dimensão do nome não bate com o arquivo"
```

---

## 6. Problemas já resolvidos (troubleshooting)

| # | Sintoma | Causa | Solução |
|---|---|---|---|
| 1 | `Read` falha no `index.html` | Arquivo de 20 MB, `DATA` em 1 linha | Editar via Python (§4.1) |
| 2 | Total de impressões não bate com a soma dos flights | Base traz ajustes de bonificadas | Modelo **aditivo** + custo ancorado (§3) |
| 3 | Rentabilidade/CPM saem errados | Média de médias | Ponderar pelo custo efetivo (§3.2) |
| 4 | Contador da capa desatualizado após novo flight | Big numbers são **HTML fixo** | Checklist §4.3 |
| 5 | Ícone da feature sumiu ao renomear | `ICONS` é chaveado pelo nome | Renomear a chave junto |
| 6 | Card de vídeo mostra "recusou conexão" | DV360 bloqueia framing | Usar `.mp4` hospedado (§5.3) |
| 7 | `<video>` não toca na validação | Chromium headless sem H.264 | Validar HTTP 200 do asset; confiar no navegador real |
| 8 | `/videos/x.mp4` retorna o HTML do dashboard | Catch-all do `vercel.json` | Rota de assets **antes** do catch-all |
| 9 | Página inicial pesada (24 MB) | Vídeos em base64 | Hospedar em `videos/` (19 MB) |
| 10 | Campanha sem imagem de capa | `firstCreativeFor` só usa `type:"image"` | Compartilhar `creativeCamp` (§5.4) |
| 11 | `curl` no Drive → 403 | Política de egresso do proxy | Usar MCP do Drive ou `.zip` no chat |
| 12 | Base64 gigante estourando o contexto | Resultado do MCP vem inline | Resultados grandes são salvos em disco: processar com Python; nunca imprimir `src` |
| 13 | Arquivo errado escolhido no Drive | Duplicatas, inclusive `[NÃO-USAR]` | Filtrar por `parentId` da pasta + sufixo combinado |
| 14 | `pip install` falha / `pypdf` quebra | Rede restrita; `_cffi_backend` ausente | Só `pypi.org` é liberado; `pip install cffi` conserta o `pypdf` |
| 15 | Erro de JS na validação assusta | `addEventListener` null é **pré-existente** | Comparar contra `git show HEAD:index.html` |
| 16 | Trabalho "sumiu" da produção | PR mergeado **antes** dos commits seguintes | §10 |

---

## 7. Playbook: novo dashboard para outro cliente

### Fase 0 — Coleta (antes de escrever qualquer código)

Peça: **(a)** resumo/planilha de performance por campanha, **(b)** pós-vendas (Slides/PDF), **(c)** criativos (`.zip`), **(d)** identidade visual, **(e)** respostas do §9.

### Fase 1 — Base do projeto

1. Copie `index.html` + `vercel.json` da Claro como ponto de partida (é o template real).
2. **Esvazie o `DATA`** — mantenha as chaves, zere as listas. **Nunca leve dados de outro cliente.**
3. Repositório novo (um por cliente) + projeto Vercel próprio.

### Fase 2 — Branding

| Onde | O que trocar |
|---|---|
| CSS `:root` | Paleta (na Claro, `--ac`/vermelho `#DA291C`), superfícies, bordas |
| `<title>` | "Cliente × HYPR · Consolidado de Campanhas" |
| Logos | `gate-logo` e `tb-logo` (base64) |
| Fontes | Só se a marca exigir |
| Vocabulário | Nome do eixo de agrupamento (na Claro, "Vertical") e seus valores |

### Fase 3 — Gate e analytics

1. Allowlist de domínios do cliente + agência.
2. Novo Apps Script + planilha; atualize `ANALYTICS_ENDPOINT`, `SHEET_ID`, `SHEET_GID`.
3. `ADMIN_DOMAIN` segue `@hypr.mobi`.

### Fase 4 — Dados

1. Preencha `flights[]` (1 linha por entrega).
2. Calcule `totals` — **no cliente novo, a base parte do zero**, então aqui a soma direta *é* a verdade; o modelo ancorado (§3.2) passa a valer das próximas inclusões em diante.
3. `features[]`, depois `brandlift[]` se houver survey.
4. Atualize os textos/contadores fixos (§4.3).

### Fase 5 — Criativos e materiais

1. Imagens em base64; vídeos em `videos/` + rota; interativos por embed.
2. `creativeCamp` ↔ `creatives[].camp` consistentes.
3. Materiais (`buildMateriais`): IDs de Google Slides; **confirme o compartilhamento** ("qualquer pessoa com o link"), senão thumbnail e link falham em produção.

### Fase 6 — Seções sob medida

Se o cliente tiver blocos que a Claro não tem (RMN/ROAS, daily performance, ad size, Google Analytics — §8.3), crie uma seção nova: um `build<Secao>()` + um card no hub + uma rota `data-go`. **Não force métrica nova dentro de `flights[]`** se ela tiver granularidade diferente (por dia, por tamanho, por audiência) — vira lista própria no `DATA`.

### Fase 7 — Validação e deploy

Rode a bateria do §4.4, confira o preview da Vercel em navegador real (vídeos e iframes!) e só então mergeie.

---

## 8. De onde tirar os dados (resumo + pós-venda → `DATA`)

Os dois materiais padrão da HYPR se complementam: **o resumo tem os números; o pós-venda tem a narrativa, os benchmarks e o survey.**

### 8.1 Planilha "Resumo" (exportada em HTML/Sheets) → `flights[]`

Blocos observados (exemplo: Mercado Livre 7.7):

| Bloco do resumo | Vai para |
|---|---|
| Cabeçalho: PI, Cliente, Campanha, Agência, Formatos, Total Investido | `camp`, contexto, `investido_cliente` |
| **DISPLAY PERFORMANCE**: datas, budget contratado, CPM negociado, impressões negociadas/bonificadas, custo efetivo, CPM efetivo, impressões entregues, clicks, CTR, CPC, custo+over, rentabilidade | Quase 1:1 com os campos de display do flight |
| **VIDEO PERFORMANCE**: CPCV negociado/efetivo, views completos, VTR, start views, custo+over, rentabilidade | Campos `v_*` |
| **AUDIENCES**: por segmento (impressões, clicks, CTR, VTR) | `features[]` (ver §8.3) |
| **AD SIZE PERFORMANCE**: por tamanho de criativo | *Não existe hoje* — decidir (§9) |
| **DAILY PERFORMANCE**: série por dia | *Não existe hoje* — decidir |
| **RMN**: Total Sales, Product View, purchases, Average Ticket, ROAS | *Não existe hoje* — decidir |
| **PDOOH**: parceiros (ex.: Helloo, JCDecaux), impressions, nº de plays | `features[]` tipo PDOOH (campo `plays`) |
| **GOOGLE ANALYTICS**: Sessions, Users, Bounce Rate, CPVisit | *Não existe hoje* — decidir |

⚠️ **Cuidado com a coluna certa:** o resumo traz `impressions`, `measurable_impressions` e `viewable_impressions`. O dashboard usa **impressões visíveis** (`viewable`). Mesma lógica para `investimento_previsto` × `investimento_efetivo` → use o **efetivo** como `custo_over`/base do CPM efetivo.

### 8.2 Pós-venda (Slides/PDF) → narrativa, features e brand-lift

| Seção do pós-venda | Vai para |
|---|---|
| Capa (campanha, período, agência) | `camp`, `periodo` |
| "Estrutura de campanha" / "Resultados de mídia" (contratado × entregue, % overdelivery) | Conferência de `budget`/`impr` |
| "Análise de resultados" (KPIs + benchmarks + comparação com campanha anterior) | Validação cruzada dos números |
| "Análise de audiências" (tabela por segmento) | `features[]` |
| "Features" (Downloaded Apps, CTV, Topics…) | `features[]` + `obs` |
| "Survey Performance" (exposto × controle, lift, n) | `brandlift[]` |
| "Teste bonificado" | `features[]` com `obs` explicando |

### 8.3 Audiências × Features — a ambiguidade mais comum

No modelo da Claro, `features[]` guarda **capabilities da HYPR** (Downloaded Apps, Topics, Tap-to-Map, PDOOH, Survey). Em clientes como o ML, o resumo separa **audiências/categorias** (Eletrodomésticos, Moda & Estilo…) de **features** (Downloaded Apps, CTV, Topics). Opções:
- **(A)** só capabilities viram `features[]`; categorias viram seção nova "Audiências";
- **(B)** tudo em `features[]` com um campo `tipo: "audiência" | "feature"`;
- **(C)** só o que o cliente considera diferencial.

**Sem resposta explícita, adote (A)** — preserva o significado do contador "Features ativadas".

### 8.4 Precedência quando os números divergem

Resumo e pós-venda podem divergir (arredondamento, corte de data, atualização posterior — RMN muda até 14 dias). **Precedência: resumo/planilha > pós-venda**, e registre a diferença em `buildInconsistencias()`. **Nunca invente** um número que não esteja em nenhum dos dois: pergunte.

---

## 9. Perguntas a fazer por cliente

Perguntas que o **resumo + pós-venda normalmente NÃO respondem**. Agrupadas; cada uma traz o *default* que se pode assumir se não houver resposta.

### 9.1 Estrutura e vocabulário
1. Qual o **eixo de agrupamento**? (Claro: "Vertical" = PME/Cooperados/Institucional.) Se o cliente não tem, agrupamos por produto, categoria ou data comemorativa? *Default: agrupar por produto/linha.*
2. Uma campanha pode ter **vários flights**? Como nomear (mês? PI?). *Default: 1 campanha = 1 flight; `code` = código do report.*
3. O **nome interno** e o nome usado pelo cliente coincidem? *Default: usar o rótulo do cliente e registrar a diferença nas inconsistências.*
4. A **agência** intermediária deve aparecer? *Default: não exibir.*

### 9.2 Cálculo
5. **CPM negociado padrão** para a rentabilidade (Claro: 14,40) — fixo por cliente ou varia por PI? *Default: usar o `cpm_neg` de cada flight; se variar, não exibir rentabilidade agregada.*
6. **CPCV negociado padrão** (Claro: 0,36).
7. Budget de **vídeo é separado** do display ou está dentro do budget geral? (Evita contar investimento em dobro.)
8. "Custo + Over" deve ser exibido como gasto? *Default: **não** — o cliente paga o budget; o excedente é bonificado.*
9. Usar impressões **viewable** (padrão) ou `impressions`?

### 9.3 Métricas que a Claro não tem (decidir se entram)
10. **Pacing de entrega**, **Alcance (D-1)**, **Frequência (D-1)**.
11. **Start views**, **Brand Exposure Hours**, quartis (25/50/75/100%).
12. **Conversions** e **View-Through Conversions** (+ definição de página tageada).
13. **RMN / Retail Media**: Total Sales, Product View, nº de compras, ticket médio, **ROAS** — vira seção própria?
14. **Google Analytics**: Sessions, Users, New Users, Bounce Rate, Connect Rate, CPVisit.
15. **Daily performance** (série temporal) — vale um gráfico?
16. **Ad size performance** (por tamanho de criativo) — vira tabela?
17. **Benchmarks HYPR** (ex.: CTR 0,70%, VTR 80%) — exibir comparativo? *Default: não exibir; se sim, precisa da tabela oficial de benchmarks.*
18. **Comparação com campanha anterior** — o dashboard deve calcular? *Default: não; fica na narrativa do pós-venda.*

### 9.4 Features, audiências e PDOOH
19. Audiências e features: modelo (A), (B) ou (C) do §8.3?
20. Lista **canônica de features** do cliente (para nomes/ícones estáveis).
21. **PDOOH**: exibir parceiros (Helloo, JCDecaux…)? Métrica principal: impressões ou plays?
22. Feature sem cliques (ex.: PDOOH, CTV) — o que mostrar no lugar do CTR?

### 9.5 Survey / brand-lift
23. Houve survey? Quais **tipos** (Awareness, Intenção, Favoritismo)?
24. Enunciado das perguntas, opções, **n** de exposto e controle.
25. Exibir **concorrentes** citados no favoritismo? *Default: exibir como no pós-venda.*

### 9.6 Criativos e materiais
26. Formatos: imagens, vídeos (**pedir `.mp4`**, não link DV360), interativos (**link de embed**).
27. Criativos são **compartilhados** entre flights da mesma campanha ou exclusivos por mês?
28. Pós-vendas/estudos no Drive — com permissão "qualquer pessoa com o link"?
29. Há material que **não pode** ir para o dashboard (confidencial/NDA)?

### 9.7 Acesso, marca e operação
30. **Domínios liberados** no gate (cliente + agência + HYPR).
31. Analytics: criar Apps Script + planilha próprios (obrigatório — não reutilize os da Claro).
32. **Identidade visual**: paleta, logo (PNG/SVG), tipografia.
33. Domínio/URL de publicação e quem administra a Vercel.
34. **Cadência de atualização** (mensal? por campanha?) e quem envia os dados.
35. Moeda/locale (padrão `pt-BR`, R$) e formato de datas.

---

## 10. Convenções de trabalho (git, PR, validação)

- **1 commit por etapa lógica** (dados → features → criativos → infra). Facilita reverter.
- Mensagem de commit registra **o que foi calculado** (ex.: "cliques 9.112 derivados do CTR 0,74%") — vira documentação.
- **Nunca abra PR sem o usuário pedir.**
- ⚠️ **Um PR mergeado não pode ser reaproveitado.** Se novos commits chegam depois do merge, eles **não** vão para produção sozinhos. Rebaseie sobre a `main` atual e abra um PR novo:
  ```bash
  git fetch origin main && git rebase origin/main
  git push --force-with-lease -u origin <branch>
  ```
  Isso já aconteceu duas vezes neste projeto — **antes de dizer "está publicado", confirme**:
  ```bash
  git merge-base --is-ancestor <commit> origin/main && echo "está na main"
  ```
- **Valide antes de commitar** (§4.4) e diga com honestidade o que **não** deu para validar no ambiente (vídeo H.264, embed de iframe).

---

## 11. Glossário de métricas

| Termo | Definição |
|---|---|
| **Impressões visíveis** (viewable) | Impressões que atenderam ao critério de visibilidade. É a base do dashboard. |
| **Measurable impressions** | Impressões mensuráveis por visibilidade (≥ viewable). |
| **CPM negociado × efetivo** | Contratado no PI × o que a entrega real custou (custo efetivo ÷ impressões × 1000). |
| **Rentabilidade do CPM** | `(CPM negociado − CPM efetivo) / CPM negociado`. Ganho de eficiência. |
| **Custo + Over** | Custo efetivo + sobre-entrega bonificada. **Não é gasto do cliente.** |
| **Bonificada** | Entrega acima do contratado, sem cobrança. |
| **CPCV** | Custo por view completo (100% assistido). Modelo de cobrança de vídeo da HYPR. |
| **VTR** | Views completos ÷ start views. |
| **Pacing** | Ritmo de entrega vs. o previsto (>100% = adiantado). |
| **CTR / CPC** | Cliques ÷ impressões · Custo ÷ cliques. |
| **Brand lift** | Diferença exposto × controle (em p.p. e em %). |
| **ROAS** | Receita ÷ investimento (Retail Media). |
| **Click-Through Conversion** | Conversão após clique no anúncio. |
| **View-Through Conversion** | Conversão após exposição, sem clique. |

---

## Apêndice — Anatomia do `index.html` (mapa de linhas, Claro)

Ordem estável; os números mudam conforme o arquivo evolui.

| Trecho | Conteúdo |
|---|---|
| topo | `<head>`, preconnect de fontes, Chart.js |
| CSS | Tokens em `:root`, componentes (`.bn`, `.campbox`, `.kcard`, `.featcard`, `.ccard`, `.media-lazy`, `.gate-*`) |
| `#email-gate` | Markup do gate |
| Hub | Big numbers (**HTML fixo**) + cards de navegação |
| **`const DATA = {…}`** | **Uma linha** com todos os dados e base64 |
| `ICONS` | Ícones por nome de feature (base64) |
| `build*()` | Renderização |
| `CAMP_CREATIVE`, `firstCreativeFor` | Vínculo campanha ↔ criativos |
| Listeners | Roteamento, filtros, lazy de vídeo/iframe |
| Gate + Analytics | `initGate`, `submitGate`, `logAccess`, painel admin |

---

*Documento gerado a partir da implementação real do dashboard Claro × HYPR e da análise dos materiais-padrão de campanha da HYPR (resumo + pós-venda).*
