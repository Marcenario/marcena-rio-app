# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project shape

Planeja.ai is a single-file PWA for managing planned-furniture (móveis planejados) sales, production, delivery, and aftermarket. The entire frontend lives in `index.html` (~15k lines, ~870 KB) — HTML, CSS, and JS are all inlined. There is **no build step, no bundler, no package manager, no test suite, no linter**. You edit `index.html` directly and reload.

Backing services:
- **Supabase** (project `gnjykqkqtcqqdadhyvcw`) — Postgres tables, Auth, Storage, and Edge Functions. URL and anon key are hardcoded near the top of the `<script>` block (around line 2143). Loaded via CDN: `<script src="https://cdn.jsdelivr.net/npm/@supabase/supabase-js@2">`.
- **PWA shell** — `manifest.json` + `service-worker.js` (stale-while-revalidate over GET).
- **Android TWA** — built by GitHub Actions, points at the production URL `https://appplanejaai.com.br`.

## Deploy & build (GitHub Actions on push to `main`)

- `.github/workflows/main.yml` — FTP deploys the whole repo to `/public_html/app/` on `191.252.83.224` (Localweb). Secrets: `FTP_PASS`. There is no staging — `main` is production.
- `.github/workflows/build-apk.yml` — generates an Android TWA project on-the-fly, signs with a keystore created from `KEY_PASSWORD`, and uploads the APK as a build artifact. The TWA wraps `https://appplanejaai.com.br`, so the APK only works once the web deploy has shipped the matching `index.html`.

There is no local dev server config. To test locally, serve the directory with any static server (e.g. `python3 -m http.server`) and open `index.html`; auth and DB calls go straight to the live Supabase project.

## Runtime architecture

### Single-page SPA driven by `showSection`
The body contains many `<div class="section" id="...">` blocks (`solicitar`, `projetos`, `atendimento`, `conferencia`, `montagem`, `ocorrencias`, `vistoria`, `rh`, `financeiro`, `agenda`, `marketing`, `dashboard`, `admin`, `usuarios`). `showSection(id, btn)` hides all and shows one. Nav buttons in `.nav-bar` are toggled visible/hidden in `onLogin()` based on user role.

Several sections have their own sub-nav (`showSubMontagem`, `showSubRH`, `showSubFin`, etc.) — same pattern, scoped to the parent section.

### Auth and role model
- `currentUser` and `userNivel` are module-level globals set in `onLogin(user)`.
- Roles: `'admin' | 'gerente' | 'funcionario' | 'cliente'`. Source of truth is `perfis.nivel` keyed by `auth.users.id`, with the constant `ADMIN_EMAIL='contato@marcenario.com.br'` forced to `'admin'` regardless of DB row.
- A `funcionario` whose `rh_funcionarios.ativo === false` is silently downgraded to `cliente` (so terminated staff retain only client-side access).
- Use the helpers, do not compare `userNivel` directly: `isAdmin()`, `isGerente()`, `isAdmOuGer()`, `isStaffGeral()` (admin+gerente+funcionario). New gated UI should follow the same pattern.

### Multi-tenant via `EMPRESAS`
The app serves two brands ("marcenario" / Marcena Rio and "conceptus" / Italínea Conceptus), loaded from the `empresas` table by `carregarEmpresas()` and cached in the `EMPRESAS` map keyed by `slug`. Most records (`projetos`, `atendimentos`, etc.) carry an `empresa` slug column. Always go through `nomeEmpresa(slug)` / `corEmpresa(slug)` rather than hardcoding the two brands — the seed defaults are only a fallback.

### The 23-stage project pipeline (`ETAPAS`)
`const ETAPAS = [...]` (around line 2146) defines the ordered lifecycle a `projetos` row moves through via `etapa_atual` (1-indexed). Each stage declares:
- `campo` — what kind of artifact unlocks the stage (`'data'`, `'arquivo_simples'`, `'assinatura'`, `'data_producao'`, or `null`)
- `key` / `keyHorario` — the column on `projetos` where the artifact's date/URL/time is stored

`avancarEtapa(id, novaEtapa)` is the only sanctioned way to move a project forward/back; it also fires the `notificar-etapa` Edge Function. When adding a stage, you must update both `ETAPAS` and any switch/if chain that branches on the stage index (search for `etapa_atual` and `ETAPAS_TOTAL`).

### Supabase surface area
- ~40 tables. Frequently touched: `projetos`, `atendimentos`, `pedidos_montagem`, `at_tarefas`, `rh_funcionarios`, `rh_pontos`, `fin_despesas`, `fin_receitas`, `funis`/`funis_etapas`, `perfis`, `empresas`, `solicitacoes_empresa`, `solicitacoes_equipe`.
- Edge Functions invoked via `fetch(\`${SUPABASE_URL}/functions/v1/<name>\`)` with the anon key as bearer. Current set: `admin-excluir-usuario`, `assinar-cartao-ponto`, `assinar-projeto-atendimento`, `assinar-relatorio`, `criar-funcionario-rh`, `email-aniversario`, `email-boas-vindas`, `email-nutricao`, `email-reuniao`, `gerar-posts-marketing`, `notificar-etapa`, `sincronizar-assinatura`, `sincronizar-assinatura-projetos`, `sincronizar-cartao-ponto`, `whatsapp-nutricao`. **Edge Function source is not in this repo** — it lives in the Supabase project. Treat function names and request shapes here as the contract.
- Storage buckets in use: `projetos` (logos, project PDFs) and `arquivos` (manuals at `arquivos/manuais/manual_<empresa>.pdf`).

### Background notifications
`iniciarAvisosTarefas()` (started only for staff in `onLogin`) polls `at_tarefas` on an interval and renders cards into `#painel-avisos`. The interval handle is held in `_avisoInterval`; clear it on logout if you add new lifecycle hooks.

## Editing conventions (from the existing code)

- All user-facing strings are pt-BR. Keep that.
- The codebase is single-file by design — do not split `index.html` into modules without an explicit ask. The FTP deploy and the TWA both expect this exact file at the root.
- New DB-bound features should be wrapped in role checks via `isStaffGeral()` / `isAdmOuGer()` / etc., matching the way `onLogin` toggles nav visibility. Server-side RLS is assumed but the UI is the first gate.
- When you add a column read/write, the table almost certainly already exists — grep `.from('<table>')` before assuming you need to create one.
- `notify(msg, type)` is the toast helper; `setAlert(id, msg, type)` writes into a placeholder div. Prefer these over `alert()`.
