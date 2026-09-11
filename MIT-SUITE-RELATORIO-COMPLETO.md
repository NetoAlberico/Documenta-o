# MIT Suite — Relatório Completo do Projeto

> Documento gerado para dar continuidade ao desenvolvimento em uma nova conversa.
> Contém: visão geral, arquitetura, tudo que foi implementado, bugs corrigidos (com causa raiz — importante pra não repetir o mesmo erro), e pendências.

---

## 1. Visão geral

MIT Suite é um conjunto de ferramentas web pra equipes de louvor/produção de igreja, publicadas como Cloudflare Workers. São **3 projetos publicados separadamente**:

| Projeto | O que é | Nome do Worker na Cloudflare |
|---|---|---|
| **mit-app** | Painel único com 6 ferramentas (Mídia, Multitracks/Vs, Letras, Cifras, Escala, Nuvem), cada uma carregada num iframe | `mit-music` |
| **mit-cloud-sync** | Backend (Worker + banco D1) — sincronização em nuvem das listas compartilhadas **e** todo o sistema de licenças/login | `mit-cloud` |
| **mit-license-admin-app** | Painel de administração de licenças (gera tokens, contas de usuário, agora com tela de login própria) | Publicado sob o domínio que aparece como `mit-license-client.gomesalberico.workers.dev` nos logs — confirme o nome exato do Worker no painel da Cloudflare |

Repositório GitHub do usuário: `NetoAlberico/MIT-Music` (confirmado num log de deploy).

---

## 2. Arquitetura de cada projeto

### 2.1 `mit-app` (painel — Worker `mit-music`)

```
mit-app/
  wrangler.jsonc          <- tem "assets" + "main" (Worker HÍBRIDO: serve estático E roda código)
                              tem "keep_vars": true (protege GENIUS_ACCESS_TOKEN/VAGALUME_API_KEY de deploys)
  src/
    worker.js              <- roteador: /api/lyrics* e /api/cifra* vão pros handlers; resto cai nos arquivos estáticos
    lyrics.js               <- busca de letra online (Vagalume + Genius)
    cifra.js                <- busca de cifra online (CifraClub via DuckDuckGo)
  public/
    index.html, shell.js, shell.css   <- painel/dashboard com os 6 cartões
    tools/
      midia/    <- MIT Mídia (mapa de palco, patch list, rider) — SEM sync de nuvem "playlist" (não aplicável)
      vs/       <- MIT Multitracks (player/editor de faixas) — app complexo, mobile ainda não totalmente responsivo
      letras/   <- MIT Letras (repertório de música + letras)
      cifras/   <- MIT Cifras (cifras com transposição, cifra online)
      escala/   <- MIT Escala (escala de equipe)
      nuvem/    <- MIT Nuvem (gerenciar listas compartilhadas — cria/renomeia/exclui listas, adiciona membros)
```

Cada ferramenta em `tools/<nome>/` tem seus próprios `index.html`, `app.js` (ou script inline), `mit-license.js`, `mit-cloud-sync.js`, `mit-cloud-bridge.js`.

**IMPORTANTE**: `mit-license.js`, `mit-cloud-sync.js`, `mit-cloud-bridge.js` são **cópias idênticas** entre as 6 ferramentas (exceto `STORAGE_KEY`/`APP_ID` em `mit-license.js`, que são únicos por app — ver seção 4). Ao corrigir um bug num desses 3 arquivos, replicar pros outros 5.

### 2.2 `mit-cloud-sync` (backend — Worker `mit-cloud`)

```
mit-cloud-sync/
  wrangler.toml            <- "keep_vars = true" (ANTES de qualquer [secão]!), binding D1 "DB", Durable Object "LISTA_DO"
  schema.sql               <- TODAS as tabelas (rodar de novo no D1 sempre que uma nova tabela for adicionada)
  src/
    index.js                <- TODAS as rotas (ver seção 3)
    lista-durable-object.js <- coordena WebSocket em tempo real por lista
  client/
    mit-cloud-sync.js       <- módulo cliente (copiado pra dentro de cada tools/<app>/)
    mit-cloud-bridge.js     <- camada de mais alto nível sobre o mit-cloud-sync.js
```

**Segredos configurados no Worker `mit-cloud`** (Configurações → Variáveis e Segredos):
- `MIT_LICENSE_SECRET` — assina/verifica os tokens (precisa ser IGUAL à "Chave secreta" do mit-license-admin-app)
- `MIT_ADMIN_KEY` — protege as rotas administrativas (`/admin/*`, `/users`, `/tokens`)

### 2.3 `mit-license-admin-app` (painel de licenças)

```
mit-license-admin-app/
  public/
    index.html    <- tudo num arquivo só (HTML+CSS+JS inline), organizado em abas + tela de login
```

---

## 3. Todas as rotas do backend (`mit-cloud-sync/src/index.js`)

### Sistema de licenças (sem prefixo `/api`)
| Rota | Método | Autenticação | O que faz |
|---|---|---|---|
| `/activate` | POST | nenhuma (recebe token no body) | Ativa um token direto num dispositivo (checa limite de dispositivos) |
| `/login` | POST | nenhuma | Login usuário+senha → devolve um token assinado |
| `/account-status` | GET | nenhuma | Checa se uma conta ainda está ativa |
| `/token-status` | GET | nenhuma | Checa se um token (por nonce) foi revogado |
| `/devices`, `/reset`, `/tokens/revoke`, `/tokens/unrevoke` | GET/POST | `x-admin-key` | Gerenciamento de dispositivos/revogação |
| `/users` | POST/GET/DELETE | `x-admin-key` | CRUD de contas usuário+senha dos CLIENTES |
| `/users/reset-devices` | POST | `x-admin-key` | Zera contagem de dispositivos de uma conta |
| `/tokens` | POST/GET/DELETE | `x-admin-key` | Registro de tokens BRUTOS gerados (metadados + o token completo). **DELETE agora também revoga de verdade** (insere em `tokens_revogados`) |

### Login de administrador (pro próprio mit-license-admin-app)
| Rota | Método | Autenticação | O que faz |
|---|---|---|---|
| `/admin/existe` | GET | nenhuma | Diz se já existe algum login de admin cadastrado (+ diagnóstico: `adminKeyConfigurada`, `tamanhoDaChave`) |
| `/admin/criar` | POST | `x-admin-key` | Cria/atualiza um login de admin (usuário+senha) |
| `/admin/login` | POST | nenhuma | Login usuário+senha → devolve a `MIT_ADMIN_KEY` de verdade |
| `/admin/usuarios` | GET/DELETE | `x-admin-key` | Lista/remove logins de admin |

### Busca de usuário / configuração
| Rota | Método | Autenticação | O que faz |
|---|---|---|---|
| `/api/usuarios/buscar` | GET | Bearer token normal | Autocomplete de membro na Nuvem — busca em `usuarios` E `tokens_gerados` juntos |
| `/api/config` | GET/POST | Bearer token normal | Configurações por usuário (chave-valor livre) |

### Listas compartilhadas (MIT Nuvem)
| Rota | Método | Autenticação | O que faz |
|---|---|---|---|
| `/ws` | upgrade | Bearer token (query `?token=`) | WebSocket em tempo real por lista |
| `/api/listas` | POST/GET | Bearer token | Criar / listar minhas listas |
| `/api/listas/:id` | PATCH/DELETE | Bearer token (só responsável) | Renomear / excluir lista |
| `/api/listas/:id/membros` | GET/POST/DELETE | Bearer token | Listar/adicionar/remover membro |
| `/api/listas/:id/itens` | GET/POST/DELETE | Bearer token | Itens sincronizados (tipo: musica, cifra, escala, playlist, etc.) |

---

## 4. Tabelas do banco D1 (`schema.sql`)

| Tabela | Pra quê |
|---|---|
| `listas`, `lista_membros`, `itens` | Núcleo do MIT Nuvem |
| `configuracoes_usuario` | Config livre por usuário |
| `usuarios` | Contas usuário+senha dos CLIENTES (cliente, app_id, tipo, início, fim, max_dispositivos, ativo) |
| `dispositivos_ativados` | Rastreio de quais `device_id` já usaram cada `nonce:` (token) ou `user:` (conta) |
| `tokens_revogados` | Nonces revogados (tokens diretos que devem parar de funcionar) |
| `tokens_gerados` | Metadados + o token completo de cada "Gerar novo token" — pra aparecer em qualquer aparelho |
| `admin_usuarios` | Login (usuário+senha) de quem administra o painel — SEPARADO da tabela `usuarios` (essa é dos clientes) |

**Sempre que uma tabela nova for adicionada ao `schema.sql`, é preciso rodar o arquivo de novo no Console do D1** (`mit-cloud-db` → aba Console → colar o `schema.sql` inteiro → Executar). Isso NÃO apaga dados existentes (tudo usa `CREATE TABLE IF NOT EXISTS`).

---

## 5. Sistema de licenças — os DOIS caminhos que existem hoje

Isso é importante porque foi fonte de confusão real durante o projeto:

1. **"Gerar novo token"** — token bruto, assinado no navegador (HMAC), **sem senha**. Histórico local + espelhado no banco (`tokens_gerados`) pra aparecer em qualquer aparelho e pra busca de membro funcionar.
2. **"Criar conta de usuário (usuário + senha)"** — conta de verdade no banco (`usuarios`), com login/senha, limite de dispositivos aplicado de forma mais robusta.

Os dois convivem. Qualquer um dos dois pode ser adicionado como membro de uma lista da Nuvem (a busca de membro pesquisa nas duas tabelas).

**"Remover" um token bruto agora também revoga de verdade** (antes só tirava da lista de organização, sem cortar o acesso — isso foi corrigido).

---

## 6. Login isolado por app — decisão de segurança importante

Cada um dos 6 apps tem sua **própria chave de armazenamento** pro token de licença:
```
mit_license_v1_vs, mit_license_v1_midia, mit_license_v1_letras,
mit_license_v1_cifras, mit_license_v1_escala, mit_license_v1_nuvem
```
Isso foi uma decisão EXPLÍCITA do usuário: entrar com token/usuário-senha em UM app **não deve** liberar os outros — cada app exige login separado, sem exceção (nem Mídia, Escala ou Multitracks).

O `DEVICE_ID_KEY` (identifica o aparelho físico, não a sessão) continua **compartilhado** entre os 6 apps — é assim que o limite de dispositivos funciona corretamente (um mesmo aparelho abrindo 3 apps diferentes conta como 1 dispositivo, não 3).

A URL do "servidor" (MIT Cloud Sync) também é salva **por navegador/aparelho**, não globalmente — por isso, ao abrir num aparelho novo, é preciso configurar o campo "Servidor" (em cada app, ou pelo menos no MIT Nuvem) apontando pra `https://mit-cloud.gomesalberico.workers.dev`.

---

## 7. Checagem de revogação em tempo real

Todos os 6 `mit-license.js` agora disparam a checagem de "ainda estou autorizado?" (`checkAccountStillActive`/`checkTokenNotRevoked`) em 3 momentos:
1. Ao abrir o app (como sempre foi)
2. A cada 1 hora (`RECHECK_MS`)
3. **Novo**: assim que a internet volta (`window.addEventListener("online", ...)`) e assim que o app volta a ficar visível (`visibilitychange`) — com uma guarda que evita checagens simultâneas

---

## 8. Sincronização de playlists entre apps

MIT Letras e MIT Cifras: quando você conecta numa lista da Nuvem, o app cria/atualiza automaticamente uma **"Minha Lista" local com o mesmo nome da lista da Nuvem**, contendo as músicas/cifras sincronizadas — o vínculo é feito pelo **ID da lista** (`nuvemListaId`), não pelo nome, então renomear na Nuvem atualiza a mesma lista local em vez de duplicar. Isso também atualiza em tempo real (`MitCloudBridge.onGrupoAtualizado`) conforme outras pessoas adicionam/removem itens.

MIT Escala, Mídia e Vs sincronizam seus próprios tipos de dados (`escala`, `mit_midia_projeto`, `vs_perfil`) mas **não** têm esse conceito de "Minha Lista automática" — só Letras/Cifras têm, porque só eles têm "Minhas Listas" como recurso interno próprio.

---

## 9. Tela de login do MIT License (implementada recentemente)

O painel inteiro (`mit-license-admin-app`) agora fica bloqueado por um **gate de login em tela cheia** até:
- Login usuário+senha (via `/admin/login`), OU
- "Ainda não criei um login — usar a chave direto" (cola a `MIT_ADMIN_KEY` crua) — **essa opção agora VALIDA a chave com o servidor antes de aceitar** (chamando `/admin/usuarios`); antes aceitava qualquer coisa digitada sem checar, o que causava bugs sérios (sobrescrevia silenciosamente uma chave correta com uma errada)

Depois de logado, aparece um botão **"Sair"** fixo no canto que limpa a chave salva e volta pro gate.

**Criar o primeiro login de admin**: precisa da `MIT_ADMIN_KEY` crua (colada via "chave direta") pra autorizar — depois disso, usuário+senha funciona em qualquer aparelho.

---

## 10. Bugs corrigidos — histórico com causa raiz (útil pra não repetir)

| # | Bug | Causa raiz | Correção |
|---|---|---|---|
| 1 | Token "não pôde ser ativado" | `mit-cloud-sync.js` só guardava o token dentro de `conectar()`, que exige lista já escolhida — mas criar a 1ª lista roda ANTES disso | Criada `MitCloudSync.definirToken()`, chamada antes de qualquer ação que precise do token |
| 2 | `Failed to fetch` genérico escondendo o erro real | Exceções inesperadas no Worker não tinham cabeçalho CORS na resposta de erro padrão da Cloudflare | `try/catch` no nível mais alto do `fetch()`, sempre devolvendo JSON com CORS mesmo em erro inesperado |
| 3 | `database_id`/variáveis "voltando" pro valor de exemplo | Eu, o assistente, recriei arquivos do zero em sessões novas sem perceber que já existiam versões corretas anteriores | Sempre `grep`/comparar TODAS as cópias existentes antes de sobrescrever um arquivo crítico |
| 4 | `GENIUS_ACCESS_TOKEN`/`MIT_ADMIN_KEY` "desaparecendo" depois de um novo deploy | Deploys via GitHub (Workers Builds) substituem variáveis/segredos configurados pelo painel se não houver `keep_vars = true` na config | Adicionado `keep_vars = true` em AMBOS `mit-app/wrangler.jsonc` e `mit-cloud-sync/wrangler.toml` (tem que ficar ANTES de qualquer seção `[tabela]` no `.toml`) |
| 5 | Busca de letra/cifra não funcionava dentro do painel unificado | `mit-app` era só "arquivos estáticos", sem nenhum código de servidor — `/api/lyrics`/`/api/cifra` não existiam ali | Criado `mit-app/src/worker.js` + `lyrics.js` + `cifra.js`, com `wrangler.jsonc` novo (`"assets"` + `"main"` juntos, `run_worker_first: true`) |
| 6 | Busca de cifra falhando "de vez em quando" | Regex de extração dos resultados do DuckDuckGo exigia uma classe CSS específica (`result__a`) que nem sempre está presente | Regex generalizado pra aceitar qualquer link pra uma música do CifraClub, independente da classe |
| 7 | "Tom Original" não detectado em algumas cifras | Só 4 padrões de regex tentavam achar o tom no HTML da página — poucos pra cobrir as variações do CifraClub | Adicionados mais 5 padrões alternativos + aviso visível (⚠️) quando mesmo assim não detecta |
| 8 | Campo "Tom Original" aparecendo vazio / botões desalinhados no Cifras (mobile) | `.field-row` sem quebra responsiva — o botão "♭/♯ Automático" (largura fixa) espremia o `<select>` até sumir o texto | `min-width` no select + `@media (max-width:420px){ .field-row{flex-direction:column} }` |
| 9 | Linhas de cifra aparecendo cortadas no início (ex.: "Era uma manhã" → "ma manhã") | Área de rolagem horizontal (`.songsheet-scroll`) iniciando com `scrollLeft` diferente de zero | `el.songsheetScroll.scrollLeft = 0` forçado a cada `renderSongsheet()` |
| 10 | Selo "Logado como" não aparecia em Vs/Mídia/Escala/Nuvem | Código tentava `document.getElementById(...)` ANTES do navegador processar até aquele ponto do HTML (script antes do elemento no documento) | Função de exibição agora espera `DOMContentLoaded` antes de tocar no DOM |
| 11 | Lista "desaparecendo" do MIT Nuvem depois de usar outro app | Não era bug — o backend corretamente só mostra listas onde a IDENTIDADE ATUAL (do token) é membro; usuário estava testando com identidades diferentes sem perceber | Adicionado selo "Logado como" em todos os apps pra deixar a identidade atual visível |
| 12 | App tentando reconectar pra sempre numa lista sem acesso (403 infinito) | `MitCloudBridge.conectar()` não tratava esse erro como definitivo, deixando o WebSocket reconectar pra sempre | Detecta a mensagem "não tem acesso a esta lista", desiste, limpa a última lista salva, mostra erro claro |
| 13 | Painel de administração de licenças "vazio" em outro aparelho | Histórico de tokens/contas só existia no `localStorage` daquele navegador específico | Novo fluxo busca do SERVIDOR (`GET /tokens`, `GET /users`) ao abrir o painel, mesclando com o que houver localmente |
| 14 | "Erro 404" ao criar lista só no celular | Configuração de "Servidor" (URL do mit-cloud) é salva por aparelho — celular tinha uma URL antiga/errada | Reconfigurar o campo "Servidor" nesse aparelho específico |
| 15 | Tela de login aceitava qualquer chave sem checar, sobrescrevendo silenciosamente uma chave correta | Nenhuma validação contra o servidor antes de salvar | Ambos os pontos de entrada de chave agora chamam `/admin/usuarios` pra confirmar antes de aceitar |
| 16 | GitHub upload "funcionando" mas Cloudflare não atualizava (só no `mit-license-client`) | Cada Worker (mit-music, mit-cloud, mit-license-client) está conectado a um repositório GitHub **diferente** — `NetoAlberico/MIT-License-Client` é separado de `NetoAlberico/MIT-Music`. Causa exata da falha específica ainda não confirmada (usuário estava subindo no repo certo, commit aparecia no GitHub, mas nenhum build novo disparava na Cloudflare) — **investigação parada no meio**, próximo passo era checar Settings → Webhooks → Recent Deliveries do repositório no GitHub | Não resolvido ainda — ver pendência #4 abaixo |
| 17 | **[EM ABERTO]** Texto da cifra vazando pro fundo escuro fora da folha (folha creme), em telas estreitas — linhas longas de cifra/letra não ficam contidas | Tentativa 1: faltava `min-width:0` em `.main-stage` (item de CSS grid) — corrigido, mas não resolveu completamente | Tentativa 2: adicionado `min-width:0` também em `.songsheet-scroll`, e a zeragem do `scrollLeft` passou a rodar duas vezes (na hora + após `requestAnimationFrame`) — **usuário testou no celular de verdade e confirmou que AINDA não funciona**. Testes isolados (fora do app real, recriando a estrutura HTML/CSS à mão) mostraram a correção funcionando — ou seja, existe alguma diferença entre a estrutura real do DOM e a réplica usada nos testes que ainda não foi identificada |

---

## 11. Pendências conhecidas (não resolvidas ainda)

1. **MIT Multitracks (Vs) no celular**: é um editor de múltiplas faixas de áudio, com painéis de largura fixa pensados pra tela grande. Torná-lo totalmente responsivo é um redesenho, não um ajuste rápido de CSS — ainda não abordado.
2. **MIT Mídia no celular**: nunca chegou a ser auditado de verdade pra responsividade (é um app de mapa de palco com arrastar-e-soltar, categoria diferente dos formulários simples).
3. Ainda não foi feita uma auditoria completa e sistemática de TODOS os formulários/telas de TODOS os 6 apps — os ajustes até agora foram pontuais, reagindo a prints específicos enviados pelo usuário.
4. **[URGENTE — investigação em andamento]** GitHub → Cloudflare não estava disparando build novo pro Worker `mit-license-client`, mesmo com commit confirmado no GitHub, repositório certo (`NetoAlberico/MIT-License-Client`), branch certa (`main`), e "commit direto na main" confirmado (não "nova branch"). Próximo passo pendente: usuário ia checar Settings → Webhooks → Recent Deliveries desse repositório no GitHub, pra ver se o webhook da Cloudflare existe e se os deliveries mais recentes aparecem com sucesso (verde) ou falha (vermelho). Se não tiver webhook nenhum listado, a solução é desconectar e reconectar o Git na aba Configurações → Build do Worker.
5. **[EM ANDAMENTO — reaberto]** Texto de cifra/letra vazando pro fundo escuro em telas estreitas (ver bug #17 na tabela acima). Duas tentativas de correção (`min-width:0` em `.main-stage` e depois também em `.songsheet-scroll`, mais reforço no reset de `scrollLeft`) não resolveram o problema real no aparelho, apesar de terem funcionado em testes isolados/recriados à mão. **Próximo passo planejado**: pedir pro usuário abrir o site num COMPUTADOR, usar o modo de emulação de celular do DevTools (F12 → Ctrl+Shift+M), inspecionar o elemento que está vazando, e olhar os valores computados de CSS (width, overflow, min-width) direto no elemento real — isso vai revelar a diferença exata entre a estrutura assumida nos testes e a estrutura real do DOM. Usuário concordou em fazer esse teste, mas ainda não enviou o resultado.

---

## 12. Como fazer deploy de cada projeto (resumo)

### mit-app (Worker `mit-music`)
- Sobe a pasta inteira via GitHub (Git conectado, deploy automático via `npx wrangler deploy`)
- Segredos/variáveis no painel: `GENIUS_ACCESS_TOKEN` e `VAGALUME_API_KEY` (opcionais, melhoram a busca de letra) — **ou**, mais confiável, direto no `wrangler.jsonc` em `"vars"` (foi o que precisou ser feito pro Genius, porque o painel não estava persistindo)

### mit-cloud-sync (Worker `mit-cloud`)
- Sobe a pasta via GitHub
- **Sempre que uma tabela nova for adicionada**: rodar o `schema.sql` de novo no Console do D1 (`mit-cloud-db`)
- Segredos: `MIT_LICENSE_SECRET`, `MIT_ADMIN_KEY` (Configurações → Variáveis e Segredos)
- `wrangler.toml` tem `keep_vars = true` — não remover essa linha

### mit-license-admin-app
- Sobe a pasta via GitHub
- Sem segredos próprios — usa a URL do `mit-cloud` + login (usuário/senha ou chave crua) configurados dentro do próprio app (fica salvo no navegador)

---

## 13. Detalhes técnicos — investigação em andamento do vazamento de texto no Cifras

Contexto pra quem for continuar essa investigação específica (item #5 das pendências, seção anterior):

**Estrutura HTML relevante** (`mit-app/public/tools/cifras/index.html`):
```html
<div class="app-shell">          <!-- display:grid -->
  <div class="main-stage">        <!-- grid-area:main; display:flex; flex-direction:column; align-items:center -->
    <div class="songsheet-scroll" id="songsheetScroll">   <!-- overflow-x:auto (só no breakpoint mobile) -->
      <div class="songsheet" id="songsheet">                <!-- fundo creme -->
        <div class="songsheet-head">
          <div class="title" id="sheetTitle"></div>
        </div>
        <div class="cifra-body" id="cifraBody"></div>       <!-- linhas com white-space:pre (não quebram) -->
      </div>
    </div>
  </div>
</div>
```

**O que já foi tentado** (arquivo `mit-app/public/tools/cifras/styles.css`):
1. `.main-stage` recebeu `min-width:0` (era item de grid sem isso — não encolhia)
2. `.songsheet-scroll` recebeu `min-width:0` também
3. Em `app.js`, dentro de `renderSongsheet()`, o reset de `scrollLeft = 0` passou a rodar duas vezes (direto + via `requestAnimationFrame`)

**Ainda não verificado / próximos passos sugeridos**:
- Inspecionar via DevTools (modo celular) os valores COMPUTADOS de `width`, `overflow-x`, `min-width` direto no elemento `.songsheet` e `.cifra-body` reais, comparando com o que o CSS declara — pra saber se as regras estão sendo aplicadas de verdade ou se tem algo sobrescrevendo
- Verificar se existe algum CSS mais específico (maior especificidade, ou `!important`) em outro lugar do arquivo que possa estar competindo com essas regras
- Considerar se `.cifra-body` (que tem `white-space:pre-wrap` na base, mas `.cifra-line` sobrescreve pra `white-space:pre` — ver arquivo, por volta da linha 244-246) precisa de um tratamento diferente: talvez width explícito com JS calculando a linha mais longa, ou uma abordagem de zoom/escala em vez de depender só de flexbox/grid pra conter o conteúdo
- Os testes que "passaram" foram feitos recriando a estrutura HTML à mão (fora do app de verdade) com `wkhtmltoimage` — pode haver CSS carregado de outro arquivo, ou uma diferença estrutural real no app que não foi replicada nesses testes isolados
- Usuário concordou em testar com DevTools no computador (modo celular, F12 → Ctrl+Shift+M) mas ainda não enviou o resultado — esse é o próximo passo imediato

---

## 14. Convenções e coisas importantes pra quem for continuar

- Todo texto de interface é em **português brasileiro**, tom direto e caloroso, explicando o "porquê" nos comentários de código
- Sempre que possível, **testar a lógica isoladamente** (Node.js puro, sem precisar de navegador) antes de entregar — várias correções neste projeto foram verificadas assim
- Ao editar arquivos que existem em **múltiplas cópias idênticas** (os 3 módulos client-side compartilhados entre os 6 apps), sempre propagar a mudança pra todas as cópias
- Cuidado ao recriar arquivos do zero em sessões novas — **sempre comparar com cópias já existentes** antes de sobrescrever (isso já causou retrabalho real neste projeto)
- `wrangler.toml`: chaves de nível raiz (como `keep_vars`) precisam vir ANTES de qualquer `[seção]`, senão o TOML as associa à seção errada
