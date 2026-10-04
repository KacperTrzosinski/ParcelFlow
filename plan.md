# ParcelFlow: system zarządzania przesyłkami dla firmy kurierskiej

**Przedmioty:** Programowanie obiektowe II oraz Bazy danych (jeden wspólny projekt)

**Zespół:** Igor Żurawski, Kamil Krysztoforski, Kacper Trzosiński

**Repozytorium:**: [Link do repozytorium](https://github.com/KacperTrzosinski/ParcelFlow)

---

## 1. Opis projektu

ParcelFlow to uproszczony system klasy TMS/ERP dla firmy kurierskiej. Pozwala zarządzać klientami, przesyłkami, magazynami, kurierami, pojazdami, trasami doręczeń i fakturami. Dyspozytor widzi stan firmy na dashboardzie, a każda przesyłka ma numer śledzenia i historię zmian statusu.

System składa się z trzech części:

- **backend i REST API w C++**,
- **baza danych PostgreSQL**,
- **webowy dashboard** (aplikacja jednostronicowa).

## 2. Stos technologiczny

| Warstwa       | Technologia                                                                           |
| ------------- | ------------------------------------------------------------------------------------- |
| Backend i API | C++20, framework Drogon, CMake                                                        |
| Baza danych   | PostgreSQL (Docker)                                                                   |
| Frontend      | React, TypeScript, Vite, Tailwind CSS, Recharts                                       |
| Wdrożenie     | Docker Compose na darmowej maszynie Oracle Cloud (Always Free), reverse proxy z HTTPS |
| Dokumentacja  | LaTeX (sprawozdanie), dbdiagram.io (ERD), README                                      |
| Narzędzia     | Git (osobne gałęzie i pull requesty), GitHub Actions                                  |

## 3. Zakres funkcjonalny

1. **Klienci:** dodawanie, edycja, usuwanie, wyszukiwanie, lista przesyłek klienta.
2. **Przesyłki:** pełny CRUD, numer śledzenia, nadawca i odbiorca, waga, wymiary, typ (standardowa, ekspresowa, delikatna), automatyczna wycena zależna od typu.
3. **Statusy przesyłek:** przepływ _utworzona → w magazynie → w doręczeniu → dostarczona / nieudana_, z historią zmian i walidacją dozwolonych przejść.
4. **Magazyny (oddziały):** CRUD, lista przesyłek w danym magazynie.
5. **Kurierzy i pojazdy:** CRUD, udźwig pojazdu, dostępność kuriera.
6. **Trasy:** utworzenie trasy z przypisaniem kuriera, pojazdu i listy przesyłek. System sprawdza udźwig i dostępność, a całość odbywa się w jednej transakcji.
7. **Faktury:** wygenerowanie faktury z dostarczonych, nierozliczonych przesyłek klienta (transakcja) oraz oznaczenie jako opłaconej.
8. **Dashboard:** liczba przesyłek wg statusu, przesyłki dzisiejsze, przychód miesięczny, obłożenie kurierów, najlepsi klienci, wykresy.
9. **Funkcje wspólne:** filtrowanie, sortowanie i paginacja list, czytelne komunikaty błędów (naruszenia ograniczeń bazy nie ujawniają surowych błędów SQL).

Poza zakresem projektu pozostają: uwierzytelnianie i role użytkowników, mapy, powiadomienia w czasie rzeczywistym oraz generowanie plików PDF.

## 4. Projekt bazy danych

Schemat w trzeciej postaci normalnej, 9 tabel:

- `customers`, `depots`, `vehicles`, `couriers`,
- `shipments`: powiązana z klientami, magazynem i fakturą,
- `shipment_events`: historia statusów, relacja **1:N** do `shipments`,
- `routes`: powiązana z kurierem, pojazdem i magazynem startowym,
- `route_stops`: relacja **N:M** między `routes` a `shipments` (z kolejnością i statusem przystanku),
- `invoices`.

Zastosowane ograniczenia: klucze główne i obce, `NOT NULL`, `UNIQUE` (numer śledzenia, numer faktury, e-mail), `CHECK` (waga dodatnia, kwoty nieujemne, dozwolone statusy).

Dane testowe: skrypt seed generujący od kilkudziesięciu do kilkuset rekordów na tabelę.

Zestaw co najmniej 20 zapytań SQL w osobnym pliku `queries.sql` obejmuje: różne operatory `WHERE`, `ORDER BY` i `LIMIT`, złączenia (w tym czterech tabel), `GROUP BY`/`HAVING`, podzapytania, `UPDATE`/`DELETE` z warunkiem oraz transakcje (tworzenie trasy, wystawianie faktury) z przykładem `ROLLBACK`.

Wszystkie zapytania w aplikacji są parametryzowane.

## 5. Zagadnienia kwalifikacyjne (PO2)

| #   | Zagadnienie                 | Planowana realizacja                                                                                                                                                                  |
| --- | --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| 1   | Hermetyzacja i enkapsulacja | Klasy domenowe z prywatnymi polami i kontrolowanym dostępem przez metody                                                                                                              |
| 2   | Konstruktory                | Konstruktory z parametrami i walidacją danych w encjach                                                                                                                               |
| 3   | Dziedziczenie pojedyncze    | `Person` → `Customer`, `Courier`; `Shipment` → `StandardShipment`, `ExpressShipment`, `FragileShipment`                                                                               |
| 4   | Dziedziczenie wielokrotne   | `Courier` dziedziczy po `Person` oraz `IAuditable`                                                                                                                                    |
| 5   | Interfejsy                  | `IRepository`, `IPricingStrategy`, `IAuditable` (klasy czysto wirtualne)                                                                                                              |
| 6   | Klasy i metody abstrakcyjne | `BaseEntity` z czysto wirtualnym `toJson()`, `Shipment::calculatePrice()`                                                                                                             |
| 7   | Polimorfizm i przesłanianie | Wycena przesyłek różnych typów przez wspólny typ bazowy (`unique_ptr<Shipment>`)                                                                                                      |
| 8   | Kompozycja i agregacja      | `Route` zawiera `RouteStop`, `Invoice` zawiera pozycje, `Address` jako obiekt wartości                                                                                                |
| 9   | Funkcje anonimowe           | Lambdy w filtrowaniu, sortowaniu i obsłudze żądań HTTP                                                                                                                                |
| 10  | Wyjątki                     | Własne typy: `ValidationException`, `NotFoundException`, `BusinessRuleException`, mapowane na kody HTTP                                                                               |
| 11  | Refleksja                   | Język C++ nie udostępnia refleksji w czasie wykonania. Wykorzystamy RTTI (`typeid`, `dynamic_cast`) oraz własny rejestr typów jako ograniczoną namiastkę i opiszemy to w sprawozdaniu |
| 12  | Przeciążanie operatorów     | Klasy `Money` i `Weight` (`+`, `<`, `==`, `operator<<`)                                                                                                                               |
| 13  | Elementy statyczne          | Konfiguracja aplikacji, generator numerów śledzenia, statyczne metody fabrykujące                                                                                                     |
| 14  | Typy generyczne             | `Repository<T>`, `Result<T>`, szablonowe funkcje pomocnicze                                                                                                                           |

Każde zagadnienie zostanie opisane w sprawozdaniu wraz z miejscem w kodzie i uzasadnieniem.

## 6. Podział prac

| Osoba                       | Zakres                                                                                                                                                                                     |
| --------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Igor: backend**    | Struktura projektu i CMake, klasy domenowe i hierarchie, warstwa dostępu do bazy, endpointy REST, walidacja po stronie serwera, logika biznesowa (statusy, wycena, trasy)                  |
| **Kamil: frontend**   | Szkielet aplikacji i routing, widoki list i formularzy (CRUD), listy rozwijane dla kluczy obcych, wyświetlanie błędów walidacji, dashboard i wykresy, klient API                           |
| **Kacper: full-stack** | Schemat bazy, ERD, dane testowe, `queries.sql` i transakcje, Docker Compose i wdrożenie, endpointy faktur i raportów, testy integracyjne, sprawozdanie i README, końcowe szlify interfejsu |

Wszyscy członkowie zespołu przygotowują własną część wniosków projektowych oraz znają całość projektu na potrzeby prezentacji.

## 7. Harmonogram

| Termin        | Etap                                                                                               |
| ------------- | -------------------------------------------------------------------------------------------------- |
| do 11.10      | Zatwierdzenie tematu                                                                               |
| do 18.10      | Zgłoszenie repozytorium, opisu, podziału prac i stosu technologicznego                             |
| 19.10 – 1.11  | Schemat bazy i ERD, środowisko Docker, szkielet API i aplikacji webowej, uzgodnienie kontraktu API |
| 2.11 – 22.11  | Encje, endpointy CRUD, widoki CRUD, dane testowe, pierwsze wdrożenie                               |
| 23.11 – 13.12 | Statusy, trasy, faktury, transakcje, dashboard, zestaw zapytań SQL                                 |
| 14.12 – 3.01  | Obsługa błędów, walidacja, testy, dopracowanie interfejsu, przegląd zagadnień kwalifikacyjnych     |
| 4.01 – 17.01  | Sprawozdanie w LaTeX, README, dokumentacja bazy, przygotowanie prezentacji                         |

## 8. Dokumentacja i sposób pracy

- Sprawozdanie w LaTeX (PDF) z opisem funkcjonalnym, technologicznym, podziałem obowiązków, opisem wszystkich zagadnień kwalifikacyjnych, instrukcją uruchomienia lokalnego i zdalnego oraz wnioskami każdego członka zespołu.
- `README.md` z opisem tematu, instrukcją uruchomienia (Docker i aplikacja) i opisem schematu, diagram ERD oraz krótkie uzasadnienie decyzji przy normalizacji.
- Praca w repozytorium rozłożona w czasie, z regularnymi commitami wszystkich członków zespołu.
- Prezentacja na działającym, wdrożonym systemie.
