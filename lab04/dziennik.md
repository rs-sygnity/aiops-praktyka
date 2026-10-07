# Dziennik — lab04

| # | Bramka / krok | Co złapała (albo co poprawiłem sam) | Poprawka |
|---|---|---|---|
| 1 | `kubectl apply --dry-run=server -f k8s/` | Nic: serwer przyjął PDB `kantyna-orders-api` i NetworkPolicy `kantyna-postgres-ingress` (`created (server dry run)`). | Brak, pliki poprawne za pierwszym razem. |
| 2 | `trivy config k8s/` | Nic: 0 błędów konfiguracji w obu plikach (przy pierwszym uruchomieniu pobrał tylko paczkę reguł). | Brak. |
| 3 | `kubectl diff -f k8s/` | Dokładnie dwa nowe zasoby (`@@ -0,0 +…`), żaden istniejący obiekt się nie zmienia. Blok `status` w PDB (`expectedPods: 0`) to artefakt podglądu, po `apply` wartości są prawdziwe. | Brak. |
| 4 | `kubectl apply` + `setup/check.sh` | `SUKCES: 11/11`. PDB ma `ALLOWED DISRUPTIONS = 1`. Do Postgresa łączy się tylko orders-api (sprawdzone w chartcie), więc polityka niczego nie odcięła. | Brak. |
| 5 | `terraform plan` (`tf/`) | `Plan: 1 to add`, choć polityka `kantyna-postgres-ingress` już jest w klastrze. Terraform porównuje kod z własnym, pustym plikiem stanu, a nie z klastrem. Przy `apply` skończyłoby się błędem „already exists". | Nie robię `apply`. Poprawna droga to `terraform import`, po którym `plan` powinien pokazać „No changes". |
