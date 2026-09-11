# MIT Cifras — Ministério Igreja da Torre

App de cifras/cânticos instalável, responsivo e 100% offline. Mesma estrutura
de arquivos do app "MIT Letras" (Repertório) — sem build step, só arquivos
estáticos.

## O que mudou em relação à versão original

- **Sem login/contas.** Antes cada pessoa precisava criar conta com e-mail e
  senha. Agora é uma biblioteca só, local no aparelho — abre e já usa.
- **Estrutura em arquivos separados** (`index.html`, `styles.css`, `app.js`,
  `manifest.json`, `service-worker.js`) em vez de um único HTML com ícones em
  base64 embutidos — mais fácil de manter e hospedar.
- **Armazenamento em IndexedDB** (via localForage, mesma biblioteca do MIT
  Letras) no lugar de `localStorage`, sem limite de tamanho para muitas cifras.
- **Backup virou "Importar e mesclar".** Antes, importar um backup restaurava
  a conta inteira (substituindo tudo). Agora: você exporta sua biblioteca num
  `.json`, e ao importar o arquivo de outra pessoa o app **mescla**:
  - Música nova → é adicionada.
  - Música já existente (mesmo título + artista) → aparece um alerta
    perguntando se você quer **manter a que já tem** ou **substituir pela
    importada**, com opção de aplicar a mesma escolha a todos os próximos
    conflitos de uma vez.
  - Listas com o mesmo nome são mescladas (as músicas dela são unidas, sem
    duplicar).
- **PWA de verdade:** `manifest.json` + `service-worker.js` próprios (antes o
  service worker era gerado dinamicamente via Blob, o que funciona mas é mais
  frágil). Ícone do app agora é a logo do MIT.

## Como publicar (necessário para instalar)

Assim como o MIT Letras, o botão "Instalar app" e o funcionamento offline só
aparecem em **HTTPS** ou em `localhost`.

**Opção rápida — GitHub Pages (grátis)**
1. Crie um repositório e envie todos os arquivos desta pasta para a raiz.
2. Em *Settings → Pages*, ative o Pages na branch `main`.
3. Acesse a URL gerada no celular ou computador — vai aparecer a opção de
   instalar.

**Opção local (teste rápido)**
```bash
cd mit-cifras-app
python3 -m http.server 8080
# abra http://localhost:8080
```

Também funciona em Netlify, Vercel, Cloudflare Pages ou qualquer hospedagem
estática — é só subir a pasta inteira.

## Estrutura de arquivos

```
index.html          → estrutura da página
styles.css           → visual (tema escuro, papel de cifra, etc. — igual ao original)
app.js                → toda a lógica (teoria musical, diagramas, capo, backup/mesclagem)
manifest.json         → metadados de instalação do PWA
service-worker.js      → cache offline (app + fontes do Google Fonts)
localforage.min.js    → biblioteca de armazenamento local (IndexedDB), hospedada localmente
icons/                → ícones do app (logo do MIT, normal e maskable)
```

## Funcionalidades (todas mantidas da versão original)

- Transposição por semitom, seletor de tom, alternância bemol/sustenido e
  relativo maior/menor.
- Diagramas de acorde para violão/guitarra, teclado e contrabaixo — clique em
  qualquer acorde da cifra pra ver.
- Conversor de capotraste com braço visual.
- Rolagem automática com controle de velocidade.
- Colar uma cifra (acordes + letra) e salvar offline — o app reconhece
  automaticamente linhas de acorde e marcações `[Seção]`.
- Listas personalizadas ("Minhas listas") pra organizar músicas por culto ou
  ensaio.
- Exportar/Imprimir a cifra atual.
- **Backup com mesclagem** (nova): exportar sua biblioteca e importar a de
  outra pessoa sem perder nada, com alerta de conflito por música.
- Responsivo: layout em 3 colunas no desktop/tablet largo, empilhado em
  telas menores.

## Notas

- Os dados ficam no navegador (IndexedDB) do aparelho — não sincroniza sozinho
  entre dispositivos. Use Backup (exportar/importar) pra levar sua biblioteca
  de um aparelho a outro, ou pra juntar a cifra de duas pessoas num só banco.
