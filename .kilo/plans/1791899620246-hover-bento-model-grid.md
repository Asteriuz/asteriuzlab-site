# Plano — Hover elegante e suave no model grid e no bento

## Contexto

- Site em arquivo único: `public/index.html` (CSS e JS inline, sem build). Único arquivo a editar.
- Cards alvo: `.model-grid > .m-card` (3 cards, `#modelos`) e `.bento > .f-card` (5 cards, `#recursos`).
- **Bug atual:** todos os cards têm classe `reveal`, e `.js .reveal.in { transform: none }` (linhas 202–205, especificidade 0,3,0) sobrescreve `.m-card:hover { transform: translateY(-7px) }` / `.f-card:hover { ... -5px }` (0,2,0). Resultado: **o lift de hover nunca executa** — só border e box-shadow mudam (flash vermelho "seco").
- Estado atual do hover: borda vermelha saturada (`.5`/`.45`) + glow vermelho intenso, `350ms`, easing `--ease: cubic-bezier(.22,.9,.3,1)` (linha 45).
- `prefers-reduced-motion`: bloco CSS global (linhas 2441–2457, zera transições com `* { transition-duration: .01ms !important }` e força `.js .reveal { opacity:1; transform:none }`) + flag `REDUCED` no JS (linha 3102).

## Decisões (aprovadas com o usuário)

1. **Estilo:** soft lift + spotlight que segue o cursor (estilo Linear/Vercel).
2. **Acento:** spotlight quase branco + halo vermelho discreto nas sombras.
3. **Correção do conflito reveal × hover:** mover o reveal da propriedade `transform` para a propriedade individual `translate`.
4. **Escopo:** apenas `.m-card` e `.f-card`. `.route-card`, `.price-card`, FAQ e demais ficam fora do escopo.
5. Sem mudança de HTML: pseudo-elementos + variáveis CSS. JS novo vai no `<script>` inline existente (região ~3136–3145).

## Tarefas (em ordem)

### 1. Corrigir conflito reveal × hover (pré-requisito)
Em `.js .reveal` (linhas 196–200):
- Trocar `transform: translateY(26px)` por `translate: 0 26px`.
- Trocar a transição para `opacity .6s var(--ease), translate .6s var(--ease)`.

Em `.js .reveal.in` (linhas 202–205):
- Trocar `transform: none` por `translate: 0 0`.

No bloco `prefers-reduced-motion` (linhas 2454–2457):
- Trocar `transform: none` por `translate: none`.

Notas: `translate` individual é suportado desde Safari 14.1 / Chrome 104; navegadores mais antigos degradam para fade sem offset (aceitável). O `.f-pdf:hover .pdf-page` (fan-out) não é afetado — continua usando `transform` nos filhos.

### 2. Refinar lift + borda + sombra (compartilhado entre `.m-card` e `.f-card`)
Em `:root` (linha 45): adicionar `--ease-soft: cubic-bezier(.16, 1, .3, 1)`.

`.m-card` (linha 854) e `.f-card` (linha 1242) — nova transição:
```css
transition: transform .4s var(--ease-soft), border-color .4s, box-shadow .4s, background .4s;
```
(duração simétrica .4s; sem `will-change` persistente — só 8 cards, evita camadas extras.)

`.f-card`: adicionar `position: relative` (o `.m-card` já tem) para ancorar o pseudo-elemento da task 3.

`.m-card:hover` (859–863) e `.f-card:hover` (1245–1249) — valores novos (iguais nos dois):
```css
transform: translateY(-4px);
border-color: rgba(255, 255, 255, .18);
background: linear-gradient(180deg, rgba(52, 52, 52, .6), rgba(38, 38, 38, .42));
box-shadow:
  0 18px 40px -18px rgba(0, 0, 0, .55),
  0 4px 18px -8px rgba(0, 0, 0, .4),
  0 0 0 1px rgba(255, 49, 49, .14);
```
- Lift reduzido (−7/−5px → −4px), borda passa a clarear de forma neutra, sombra em camadas escura com halo vermelho bem sutil (fim do glow saturado).
- `.m-card` não tem box-shadow base; transição `none → sombra` interpola corretamente nos browsers modernos (tratada como sombra transparente). Se quiser garantia extra, definir base `box-shadow: 0 0 0 0 rgba(0,0,0,0)`.
- Manter os backgrounds base dos cards como estão (`.m-card` linha 850, `.f-card` linha 1236).

### 3. Spotlight que segue o cursor
CSS (após as regras de hover de `.m-card`/`.f-card`):
```css
.m-card::after, .f-card::after {
  content: "";
  position: absolute;
  inset: 0;
  border-radius: inherit;
  pointer-events: none;
  opacity: 0;
  background: radial-gradient(320px circle at var(--mx, 50%) var(--my, 50%), rgba(255, 255, 255, .07), transparent 60%);
  transition: opacity .4s var(--ease-soft);
}
.m-card:hover::after, .f-card:hover::after { opacity: 1; }
```
- `--mx/--my` com default `50%` → spotlight centrado (fallback CSS puro, funciona sem JS).
- Sobreposição de conteúdo: alpha 7% é imperceptível sobre texto; elementos com z-index maior (`.pdf-page`, `.pdf-chip`) pintam acima do pseudo — efeito desejado.

JS (no `<script>` inline, perto do bloco de reveals ~3141):
- Selecionar `$$('.m-card, .f-card')`.
- Gate: executar somente se `!REDUCED && matchMedia('(hover: hover) and (pointer: fine)').matches`.
- `pointermove` por card: `card.style.setProperty('--mx', e.offsetX+'px')` / `--my` (usar `getBoundingClientRect` com `clientX/clientY` se `offsetX` não bastar), com throttle via `requestAnimationFrame`.
- Sem listeners em touch/reduced motion → degrada para o fallback CSS (spotlight centrado) ou nada.

### 4. Polimento opcional (barato, só se sobrar apetite)
- Micro-scale no badge/ícone no hover do card: `.m-card:hover .m-badge, .f-card:hover .f-icon { transform: scale(1.05) }` + `transition: transform .4s var(--ease-soft)` nos mesmos.
- Descartar se ficar pesado: validar visualmente antes de manter.

## Riscos / cuidados

- **Não tocar no HTML** — tudo via CSS/JS existentes.
- O reveal com `translate` muda TODAS as 41 entradas `.reveal` do site (não só os cards) — comportamento visual idêntico (fade-up), mas é uma mudança global; testar reveals de outras seções também.
- Navegadores < Safari 14.1: sem offset no reveal (só fade). Aceito.
- Touch devices: gate `(hover: hover) and (pointer: fine)` evita spotlight/lift "grudados" no tap.
- `prefers-reduced-motion`: o bloco global já zera transições; confirmar que o spotlight não ativa (gate no JS) e que o reveal usa `translate: none`.

## Validação

1. Abrir `public/index.html` no navegador (Chrome + um WebKit se possível).
2. Hover nos 3 `.m-card` e 5 `.f-card`: lift de −4px **visível** (confirma o fix), borda clareia, sombra em camadas aparece suave, spotlight segue o cursor e some no leave.
3. Card de PDF (`.f-pdf`): fan-out das páginas no hover continua funcionando junto com o novo lift.
4. Scroll: reveals de todas as seções continuam com fade-up (sem card "travado" deslocado).
5. DevTools: emular touch → sem spotlight; emular `prefers-reduced-motion: reduce` → tudo instantâneo, sem spotlight.
6. Performance: mover o cursor rápido entre cards sem travadas (throttle rAF ativo).

## Fora do escopo

- Aplicar o tratamento em `.route-card`, `.price-card`, FAQ, botões e nav.
- Reestruturar as grids ou o conteúdo dos cards.
