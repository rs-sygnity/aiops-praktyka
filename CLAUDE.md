# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Przeznaczenie repo

Repo ćwiczeniowe szkolenia Sages „AIOps w praktyce”. Uczestnicy używają narzędzi AI do diagnozy, konfiguracji
i obsługi incydentów w **Kantynie** (system zamawiania obiadów w firmowej stołówce, EKS). Dokumentacja, laby
i runbooki są po polsku — zachowuj ten język.

- `app/` — kod Kantyny, chart Helm, runbooki
- `labs/` — instrukcje labów (cel, kroki, warunek SUKCES); `docs/` — materiały tematyczne i `docs/prompts/` (szablony promptów)
- `setup/` — przygotowanie stanowiska; `setup/check.sh` sprawdza Kantynę uczestnika (oczekiwane `SUKCES: 11/11`)

## Komendy (z katalogu `app/`)

- Testy: `make test` · `make test-orders-api` (`cd orders-api && uv run pytest -q`) · `make test-payments` (go test w kontenerze golang:1.23)
- Pojedynczy test: `cd orders-api && uv run pytest tests/test_api.py::nazwa_testu -q`
- Lokalnie: `docker compose up -d --build` (UI :8088, Swagger :8000/docs), `docker compose down -v`, ruch: `--profile loadgen`
- Obrazy: `make build push VERSION=x [VARIANT=receipt-v2|menu-v2]`, `make ecr-login`
- Helm: `make helm-lint`, `make helm-template`, `helm upgrade --install kantyna deploy/helm/kantyna -n <ns> ...`, wycofanie: `helm rollback kantyna -n <ns>`

## Architektura

web (nginx, proxy `/api/*`) → orders-api (FastAPI) → payments (Go, `POST /pay`) i PostgreSQL; orders-api publikuje
zdarzenia do SQS → worker je odbiera, zapisuje paragony w S3 (po 5 nieudanych próbach wiadomość trafia do DLQ).
loadgen generuje ruch syntetyczny o dobowym profilu (Europe/Warsaw). Dostęp do SQS/S3 przez EKS Pod Identity.

Observability: Prometheus (ServiceMonitor + PrometheusRule), logi JSON do Loki (z `trace_id`), trace'y OTLP do Tempo.
Każdy z 7 alertów ma runbook `app/runbooks/<NazwaAlertu>.md` (URL w adnotacji `runbook_url`).

**Flagi runtime** (`flags.json`): w klastrze ConfigMap `kantyna-flags` (blok `flags:` w values Helm), lokalnie
`app/deploy/local/flags.json`. Serwisy czytają plik ponownie co ~15 s, bez restartu. Sekcje per serwis
(orders-api, payments, worker) wstrzykują awarie (error_rate, memory_leak_mb_per_min, db_pool_leak,
slow_menu_query, latency_ms, fail_rate…) — na tym opierają się incydenty w labach. Każdy serwis ma własny
`flags.py` (zmienne środowiskowe jako fallback; wartości z pliku mają pierwszeństwo).

## Wersje narzędzi
- kubectl
```
clientVersion:
  buildDate: "2026-09-23T17:10:04Z"
  compiler: gc
```
- helm
```
v3.22.0+g144ca65
```
- git 
```
git version 2.53.0
```

## Zasady i pułapki

- `.claude/settings.json`: `kubectl get/describe/logs/top` i `k8sgpt analyze` bez pytania; zmiany (`set/patch/edit/
  rollout/apply`) wymagają zgody; `delete`, `exec` i odczyt sekretów są zablokowane — nie obchodź tego.
- `app/.env` zawiera **fałszywe** sekrety (do ćwiczeń). Do modelu nie wysyłaj prawdziwych haseł ani kluczy.
- Klaster jest wspólny — pracuj tylko w swoim namespace. Najpierw dowód (komenda potwierdzająca hipotezę), potem
  zmiana; jedna zmiana naraz.
- Makefile: `SERVICES` = orders-api payments worker web loadgen; `orders-api/Dockerfile.legacy` to build legacy (`make legacy-build`).
- Laby 03–07 z głównego README pojawiają się w repo kolejnymi dniami (`git pull upstream main`).
- Nie uruchamiaj polceń potencjalnie niebezpiecznych np `kubectl delete`
- komenda, którą mogę sprawdzić zmiany: `helm lint app/deploy/helm/kantyna`

