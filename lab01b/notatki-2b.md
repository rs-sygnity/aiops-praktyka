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

# Część C — komenda `/diagnoza`

## C1: własna komenda

Zapisana w `.claude/commands/diagnoza.md` (katalog główny repo). Oparta na szablonie `docs/prompts/diagnoza.md`:
- w miejscu objawu jest `$ARGUMENTS`, na końcu format tabeli `hipoteza | dowód (cytat) | komenda weryfikująca`,
- dopisana linijka z README: „Zbierz dane komendami read-only (kubectl get/describe/logs). Zredaguj sekrety przed
  analizą.”,
- poprawki z części B: redakcja obejmuje też hasła w URL-ach (`://user:hasło@host`), agent czyta tylko wskazany plik
  zredagowany, a wnioski będące dedukcją mają być oznaczone (strefa PV w B3 była dedukcją, nie cytatem),
- frontmatter z `description`, żeby komenda miała opis w podpowiedziach.

Komendy są ładowane przy starcie sesji, więc do testu potrzebna jest nowa sesja `claude`.

## C2: test (do uzupełnienia)

Polecenie testowe, tylko na pliku, nie na własnym namespace:

`/diagnoza kantyna-postgres-0 Pending, dane w lab01b/awaria-zred.txt`

- Czy komenda pojawia się w podpowiedziach i uruchamia się?
- Czy agent pracuje na `lab01b/awaria-zred.txt`, a nie na `labs/lab01b-prompty/dane/awaria.txt`?
- Czy odpowiedź ma tabelę z cytatami, które da się znaleźć przez `grep -n`?
