# a11y-plugin

Plugin de acessibilidade standalone para sistemas da Prefeitura do Rio. Vanilla JS, zero dependências, funciona em sistemas arbitrários.

Distribuído via CDN (GCP Bucket):
```
https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/a11y-plugin.min.js
```

---

## Índice

1. [O que é e como funciona](#o-que-é-e-como-funciona)
2. [Como usar](#como-usar)
   - [HTML puro](#html-puro)
   - [Next.js](#nextjs)
   - [Variáveis de ambiente](#variáveis-de-ambiente)
3. [Tokens CSS (modo daltônico)](#tokens-css-modo-daltônico)
4. [CSP (Content Security Policy)](#csp-content-security-policy)
5. [Assets no bucket](#assets-no-bucket)
6. [Como contribuir](#como-contribuir)
   - [Desenvolvimento local](#desenvolvimento-local)
   - [Build e deploy para o bucket](#build-e-deploy-para-o-bucket)

---

## O que é e como funciona

O plugin injeta um botão flutuante na página com um painel de recursos assistivos. As preferências do usuário são salvas automaticamente no `localStorage` (chave: `a11y-preferences`) e restauradas a cada visita.

**Recursos disponíveis:**

- Tamanho de fonte (3 níveis)
- Espaçamento entre letras e altura de linha (3 níveis)
- Fonte para dislexia (OpenDyslexic)
- Alto contraste
- Modo daltônico — deuteranopia, protanopia, tritanopia (matrizes Viénot 1999 + paleta Okabe-Ito)
- Destacar links
- Reduzir movimento
- Saturação (3 níveis)
- Máscara de leitura
- Esconder imagens
- VLibras integrado com botão customizado

**VLibras:** quando presente na página, o plugin substitui o botão nativo do VLibras por um botão customizado com o mesmo estilo visual.

**Modo daltônico:** usa duas camadas complementares. Um filtro SVG (matrizes de daltonização Viénot 1999) corrige as cores nos pixels de toda a página. Sistemas que usam os tokens CSS listados em [Tokens CSS](#tokens-css-modo-daltônico) recebem adicionalmente a paleta Okabe-Ito aplicada com precisão exata nas cores semânticas.

---

## Como usar

### HTML puro

Adicione ao final do `<body>` (ou no `<head>` com `defer`):

```html
<!-- VLibras (opcional, recomendado) -->
<script src="https://vlibras.gov.br/app/vlibras-plugin.js"></script>
<div vw class="enabled">
  <div vw-access-button class="active"></div>
  <div vw-plugin-wrapper><div class="vw-plugin-top-wrapper"></div></div>
</div>

<!-- Plugin de Acessibilidade -->
<script src="https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/a11y-plugin.min.js"></script>
<script>A11yPlugin.init()</script>
```

Só isso. Sem configuração adicional necessária.

### Next.js

Use o componente `Script` com `strategy="afterInteractive"`:

```tsx
'use client'

import Script from 'next/script'

const CDN_URL =
  'https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/a11y-plugin.min.js'

export function AccessibilityPlugin() {
  return (
    <Script
      src={CDN_URL}
      strategy="afterInteractive"
      onLoad={() => {
        const w = window as unknown as { A11yPlugin?: { init: () => void } }
        w.A11yPlugin?.init()
      }}
    />
  )
}
```

> **Por que `'use client'`?** O `onLoad` é um event handler — Next.js exige que o componente seja Client Component para usar handlers. Sem o `'use client'`, você verá o erro `Event handlers cannot be passed to Client Component props`.

Adicione o componente no layout principal:

```tsx
import { AccessibilityPlugin } from '@/components/accessibility-plugin'

export default function RootLayout({ children }) {
  return (
    <html>
      <head>
        <AccessibilityPlugin />
      </head>
      <body>{children}</body>
    </html>
  )
}
```

### Variáveis de ambiente

Em vez de hardcodar a URL do CDN, use variáveis de ambiente para facilitar a troca entre ambientes (staging, produção).

**Next.js** — crie ou edite `.env` / `.env.local` na raiz do projeto:

```env
NEXT_PUBLIC_A11Y_PLUGIN_URL=https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/a11y-plugin.min.js
```

> Variáveis com prefixo `NEXT_PUBLIC_` são expostas ao browser. Sem esse prefixo, a URL não estará disponível no lado do cliente.

Atualize o componente:

```tsx
'use client'

import Script from 'next/script'

const CDN_URL = process.env.NEXT_PUBLIC_A11Y_PLUGIN_URL!

export function AccessibilityPlugin() {
  if (!CDN_URL) return null

  return (
    <Script
      src={CDN_URL}
      strategy="afterInteractive"
      onLoad={() => {
        const w = window as unknown as { A11yPlugin?: { init: () => void } }
        w.A11yPlugin?.init()
      }}
    />
  )
}
```

Para habilitar/desabilitar via feature flag:

```env
NEXT_PUBLIC_A11Y_PLUGIN_ENABLED=true
NEXT_PUBLIC_A11Y_PLUGIN_URL=https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/a11y-plugin.min.js
```

```tsx
export default function RootLayout({ children }) {
  const enabled = process.env.NEXT_PUBLIC_A11Y_PLUGIN_ENABLED === 'true'
  return (
    <html>
      <head>
        {enabled && <AccessibilityPlugin />}
      </head>
      <body>{children}</body>
    </html>
  )
}
```

**HTML puro / outros frameworks**: carregue dinamicamente:

```html
<script>
  var s = document.createElement('script')
  s.src = 'URL_DO_CDN'
  s.onload = function() { A11yPlugin.init() }
  document.head.appendChild(s)
</script>
```

---

## Tokens CSS (modo daltônico)

Se o sistema usar variáveis CSS com os nomes abaixo, o modo daltônico aplica automaticamente a paleta Okabe-Ito com precisão exata:

```css
--primary
--destructive
--success
--success-secondary
--chart-1 até --chart-5
--status-badge-open / --status-badge-in-progress / --status-badge-done
--green-100 / --green-200 / --red-100 / --red-200
--blue-100 / --yellow-100 / --yellow-200
```

Sistemas sem esses tokens continuam funcionando — o filtro SVG atua nos pixels da página inteira independentemente.

---

## CSP (Content Security Policy)

O plugin carrega recursos externos do bucket GCP. Se o sistema tiver CSP configurado, adicione:

| Diretiva | Valor | Motivo |
|---|---|---|
| `script-src` | `https://storage.googleapis.com` | O próprio bundle |
| `script-src` | `https://vlibras.gov.br` | VLibras (se usar) |
| `font-src` | `https://storage.googleapis.com` | OpenDyslexic-Regular.otf |
| `img-src` | `https://storage.googleapis.com` | familia-vlibras.webp |
| `style-src` | `'unsafe-inline'` | O plugin injeta `<style>` no DOM |

Exemplo completo:

```
Content-Security-Policy:
  script-src 'self' https://storage.googleapis.com https://vlibras.gov.br;
  font-src 'self' https://storage.googleapis.com;
  img-src 'self' https://storage.googleapis.com data:;
  style-src 'self' 'unsafe-inline';
```

> O plugin injeta estilos via `document.createElement('style')` em runtime, por isso o `unsafe-inline`. Se o sistema usar CSP baseado em nonce, será necessário adaptar o plugin para aceitar `A11yPlugin.init({ nonce: 'SEU_NONCE' })` — abrir issue se necessário.

---

## Assets no bucket

Todos os assets estão em:
```
https://storage.googleapis.com/rj-escritorio-dev-public/superapp/a11y-plugin/
```

| Arquivo | Descrição |
|---|---|
| `a11y-plugin.min.js` | Bundle de produção (~58 KB) |
| `a11y-plugin.js` | Bundle de desenvolvimento, não minificado (~76 KB) |
| `OpenDyslexic-Regular.otf` | Fonte para modo dislexia |
| `familia-vlibras.webp` | Imagem do popup do botão VLibras |

---

## Como contribuir

### Desenvolvimento local

```bash
git clone https://github.com/prefeitura-rio/a11y-plugin
cd a11y-plugin
npm install
npm run build        # gera dist/a11y-plugin.js para desenvolvimento
npm run typecheck    # verificação de tipos TypeScript
```

Abra `index.html` no browser para testar — já carrega VLibras + plugin local.

### Build e deploy para o bucket

1. Gere o bundle minificado:

```bash
npm run build:min    # gera dist/a11y-plugin.min.js para colocar no bucket
```

2. Faça upload para o bucket GCP:

```bash
https://console.cloud.google.com/storage/browser/rj-escritorio-dev-public/superapp/a11y-plugin
```

> O bucket é público. O `max-age=3600` garante que browsers façam cache por 1 hora, ajuste conforme necessidade.

**Estrutura do projeto:**

```
src/
  features/     — cada feature de acessibilidade (font-size, color-blind, etc.)
  ui/           — botão, painel, estilos injetados
  state.ts      — estado persistido no localStorage
  index.ts      — entry point, exporta init()
dist/           — bundle gerado pelo build, não commitado
index.html      — página de teste local
```
