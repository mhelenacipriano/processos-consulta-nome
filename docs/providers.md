# Provider feasibility matrix

Research date: **2026-10-23**. Every observation below comes from a live request made from a cloud
container (datacenter egress IP, through a TLS-terminating proxy). Results from a different network
may differ, especially the WAF blocks; they are re-testable with the commands in each row.

Status legend: **VERIFIED WORKING** · **REQUIRES CREDENTIALS** · **UNSUPPORTED** · **UNVERIFIED**.
"Blocked from test network" means UNVERIFIED: we did not prove the route is unusable for everyone, and we
did not try to evade the block.

## Matrix

| Provider | Process by party name | Jurisprudence by text/name | Process details | Notes |
|---|---|---|---|---|
| **TJPB** | **UNSUPPORTED** for automation (reCAPTCHA) | **UNVERIFIED** (site blocked from test network) | UNSUPPORTED (same CAPTCHA) | See TJPB below |
| **STJ** | **UNVERIFIED** (no public name search found; SCON blocked) | **VERIFIED WORKING** via open-data dump (offline index, not live search) | UNVERIFIED | See STJ below |
| **STF** | **UNVERIFIED** (blocked from test network) | **UNVERIFIED** (blocked from test network) | UNVERIFIED | See STF below |
| **DataJud (CNJ)** | **UNSUPPORTED** (no party names in documents) | UNSUPPORTED (no decision text) | **VERIFIED WORKING** by process number only (metadata) | See DataJud below |
| **Jusbrasil API** | **REQUIRES CREDENTIALS** (paid contract; "consulta processual por nome" documented) | UNVERIFIED (no jurisprudence product seen in docs) | REQUIRES CREDENTIALS | See Jusbrasil below |

## TJPB

- **Public PJe consulta pública (1º grau)**: `https://consultapublica.tjpb.jus.br/pje/ConsultaPublica/listView.seam`
  - Test: `curl -L` the URL, inspect the form. Returned HTTP 200 (JSF/RichFaces page, ViewState + jsessionid).
  - The form has a party-name field `fPP:dnp:nomeParte` (name search without CPF/number is a real feature of the UI).
  - The search button runs `onclick="return executarReCaptcha();..."`: **a reCAPTCHA gates the search**.
    Automating it would mean bypassing a CAPTCHA, which this project will not do. => **UNSUPPORTED** as a provider.
  - `robots.txt` on that host returned HTTP 520 (no usable policy).
- Other hosts: `www.tjpb.jus.br/jurisprudencia` → 403; `www.tjpb.jus.br/` → 403; `pje.tjpb.jus.br/...` → 302;
  `jurisprudencia.tjpb.jus.br` and `pje2g.tjpb.jus.br` → proxy 502 (unreachable from the test network).
  The TJPB jurisprudence search could therefore **not** be evaluated: **UNVERIFIED**. To be re-tested from another network
  (e.g. the user's machine) before any adapter is written.
- DataJud has a TJPB index (`api_publica_tjpb`) but see DataJud limits below.

## STJ

- **SCON (jurisprudence search)** `https://scon.stj.jus.br/SCON/`: returns an F5/WAF "Request Rejected" body
  (HTTP 200) for `/SCON/`, `/robots.txt` with browser UA, `curl/8` and a descriptive UA. `/SCON/pesquisar.jsp` → 302 to `http://scon.stj.jus.br/SCON/`.
  => live SCON search **UNVERIFIED / blocked from test network**. No evasion attempted.
- **Consulta processual** `https://processo.stj.jus.br/processo/pesquisa/` loads (HTTP 200); it is a number-oriented form. No name-based public search
  was confirmed. **UNVERIFIED**.
- **Open-data portal (CKAN)** `https://dadosabertos.web.stj.jus.br/` — **VERIFIED WORKING**.
  - API: `GET /api/3/action/package_search?q=jurisprudencia` → `success: true`, 11 datasets ("Espelhos de acórdãos" per órgão julgador).
  - `package_show?id=espelhos-de-acordaos-segunda-turma`: license **Creative Commons Atribuição**, 54 resources (monthly JSON + ZIPs), modified 2026-09-08.
  - Downloaded `20260831.json` (3.5 MB, HTTP 200): list of 1100 acórdão records with `numeroProcesso`, `numeroRegistro`, `siglaClasse`,
    `nomeOrgaoJulgador`, `ministroRelator`, `dataDecisao`, `ementa`, `decisao`, `dataPublicacao`.
  - Consequence: this is a **bulk dataset, not a live search API**. A name/keyword search requires downloading and indexing
    (e.g. PostgreSQL full-text) the dumps locally. Data is as recent as the last published dump (latest seen: 2026-08-31).
    Party names generally do not appear in ementas; hits are textual mentions only (`match_type = text_mention`).
    A sample search for "santiago" in the Aug-2026 Segunda Turma file returned 0 mentions (no assumption that results exist).
  - Attribution (CC-BY) must be shown in the UI.

## STF

- `https://jurisprudencia.stf.jus.br/pages/search` → HTTP 200 (SPA shell, 19 KB) at the first probe, but
  `/robots.txt`, `portal.stf.jus.br` and a `POST /api/search/search` all returned F5 **403 "requested URL was rejected"**;
  `GET /api/search/search` → 405.
- Whether this is an IP-reputation block of the cloud egress or a general policy is unknown. => **UNVERIFIED**; no provider created.
  Re-test from another network and read the page's own XHR calls (browser devtools) before writing an adapter.

## DataJud (CNJ public API)

- Docs: <https://datajud-wiki.cnj.jus.br/api-publica/> (+ `/acesso/`, `/termo-uso/`, `/endpoints/`).
- Auth: header `Authorization: APIKey <key>`. The key is **published in the wiki** and "may be changed by CNJ at any time" → read it from env (`DATAJUD_API_KEY`), do not hard-code.
  Without a key: HTTP 401.
- Endpoint per court: `https://api-publica.datajud.cnj.jus.br/api_publica_tjpb/_search` (Elasticsearch DSL). Also `api_publica_stj`; STF is not in the list seen.
- **Terms**: lawful, non-commercial use only; no redistribution/commercial exploitation of derived information; CNJ gives no guarantee of accuracy/currency.
- Observed (POST `_search`, `match` on `classe.codigo`): HTTP 200 but `took` 35–42 s; a second call returned `es_rejected_execution_exception` for one shard
  (partial result with `failed: 1`), another call timed out at 40 s. => slow, load-shedding: needs long timeouts, retries and **explicit partial-failure handling**.
- Fields seen in `_source`: `tribunal`, `numeroProcesso`, `dataAjuizamento`, `classe{codigo,nome}`, `assuntos[]`. **No party names** (wiki: "resguardo de processos sigilosos e dados de partes").
  => name-based party search is **UNSUPPORTED**. Useful only to enrich a CNJ number obtained elsewhere (only a small sample was inspected).

## Jusbrasil (official API)

- Docs: <https://api.jusbrasil.com.br/docs/index.html> (HTTP 200). Sections seen: authentication (API key), "Consulta processual por nome" (reports),
  "Distribuição de novos processos" (monitored parties), process monitoring, "Como contratar".
- Commercial/contracted product → **REQUIRES CREDENTIALS**; not tested live (no credentials). Optional adapter, env `JUSBRASIL_API_KEY`, disabled by default.
- Jurisprudence search: not found in the docs index → **UNVERIFIED**; not assumed.

## Decision

The honest MVP given these findings:

1. **STJ jurisprudence via the open-data dataset** (ingest + local full-text search) is the only verified, lawful, programmatic name/keyword route.
2. **TJPB, STF**: no lawful verified route from the test network. No fake providers; they appear in `/api/providers` as `unsupported` / `unverified` with the reason.
3. **DataJud**: optional enrichment by CNJ number only; never used for name search.
4. **Jusbrasil**: optional, credential-gated, off by default.
5. Next research step (needs a different network): re-test TJPB jurisprudence, STF jurisprudence and STJ SCON from an unblocked IP.
