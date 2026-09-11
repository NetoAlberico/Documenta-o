# Suite MIT — Rebrand e White-Label (pacote completo)

> 📘 Para a visão de conjunto — arquitetura, regras de negócio e passo a passo de uso de cada app — veja [`ARQUITETURA-E-REGRAS-DE-NEGOCIO.md`](./ARQUITETURA-E-REGRAS-DE-NEGOCIO.md).

Este pacote contém os 5 aplicativos já atualizados:

```
mit-suite/
├── mit-midia-app/          (era: "Stage map manager")
├── mit-vs-app/             (era: vs-web)
├── mit-license-admin-app/  (era: mit-license-admin-site)
├── mit-repertorio-app/     (era: repertorio-app)
└── mit-cifras-app/         (nome mantido, já estava correto)
```

## O que foi feito em TODOS os projetos

1. **"Ministério Igreja da Torre" removido** de títulos de aba, textos de marca no cabeçalho, `manifest.json` (nome do PWA) e READMEs. A única string que **não** foi tocada é a `SECRET` dentro de `mit-license.js` (`Torre-MIT-2026-...`) — isso é uma chave técnica de validação de licença compartilhada entre os apps, não texto de marca. Mudar isso quebraria a licença entre o painel admin e os apps.

2. **Logo/ícone trocado** em todos os arquivos de favicon/PWA (`icon-64`, `icon-192`, `icon-512`, versões *maskable*, `favicon-32`, `favicon-64`, `apple-touch-icon`, `logo-mit.png`, `logo-original.png`) pela nova marca (grade de 4 pontos coloridos — a mesma que já estava em uso no mit-midia-app). Os nomes de arquivo foram mantidos iguais, então nenhum HTML precisou mudar caminho de imagem.

3. **Marca do cliente (white-label)** — implementada em todos os projetos através de um módulo compartilhado, `mit-brand.js`, presente em cada `public/`. Cada app agora tem um botão **🏷️ Marca** no cabeçalho que abre um formulário para configurar:
   - Nome da produtora/igreja/empresa
   - Subtítulo (evento, turnê, culto...)
   - Logo (upload de imagem)

   Tudo fica salvo só no navegador de quem está usando (`localStorage`), nunca é enviado a servidor. Cada app usa uma chave de armazenamento própria (`mit_brand_vs`, `mit_brand_cifras`, `mit_brand_letras`, `mit_brand_admin`, `mit_midia_brand_v1`), então configurar a marca em um app não mistura com os outros.

   No **mit-midia-app**, essa funcionalidade já existia com implementação própria (não usa o `mit-brand.js` compartilhado, mas se comporta de forma idêntica) — foi só mantida e revisada.

## O que foi renomeado

| Pasta antiga | Pasta nova | Título do app | Nome interno (`package.json`/`wrangler`) |
|---|---|---|---|
| Stage map manager | `mit-midia-app` | **MIT Mídia** | `mit-midia` *(novo, nunca publicado)* |
| `vs-web` | `mit-vs-app` | MIT VS | `mit-vs` *(mantido)* |
| `mit-license-admin-site` | `mit-license-admin-app` | MIT License | `mit-license-client` *(mantido)* |
| `repertorio-app` | `mit-repertorio-app` | MIT Letras | `mit-letras` *(mantido)* |
| `mit-cifras-app` | `mit-cifras-app` | MIT Cifras | `mit-cifras` *(mantido)* |

**⚠️ Importante sobre os nomes internos:** só renomeei a **pasta** de cada projeto. O campo `name` dentro de `package.json` e `wrangler.jsonc` foi **mantido igual ao que já estava**, porque esse nome é o que identifica o Worker já publicado na Cloudflare — trocar esse valor faria a Cloudflare tentar criar um Worker novo em vez de atualizar o que já existe, e você perderia o vínculo com o domínio/projeto atual.

Se quiser também renomear o Worker na Cloudflare (não só a pasta local), o caminho é:
1. Trocar o campo `"name"` no `wrangler.jsonc` do projeto.
2. Rodar `npx wrangler deploy` — isso cria um Worker novo com o nome novo.
3. Reconectar o domínio customizado (se tiver um) do Worker antigo para o novo.
4. Só depois apagar o Worker antigo no painel da Cloudflare.

O `mit-midia-app` é exceção: como esse projeto ainda não tinha sido publicado (era o pacote que resolvemos o erro de deploy), o nome `mit-midia` já pode ser usado de primeira, sem esse cuidado.

## mit-midia-app — o que mudou além do rebrand

- `public/index.html` agora é a ferramenta em si (igual ao padrão dos outros apps da suite — um único `index.html` como página principal).
- A antiga landing page de vendas (proposta "StageGrid" como produto para revenda) foi mantida como página opcional em `public/vender.html`, com os textos já trocados para "MIT Mídia". Ela não é obrigatória — se você só quer o app funcionando internamente, pode ignorá-la ou até apagá-la.
- Adicionei `package.json` e `wrangler.jsonc` (o projeto antes só existia como HTML solto, sem esses arquivos).

## Como publicar cada projeto

Mesmo processo em todos:
```bash
cd mit-nome-do-app
npm install
npx wrangler deploy
```

Se o deploy for via Git conectado à Cloudflare (em vez de `wrangler deploy` local), basta subir a pasta inteira do projeto para a raiz do repositório correspondente e o build da Cloudflare cuida do resto — os `wrangler.jsonc`/`wrangler.toml` de cada um já apontam `public/` como pasta de arquivos estáticos.

## Testando localmente antes de publicar

```bash
cd mit-nome-do-app
npx wrangler dev
```

Abra o endereço local mostrado no terminal e confira: ícone da aba do navegador, cabeçalho sem "Ministério Igreja da Torre", e o botão 🏷️ Marca funcionando (abra o modal, digite um nome, suba uma logo, salve e recarregue a página pra confirmar que persistiu).

---

## Atualização — correções e novos recursos

### mit-repertorio-app
Nenhuma mudança de código necessária — o recurso de **criar blocos com nome personalizado e renomeá-los depois** já estava implementado no `app.js` que você enviou (veja "Ou crie um bloco com nome personalizado" no modal de adicionar bloco, e o ✎ em cada bloco pra renomear). Se o seu site publicado ainda mostra a versão antiga (só os 4 botões fixos), é o **service worker** guardando a versão anterior em cache — bumpei `CACHE_NAME` para `repertorio-cache-v9` neste pacote; depois de publicar, um recarregamento (às vezes dois) resolve.

### mit-cifras-app
Corrigido um bug real: o modo "♭/♯ Automático" sempre usava bemol, nunca decidia de fato pelo tom da música. Agora ele escolhe sustenido ou bemol seguindo a convenção musical real — cada tom (maior e seu relativo menor) tem uma preferência própria: G#m/Bm usam sustenido (ex: G#m → C#m7, D#), Abm/Dm usam bemol, etc. A escolha manual ("♯ Sustenidos" fixo) continua funcionando igual, sem mudança. Ajustei todos os pontos que nomeiam notas: seletor de tom, rótulo do tom na folha, capotraste, edição de cifra e o toggle Maior/Menor do modal de importar. Também bumpei o cache do service worker (`mitcifras-cache-v2`) para essa correção chegar aos usuários.

### mit-license-admin-app
1. **Upload de logo não aparecia** — bug real: o próprio CSS da página tinha uma regra global (`input[type=file]{display:none}`) usada para esconder inputs de arquivo nativos em outros lugares do site, e ela acabava escondendo também o campo de logo do modal "Configurar Marca do Cliente". Corrigido com uma regra mais específica que sempre vence essa regra global — o mesmo ajuste foi replicado no `mit-brand.js` do mit-vs-app, que tinha exatamente o mesmo problema (o mit-cifras-app e o mit-repertorio-app não tinham esse conflito, mas receberam o arquivo `mit-brand.js` atualizado mesmo assim, por segurança).
2. **Exportar/Importar contas** — adicionados botões ao lado de "Contas existentes". "Exportar contas" baixa a lista atual (cliente, app, vigência, usuário) direto do servidor de licenças. "Importar contas" lê um arquivo desses e **recria** as contas no servidor — cada uma com usuário/senha novos, baixados num arquivo à parte no fim. Isso não é uma restauração 1:1: o servidor de licenças nunca devolve senhas depois de criadas, então não existe forma de "restaurar" a senha original de ninguém — o import serve para recriar rapidamente o acesso dos mesmos clientes caso o banco de dados do servidor seja perdido.

---

## Atualização 2 — tipos de bloco personalizados e correção de tom (sustenido/bemol)

### mit-repertorio-app
Reorganizei como os **tipos de bloco** funcionam, separando duas coisas que antes estavam misturadas no mesmo modal:
- **"Adicionar bloco ao repertório"** agora mostra só a lista de tipos disponíveis (Celebração, Harpa, Adoração, Final + qualquer tipo que você criar) — sem campo de nome personalizado ali.
- Um botão novo, **"⚙️ Criar/gerenciar tipos de bloco"**, dentro desse mesmo modal, abre uma tela separada onde você cria tipos novos (ex: Ceia, Batismo, Ministração) e remove os que não usa mais. Tudo o que você criar ali aparece automaticamente tanto em "Adicionar bloco ao repertório" quanto no campo "Título musical" da música — os dois já usam a mesma lista por trás.
- A lista de tipos agora é salva (`localForage`, junto com o resto dos dados do app) — os tipos que você criar sobrevivem a um recarregamento da página.
- Backups importados de outro navegador/dispositivo que usem um tipo de bloco que você ainda não tem são registrados automaticamente na sua lista, em vez de serem descartados.
- Cache do service worker atualizado (`repertorio-cache-v10`) para essa mudança chegar aos usuários.

### mit-cifras-app
Dois ajustes a partir do que você reportou com "Oceanos" (Bm) e "Eu Navegarei" (G#m):

1. **Bug real corrigido:** quando o app detectava automaticamente o tom de uma cifra buscada no CifraClub (ex: "G#m"), ele atualizava o valor selecionado mas **não atualizava o texto das opções do próprio seletor de tom** — por isso continuava aparecendo "Ab" mesmo já estando internamente correto como sustenido/menor. Agora o seletor é reconstruído com a grafia certa (sustenido ou bemol, conforme o tom) antes de selecionar o valor.
2. **Clareza para quem está começando:** o botão "⇄ Relativo" no modal de salvar/editar cifra não fazia relação de relativo nenhuma — só alternava entre maior e menor do mesmo tom, só que com um nome que sugeria outra coisa. Troquei por um botão que mostra literalmente **"Maior"** ou **"Menor"**, com o campo chamado "Maior ou menor?". O botão "⇄ Relativo" da folha principal (que faz a troca de relativo de verdade — ex: G ⇄ Em, mesmas notas) não foi tocado, continua correto como estava.
- Cache do service worker atualizado (`mitcifras-cache-v3`).

---

## Atualização 3 — sincronização de renomear/apagar tipos (mit-repertorio-app) e Rider Técnico Profissional (mit-midia-app)

### mit-repertorio-app
- **Renomear agora propaga.** Se o bloco que você está renomeando usa um tipo que existe na lista de "Tipos de bloco", renomear atualiza esse tipo em todo lugar de uma vez: a própria lista de tipos, qualquer outro bloco repetido com o mesmo tipo no repertório, e o "Título musical" de qualquer música que usava esse nome. Se o bloco tiver um nome que não está na lista (um caso legado, de antes desse sistema), ele continua sendo renomeado só naquela instância, como antes.
- **Apagar um tipo agora remove em cascata.** Excluir um tipo pela janela "Tipos de bloco" também remove os blocos com esse nome da tela principal do repertório e desmarca esse "Título musical" das músicas que o usavam — sempre com uma confirmação mostrando quantos blocos/músicas serão afetados antes de executar. As músicas em si nunca são apagadas, só o vínculo com aquele bloco.
- Cache do service worker atualizado (`repertorio-cache-v11`).

### mit-midia-app — nova camada "⑤ Rider Técnico Profissional"
Adicionei uma quinta aba completa, criada do zero, sem alterar as 4 já existentes (Cenografia, Áudio, Luz, Energia continuam iguais). Essa aba nova implementa:

1. **Menu lateral categorizado e colapsável** com 4 grupos (👤 Músicos e Linha de Frente, 🎸 Backline, 🔊 Monitoração & Áudio, 🏗️ Infraestrutura & Energia), cada um com os itens exatamente como especificado.
2. **3 barras de diagnóstico** no topo (Conformidade do Rider, Clareza do Sistema, Carga Elétrica da Linha) que trocam de cor (verde/amarelo/vermelho) e largura dinamicamente via `atualizarDiagnostico(id, porcentagem, tipo)` — com os limiares exatos pedidos (cobertura: >70% verde, 40–70% amarelo, <40% vermelho; carga: <70% verde, 70–90% amarelo, >90% vermelho).
3. **Validação de fase/microfonia** (`verificarAlinhamentoAudio(mic, mon)`): distância euclidiana real em pixels + ângulo da cápsula do microfone (editável, com botões ↺ ↻) contra a direção do monitor, com cone de cobertura de 45° (±22,5° do eixo). Testei os casos de borda manualmente antes de fechar.
4. **Distribuição automática de energia**: cada equipamento com consumo (W) se conecta ao Ponto de Energia mais próximo dentro do raio de cabeamento; ultrapassar a capacidade do ponto marca "Sobrecarga de Circuito" nele.
5. **Peso sobre praticáveis e canais/DI**: cada Praticável soma o peso de tudo que está posicionado sobre sua área e compara com o limite; o app conta canais (microfones + DI) contra o limite de 16, e avisa quando um instrumento que precisa de DI (Teclado/Synth) não tem uma por perto (raio de 100px).
6. **Painel de alertas + tooltips**: barra lateral fixa à direita lista todos os problemas ativos em tempo real, clicável pra rolar até o item; qualquer item com problema ganha um ícone vermelho "!" e mostra o diagnóstico completo ao passar o mouse (tooltip nativo).
7. **Persistência híbrida**: toda alteração salva instantaneamente em `localStorage` e dispara, em segundo plano, um `fetch()` para `/api/save-rider` — se não houver servidor respondendo nesse endpoint ainda, o erro de rede é só registrado no console, sem travar nada (o backup local já está garantido antes da tentativa de rede).

**Importante:** o passo 7 assume que existe (ou vai existir) um endpoint `/api/save-rider` no seu backend — como o mit-midia-app hoje é 100% estático (sem Worker próprio, só arquivos em `public/`), essa chamada vai falhar silenciosamente até você criar esse endpoint (o app continua funcionando normalmente com o backup local enquanto isso). Se quiser, posso te ajudar a montar esse endpoint como um Cloudflare Worker depois.

---

## Atualização 4 — correção de impressão do Rider + licença no mit-midia-app

### mit-midia-app — bug de impressão corrigido
A aba "⑤ Rider Técnico Profissional" imprimia em branco (só o título aparecia). Causa: o canvas tinha ficado sem querer dentro da mesma `div` marcada para não imprimir (`no-print`) que envolvia o menu lateral de arrastar. Corrigido — agora só o menu lateral (que não faz sentido no papel) fica de fora; o canvas, as 3 barras de diagnóstico e a lista de alertas técnicos saem no PDF normalmente. Como o canvas do Rider tem tamanho fixo em pixels (por causa dos cálculos de distância/fase), adicionei um ajuste de escala só para impressão, pra caber certinho numa folha A4 paisagem.

### mit-midia-app — agora com login/licença (igual aos outros 3 apps)
Só este projeto foi alterado — **vs-app, repertorio-app e cifras-app continuam exatamente como estavam**, sem nenhuma mudança.

- Copiei o mesmo módulo `mit-license.js` (usado em mit-vs-app, mit-repertorio-app, mit-cifras-app) para dentro do mit-midia-app, com `APP_ID: "midia"` e a mesma `SECRET` padrão dos outros três — então o mesmo painel `mit-license-admin-app` já gera tokens/contas pra ele.
- O módulo é totalmente auto-contido (injeta a própria tela de login, sem precisar mexer no HTML do app) — mesmo comportamento dos outros: pede o token na primeira vez, lembra depois, avisa antes de vencer, bloqueia automaticamente quando vence.
- **mit-license-admin-app**: adicionei "Mídia" como opção nos dois seletores de app (gerar token solto e criar conta usuário/senha) — agora dá pra emitir acesso pro mit-midia-app do mesmo jeito que pros outros três.
- Antes de publicar de verdade, troque a `SECRET` padrão (tanto no `mit-license.js` de cada app quanto no painel admin) por uma só sua — a que está aí é só o valor de exemplo que já vinha nos outros projetos, mantida igual para os quatro continuarem compatíveis entre si.

---

## Atualização 5 — mit-vs-app virou "MIT Multitracks" (Playback Profissional)

Transformei o mit-vs-app (que já era um player de VS/backing-track bem completo) num produto de multitracks profissional. Sendo direto sobre o que é real e o que precisa de peça externa:

### O que funciona de verdade, 100% no navegador
- **Click track automático** — sintetizado a partir do BPM/compasso (que já existiam no app), sem precisar mais gravar um `click.wav`. Acento no tempo 1, volume e subdivisão ajustáveis, integrado ao PANIC e a todos os modos de roteamento de saída.
- **Voz guia automática** — usa a Web Speech API do navegador (TTS nativo) pra falar o nome da seção (ex. "Refrão") quando não houver um áudio gravado pra ela naquele idioma. Toca ao vivo; não entra no arquivo exportado (deixei isso explícito na interface).
- **Velocidade de reprodução** — separada da transposição de tom (que já existia). Só se aplica na hora de exportar, pra não bagunçar a sincronia de seções/loop/click durante o ensaio ao vivo. Deixei bem claro que muda o tom junto (não é um time-stretch independente).
- **Dividir/cortar trechos** — novo botão ✂️ em cada faixa do Editor de Trilhas (inclusive nas faixas "arquivo inteiro" importadas da pasta). Divide no ponto exato onde o play está; depois é só remover a metade que não quiser com o × que já existia.
- **Exportar em .mp3** e **Exportar stems em .zip** (um .wav por faixa, respeitando o volume de cada uma), além do .wav de mixagem completa que já existia — via `lamejs` e `JSZip` carregados por CDN.

### Separação de Stems — agora 100% local, sem link/API nenhuma
Na atualização anterior, essa função dependia de configurar um serviço externo (URL + chave). Troquei por uma implementação que roda **inteira dentro do navegador**, sem enviar nada a lugar nenhum: filtros de frequência (passa-baixa/passa-alta com a fórmula clássica "cookbook" de EQ) + detecção de transiente por comparação de envelopes rápido/lento (a mesma ideia de um transient shaper de estúdio). Sendo direto sobre o que isso é: **não é uma rede neural treinada** (tipo Demucs/Spleeter) — é processamento de sinal clássico, então o resultado é uma estimativa por faixa de frequência (vocal, baixo, bateria, outros), não uma separação instrumento-por-instrumento perfeita. Testei a lógica com sinais sintéticos (grave+voz+cliques de bateria) antes de integrar — a detecção de transiente reconheceu corretamente os cliques com ~3x mais energia que os trechos sustentados, e o processamento de uma música de 4 minutos leva alguns segundos. As 4 faixas resultantes já entram direto no mixer, prontas pra ajustar/exportar.

### O que não dá pra fazer sem servidor próprio
"Compartilhamento simples por link" (mandar a multitrack pra banda com um link) exigiria hospedar os arquivos em algum lugar — isso é backend de verdade, fora do escopo de um app 100% estático. Não implementei uma versão falsa disso; se quiser essa função de verdade no futuro, precisa de um servidor de arquivos por trás.

---

## Atualização 6 — Meus VS, Song Sections e Editor de Trilhas

### Meus VS — proteção contra perda de trabalho
- Botão novo **"💾 Salvar estado atual"** no transporte — salva imediatamente tudo que está feito (seções, faixas, tom, mixer), sem esperar.
- Corrigido um risco real: abrir uma pasta nova ou reabrir um VS salvo agora força o salvamento do que estava em edição *antes* de trocar. Antes disso, o salvamento automático tinha um pequeno atraso (debounce) que podia perder a última alteração se a pasta fosse trocada rápido demais depois de editar algo.

### Estrutura da Música ↔ Editor de Trilhas
Investiguei o relato de que montar a trilha no Editor não refletia na "Estrutura da Música" — a causa era que essa barra sempre mostrou só as *seções marcadas* (Intro/Verso/Refrão), nunca teve ligação nenhuma com os clipes do Editor de Trilhas (não era bug de dado perdido, era mesmo uma função que faltava). Adicionei uma segunda faixa fina logo abaixo da barra de seções, mostrando automaticamente onde cada clipe de cada faixa (pads, guias, cliques manuais) cai no tempo, colorida por faixa e clicável para pular pra lá.

### Song Sections — arrastar e redimensionar
Os blocos coloridos da "Estrutura da Música" agora são interativos:
- **Arrastar o bloco** reposiciona o início daquela seção no tempo exato onde você soltar.
- **Puxar a borda direita** ajusta onde a *próxima* seção começa (a "duração" de uma seção sempre foi implícita — do seu início até o início da seguinte — então isso é o equivalente real a redimensioná-la).
- Tudo salvo junto com o VS normalmente, e a lista de seções/reagendamento de cues é recalculada automaticamente ao soltar.

### Editor de Trilhas — reescrito
- **Reordenar faixas**: arraste a linha inteira pela alça "⠿" pra cima/baixo (drag-and-drop nativo).
- **Excluir faixa**: botão 🗑 por linha, mais seleção múltipla (checkbox por linha + barra "Excluir selecionadas") para limpar várias de uma vez.
- **Esticar/encolher duração**: alça de redimensionar na borda direita de cada clipe — e também nas faixas de arquivo inteiro (a música original, faixas de click importadas, etc.). Encolher corta o final; esticar repete (loop) o conteúdo até preencher o novo tamanho — não é um time-stretch de verdade (não muda a velocidade/altura do que já toca), deixei isso claro no código pra não prometer mais do que entrega.
- Corrigido de quebra um bug real descoberto no processo: a duração total da música (`recalcularDuracaoTotal`) usava um valor "congelado" de quando a pasta foi carregada, que não se atualizava se uma faixa de arquivo inteiro fosse esticada/encolhida depois — agora recalcula do zero a cada vez, a partir do tamanho atual de cada faixa.

---

## Atualização 7 — MIT Suite: os 4 apps + o novo MIT Escala virando um produto só

O pedido: unir mit-midia-app, mit-vs-app, mit-repertorio-app, mit-cifras-app e o novo mit-escala-app (o `escala-ministerio.html` que você mandou) num produto comercial único — o **mit-license-admin-app continua separado**, por ser uma ferramenta interna de administração, não algo que o cliente final usa.

### A arquitetura escolhida: um "shell" que reúne as 5 ferramentas já prontas

Em vez de reescrever os 5 apps do zero misturados num arquivo só (um risco real de quebrar código já funcionando — cada um tem 2 a 4 mil linhas de JavaScript próprias, com nomes de função/variável que podem colidir se forem simplesmente concatenados), criei **`mit-app/`**: um painel/shell novo que carrega cada ferramenta já existente dentro de uma `<iframe>`, por trás de uma interface única, contínua e com identidade visual coesa. É o mesmo padrão usado por várias suítes comerciais reais (às vezes chamado de "micro-frontends") — cada ferramenta continua sendo o código já testado e funcionando, e o que muda é a experiência de quem usa.

**O que o shell (`mit-app/public/`) entrega:**
- Um **painel/dashboard** com as 5 ferramentas em cartões, cada uma com sua cor de destaque (usando a mesma paleta da logo: roxo, teal, rosa, âmbar) e um elemento de assinatura visual — linhas conectando pontos no fundo do cabeçalho, ecoando o próprio desenho da logo (o X entre 4 pontos).
- **Navegação por abas** no topo pra trocar de ferramenta sem voltar ao painel toda hora.
- **Uma marca só pra tudo**: descobri que os 4 apps antigos já usavam o mesmo formato de dados de marca (nome/subtítulo/logo) — só precisei unificar a chave de `localStorage` (`mit_brand_v1`) entre eles. Configurar a marca uma vez (no shell, ou em qualquer ferramenta) aparece em todas.
- **Um token de acesso só pra tudo**: os apps já usavam a mesma chave de licença (`mit_license_v1`) — dentro do mesmo domínio, gerar um token como "Qualquer app MIT" no painel de administração desbloqueia as 5 ferramentas de uma vez, sem precisar validar em cada uma.

### MIT Escala — rebrandizado e com um bug crítico corrigido

O arquivo enviado (`escala-ministerio.html`) usava uma API de armazenamento (`window.storage`) que só existe dentro do ambiente de artifacts do Claude — **fora dali, em um site publicado de verdade, os dados desapareceriam a cada recarregamento da página**, sem aviso nenhum. Troquei isso por `localStorage` de verdade, igual ao resto da suíte, e confirmei com um teste isolado (via jsdom) que a tela renderiza e os dados persistem corretamente antes de integrar.

Também apliquei a mesma identidade visual: logo, favicons, `mit-license.js` (com `APP_ID: "escala"`) e a linha de marca compartilhada no cabeçalho — mantendo o visual próprio do app (a serifada Fraunces, o layout de "pauta musical" de fundo) que já era bonito, só alinhando a marca/ícone ao restante da suíte, como foi pedido.

### mit-vs-app: faixas importadas agora tão editáveis quanto as criadas na hora

As faixas importadas da pasta (o arquivo inteiro) agora nascem já como faixas *de clipe* — o mesmo modelo de dado das faixas criadas manualmente — então ganham de graça tudo que esse modelo já tinha: arrastar pro lado, esticar/encolher a duração, dividir. Cuidei também da persistência (pra não duplicar a faixa nem perder a posição ajustada ao recarregar a página) e testei essa lógica de restauração isoladamente antes de aplicar.

### O que ainda não existe (sendo direto sobre os limites)

- **Um merge de código de verdade** (os 5 apps virando um único JavaScript, sem iframes) não foi feito — seria um projeto de reescrita bem maior, com risco real de introduzir bugs em funcionalidades que já funcionam. O shell entrega a experiência unificada sem esse risco.
- **Publicação/deploy**: o shell e as 5 ferramentas precisam estar hospedados sob o mesmo domínio pra marca e licença serem compartilhadas automaticamente — isso é responsabilidade de onde você publicar (ex.: um único projeto no Cloudflare Pages apontando pra pasta `mit-app/public/`).
