# lab01b — notatki do części A

## Trzy różnice między `zly.yaml` a `dobry.yaml`

1. **Obraz.** `zly.yaml` ma `kantyna/orders-api:1.0.0`: tag wymyślony przez model, bez rejestru. `dobry.yaml` ma
   `123456789012.dkr.ecr.eu-central-1.amazonaws.com/kantyna/orders-api:0.0.0-dev`, czyli wartość z `values.yaml` i
   szablonu chartu.
2. **Pełna konfiguracja z chartu.** `dobry.yaml` ma `serviceAccountName: kantyna-orders-api` (dostęp do SQS/S3 przez
   Pod Identity), wolumen `flags` z ConfigMap `kantyna-flags` oraz zmienne `FLAGS_FILE`, `QUEUE_URL`,
   `RECEIPTS_BUCKET`, `AWS_REGION`, `OTEL_*`. W `zly.yaml` ich nie ma, więc aplikacja nie dostałaby uprawnień do AWS
   ani flag runtime.
3. **Etykiety i namespace.** `zly.yaml` używa etykiet `app: orders-api` i nie ustawia namespace'a (nazwa `orders-api`
   może kolidować z istniejącym zasobem). `dobry.yaml` używa etykiet `app.kubernetes.io/*` jak w chartcie, nazwy
   `orders-api-lab`, namespace'a `robert` i innych etykiet selektora niż chart, więc nie przejmuje podów
   `kantyna-orders-api`.

Dodatkowo: `zly.yaml` ma 3 repliki i jawną strategię rolling update, `dobry.yaml` 2 repliki jak w chartcie.

## Czego walidator nie sprawdził

`--dry-run=server` sprawdza schemat i uprawnienia, a nie to, czy odwołania wskazują na istniejące rzeczy. Przy
`zly.yaml` (`orders-api created (server dry run)`) przeszedł plik, w którym:

- obraz `kantyna/orders-api:1.0.0` prawdopodobnie nie istnieje (pody skończyłyby w `ImagePullBackOff`),
- Secret `kantyna-db`, usługi `kantyna-payments:8080` i `kantyna-postgres:5432` to nazwy zgadnięte z założenia, że
  release nazywa się `kantyna`; walidator nie sprawdza, czy takie zasoby są w namespace.

Wniosek: przejście walidacji nie znaczy, że Deployment zadziała. Nie zweryfikowałem tego na klastrze, bo labowi
wolno tylko `--dry-run`.

# Część B — diagnoza z `awaria-zred.txt`

## Redakcja sekretów (B1)

Wzorzec z README (`(password|token|secret)=\S+`) nie zamaskował hasła zapisanego w URL-u bazy
(`postgresql://kantyna:<hasło>@kantyna-postgres...`, wiersz 20). Wykryłem to dopiero przy czytaniu pliku przed
diagnozą. Hasło zamaskowałem dodatkowym wzorcem `://user:hasło@` i sprawdziłem, że w pliku nie zostały hasła ani
tokeny. Wniosek: sprawdzaj wynik redakcji osobnym `grep`em po innych kształtach sekretów, nie tylko tym, który
sam napisałeś.

## Wynik diagnozy (B2)

Hipotezy od najbardziej prawdopodobnej:
1. PV bazy leży w innej strefie (AZ) niż wszystkie nowe węzły (wszystkie w `eu-central-1a`).
2. Przyczyna hipotezy 1: nowa node group ma tylko jedną podsieć/AZ.
3. Affinity PV odwołuje się do innej etykiety niż strefa (np. typ instancji lub hostname starego węzła).

Łańcuch awarii: `kantyna-postgres-0` Pending → brak endpointów `kantyna-postgres` → orders-api kończy startup po
5 próbach (`CrashLoopBackOff`) → 502 w web. orders-api jest skutkiem, nie przyczyną.

## Sprawdzenie dowodów (B3)

Cztery cytowane fragmenty znalazłem w pliku: `volume node affinity conflict` (wiersze 49–50), `eu-central-1a`
(58–60), `Bound` (54), `attempt 5/5 failed` (27). Nie znalazłem zmyślonego dowodu.

**Wniosek bez pokrycia w danych:** strefa PV. W danych jej nie ma. Hipoteza „PV w innej AZ” to dedukcja z komunikatu
o konflikcie affinity, a nie cytat. Rozstrzygnie to dopiero:
`kubectl get pv pvc-3e8a91c4-27d5-4b0f-9c6e-5a1f0d7b2e48 -o jsonpath='{.spec.nodeAffinity}'`.
Podobnie brakuje listy podsieci node group, więc hipoteza 2 jest niepotwierdzona.

## Do uzupełnienia po B4

- Czy po `/clear` kolejność hipotez i dowody są takie same?
