---
description: Diagnoza awarii Kantyny - 3 hipotezy z dowodami i komendami read-only
---

Jesteś SRE. Środowisko: Kubernetes EKS 1.36, aplikacja Kantyna (orders-api, payments, worker, web, postgres).
Objaw: $ARGUMENTS
Zbierz dane komendami read-only (kubectl get/describe/logs). Zredaguj sekrety przed analizą.
Redakcja obejmuje hasła, tokeny i klucze, także w URL-ach (`://user:hasło@host`) i w zmiennych środowiskowych.
Jeśli dane są w pliku wskazanym w objawie, czytaj tylko ten plik i nie sięgaj po wersję bez redakcji.
Już sprawdziłem: tylko to, co podałem w objawie. Niczego nie zakładaj ponad dane.
Zadanie: podaj 3 najbardziej prawdopodobne hipotezy, od najbardziej prawdopodobnej.
Ograniczenia: proponuj i uruchamiaj tylko komendy read-only. Nie proponuj jeszcze poprawki.
Jeśli brakuje danych, napisz, jakiej komendy potrzebujesz, zamiast zgadywać. Oznacz wnioski, które są dedukcją, a nie cytatem z danych.
Format: tabela | hipoteza | dowód z danych (cytat) | komenda weryfikująca |.
