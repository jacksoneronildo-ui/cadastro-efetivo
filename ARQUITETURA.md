# Cadastro de Efetivo — 9ª CIPM

App de página única (`index.html`), hospedado no GitHub Pages, sem build/bundler.
Toda a lógica roda no navegador do usuário e fala direto com o Firebase (Firestore + Authentication).

- **Repositório**: https://github.com/jacksoneronildo-ui/cadastro-efetivo
- **Link publicado**: https://jacksoneronildo-ui.github.io/cadastro-efetivo/
- **Arquivo principal**: `index.html` (contém HTML + CSS + JS, tudo inline)
- **Projeto Firebase**: `cadastro-efetivo-9cipm-d59a2`

## Stack

- Firebase Firestore (compat SDK) — banco de dados
- Firebase Authentication (compat SDK, e-mail/senha) — login
- jsPDF (via CDN, cdnjs) — geração de relatórios em PDF
- SheetJS/xlsx (via CDN, cdnjs) — exportação em Excel
- Sem framework, sem build step — editar o `index.html` direto

## Armazenamento (Firestore)

### Coleção `kv` (chave-valor genérica)

Cada documento: `{ value: "<JSON string>", updatedAt: timestamp }`. O campo `value` sempre precisa
de `JSON.parse()`/`JSON.stringify()` na aplicação (funções `storageGet`/`storageSet`).

Chaves usadas:

- `launches:<YYYY-MM-DD>` → array de lançamentos diários daquela data
- `event-launches:<YYYY-MM-DD>` → array de lançamentos de evento/operação daquela data
- `event-index` → array de nomes de eventos já usados (autocomplete)
- `officer-index` → array `[{ nome, re, patente, tel }]` (autocomplete de policiais)
- `vehicle-index` → array `[{ placa, modelo, pat }]` (autocomplete de viaturas)
- `command-static` → `{ comando, subcomando }` (fixo, não muda por data)
- `edit-lock-hour` → number (0-23), horário do dia seguinte até quando dá pra editar/remover
- `active-window-hours` → number (0-24), horas após o término em que a equipe/posto ainda conta como ativa

### Coleção `permissions`

Doc ID = e-mail do usuário (minúsculo). Campos: `{ nome, nivel, criadoPor, criadoEm }`.
`nivel` é `'admin' | 'reduzido' | 'geral'`.

## Autenticação e permissões

- Login por e-mail/senha (Firebase Auth). Sessão persiste no navegador.
- Bootstrapping: o primeiro admin precisa ser criado manualmente (Auth + doc em `permissions`).
- Para admin/reduzido criarem outros usuários **sem se deslogar**, o app usa uma instância
  secundária do Firebase (`getSecondaryAuth()`) — cria a conta ali e desloga só dela.
- Qualquer usuário logado pode trocar a própria senha (reautentica com senha atual + `updatePassword`).

### Matriz de permissões

| Ação | Admin | Reduzido | Geral |
|---|---|---|---|
| Cadastrar lançamento | ✅ | ✅ | ✅ |
| Editar (dentro do prazo) | ✅ | ✅ | ✅ |
| Editar/remover fora do prazo | ✅ (sem trava) | ❌ | ❌ |
| Remover cadastro | ✅ | ✅ | ❌ |
| Ver relatórios/quantitativo/busca/PDF/Excel | ✅ | ✅ | ✅ |
| Editar Comando/Subcomando | ✅ | ✅ | ❌ |
| Mudar horário limite / janela de atividade | ✅ | ❌ | ❌ |
| Alternar status Ativa/Encerrada de equipe/posto | ✅ | ✅ | ❌ (só visualiza) |
| Cadastrar usuário nível admin/reduzido | ✅ | ❌ | ❌ |
| Cadastrar usuário nível geral | ✅ | ✅ | ❌ |
| Ver aba Auditoria | ✅ | ❌ | ❌ |

Funções helper: `isAdmin()`, `isReduzido()`, `isGeral()`, `canRemoveLaunch()`, `canEditComando()`,
`canChangeLockHour()`, `canManageUsers()`, `canToggleAtivo()`, `creatableLevels()`.

## Regras de segurança do Firestore (já publicadas)

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /kv/{document} {
      allow read, write: if request.auth != null;
    }
    match /permissions/{email} {
      allow read: if request.auth != null;
      allow write: if request.auth != null &&
        exists(/databases/$(database)/documents/permissions/$(request.auth.token.email)) &&
        get(/databases/$(database)/documents/permissions/$(request.auth.token.email)).data.nivel in ['admin', 'reduzido'];
    }
  }
}
```

## Modelo de dados: objeto "launch" (lançamento)

```js
{
  id, date,               // "YYYY-MM-DD"
  evento,                 // só em lançamentos de evento/operação
  inicio, fim,             // "HH:MM"
  gt, cidade, codigo,      // cabeçalho do serviço
  efetivo: [{
    patente, re, nome, tel, sexo, motorista,
    viatura: null | { modelo, km, pat, placa, armadoCartao },
    observacao
  }],
  viatura: { modelo, km, pat, placa, armadoCartao }, // viatura geral (4 rodas)
  gcoi, gcoiAt,
  ativo,                   // true/false — status manual de equipe/posto ativo
  ativoAlteradoPor, ativoAlteradoEm,
  criadoPor, createdAt,
  edited, editedAt, editadoPor,
  original,                 // snapshot pré-edição (só na 1ª edição)
  removed, removedAt, removedReason, removidoPor
}
```

## Abas do app (todas dentro de um único `#app`)

1. **Lançamento diário** — cadastro
2. **Lan. em Evento/ Operação** — cadastro (mesmo formulário + campo Evento/Operação)
3. **Relatório de serviços diários** — consulta por data, PDF, Excel, card "Serviço diário" (comando)
4. **Relatório de Evento/ Operação** — consulta por data, agrupado por evento
5. **Quantitativo de Lançamentos** — soma por posto/graduação, viaturas, equipes/postos (com contagem de "ativos")
6. **Buscar Policial** — busca por nome/matrícula num intervalo de datas
7. **Usuários** (admin/reduzido) — cadastro de acesso (nome, e-mail, senha inicial, nível)
8. **Equipes Ativas** (admin/reduzido) — lista o que está `ativo:true` hoje/ontem, com botão de encerrar
9. **Auditoria** (admin) — quem fez o quê (cadastro/edição/remoção/status), com export PDF/Excel

## Funcionalidades importantes

- **Trava de edição por horário** (`isLaunchEditable`): lançamento só editável/removível até
  `edit-lock-hour` do dia seguinte à data do serviço. Admin ignora a trava.
- **Janela de atividade** (`computeServiceWindow`, `isWithinActiveWindow`): considera o serviço
  "possivelmente ativo" até `activeWindowHours` depois do horário de término previsto (trata virada
  de meia-noite quando `fim < inicio`).
- **Conflito de horário** (`detectScheduleConflicts`): mesmo policial em 2+ lançamentos do mesmo dia
  com horários que se cruzam (ignora lançamentos com `ativo === false`). Nome fica destacado em
  vermelho no card e aparece aviso no topo do relatório diário.
- **Autocompletar**: nome de policial (preenche patente/matrícula/telefone), placa de viatura
  (preenche modelo/PAT), nome de evento — tudo via `<datalist>` alimentado pelos índices no Firestore.
- **Compartilhar WhatsApp** (`shareWhatsApp`): gera texto no formato do boletim usado pela unidade
  (🚨 CADASTRO DE EFETIVO, EFETIVO, VIATURA, GCOI ✅/❌, CMD/SUBCMD, lema) e abre `wa.me`.
- **Cache local de emergência**: `storageGet`/`storageSet` gravam uma cópia em `localStorage` do
  navegador; só é usada se o Firestore estiver genuinamente inacessível (falha de rede), nunca quando
  o servidor responde "não encontrado" de propósito.
- **Persistência offline do Firestore** ativada (`db.enablePersistence`).

## Coisas para ter cuidado ao mexer

- O arquivo publicado no GitHub **precisa se chamar `index.html`** (GitHub Pages).
- `firebaseConfig` está hardcoded no arquivo (é público por natureza do Firebase Web SDK — a
  segurança real vem das regras do Firestore, não do sigilo dessa config).
- Sempre validar a sintaxe do JS antes de publicar (o `<script>` é um bloco só, um erro quebra o app inteiro).
- Testar mudanças relacionadas a permissão logando com usuários de cada nível (admin/reduzido/geral).
- Se mudar a estrutura de alguma chave do `kv`, dados antigos gravados no formato anterior ficam
  "órfãos" — não há migração automática.
