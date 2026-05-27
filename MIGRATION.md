# MIGRATION.md — PINs → Firebase Auth + RBAC

Migração do Mahau CRM do modelo de **PINs em texto plano** (identidade 100% client-side,
Firestore aberto) para **Firebase Auth (email/senha) + papéis via custom claims + Firestore
Security Rules default-deny**.

> **Regra de ouro:** nada em produção sem aprovação explícita. A ordem abaixo é
> desenhada para permitir **rollback** a cada passo — os passos irreversíveis
> (aplicar rules, limpar PINs) vêm **por último**, só depois de tudo testado.

---

## Estado atual (antes da migração)

- App: `index.html` servido por **GitHub Pages** a partir da `main`.
- Identidade: **Firebase Auth + custom claim `role`** (código já no branch `feat/firebase-auth-rbac`).
- Papéis: `admin` (Sócio), `manager` (Gerente), `promoter` + claim `promoter` ∈ {Ygor, Matheus, Mahau}.
- Rules-alvo versionadas em `firestore.rules` (**ainda não aplicadas** no Console).

### O que está no branch `feat/firebase-auth-rbac` (commits acumulados, sem push)
| Sub-entrega | Commit | Conteúdo |
|---|---|---|
| A | `firestore.rules`, `firebase.json`, `firestore.indexes.json` | rules RBAC versionadas |
| B | `index.html`, `recover-bday.html` | Firebase Auth + login email/senha |
| C | `index.html` | remoção do código PIN morto + `roleFromClaims` |
| D | `.gitignore` | ignora `clean-pins.html` |
| E | `set-claims.mjs`, `MIGRATION.md`, `.gitignore` | tooling de claims + esta doc |

---

## Pré-requisitos (ambiente local)

- **Node.js** — confirmado v24.16.0.
- **firebase-admin** — `npm install firebase-admin` (gera `node_modules/`, já gitignorado).
- (Opcional, para aplicar rules via CLI) **firebase-tools** — `npm install -g firebase-tools`.
- Acesso de **owner/editor** ao projeto Firebase `mahaucrm` no Console.

---

## Ordem cronológica de cutover

### Passo 0 — Backup do estado atual  ·  *reversível*
1. **Rules atuais:** Console → Firestore Database → aba **Regras** → copie o conteúdo
   atual e salve num arquivo local (ex.: `rules-backup-ANTES.txt`). É o seu rollback do Passo 7.
2. **Dados:** rode o backup JSON do app (admin → aba **Exportar**) e guarde o arquivo.
3. **Branch:** garanta que a `main` está intacta (`git log main` deve terminar no commit
   `c5335d0`, sem nada da migração). O código novo vive só no branch.

> **Rollback:** nada foi alterado ainda.

---

### Passo 1 — Versionar as rules  ·  ✅ *já feito (Sub-entrega A)*
`firestore.rules` + `firebase.json` + `firestore.indexes.json` já estão no branch.
**Nada aplicado no Console.** Sem efeito em produção.

> **Rollback:** trivial — são arquivos no branch.

---

### Passo 2 — Habilitar o provedor Email/Senha  ·  *reversível*
Console → **Authentication** → **Sign-in method** → habilite **Email/senha**.

> **Rollback:** desabilitar o provedor. O app PIN antigo ignora o Auth — sem impacto.

---

### Passo 3 — Criar os usuários  ·  *reversível*
Console → **Authentication** → **Users** → **Add user** (email + senha) para cada conta.
Use senhas fortes e guarde-as com segurança.

| Email | Papel pretendido |
|---|---|
| `vitorcordeiroo@icloud.com` | admin (Sócio) — **real** |
| `socio@mahau.com.br` | admin (Sócio) — *placeholder* |
| `gerente@mahau.com.br` | manager (Gerente) — *placeholder* |
| `ygor@mahau.com.br` | promoter / Ygor — *placeholder* |
| `matheus@mahau.com.br` | promoter / Matheus — *placeholder* |
| `promoter@mahau.com.br` | promoter / Mahau — *placeholder* |

> São **6 contas** (2 admins: seu iCloud + um "socio" genérico). Ajuste à vontade.
> Os emails placeholder podem ser trocados pelos reais depois — atualize o `ROLES`
> em `set-claims.mjs` (ou crie um `roles.json`) e rode o Passo 5 de novo.
>
> **Rollback:** apagar os usuários. O app PIN antigo não usa Auth — sem impacto.

---

### Passo 4 — Baixar a chave de service account  ·  *reversível*  ⚠️ SEGREDO REAL
Necessária para o `set-claims.mjs` setar claims (Admin SDK, grátis no plano Spark).

1. Console → ⚙ ao lado de **Visão geral do projeto** → **Configurações do projeto**.
2. Aba **Contas de serviço**.
3. Botão **Gerar nova chave privada** → confirme em **Gerar chave**.
4. Baixa um JSON, ex.: `mahaucrm-firebase-adminsdk-xxxxx-yyyyyyyyyy.json`.
5. Coloque o arquivo **na pasta do repositório** (ou aponte a env `GOOGLE_APPLICATION_CREDENTIALS`
   para o caminho dele). O `set-claims.mjs` detecta automaticamente um `*-firebase-adminsdk-*.json`.
6. ⚠️ **NUNCA commite essa chave.** Já está coberta no `.gitignore`
   (`*-firebase-adminsdk-*.json`, `service-account-key.json`). Diferente da config web
   do Firebase (que é pública), **esta chave dá acesso administrativo total** ao projeto.

> **Rollback / higiene:** após terminar a migração, você pode **revogar** a chave em
> Contas de serviço → Gerenciar chaves, e/ou apagar o arquivo local.

---

### Passo 5 — Setar os custom claims  ·  *reversível*
No diretório do repo:
```bash
npm install firebase-admin
node set-claims.mjs --dry-run    # confere o mapeamento email -> claims, sem alterar
node set-claims.mjs              # aplica
```
O script valida os papéis, acha a chave, seta os claims e relê para confirmar.

- **Usuário não existe?** O script avisa — volte ao Passo 3 e crie a conta.
- **Token:** claims só entram no token após **deslogar/logar** (ou ~1h). Relevante no Passo 6.

> **Rollback:** rode com um `roles.json` setando `{}`/removendo, ou via Console/Admin SDK
> limpe os claims. O app PIN antigo não depende de claims.

---

### Passo 6 — Deploy do cliente novo (merge → main)  ·  *REVERSÍVEL (ponto-chave)*
Aqui o app passa a exigir login email/senha. As **rules ainda estão abertas** — então,
mesmo que algum claim esteja faltando, o Firestore não bloqueia (a UI mostra "Acesso
pendente" para quem não tem papel, mas não há perda de acesso aos dados).

```bash
git checkout main
git merge --no-ff feat/firebase-auth-rbac
git push origin main          # GitHub Pages publica o cliente novo
```
**Teste em produção (rules ainda abertas):**
- Login com cada conta → papel correto (Sócio vê tudo, Gerente cadastra, promoter só Brief).
- Logout (botão Sair) → volta ao login.
- Conta sem claim → tela "Acesso pendente".
- `recover-bday.html` → exige login antes de rodar.

> **Rollback (se o cliente novo quebrar):**
> ```bash
> git revert -m 1 HEAD        # desfaz o merge
> git push origin main
> ```
> Volta o `index.html` PIN antigo. Como **rules seguem abertas e PINs seguem no doc**,
> o app PIN volta a funcionar 100%. Investigue, corrija no branch, repita o Passo 6.

---

### Passo 7 — Aplicar as Security Rules  ·  *reversível (via backup do Passo 0)*
Só depois do Passo 6 validado. A partir daqui, o Firestore **exige** auth + claim correto.

**Opção A — Firebase CLI (usa os arquivos do repo):**
```bash
npm install -g firebase-tools
firebase login
firebase use mahaucrm
firebase deploy --only firestore:rules
```
**Opção B — Console:** Firestore Database → **Regras** → cole o conteúdo de
`firestore.rules` → **Publicar**.

**Teste pós-rules:**
- admin/gerente: leem e escrevem `crm`; leem/escrevem `bday_*`; backup (admin) funciona.
- promoter: vê o Brief (lê `crm`), **não** escreve, **não** acessa `bday_*`.
- aba sem login / coleção desconhecida: negado.

> **Rollback:** reaplique as rules salvas no Passo 0 (Console → Regras → cole o backup →
> Publicar), ou ajuste `firestore.rules` e re-deploy. O cliente novo continua funcionando
> com rules abertas.

---

### Passo 8 — Limpar os PINs legados  ·  ⚠️ *menos reversível — fazer por último*
Os PINs no doc `mahau/crm` já são **ignorados** pelo cliente novo (defense in depth),
mas continuam lá em texto plano. Remova-os:

1. Sirva o repo localmente (`python -m http.server`) ou abra via a URL do Pages.
2. Abra **`clean-pins.html`** (arquivo local, não versionado).
3. Faça login como **Sócio/Gerente**.
4. Ele lista os campos legados presentes. Digite **`LIMPAR`** e clique em
   **Remover PINs legados**. Remove `adminPin`, `managerPin`, `promoterPins`, `configured`
   via `deleteField()`. **Não toca** clientes, visitas, metas nem `waLogs`.

> **Por que por último:** após remover os PINs, voltar ao app PIN antigo não te deixaria
> entrar (não há mais PIN no doc). Só faça este passo com o caminho de Auth 100% validado.
> Mesmo assim, o cliente novo **não quebra** se restarem PINs — então não há pressa.

---

### Passo 9 — Higiene final  ·  *recomendado*
- Apague/arquive o backup `rules-backup-ANTES.txt` quando não precisar mais do rollback.
- **Revogue** a chave de service account (Passo 4) se não for reusá-la em breve, ou
  guarde-a fora do repo em local seguro. Confirme que ela **não** foi commitada:
  `git log --all --name-only | grep -i adminsdk` (deve não retornar nada).
- Considere apagar `recover-bday.html`/`clean-pins.html` locais (one-offs).

---

## Matriz de testes (resumo)

| Papel | Login | `crm` leitura | `crm` escrita | `bday_*` | Backup (coleção) | Brief |
|---|---|---|---|---|---|---|
| admin   | ✓ | ✓ | ✓ | ✓ | ✓ | ✓ |
| manager | ✓ | ✓ | ✓ | ✓ | ✓ (via UI admin-only) | ✓ |
| promoter| ✓ | ✓ | ✗ | ✗ | ✗ | ✓ (só Brief) |
| sem claim | ✓ login, "Acesso pendente" | ✗ | ✗ | ✗ | ✗ | ✗ |
| não autenticado | — | ✗ | ✗ | ✗ | ✗ | ✗ |

> Antes de aplicar em produção, vale validar a mesma matriz no **Firestore Emulator**
> (`firebase emulators:start --only firestore`) ou no **Rules Playground** do Console.

---

## Troubleshooting

- **Logou mas caiu em "Acesso pendente":** o token não tem o claim. Rode `node set-claims.mjs`
  e **deslogue/relogue**. Confira com `node set-claims.mjs --dry-run` (mostra os claims atuais).
- **"Missing or insufficient permissions" após o Passo 7:** o usuário não tem o claim
  esperado, ou está tentando algo fora do papel (ex.: promoter acessando `bday_*`). Verifique
  o claim e as rules.
- **`auth/user-not-found` no set-claims:** crie a conta no Console (Passo 3) antes.
- **Promoter entra mas o Brief não abre no nome certo:** o claim `promoter` precisa ser
  exatamente `Ygor`, `Matheus` ou `Mahau` (espelha `PROMOTERS` no `index.html`).
