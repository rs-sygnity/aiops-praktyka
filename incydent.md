# Dziennik incydentu — lab07 (namespace `robert`)

**Zgłoszenie:** menu ładuje się bardzo wolno, czasem błąd.
**Start (wykrycie):** 2026-10-07 ok. 15:33 CEST

## 1. Wykrycie

- `setup/check.sh` → **8/11**: `kantyna-orders-api` gotowe 0/2; „menu przez web” i „zamówienie przez web” nie przechodzą.
- Oba pody orders-api: `Running`, `0/1 Ready`, 0 restartów, wiek 26 h.
- Loki: `Readiness check failed … couldn't get a connection after 30.00 sec` (orders-api, co ok. 5 s); loadgen: `ConnectError: All connection attempts failed`.
- Najwcześniejszy wpis „Readiness check failed” w Loki: 2026-10-07 13:25:28 UTC (15:25 CEST). Prometheus: `kube_pod_status_ready` dla obu podów orders-api = 1 do ok. 13:19 UTC (15:19 CEST), a 0 w próbce 13:38 UTC, więc pody wyszły z gotowości ok. 15:25 CEST, a nie 26 h temu (wiek podów to nie czas awarii). Dokładną minutę trzeba doprecyzować.
- Prometheus (`namespace="robert"`, rate 5 min): 200 ≈ 0,86 rps, 201 ≈ 0,12 rps, **503 ≈ 0,22 rps**, 500 ≈ 0,02 rps, czyli ok. 20% odpowiedzi to błędy. p95 czasu odpowiedzi = **10 s** (górna granica histogramu, więc realnie ≥10 s) dla `/api/menu`, `/api/menu/search`, `/api/orders`, `/api/orders/{id}`, `/readyz`; `/healthz` i `/metrics` ≈ 5 ms.
- Dashboard `robert-kantyna-demo` pokazuje „brak problemów”, bo wszystkie zapytania mają `namespace="demo"` zamiast `robert`, więc nie widzi tego namespace'u (do poprawy).

## 2. Pierwszy komunikat

Odbiorca: zespoły. Wersja do wysłania na kanał (szkic AI, zgodny z `docs/prompts/komunikat-incydentu.md`; godzina następnej aktualizacji wybrana przez mnie):

> **[Incydent] Kantyna: menu i zamówienia działają wolno lub zwracają błędy**
>
> 1. Od około 15:25 część użytkowników Kantyny widzi bardzo wolne ładowanie menu, a część zapytań kończy się błędem.
> 2. Pierwsze objawy zaobserwowaliśmy ok. 15:25; zgłoszenie potwierdziliśmy o 15:33.
> 3. Z naszych pomiarów ok. 1 na 5 odpowiedzi serwisu zamówień kończy się błędem, a pozostałe mogą trwać 10 sekund lub dłużej; składanie zamówień i przeglądanie menu jest utrudnione.
> 4. Badamy przyczynę i przygotowujemy przywrócenie normalnego działania; na tym etapie nie znamy jeszcze przyczyny.
> 5. Następną aktualizację podamy do 16:15, a wcześniej, jeśli sytuacja się zmieni.

Źródła liczb: Prometheus (503 ≈ 0,22 rps + 500 ≈ 0,02 rps, średnia z 5 min; p95 = 10 s to górna granica histogramu). Początek ok. 15:25 z Loki i `kube_pod_status_ready`.

## 3. Diagnoza: hipotezy i dowody

| # | Hipoteza | Dowód (komenda / zapytanie → wynik) | Status |
|---|---|---|---|
| 1 | Pula połączeń DB orders-api wyczerpana przez flagę `db_pool_leak` (ścieżka `/api/menu`, `app/orders-api/app/main.py:147`, zabiera połączenie z p=0,3 i go nie zwraca) | Prometheus `max by (pod) (db_pool_connections_in_use{namespace="robert"}) / max by (pod) (db_pool_size{namespace="robert"})` → 1 na obu podach od ok. 15:27 (wcześniej 0); Loki `{namespace="robert"} \|~ "(?i)error\|warn"` → `couldn't get a connection after 30.00 sec`; `kubectl -n robert describe pod` → `Readiness probe failed … context deadline exceeded` (x185; `/readyz` robi `db.ping()` i czeka na pulę dłużej niż timeout probe 3 s) | potwierdzona |
| 2 | Flagi włączone ręcznie na klastrze, a nie z repo (drift) | `kubectl -n robert get cm kantyna-flags -o jsonpath='{.metadata.managedFields}'` → `kubectl-patch Update 2026-10-07T13:10:17Z` (15:10 CEST); `values.yaml` w repo: `db_pool_leak: false`, `slow_menu_query: false`; `helm get values kantyna` nie zawiera `flags`; `helm history`: rev. 5 (15:24) nie cofnął flag. Kolejność: patch 15:10 → pula zapełnia się losowo (30% `/api/menu`) → readiness pada ok. 15:25. Kto patchował: nieustalone | potwierdzona (autor zmiany nieznany) |
| 3 | `slow_menu_query: true` (N+1 w `/api/menu`, `app/orders-api/app/runtime.py:53`): druga, niezależna przyczyna wolnego menu | `kubectl -n robert get cm kantyna-flags -o jsonpath='{.data}'` → flaga `true`; brak pomiaru średniej latencji menu, bo ruch blokuje pula | flaga potwierdzona, wpływ na latencję do zmierzenia po mitygacji |
| (wykluczone) | NetworkPolicy blokuje ruch | `kubectl -n robert get networkpolicy -o yaml` → `kantyna-orders-api-ingress` (z web), `kantyna-postgres-ingress` (z orders-api), `lab07-loadgen-i-monitoring` (loadgen, monitoring) pozwalają na potrzebny ruch; `/healthz` odpowiada | wykluczone |
| (wykluczone) | OOM / CPU | `kubectl -n robert top pods` → ~65 Mi, 2 m CPU, 0 restartów | wykluczone |

Pozostałe ustalenia z objawów (punkt 1): dashboard `robert-kantyna-demo` miał zapytania na `namespace="demo"` (nowy: `robert-kantyna-robert`); 503 ≈ 0,22 rps, p95 = 10 s (górna granica histogramu).

## 4. Mitygacja i naprawa

### 4.1 Mitygacja (zrobiona; przyczyna w repo i dryf flag zostają do naprawy w 4.2)

| Godzina | Zmiana | Komenda | Wynik |
|---|---|---|---|
| 15:51 | Zmiana 1: `db_pool_leak` i `slow_menu_query` → `false` w ConfigMapie (reszta flag bez zmian; kopia poprzedniej wartości w scratchpadzie sesji) | `kubectl -n robert patch cm kantyna-flags --type merge --patch-file …` | ConfigMap zawiera `false`/`false`; pody nadal `0/1` (wyciekniętych połączeń zmiana flagi nie zwalnia) |
| 15:53 | Zmiana 2: restart orders-api | `kubectl -n robert rollout restart deploy/kantyna-orders-api` | `successfully rolled out`, nowe pody `1/1` po ok. 15 s |
| 15:53 | Sprawdzenie | `setup/check.sh`; Prometheus `max by (pod) (db_pool_connections_in_use{namespace="robert"})` | **11/11**; nowe pody: pula 0/10, stare (kończące się): 10/10 |

Usługa przywrócona ok. 15:53 (czas trwania incydentu od ok. 15:25: ok. 28 min). Metryki ustabilizowane jeszcze niezweryfikowane (punkt 5).

**Uwaga:** ręczny patch ConfigMapy zrobił z `kubectl-patch` właściciela pól flag. Repo ma `false`, więc teraz klaster zgadza się z repo, ale ręczna zmiana nie jest śladem w gicie.

### 4.2 Naprawa commitem

Ustalenie: w repo nie było błędu (repo i release Helm miały `db_pool_leak: false`, `slow_menu_query: false`). Przyczyną był ręczny `kubectl patch` flag o 15:10, poza repo, a awaria nie wywołała alertu, bo `prometheusRule.enabled: false` (jawnie zapisane w wartościach release'u, więc zmiana domyślnej wartości w `values.yaml` przy `--reuse-values` nic by nie dała). Commit nie usuwa przyczyny (ją zamknęła mitygacja 4.1), tylko sprawia, że podobna awaria zostanie wykryta alertem.

Zmiana (`git diff` pokazany i zaakceptowany przed commitem):
- `.github/workflows/ci.yml`: `helm upgrade … --set prometheusRule.enabled=true`
- `app/deploy/helm/kantyna/values.yaml`: `runbookBaseUrl` → `https://github.com/rs-sygnity/aiops-praktyka/blob/main/app/runbooks` (poprzedni adres wskazywał na nieistniejące repo)

- Commit: `248a4a3` (`ci: włącz PrometheusRule przy deployu i popraw runbookBaseUrl (lab07)`)
- Przebieg pipeline'u: https://github.com/rs-sygnity/aiops-praktyka/actions/runs/37632964919 (**zielony**, w tym „Wdrożenie (helm)”)
- Efekt na klastrze (16:01): `helm history` rev. 6 `deployed` (14:00:45 UTC); PrometheusRule `kantyna` istnieje z 7 alertami, w tym `KantynaDBPoolExhausted`; flagi nadal `false`/`false`; pody orders-api `1/1`; `setup/check.sh` → 11/11.
- Bez `--force-conflicts`, konfliktu z `kubectl-patch` nie było (wartości flag w repo i klastrze są identyczne).

## 5. Weryfikacja

_Jeszcze niezrobione._ Cel: `setup/check.sh` → 11/11 i metryki stabilne przez 5 min.

## 6. Post-mortem

_Jeszcze niezrobione._
