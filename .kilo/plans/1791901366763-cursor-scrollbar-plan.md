# Plano — Cursor customizado (dot + anel) e scrollbar com gradiente da marca

## Contexto

- Site em arquivo único: `public/index.html` (CSS inline linhas 19–2521, JS inline linhas 3149–3539, sem build). Único arquivo a editar.
- Hoje **não existe** scrollbar nem cursor customizado (apenas `cursor: pointer` em `button` linha 85 e `.pdf-page` linha 1446).
- Identidade: dark `#141414`, acento `--red: #FF3131`, gradiente assinatura red→rose (`#FF3131 → #FF7A9E`, usado em `.grad` linha 168 e `.hairline` linha 194), cantos 16px, glows sutis.
- Seção clara de preço (`--paper: #F6F4EE`, linhas 1760+) — o cursor precisa continuar visível sobre ela.
- Infra de interações já existente para reutilizar:
  - `REDUCED` — flag de `prefers-reduced-motion` (linha 3155).
  - `FINE_POINTER = matchMedia('(hover: hover) and (pointer: fine)')` (linha 3197) — mesmo gate do spotlight dos cards.
  - Padrão de rAF + CSS vars do spotlight (`--mx/--my`, linhas 1275–1292) como referência de estilo de código.
- `<html>` recebe classe `js` sincronamente (linha 18) — base para esconder o cursor nativo **só** quando o JS de fato ativa o cursor custom.

## Decisões (aprovadas com o usuário)

1. **Cursor:** dot instantâneo (`#FF3131`, ~6px) + anel (~28px) que segue com atraso suave (lerp), estilo Linear/Vercel.
2. **Hover em interativos** (`a`, `button`, `.pdf-page`): anel cresce para ~44px **e preenche** com `var(--red-soft)`; dot permanece.
3. **Scrollbar:** fina e arredondada, thumb com o **gradiente red→rose**, track transparente. Firefox degrada para vermelho sólido (`scrollbar-color` não aceita gradiente).
4. **Sem `mix-blend-mode`:** o vermelho `#FF3131` tem contraste suficiente no dark e no paper — evita cores imprevistas no hover preenchido.
5. **Escopo de gating:** cursor custom só ativa com `FINE_POINTER && !REDUCED` (mesma condição do spotlight). Touch, reduced-motion e no-JS mantêm comportamento nativo.

## Tarefas (em ordem)

### 1. Scrollbar (CSS puro, zero JS)
Inserir bloco novo na seção base do CSS (após `::selection`/`:focus-visible`, ~linha 101):

```css
html {
  scrollbar-width: thin;
  scrollbar-color: #FF3131 transparent; /* fallback Firefox */
}

::-webkit-scrollbar { width: 10px; height: 10px }
::-webkit-scrollbar-track { background: transparent }
::-webkit-scrollbar-thumb {
  background: linear-gradient(180deg, #FF3131, #FF7A9E);
  border-radius: 99px;
  border: 2px solid transparent;           /* padding óptico via background-clip */
  background-clip: padding-box;
}
::-webkit-scrollbar-thumb:hover { filter: brightness(1.12) }
::-webkit-scrollbar-corner { background: transparent }
```

Notas:
- Track transparente deixa o fundo da seção aparecer por trás → o gradiente lê bem no dark e no paper.
- `html` já tem regra em linha 55 (`scroll-behavior`); adicionar as propriedades de scrollbar nessa mesma regra em vez de duplicar o seletor.

### 2. HTML do cursor
Logo após a abertura de `<body>` (antes do `#nav`):

```html
<div id="cursor-dot" aria-hidden="true"></div>
<div id="cursor-ring" aria-hidden="true"></div>
```

### 3. CSS do cursor (seção nova, perto da base)
```css
html.cursor-on, html.cursor-on * { cursor: none }

#cursor-dot, #cursor-ring {
  position: fixed;
  top: 0; left: 0;
  z-index: 100;                 /* acima do nav (z-60) e chat-card */
  pointer-events: none;
  border-radius: 50%;
  will-change: transform;
  opacity: 0;                   /* JS controla visibilidade */
  transition: opacity .3s;
}

#cursor-dot {
  width: 6px; height: 6px;
  margin: -3px 0 0 -3px;        /* centra no ponto exato do mouse */
  background: var(--red);
}

#cursor-ring {
  width: 28px; height: 28px;
  margin: -14px 0 0 -14px;
  border: 1.5px solid rgba(255, 49, 49, .65);
  transition: opacity .3s, width .3s var(--ease-soft), height .3s var(--ease-soft),
              margin .3s var(--ease-soft), background-color .3s, border-color .3s;
}

#cursor-ring.hover {
  width: 44px; height: 44px;
  margin: -22px 0 0 -22px;
  border-color: rgba(255, 49, 49, .45);
  background: var(--red-soft);
}
```

Notas:
- Crescimento via `width/height/margin` + centrado por margem negativa (sem recalcular transform de offset) — o transform fica 100% dedicado à posição.
- Sem cursor nativo, seleção de texto perde o I-beam; página não tem inputs reais (chat é mock) — trade-off aceito.

### 4. JS do cursor (no `<script>` inline, junto do bloco FINE_POINTER ~linha 3197)
- Gate: `if (FINE_POINTER && !REDUCED) initCursor();` e `document.documentElement.classList.add('cursor-on')` **dentro** de `initCursor` (cursor nativo nunca some se o JS falhar).
- Estado: `dotX/dotY` atualizados direto no `pointermove` (sem lerp); `ringX/ringY` com `ringX += (dotX - ringX) * .16` por frame em um único loop `requestAnimationFrame`.
- Posição via `style.transform = translate3d(x, y, 0)`.
- Hover: delegação com `mouseover`/`mouseout` no `document`; `const t = e.target.closest('a, button, .pdf-page')` → `ring.classList.toggle('hover', !!t)`.
- Visibilidade: `document.addEventListener('mouseleave')` → opacity 0; `mouseenter` → opacity 1 (transição CSS cuida do fade). Inicial: primeira leitura de `pointermove` revela (evita anel "voando" do canto — setar posição inicial no primeiro evento antes de mostrar).
- Pressed (polimento barato): `mousedown` → dot `scale(.75)`, `mouseup` → volta, via classe no dot.
- Pausar o rAF quando `document.hidden` (visibilidade) — opcional, mas recomendado.

### 5. prefers-reduced-motion
- Cursor custom nem inicia (gate cobre). Nada a mudar no bloco existente (linha 2494).
- Scrollbar é estática — compatível.

## Riscos / cuidados

- **Não mexer no HTML existente** — só adicionar os 2 elementos do cursor.
- `cursor: none` global só entra com a classe `cursor-on` (JS ativo); nunca em touch/reduced-motion/no-JS.
- `::-webkit-scrollbar` no Firefox é ignorado — o fallback `scrollbar-width/scrollbar-color` cobre; demais browsers usam o WebKit block.
- z-index 100 acima de tudo (nav 60); conferir que não cobre modais — página não tem modais.
- Não conflita com o spotlight dos cards (`--mx/--my`) — listeners distintos, sem estado compartilhado.
- Movimento do anel é decorativo; dot segue 1:1 → precisão de clique não é afetada.

## Fora do escopo

- Efeito magnético em botões, labels textuais no cursor, cursor sobre `iframe`/inputs de arquivo.
- Estilizar scrollbars de containers internos (não há containers roláveis além do documento).
- Alterar `scroll-behavior: smooth` ou âncoras.

## Validação

1. Abrir `public/index.html` no navegador (Chrome/Edge + Firefox).
2. Scrollbar: fina, arredondada, gradiente visível; hover brilha; no Firefox thumb vermelho sólido fino.
3. Cursor: nativo escondido, dot segue instantâneo, anel chega com atraso suave.
4. Hover em nav, links, `.btn`, FAQ e `.pdf-page`: anel cresce a ~44px e preenche com vermelho translúcido; ao sair, volta.
5. Seção de preço (fundo claro): dot e anel visíveis.
6. Mouse sai da janela: cursor some; volta ao reentrar.
7. DevTools: emular touch → cursor nativo normal, dot/anel ausentes; emular `prefers-reduced-motion: reduce` → idem.
8. Clicar: dot dá o scale de press; cliques funcionam normalmente (pointer-events: none nos elementos do cursor).
9. Reveal/scroll das seções inalterados; sem layout shift com a scrollbar nova.
