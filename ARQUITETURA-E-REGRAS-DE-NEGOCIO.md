# Suite MIT — Arquitetura, Regras de Negócio e Guia de Uso

**Última atualização:** parte deste documento reflete o estado dos apps neste momento do desenvolvimento; sempre que uma nova funcionalidade grande for adicionada, vale revisar as seções correspondentes.

---

## 1. Visão geral — como os 5 apps se relacionam

A suite é composta por **5 aplicações independentes**, cada uma publicada como seu próprio projeto na Cloudflare, mais **1 peça de infraestrutura compartilhada que não está dentro dos pacotes que venho entregando** (o "license-server" — ver seção 3).

```
                         ┌───────────────────────────┐
                         │   license-server (Worker) │  ← infraestrutura externa,
                         │  mit-license.gomesalberico │    NÃO incluída nos zips
                         │      .workers.dev          │    que venho gerando
                         └─────────────┬─────────────┘
                                       │ valida limite de dispositivos
                                       │ (só na ativação do token)
              ┌────────────────────────┼─────────────────────────┐
              │                        │                         │
     ┌────────▼────────┐     ┌─────────▼────────┐      ┌─────────▼─────────┐
     │  mit-vs-app      │     │ mit-repertorio-  │      │  mit-cifras-app   │
     │  (Backing Tracks)│     │ app (Letras)     │      │  (Cifras)         │
     └────────┬─────────┘     └────────┬─────────┘      └─────────┬─────────┘
              │ mit-license.js (SECRET compartilhado)             │
              └─────────────────────────┬───────────────────────--┘
                                        │
                          Tokens gerados/gerenciados por
                                        │
                         ┌──────────────▼──────────────┐
                         │   mit-license-admin-app      │
                         │   (painel interno, você usa) │
                         └───────────────────────────────┘

     ┌──────────────────────────────────────────────────┐
     │   mit-midia-app (Stage Map / Rider Técnico)       │
     │   — usa mit-license.js (APP_ID: "midia")           │
     └──────────────────────────────────────────────────┘
```

**Como eles se conectam de fato:**

1. **mit-vs-app, mit-repertorio-app e mit-cifras-app** compartilham o mesmo módulo `mit-license.js` — um "portão" de acesso por token que bloqueia a tela até a pessoa digitar um código válido. Os três usam a **mesma `SECRET`** (a chave que assina/valida o token) e cada um se identifica com um `APP_ID` diferente (`vs`, `repertorio`, `cifras`) — assim um token gerado para um app não funciona automaticamente nos outros, a não ser que você gere com `APP_ID: "any"`.
2. **mit-license-admin-app** é o painel que **você** usa para gerar esses tokens e (opcionalmente) criar contas de usuário/senha. Ele não tem lógica de negócio do produto em si — é uma ferramenta interna de gestão de licenças.
3. **mit-midia-app** (Stage Map / Rider Técnico) agora também usa o mesmo `mit-license.js` dos outros três (`APP_ID: "midia"`) — o painel `mit-license-admin-app` já gera token/conta pra ele do mesmo jeito.
4. **O "license-server"** é uma peça de infraestrutura que o `mit-license.js` e o `mit-license-admin-app` **referenciam por URL** (`LICENSE_SERVER_URL` / campo "servidor de licenças" no painel admin), usada só para o limite de dispositivos por token e para a lista de contas usuário/senha. Eu não tenho o código-fonte desse servidor — ele já existia antes de eu entrar no projeto, ou foi montado por você separadamente. Isso significa: **eu não posso garantir o comportamento exato dele**, só documentar como os outros apps o consomem.

---

## 2. Arquitetura técnica de cada app

Todos os 5 são **aplicações client-side** (HTML/CSS/JS puro, sem framework, sem build step) — a lógica de produto roda inteira no navegador da pessoa. Dois deles (mit-cifras-app e mit-repertorio-app) têm, além disso, um **Cloudflare Worker** próprio só para uma tarefa específica: buscar conteúdo em sites de terceiros (contornando CORS/bloqueio de navegador), nunca para guardar dados do usuário.

| App | Deploy | Tem Worker próprio? | Armazenamento | Módulo de licença |
|---|---|---|---|---|
| **mit-midia-app** | Cloudflare Pages/Workers (estático) | Não | `localStorage` (marca, paletas, projeto ativo) + exportação manual em `.json` | Sim (`APP_ID: "midia"`) |
| **mit-vs-app** | Cloudflare Workers (estático, `assets` direto) | Não | Nenhum — lê arquivos de áudio direto da pasta que a pessoa abre no computador dela | Sim (`APP_ID: "vs"`) |
| **mit-license-admin-app** | Cloudflare Workers (estático) | Não | `localStorage` (chave secreta, URL do servidor, histórico de tokens gerados) | — (é o próprio painel) |
| **mit-repertorio-app** | Cloudflare Workers | Sim (`src/worker.js` → `/api/lyrics*`) | `localForage`/IndexedDB (músicas, blocos de repertório, listas) | Sim (`APP_ID: "repertorio"`) |
| **mit-cifras-app** | Cloudflare Workers | Sim (`src/worker.js` → `/api/cifra*`) | `localForage`/IndexedDB (cifras, capotraste, listas) | Sim (`APP_ID: "cifras"`) |

**Por que dois deles têm um Worker "de verdade":** mit-repertorio-app busca letras de música e mit-cifras-app busca cifras (acordes + letra) em sites como o CifraClub — como esses sites não têm uma API pública, o Worker faz essa busca no lugar do navegador da pessoa (evita bloqueio de CORS) e devolve só o texto relevante. Fora essa única função, o resto de cada app é 100% estático.

**Por que os outros três não precisam de Worker:** mit-midia-app e mit-vs-app não buscam nada de fora — todo dado é local (arrastar elementos no palco, ou abrir uma pasta de áudio no próprio computador). mit-license-admin-app só conversa com o license-server externo, direto do navegador.

---

## 3. Dependência externa: o "license-server"

`mit-license.js` (nos 3 apps) e `mit-license-admin-app` fazem chamadas HTTP para uma URL configurável (ex: `https://mit-license.gomesalberico.workers.dev`), esperando destes endpoints:

- `POST /users` — cria uma conta usuário/senha (recebe cliente, app, vigência, limite de dispositivos; devolve usuário e senha — a senha só uma vez).
- `GET /users` — lista as contas existentes.
- `DELETE /users?username=...` — revoga uma conta.
- `POST /users/reset-devices` — zera a contagem de dispositivos ativados de uma conta.
- Validação de limite de dispositivos por token (chamada feita pelo `mit-license.js` na hora de ativar um token novo num aparelho).

**Isso é uma peça real de infraestrutura que precisa estar no ar** para essas funções (contas de usuário, limite de dispositivos) funcionarem — sem ela configurada, o resto de cada app funciona normalmente, só essas duas coisas específicas ficam inativas. Se um dia você quiser, posso ajudar a especificar ou construir esse servidor do zero — hoje eu não tenho o código dele, então não posso garantir nem alterar o que ele já faz.

---

## 4. Regras de negócio por aplicação

### 4.1 mit-midia-app — Stage Map / Rider Técnico Profissional
- **Para quem é:** produtoras, igrejas, técnicos de palco que precisam documentar visualmente onde cada músico, microfone, luz e tomada vai ficar — de um jeito que outra pessoa (um técnico substituto) consiga montar o evento só lendo o mapa.
- **Regra central:** cada elemento arrastado pra um dos 4 mapas (Cenografia, Áudio, Luz, Energia) gera automaticamente uma linha nas tabelas técnicas (Input List, Patch List de Luz, Lista de Voltagem) — a pessoa nunca preenche a tabela na mão, ela só arruma o palco.
- **Regra de segurança elétrica:** toda tomada/régua exige uma cor (110V azul, 110V amarelo, 220V vermelho) escolhida no momento em que é colocada no palco — não dá pra deixar sem, evitando ligação errada.
- **Marca própria:** cada instalação pode ter nome/logo configurados livremente (white-label) — isso é por navegador, não por conta de usuário (não tem login).
- **Rider Técnico Profissional (aba 5):** camada avançada com validações automáticas — fase/microfonia entre microfone e monitor (por distância + ângulo), sobrecarga de energia por ponto, peso sobre praticáveis, limite de canais/necessidade de DI — cada uma dispara um alerta visível na barra lateral de notificações.
- **Controle de acesso:** protegido por token igual aos outros três (`APP_ID: "midia"`), gerado pelo mesmo `mit-license-admin-app`.

### 4.2 mit-vs-app — MIT VS (Backing Tracks)
- **Para quem é:** músicos/técnicos de som que tocam junto de faixas de apoio (backing tracks) e precisam transpor o tom delas em tempo real, sem editar o arquivo original.
- **Regra central:** o processamento de áudio (pitch-shift/transposição) acontece 100% no navegador — nada é enviado a servidor algum, então funciona até sem internet depois de carregado.
- **Origem dos arquivos:** a pessoa abre uma pasta local do computador dela (`webkitdirectory`) — o app nunca hospeda os áudios.
- **Licenciamento:** protegido por token (`APP_ID: "vs"`) — token vencido bloqueia o app até renovar.

### 4.3 mit-license-admin-app — painel interno de licenças
- **Para quem é:** só você (ou quem administra as vendas/licenças) — não é uma tela que um cliente final deveria ver.
- **Regra central:** a mesma `SECRET` usada aqui pra gerar um token precisa ser idêntica à `SECRET` dentro do `mit-license.js` do app de destino — são as duas pontas da mesma assinatura. Trocar a chave aqui sem trocar nos apps invalida todos os tokens já emitidos (isso é intencional — é o "kill switch" caso a chave seja comprometida).
- **Duas formas de licenciar:** token solto (a pessoa cola um código) ou conta usuário/senha (via o license-server externo) — a segunda dá limite de dispositivos e permite revogar sem precisar trocar a chave secreta geral.
- **Backup:** "Assinaturas geradas" (histórico local de tokens) e "Contas existentes" (remoto, via license-server) têm export/import próprios — mas import de contas **recria** com senha nova, nunca restaura a senha antiga (o servidor não guarda senha depois de criada).

### 4.4 mit-repertorio-app — MIT Letras
- **Para quem é:** equipe de louvor organizando o repertório de um culto/evento — sequência de blocos (Celebração, Harpa, Adoração, Final, ou tipos que a própria equipe cria) com as músicas de cada um.
- **Regra central:** um "bloco" é uma INSTÂNCIA na sequência (pode repetir o mesmo tipo várias vezes, cada repetição com suas próprias músicas); um "tipo de bloco" é uma entrada na lista mestra (Celebração, Harpa...) usada tanto para criar blocos quanto para marcar o "Título musical" de uma música.
- **Regra de sincronismo:** renomear um bloco que usa um tipo cadastrado renomeia esse tipo em todo lugar (outros blocos iguais + músicas marcadas com ele); apagar um tipo pela tela de gerenciamento remove também os blocos e desmarca as músicas — sempre com confirmação antes.
- **Licenciamento:** protegido por token (`APP_ID: "repertorio"`).

### 4.5 mit-cifras-app — MIT Cifras
- **Para quem é:** instrumentistas que tocam de cifra (acordes escritos sobre a letra) e precisam transpor tom, usar capotraste, ver diagrama de acorde — tudo offline depois de salvo.
- **Regra central de notação:** o modo "Automático" de sustenido/bemol segue a convenção real de cada tom (maior e seu relativo menor compartilham a mesma grafia) — não é uma escolha arbitrária, é a mesma lógica usada em cifras profissionais/CifraClub.
- **Busca online:** o Worker do próprio app busca a cifra no CifraClub só na hora de importar — depois de salva, a cifra vive só no dispositivo da pessoa (IndexedDB), sem depender de internet.
- **Licenciamento:** protegido por token (`APP_ID: "cifras"`).

---

## 5. Passo a passo de uso — cada aplicação

### 5.1 mit-midia-app
1. Abra o app → se pedir token, cole o código recebido.
2. Escolha a aba do mapa que quer montar primeiro (recomendo começar por **Cenografia**).
3. Arraste os músicos da lista lateral pro palco; clique no ✎ de cada um pra digitar o nome real.
4. Vá para **Áudio & Monitoração** → arraste DI/Microfone/IEM pro palco, clique no ✎ pra vincular a um músico e configurar canal/fone.
5. Vá para **Iluminação** → arraste os aparelhos pras varas de fundo/frente, configure DMX/universo em cada um.
6. Vá para **Energia** → arraste tomadas/réguas, escolha a cor certa (110V azul/amarelo ou 220V vermelho) no momento em que aparecer o aviso.
7. (Opcional) Vá para **Rider Técnico Profissional** pra uma visão avançada com validação automática de fase, energia, peso e canais — resolva os alertas listados na barra lateral direita.
8. Clique em **🏷️ Marca** pra colocar o nome/logo da sua produtora ou igreja.
9. Clique em **💾 Salvar Projeto** pra baixar um `.json` de backup (ou **📂 Carregar Projeto** pra reabrir um depois).
10. Clique em **🖨️ Imprimir / PDF** — cada mapa e cada tabela sai em folha A4 separada.
11. Se quiser incluir itens novos nos menus de arrastar (ex: "Violão 1", "Backing 3"), clique em **⚙️ Incluir / editar / excluir itens** no rodapé de qualquer painel lateral.

### 5.2 mit-vs-app
1. Abra o app → se pedir token, cole o código recebido.
2. Clique em **📁 Abrir Pasta do VS** e selecione a pasta com os arquivos `.wav` no computador.
3. Escolha a faixa, ajuste o tom com o transpositor.
4. Toque direto do navegador; se precisar, exporte a versão já transposta com **💾 Exportar VS Alterado**.

### 5.3 mit-license-admin-app
1. Configure a **Chave secreta** (a mesma que está em `mit-license.js` do app de destino) e, se for usar contas/limite de dispositivo, a **URL do servidor de licenças** + chave admin.
2. Pra gerar um token solto: preencha cliente, app, período de vigência e limite de dispositivos → **Gerar token** → copie e envie pro cliente.
3. Pra criar conta usuário/senha: mesma coisa na seção "Criar conta de usuário" → **Criar conta** → copie a senha exibida (só aparece uma vez).
4. Use **Exportar/Importar backup** (tanto de tokens quanto de contas) antes de trocar de computador ou limpar o navegador.

### 5.4 mit-repertorio-app
1. Abra o app → se pedir token, cole o código recebido.
2. Cadastre as músicas no banco geral (nome, artista, tom, letra).
3. Monte a sequência do culto clicando em **+ Adicionar bloco ao repertório** e escolhendo o tipo (ou crie um tipo novo em **⚙️ Criar/gerenciar tipos de bloco**).
4. Use a busca dentro de cada bloco pra puxar músicas do banco geral.
5. Reordene blocos com as setas ↑ ↓, renomeie com ✎ se precisar.
6. Exporte um backup (JSON) periodicamente pela tela de backup.

### 5.5 mit-cifras-app
1. Abra o app → se pedir token, cole o código recebido.
2. Clique em **+ Colar/salvar uma cifra offline** → cole o texto da cifra (ou use **🔎 Buscar cifra online** pra puxar direto do CifraClub).
3. Confirme título, artista, tom original e se é maior ou menor.
4. Salve — a partir daí a cifra funciona offline, com transposição, capotraste e diagramas de acorde.

---

## 6. O que ainda depende de você / próximos passos possíveis

- **license-server**: eu não tenho o código dele — se algo nessa ponta específica (contas/limite de dispositivo) não estiver funcionando como descrito, é lá que precisa ser verificado, não nos 5 apps que venho editando.
- **`/api/save-rider`** (mit-midia-app, aba Rider Técnico): endpoint ainda não existe — o app funciona com backup local enquanto isso; posso montar esse endpoint como Worker quando quiser.
- Cada app tem seu próprio `README.md` dentro do respectivo projeto com detalhes de deploy — este documento aqui é a visão de conjunto; para o passo a passo de publicar cada um na Cloudflare, veja o README de cada pasta.
