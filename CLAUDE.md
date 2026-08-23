# XPLuna — tema Windows XP (Luna) para Discord + Vencord

Este arquivo é o manual de manutenção do tema. Leia antes de mexer no CSS:
quase todo problema aqui já foi encontrado uma vez, e a razão está documentada.

Idioma do projeto: **português**. Comentários no CSS e mensagens de commit em pt-BR.
Os comentários do CSS são sem acento de propósito (o arquivo é lido dentro do
Discord e já tomei mojibake antes); este arquivo aqui pode ter acento normal.

---

## O que é

`XPLuna.theme.css` — skin do Discord com a cara do Windows XP tema Luna (azul):
barra de título com gradiente, barra de menu, abas de janela, botões biselados,
campos afundados e barra de rolagem com setas.

Referência visual: <https://botoxparty.github.io/XP.css/>

## Como está montado

```
XPLuna.theme.css
├── @import midnight (refact0r)      ← motor de base, resolve hash rotation
├── variáveis (:root / body)         ← paleta Luna + variáveis do midnight
├── seções (1)…(5)                   ← janela, barra de título, menu, header
└── seções (A)…(AI)                  ← cada tela/componente, em ordem cronológica
```

O tema **não parte do zero**: ele importa o
[midnight](https://refact0r.github.io/midnight-discord/build/midnight.css) do
refact0r e escreve por cima. O midnight já acompanha a rotação de hashes do
Discord, o que economiza metade da manutenção. Não remova o `@import`.

### Mapa das seções

| | assunto |
|---|---|
| (1)–(5) | moldura da janela, barra de título, barra de menu, header da conversa |
| (A)–(D) | separador do header, painel do usuário, cartão de perfil, presença |
| (E) | tela de configurações (Painel de Controle) |
| (F) | tela de Amigos (lista do Explorer) |
| (G)–(I) | consistência entre telas, coluna da direita, a aba como caixa |
| (J)–(M) | aba menor, markdown (code block, quote, spoiler), atividade |
| (N)–(S) | toolbar do header, faixa branca, encaixe da aba, busca, avatar da aba |
| (T)–(V) | botões: verde, componente novo `.button_a22cb0`, hierarquia primary |
| (W)–(X) | modais com chrome de janela (duas famílias diferentes) |
| (Y)–(AA) | Loja, Missões, Nitro |
| (AB) | barras de rolagem |
| (AC)–(AD) | visualizador de imagem, controles de vídeo |
| (AE) | **coisa que não foi feita**, com a medição que explica por quê |
| (AF)–(AI) | barra de menu real, canal com tópico, tela de voz/chamada |

As seções estão em ordem de descoberta, não por assunto. Isso é proposital:
regra que veio depois vence por ordem de cascata, e várias seções tardias
corrigem seções antigas. **Não reordene o arquivo.**

---

## Regras de ouro

### 1. `:has()` com argumento descendente é proibido

Medido por CDP neste tema: uma única regra `:has()` cujo sujeito é raso custa
~0. Uma regra cujo sujeito é o `#app-mount` levou a inserção de um nó no DOM de
**6,5 ms para 89,2 ms**. O Discord muta o DOM o tempo todo, então isso trava o
app inteiro.

- `:has(> filho)` → barato.
- `:has(descendente)` → caro.
- `:has()` com sujeito perto da raiz → catastrófico, mesmo com combinador filho.

Antes de usar `:has()`, procure um `data-*`, um `aria-*` ou uma segunda classe
no próprio elemento. Âncoras sem hash que já provaram valer aqui:

`data-window-chrome="true"`, `data-settings-sidebar-item`, `data-list-item-id`,
`aria-label`, `aria-expanded`, `.user-profile-popout`, `.user-profile-modal-v2`,
`.user-profile-sidebar`, `code.inline`.

### 2. Como medir performance

Está tudo em `scratchpad/perf3.js` (fora do repo). O padrão:

```js
sheet = document.getElementById('vencord-themes').sheet;
sheet.disabled = true;               // baseline
host.appendChild(node); document.body.offsetHeight;   // cronometrar isso
```

Compare com o tema ligado. Bissecte injetando uma regra por vez num `<style>`.
Alvo: **abaixo de 10 ms** por inserção de nó. Hoje o tema está em ~6 ms.

### 3. Nunca tocar na cor do usuário

Banner de perfil, cor de accent, `.background_fb62e2` na chamada — tudo isso é
do usuário. O tema mexe em forma (máscara, raio, moldura), nunca na cor.
Isso já rendeu bronca uma vez.

### 4. Inspecione antes de escrever

Ver "Ferramentas" abaixo. Escrever CSS com base em palpite de nome de classe
desperdiça rodadas: `[class*="subtitleContainer_"]` chegou a ter 19 regras
mirando uma classe que naquele momento não existia no DOM.

---

## Ferramentas

### CDP (obrigatório)

Nunca inspecione clicando no DevTools por screenshot. Suba o Discord com porta
de debug e fale CDP direto:

```bash
taskkill //IM Discord.exe //F
"$LOCALAPPDATA/Discord/app-<versao>/Discord.exe" --remote-debugging-port=9222 &
```

`curl http://127.0.0.1:9222/json` lista os alvos. No scratchpad há um cliente
WebSocket em stdlib pura (`cdp.py`) e um `shot.py` para screenshot:

```bash
python cdp.py "document.querySelector('[class*=\"tile_\"]').className"
python cdp.py - <<'JS'
(()=>{ /* ... */ })()
JS
CDP_URL_MATCH=popout python cdp.py "location.href"   # escolhe a janela destacada
python shot.py saida.png
```

Precisa de `PYTHONIOENCODING=utf-8` quando a saída tem acento.

Cuidados:
- `location.reload()` recarrega o tema, mas **fecha as janelas destacadas**.
- Trocar `enabledThemes` em `Vencord/settings/settings.json` só vale depois de
  reiniciar o app inteiro (o processo principal lê no boot).
- Se você remover nós do DOM para testar, **restaure**. Eu já deixei a barra de
  menu sumida por ter falhado no `insertBefore` de volta.

### Dicionário de classes

O bundle CSS completo (incluindo chunks lazy) sai assim: extrair o mapa de
hashes de chunk CSS do `djs/web.*.js` e baixar `https://discord.com/assets/<hash>.css`.
Dá ~5 MB de CSS e serve para achar o nome real de um componente sem precisar
que ele esteja na tela.

---

## Armadilhas já encontradas (não repita)

**Existem duas `.bar_c38106`.** Uma é `.systemBar_.fixed_` e alterna com a
outra dependendo do estado da janela. Ancore em `[data-window-chrome="true"]`.

**`-webkit-app-region`.** A barra de título é `drag`. Qualquer botão que você
mover para dentro dela para de receber clique — o Electron trata como arrastar
a janela. Marque `no-drag` explicitamente. E o contrário também morde: um
elemento `no-drag` esticado de ponta a ponta impede arrastar a janela.

**`align-items` depende da direção do flex.** O `<section>` do header é
flex-column; pôr `align-items: flex-end` nele empurra tudo para a direita.
Já caí nessa duas vezes.

**Listas virtualizadas** (Amigos, membros) têm `height` inline por linha.
Nunca mexa em `height`/`position` dessas linhas — só cor.

**Modais são de duas famílias.** `.focusLock__ > .root__` e
`.outerContainer__ > .container__`. Estilizar só uma deixa metade dos modais
sem moldura.

**Offset de absolute nem sempre sai do padding box.** No visualizador de imagem
o deslocamento saiu da caixa de conteúdo. Meça em vez de deduzir.

**`overflow` corta ideia boa.** No modal da loja, `previewContainer` é
`overflow: hidden` e `modalRoot` é `overflow: clip`, ambos começando onde a
faixa de título termina — não dá para subir botão nenhum para a barra.

**Itens de menu não são filhos diretos do `.trailing_`.** Ficam dentro de
`.iconWrapper__` e de um `<a>`. Um `:not(:has(> .clickable__))` casa na janela
principal também e derruba a barra de menu.

**Canal com tópico estoura a aba.** O `.topic__` fica dentro do `.children__`,
que é a aba; com `flex: 1 1 auto` a aba vai a 2700px e joga a toolbar para fora
da tela. Está escondido na seção (AG).

**Máscara de avatar.** `foreignObject { mask: none }` é o que deixa o avatar
quadrado — e é a mesma máscara que o plugin RadialStatus usa para recortar o
anel de presença. Os dois não convivem.

---

## Convenções de estilo

Paleta (em `:root`):

| variável | valor | uso |
|---|---|---|
| `--xp-face` | `#ece9d8` | cinza das janelas e botões |
| `--xp-light` | `#ffffff` | luz do biselado |
| `--xp-shadow` | `#aca899` | sombra do biselado |
| `--xp-frame-dark` | `#003c74` | borda de botão |
| `--xp-select` | azul de seleção | item selecionado |

Receitas que se repetem:

```css
/* botão XP */
background: linear-gradient(180deg, #ffffff, #ecebe5 86%, #d8d0c4);
border: 1px solid var(--xp-frame-dark);
border-radius: 3px;

/* brilho âmbar do hover */
box-shadow: inset -1px 1px #fff0cf, inset 1px 2px #fdd889,
            inset -2px 2px #fbc761, inset 2px -2px #e5a01a;

/* campo afundado */
background: #ffffff;
border: 1px solid var(--xp-shadow);
box-shadow: inset 1px 1px #7f7c6e, inset -1px -1px var(--xp-light);

/* botão de toolbar: sem moldura em repouso, moldura só no hover */
border: 1px solid transparent;
background: transparent;
```

Tipografia: Tahoma 11px no chrome, 12px em aba e menu. Nada de peso 600.

Escreva comentário explicando **por que**, não o que. O que o CSS já diz.
Quando uma medição motivou a regra, ponha o número no comentário.

---

## Rotina de trabalho

1. Inspecionar o DOM por CDP e confirmar as classes reais.
2. Escrever a regra numa seção nova no fim do arquivo, com comentário.
3. `location.reload()` por CDP.
4. Um screenshot no fim, não a cada edição.
5. Se mexeu em `:has()` ou em seletor largo, rodar o benchmark.

O dono do projeto pediu explicitamente: **não tirar print a cada mudança**.
Agrupe as edições e verifique uma vez.
