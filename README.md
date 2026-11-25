# PROJECT PROMETHEUS
## Specyfikacja wykonawcza dla Codexa — v0.1

**Data:** 25 listopada 2026 · **Status:** brief do implementacji · **Target:** MacBook, Docker, Portainer Standalone · **Tryb:** długotrwałe autonomiczne badania obliczeniowe pod nadzorem właściciela.

## 1. Misja i zakres

Stworzyć osobiste laboratorium badawcze, które prowadzi powtarzalne eksperymenty, generuje i falsyfikuje hipotezy, kumuluje wiedzę, dobiera narzędzia oraz optymalizuje własne metody badawcze. Rezultatem ma być dowód, reprodukowalny eksperyment, użyteczna obserwacja albo dobrze udokumentowany wynik negatywny — nie liczba wygenerowanych publikacji.

Właściciel wyznacza kierunki, udostępnia modele i budżet, a następnie nadzoruje pracę przez panel WWW. Nie zakładamy dostępu do infrastruktury dużych laboratoriów AI. MacBook jest serwerem sterującym i wykonawcą lekkich obliczeń, a zewnętrzne modele realizują zadania wymagające rozumowania. System nie wymaga GPU, treningu LLM ani stałej obecności użytkownika.

**Kryterium nadrzędne:** autonomia merytoryczna w ograniczonych, bezpiecznych dziedzinach, bez autonomicznego rozszerzania uprawnień, budżetów, dostępów do sieci ani modyfikowania mechanizmów nadzoru. Nie ma automatycznych działań fizycznych, publikacji zewnętrznych ani operacji na cudzych systemach.

## 2. Programy badawcze

Każdy program jest niezależną konfiguracją danych, a nie osobną aplikacją. LIFE pozostaje poza zakresem projektu.

| Program | Misja | Pierwsze eksperymenty | Wynik podlegający weryfikacji |
|---|---|---|---|
| MORPHOS | Geometria generatywna, matematyczna sztuka, cyfrowa emergencja | Fraktale 2D/3D, parkietaże, niebiologiczne automaty komórkowe, geometryczne formy | Reguła, seed, render, miary złożoności i stabilności, porównanie z istniejącą rodziną |
| LOGOS | Język programowania dogodny dla AI i zrozumiały dla ludzi | Porównanie dialektów małego języka obliczeniowego; reprezentacji tekstowej, AST i czytelnego pseudokodu | Poprawność, liczba tokenów, koszt naprawy, czytelność, wydajność; testy na niezależnych zadaniach |
| MATH | Użyteczne twierdzenia, kombinatoryka, tożsamości i algorytmy | Tożsamości, własności grafów, nowe przybliżenia i metody obliczeń | Dowód formalny lub kontrprzykład, wyniki porównawcze |
| ENERGY | Czysta energia i efektywność energetyczna | Bilans cieplny, pasywne chłodzenie, odzysk ciepła, niskotemperaturowa konwersja energii, analiza ekonomiczna OZE | Zgodność z termodynamiką, bilans energetyczny, analiza wrażliwości i porównanie z literaturą |
| SPACE | Fundamentalna fizyka i matematyka czasoprzestrzeni | Analiza literatury o metrykach, warunkach energetycznych i ograniczeniach relatywistycznych; bezpieczne modele abstrakcyjne | Obliczenia symboliczne, kontrola jednostek i granic, zgodność ze znanymi równaniami |

MORPHOS jest pierwszą ścieżką wdrożeniową, LOGOS drugą. ENERGY, MATH i SPACE otrzymują kompletne definicje i działające szablony eksperymentów, ale startują w stanie wyłączonym. Wszystkie programy początkowo są wyłączone do momentu świadomego uruchomienia przez właściciela.

### Konfiguracja programu

- UUID, nazwa, opis, cele, hipotezy, zakres dozwolonych tematów, wykluczenia, kryteria sukcesu.
- enabled, priorytet, dzienny i miesięczny limit kosztu, limit czasu CPU, liczba jednoczesnych zadań, limit wielkości artefaktów.
- Polityka modeli i narzędzi, preferowane metody weryfikacji, źródła literatury.
- Stan: disabled, running, pausing, paused, blocked, budget_exhausted, archived.
- Wyłączenie zatrzymuje przydział nowych zadań natychmiast; istniejące dochodzą do checkpointu lub są anulowane po określonym czasie. Dane pozostają. Hard stop anuluje pracę lokalną bez oczekiwania; zewnętrzne zapytanie może już generować koszt.
- Nowy program można zdefiniować w panelu bez zmiany kodu; pozostaje wyłączony do zatwierdzenia jego zakresu przez właściciela. System może sugerować programy i cele, lecz nie aktywować samodzielnie nowych domen.
- Programy współdzielą indeks ogólnej wiedzy i narzędzia, przy zachowaniu osobnej historii, budżetu i pochodzenia wyników.

## 3. Architektura docelowa

```mermaid
flowchart TD
    UI[Panel WWW] --> CORE[Core API i planista]
    CORE --> DB[(PostgreSQL)]
    CORE --> ROUTER[Model broker]
    ROUTER --> AI[Zatwierdzeni dostawcy AI]
    CORE --> WORK[Worker badawczy]
    WORK --> LAB[Izolowany wykonawca eksperymentów]
    WORK --> STORE[(Artefakty i wyniki)]
    WORK --> DB
    CORE --> EVAL[Ocena i wersjonowanie metod]
```

Preferowany monorepo, Python 3.12+ (wersja przypięta), FastAPI, SQLAlchemy/Alembic, PostgreSQL, React/TypeScript/Vite, pytest, Playwright. Użyć trwałej kolejki PostgreSQL (transakcyjne lease z heartbeat); nie dodawać Redis na początku, chyba że pomiary wykażą potrzebę. Panel może być serwowany statycznie przez API, aby ograniczyć liczbę kontenerów.

| Usługa Compose | Funkcja | Dostęp |
|---|---|---|
| `web` | UI, API, uwierzytelnianie | Jedyny port użytkownika |
| `scheduler` | Przydział zadań, retry, budżety, checkpointy | DB i wewnętrzne API |
| `worker` | Wykonuje cykle badań i recenzje | Wewnętrzne API, broker, kontrolowane narzędzia |
| `model-broker` | Adaptery modeli, limity, licznik kosztu, dostęp do zatwierdzonych API | Sekrety dostawców tylko tutaj |
| `lab` | Bezpieczne obliczenia i renderowanie | Wewnętrzny, bez Internetu; bez sekretów |
| `db` | PostgreSQL | Sieć wewnętrzna |

Preferować jeden wspólny obraz aplikacji z różnymi komendami startowymi, osobne obrazy DB i lab. W MVP jeden worker i jedno zadanie laboratoryjne naraz. Lab przyjmuje walidowane zadania obliczeniowe w formie ograniczonego DSL, np. wyrażenia symboliczne, siatki, parametry renderów, ograniczone programy LOGOS. Nie wykonywać dowolnego kodu wygenerowanego przez model w kontenerach aplikacyjnych. Jeżeli potrzebne jest rozszerzenie wykonawcy, agent przygotowuje propozycję kodu do recenzji i zatwierdzenia.

## 4. Model autonomicznej pracy

Cykl zadania rozpisany jako trwała maszyna stanów:

1. **Orient:** wczytaj cel, dotychczasowe wyniki, warunki i budżet.
2. **Survey:** znajdź publikacje, porównaj wcześniejsze wyniki i zaznacz luki; zapisuj źródła i rzeczywiście dostępne fragmenty.
3. **Propose:** zgłoś konkretne, falsyfikowalne hipotezy, przewidywane wyniki, koszty oraz kontrolę negatywną.
4. **Select:** oceń wartość informacji, koszt, odtwarzalność i podobieństwo do wcześniejszych badań; wybierz mały eksperyment. Kontrola zakresu jest obowiązkowa przed wykonaniem.
5. **Test:** wykonaj dozwolone obliczenie zgodnie z zapisanym manifestem, z limitami zasobów.
6. **Critique:** osobna sesja recenzenta szuka kontrprzykładów, błędów jednostek, ukrytych założeń i istniejącej literatury; nie traktuje własnej pewności jako dowodu.
7. **Verify:** ponowne wykonanie z niezależnymi danymi/seedem, porównanie z baseline, dowód symboliczny lub formalny, jeśli dostępny.
8. **Archive:** trwały zapis artefaktów, wyniku pozytywnego lub negatywnego, kosztu i oceny.
9. **Plan:** zaproponuj kolejny krok, rozgałęzienie lub zamknięcie ślepej uliczki.

Cykl jest ograniczony czasem, liczbą wywołań i budżetem. Planista może rozpocząć kolejny cykl wyłącznie w aktywnym programie i po ponownej kontroli limitów. Nie przekazuj agentowi nieograniczonego terminala, możliwości instalowania oprogramowania ani dowolnego dostępu do usług zewnętrznych.

Status dowodu: `hypothesis`, `tested`, `reproduced`, `formally_verified`, `refuted`, `inconclusive`, `external_reviewed`. Nowość to osobna oś: `unchecked`, `prior_art_found`, `no_match_found`, `expert_reviewed`. Brak znalezionego odpowiednika nie jest dowodem pierwszeństwa. Wynik formalny wymaga sprawdzenia zgodności formalnego twierdzenia z zamierzeniem.

## 5. Model broker i dobór narzędzi

Interfejs dostawcy: `capabilities`, `estimate_cost`, `generate`, `health`, `usage`. Obsługa odpowiedzi strukturalnych, limitów kontekstu, identyfikatora modelu i wersji, retry z backoff, timeout, rozliczenia rzeczywiste. Adaptery OpenAI, Anthropic oraz opcjonalnego lokalnego serwera kompatybilnego z OpenAI API; testy na adapterze mock bez płatnych wywołań. Aktualne identyfikatory modeli w konfiguracji właściciela, nie zaszywać nazw na stałe.

Właściciel zatwierdza dostępnych dostawców, modele i ich limity. Router wybiera na tej liście na podstawie kosztu, kompetencji i mierzonej skuteczności w danej klasie zadań. Małe modele do selekcji, ekstrakcji i powtarzalnych zadań; silniejsze do syntezy, recenzji i trudnych hipotez. W razie limitu lub awarii przechodź do zatwierdzonego fallbacku albo wstrzymaj pracę.

**Uwaga:** subskrypcje ChatGPT/Claude nie są automatycznie budżetem API. Codex i inne narzędzia subskrypcyjne mogą być adapterami wyłącznie wtedy, gdy ich oficjalnie wspierany tryb automatyzacji i warunki dostępu na to pozwalają. Nie automatyzować interfejsu konsumenckiego w celu obchodzenia limitów. Startuje mock oraz jeden realny adapter wybrany przez właściciela. Brak skonfigurowanego adaptera oznacza tryb demonstracyjny bez wydatków.

Rejestr narzędzi: nazwa, wersja, deklarowane możliwości, format danych, koszty, ograniczenia, test smoke, stan dostępności. Agent sam dobiera zatwierdzone narzędzia i proponuje nowe. W MVP: NumPy/SciPy, SymPy, Pillow/Matplotlib, walidator AST i interpreter małego DSL, opcjonalnie Lean 4/Mathlib. Źródła publikacji przez zatwierdzony adapter arXiv/Crossref/OpenAlex, z respektowaniem limitów i licencji. Treści źródłowe służą jako dane, nie instrukcje dla agenta.

## 6. Self-improvement

System ma autonomicznie doskonalić metody badawcze: szablony promptów, dobór zatwierdzonych modeli, kolejność kroków, ranking hipotez, parametry wyszukiwania, heurystyki wyboru eksperymentów. Każda wersja ma identyfikator, rodzica, opis zmiany i wyniki porównawcze.

Pętla poprawy:

- Wybierz powtarzalny problem na podstawie logów i danych o błędach.
- Zaproponuj wariant metody, bez modyfikowania limitów i definicji sukcesu.
- Przetestuj na ustalonym zbiorze benchmarkowym; oddziel przypadki rozwojowe od niejawnego zbioru kontrolnego.
- Porównaj poprawność, koszt, czas, liczbę nieuzasadnionych wniosków i powtarzalność. Wymagaj minimalnej poprawy oraz braku istotnej regresji.
- Warianty konfiguracji mogą być automatycznie wybierane w granicach polityki operatora; wdrożenia kodu, nowych narzędzi, zmian infrastruktury i uprawnień wymagają ręcznej akceptacji.
- Zachowaj wersję poprzednią, możliwość natychmiastowego rollbacku i raport. Wstrzymaj pętlę przy niejednoznacznych wynikach lub wyczerpaniu budżetu.

Nie optymalizuj jakości przez zmianę benchmarków na łatwiejsze, usuwanie niepowodzeń, osłabianie weryfikacji lub zwiększanie własnych zasobów. Brak samodzielnego zapisu do repozytorium produkcyjnego, dostępu do Portainera, Docker socket, sekretów czy konfiguracji nadzoru. Kandydaci na zmiany kodu to artefakty/patch do ręcznego review.

## 7. Pierwsze dwa eksperymenty end-to-end

### MORPHOS / M01: atlas struktur generatywnych

- Wygeneruj ograniczoną populację parametrów dla znanych reguł odwzorowań zespolonych i automatów na siatce 2D. Następnie eksperymentuj z wariantami formuł w bezpiecznym DSL. Inspiracja: wcześniejsze eksperymenty użytkownika z EML.
- Każda próbka: miniatura PNG, reguła, seed, parametry, rozdzielczość, wersja renderera, czas. Mierz przykładowo różnorodność, symetrię, liczbę komponentów i stabilność.
- Dedup: hash parametrów i podobieństwo obrazów. Szukaj rodzin różniących się mierzalnie od baseline, a nie tylko atrakcyjnych miniaturek.
- Pokaż galerię najlepszych form; ocena estetyczna użytkownika jest osobna od oceny naukowej. Eksport obrazów i reproduktorów; 3D dopiero po działającym 2D.

### LOGOS / L01: test języka dla agentów

- Zdefiniuj mały deterministyczny język bez dostępu do systemu operacyjnego i sieci: liczby, bool, listy, funkcje czyste, ograniczone iteracje, jawne typy. Warianty różnią się składnią (kompaktowa vs zbliżona do języka naturalnego vs AST); semantyka wspólna.
- Zbuduj parser, sprawdzanie typów, interpreter, formatter oraz widok czytelny dla człowieka, z testem round-trip.
- Zbuduj benchmark bezpiecznych zadań: przekształcenia tablic, statystyka, kombinatoryka, małe funkcje proceduralne. Porównuj z Pythonem na równoważnych zadaniach; Python jako zatwierdzone implementacje referencyjne, nie dowolny skrypt modelu wykonywany na hoście.
- Każda próba ma ten sam budżet kontekstu i napraw, z osobnym czasem nauki specyfikacji. Zapisuj wynik testów, tokeny wejścia i wyjścia, liczbę napraw, czas i zgodność z typami. Mierz również koszt pracy człowieka na małej próbce kodu.
- Oddziel wynik dla różnych modeli; język łatwy dla jednego modelu może nie być uniwersalnie optymalny. Kontroluj znajomość Pythona wynikającą z treningu. Ewolucja wariantów odbywa się w ograniczonej przestrzeni, a nie przez dowolne uruchamianie nowych kompilatorów.
- Literatura startowa: Thera, ilo, Lingo, Fluxon, MoonBit oraz katalog https://agentlanguages.dev/. Celem jest wiarygodny wynik porównawczy, nie obietnica stworzenia najlepszego języka świata.

## 8. Dane i pamięć

Minimalne tabele: `programs`, `goals`, `hypotheses`, `sources`, `jobs`, `job_attempts`, `experiments`, `verifications`, `artifacts`, `findings`, `model_calls`, `budget_ledger`, `method_versions`, `benchmark_runs`, `audit_events`, `approvals`. Każdy wynik powiązany z programem, hipotezą, wersją metody, manifestem i źródłami. Artefakty w trwałym wolumenie, ścieżki i SHA-256 w DB. Nie przechowywać kluczy ani pełnych sekretów w logach, promptach i artefaktach.

Manifest eksperymentu: parametry, seed, wersje kodu/narzędzi, model, wersja promptu, źródła wejściowe, limit CPU/RAM/czasu, przewidywany wynik, baseline, komenda logiczna/rodzaj operacji, identyfikator artefaktu, rezultat i koszt. Wyszukiwanie pełnotekstowe w PostgreSQL na start; embeddingi opcjonalnie, tylko po zmierzeniu potrzeby. Historia jest append-only w odniesieniu do zdarzeń badawczych; poprawki to kolejne wersje.

## 9. Panel WWW i API

Język: polski interfejs lub prosty wybór PL/EN; artefakty badawcze domyślnie po angielsku. Responsywny, wygodny na telefonie. Nie wymaga kont wieloużytkownikowych w MVP: jeden operator, hasło ustawiane podczas pierwszego uruchomienia, sesja z bezpiecznym cookie, ochrona CSRF i mutacji.

| Ekran | Funkcje |
|---|---|
| Dashboard | Globalny start/pauza/stop, aktywne programy, aktualna praca, zegar budżetu, zużycie CPU/RAM, ostatnie odkrycia, problemy |
| Programs | Niezależne przełączniki, priorytety, cele, edycja zakresu, limity, kolejka, historia i tworzenie nowego programu |
| Research | Hipotezy, literatura, eksperymenty, wyniki, niepowodzenia, dowody, poziom potwierdzenia i nowości |
| Gallery | MORPHOS: miniatury, parametry, rodziny, filtrowanie, eksport i ponowne wykonanie |
| LOGOS | Ranking wariantów, testy, kod i widok człowieka, koszt, błędy, porównanie z baseline |
| Evolution | Warianty metod, metryki, benchmarki, aktualnie wybrana wersja i rollback |
| Settings | Modele, polityka kosztów, narzędzia, limity hosta, kopie zapasowe, logi |
| Approvals | Nowe cele poza dotychczasowym zakresem, narzędzia, poprawki kodu, propozycje migracji i publikacji |

API REST pod `/api/v1`, OpenAPI generowane z FastAPI. Minimalne operacje `/programs`, `/programs/{id}/enable`, `/disable`, `/pause`, `/jobs`, `/findings`, `/experiments`, `/artifacts`, `/budgets`, `/models`, `/methods`, `/approvals`, `/status`, `/stop`. Zmiany limitów i stanów audytowane. Strumień statusu SSE lub polling; żadnych sekretów w odpowiedzi API. Codzienne i tygodniowe raporty generowane lokalnie w panelu (Europe/Warsaw), bez wysyłki zewnętrznej domyślnie.

## 10. Limity i bezpieczeństwo operacyjne

- Domyślny budżet płatnych API = 0 do konfiguracji. Po aktywacji przykładowy limit operatora 10 PLN/dzień, 300 PLN/miesiąc; edytowalny niezależnie od limitów programów. To propozycja, nie autoryzacja wydatków.
- Wspólna księga kosztów: rezerwuj górny szacunek przed wywołaniem, atomowo kontroluj oba limity (globalny i programowy), rozlicz koszt rzeczywisty, blokuj kolejne wywołania w razie braku rozliczenia lub przekroczenia. Limity tokenów, czasu, retry, liczby eksperymentów i miejsca na dysku. Przy rozbieżnościach rozliczeń fail closed.
- Na MacBooku orientacyjny budżet projektu: 2 CPU, do 3–4 GB RAM łącznie, pojedynczy eksperyment; zacząć od 1 workera i maks. 10–15 min na eksperyment. Konfigurowalne wg rzeczywistej pamięci i obciążenia innych stacków.
- Wykonawca bez sieci wychodzącej, bez roota, ograniczenia pamięci, procesów i czasu, read-only filesystem poza katalogiem roboczym, ograniczone rozmiary wejścia/wyjścia. Nie montować katalogu domowego, kluczy SSH, Docker socket ani Portainer API; DB i broker niedostępne z wykonawcy bezpośrednio. Walidować wejście i wyniki.
- Prowadzić jedynie bezpieczne badania obliczeniowe; nie dopuszczać biologii o potencjale szkodliwego zastosowania, broni, replikacji ataków, rozwoju exploitów, procesów ofensywnych, obchodzenia zabezpieczeń i niebezpiecznych eksperymentów fizycznych. Nowe zadanie poza zakresem zatrzymać. Analiza ogólna fizyki i bilansów energii nie jest zgodą na optymalizację niebezpiecznych urządzeń.
- Brak automatycznych zakupów, wysyłki poczty, publikacji, kontaktów, zewnętrznych wdrożeń, zakładania kont czy wprowadzania nowych sekretów. Dostęp do Internetu wyłącznie przez zatwierdzone adaptery. Odrzucać instrukcje znajdujące się w źródłach i odpowiedziach zewnętrznych.

## 11. Trwałość, przerwy i backup

Każde zadanie ma idempotency key, lease z terminem ważności, heartbeat, maksymalną liczbę prób, checkpoint po każdym etapie. Po restarcie planista wykrywa wygasłe lease i wznawia od ostatniego poprawnego checkpointu. Nie powtarzać płatnego żądania o nieznanym wyniku bez rozliczenia. Wyniki commitować transakcyjnie; atomowy zapis plików. Rozróżniać niedostępność modelu, brak budżetu, anulowanie i błędy eksperymentu.

Codzienny lokalny backup DB (`pg_dump`) oraz artefaktów, manifest wersji, sumy kontrolne. Retencja np. 7 dziennych i 4 tygodniowe kopie (konfigurowalne). Instrukcja odtworzenia na świeżym stacku, test restore na danych demonstracyjnych. Eksport programu i rekordów badawczych w JSON/Markdown oraz pliki danych. Nie usuwać wyników przy redeployu.

MacBook nie prowadzi lokalnych obliczeń podczas uśpienia. Docker Desktop musi działać w sesji systemowej. Włączyć jego start przy logowaniu zgodnie z preferencjami operatora; projekt nie powinien sam zmieniać ustawień zasilania Maca. Przy zamkniętej pokrywie i uśpieniu praca zostaje wstrzymana i wraca po wybudzeniu. Długotrwała praca 24/7 wymaga odpowiedniej konfiguracji zasilania lub dedykowanego hosta.

## 12. Portainer — obowiązkowy sposób wdrożenia

Target: już istniejące lokalne Docker Desktop/Portainer lub równoważny silnik Docker na Macu. Nie zmieniać innych stacków użytkownika. Obsługa `linux/arm64` (Apple Silicon) i opcjonalnie `linux/amd64` (Intel / przyszła migracja). Użyć wieloarchitekturowych obrazów. Sprawdzić rzeczywistą architekturę i zasoby w trakcie instalacji.

Repozytorium musi zawierać `compose.yaml` gotowy do Portainer Standalone, `.env.example`, Dockerfile, healthchecki, migracje, `README.md`, `docs/DEPLOY_PORTAINER.md`, `docs/BACKUP_RESTORE.md`, `docs/OPERATIONS.md` i testy. Żadnych wymaganych plików spoza repozytorium; produkcyjne sekrety w zmiennych/sekretach skonfigurowanych przez właściciela, nie w Git. Opublikowane obrazy wersjonowane i przypięte do konkretnego tagu/digestu; nie polegać na automatycznie zmieniającym się `latest`.

Wolumeny nazwane: `prometheus_db`, `prometheus_artifacts`, `prometheus_backups`; spójna nazwa projektu, zachowanie wolumenów przy aktualizacji. Własne sieci stacku, brak portów DB na hoście, restart `unless-stopped`, limity `cpus`, `mem_limit`, `pids_limit` dla usług, healthcheck DB i aplikacji. Aplikacja domyślnie `127.0.0.1:8787`; dostęp z telefonu przez świadomie skonfigurowaną prywatną sieć/VPN albo autoryzowane HTTPS, nie publiczne przekierowanie portów.

Procedura handoff:

1. Codex tworzy repozytorium i testuje stack lokalnie przez `docker compose config`, build oraz uruchomienie testowe z fake providerem.
2. Przygotowuje obrazy arm64/amd64 oraz repo/registry dostępne dla użytkownika (publikacja tylko po jego zgodzie). Alternatywnie dokumentuje lokalne zbudowanie obrazów, a następnie wdrożenie Compose wskazującego te obrazy.
3. W Portainer wybrać lokalne środowisko Docker Standalone → Stacks → Add stack → nazwa `prometheus-research` → Repository albo Upload → wskazać `compose.yaml`. Unikać konfliktu z systemem monitoringu o nazwie Prometheus.
4. Skonfigurować wymagane zmienne i sekrety, uruchomić stack, potwierdzić healthy, wejść do panelu, ustawić hasło, wykonać smoke test bez AI.
5. Ustawić jeden zatwierdzony model, budżet i włączyć MORPHOS. Po weryfikacji można aktywować LOGOS lub inne programy.
6. Aktualizacja poprzez ręczne zatwierdzenie nowej wersji obrazu i redeploy w Portainer; wykonać backup przed migracją DB. Udokumentować rollback obrazów oraz kompatybilność schematu. Bez automatycznych aktualizacji produkcji przez agentów.

Dokumentacja referencyjna: https://docs.portainer.io/user/docker/stacks/add ; https://docs.portainer.io/user/docker/stacks/edit ; https://docs.docker.com/desktop/settings-and-maintenance/settings/ .

## 13. Kolejność implementacji

| Etap | Zakres i warunek zakończenia |
|---|---|
| 0 | Repo, ADR, schemat DB, Compose, mock model, CI, README; stack działa bez płatnego API |
| 1 | Panel, CRUD programów, niezależne przełączniki, trwała kolejka, checkpoint, globalny stop, budget ledger, logi i auth |
| 2 | MORPHOS M01: pełny cykl na deterministycznych danych, galeria, dedup i reprodukcja |
| 3 | Broker: pierwszy rzeczywisty adapter, routing, kontrola użycia, literatura, recenzent, audyt |
| 4 | LOGOS L01: DSL, interpreter, benchmark, raporty porównawcze; przy wyłączonym programie brak zadań |
| 5 | Ewolucja konfiguracji metod, kontrolne benchmarki, wersjonowanie i rollback, backup/restore |
| 6 | MATH, ENERGY, SPACE: szablony, bezpieczne narzędzia i przykładowy cykl; domyślnie wyłączone |
| 7 | Handoff Portainer, test na realnym MacBooku, 72-godzinny ograniczony soak test za zgodą właściciela |

Nie blokować etapu 1 skomplikowanym agent frameworkiem; prosta, obserwowalna maszyna stanów jest preferowana. Kolejne etapy wdrażać jako wystarczająco małe, testowalne pionowe przyrosty.

## 14. Kryteria odbioru

- [ ] Stack z jednego Compose można odtworzyć w Portainer; aplikacja healthy, interfejs dostępny na skonfigurowanym porcie.
- [ ] Demo działa bez kluczy API i bez kosztów; programy domyślnie wyłączone.
- [ ] Włączenie jednego programu generuje tylko jego zadania; wyłączenie ma skutek niezależny i zachowuje historię.
- [ ] Globalny stop uniemożliwia kolejne zlecenia; twarde limity działają również przy równoczesnych wywołaniach, restartach i awarii dostawcy.
- [ ] Restart stacku w połowie eksperymentu zachowuje dane i wznawia bez duplikacji wyniku lub niekontrolowanych płatnych retry.
- [ ] MORPHOS tworzy co najmniej 20 deterministycznych przykładów, każdy z manifestem, renderem i odtwarzalnością; wynik jest poprawnie oznaczony jako benchmark, nie odkrycie.
- [ ] LOGOS porównuje minimum trzy reprezentacje na minimum 20 bezpiecznych zadaniach, raportuje wyniki i ograniczenia eksperymentu.
- [ ] Pętla metod wykazuje działanie na kontrolowanym przypadku, odrzuca regresję i umożliwia rollback.
- [ ] Lab nie ma dostępu do sekretów, sieci publicznej ani hosta; testy odrzucają zadania poza zakresem i niepoprawne wejścia.
- [ ] Backup/restore odtwarza programy, wyniki i artefakty na nowym środowisku testowym; redeploy nie kasuje danych.
- [ ] Codex dostarcza plan instalacji, procedurę operacyjną, listę ograniczeń i raport z testów, odróżniając weryfikację lokalną od faktycznego wdrożenia na Macu.

## 15. Instrukcja dla agenta implementującego

Traktuj dokument jako specyfikację nadrzędną. Najpierw zinwentaryzuj repo i środowisko, zachowaj istniejące zmiany. Jeśli nie ma repo, załóż lokalny szkielet projektu. Prowadź `IMPLEMENTATION_STATUS.md` z checklistą, decyzjami i aktualnymi ograniczeniami. Rozwiązuj problemy samodzielnie w uzgodnionym zakresie, testuj każdy etap i zapisuj powtarzalne instrukcje. Preferuj małą liczbę zależności i łatwą diagnostykę. Nie wymagaj od właściciela decyzji dotyczących drobnych szczegółów technicznych; pytaj o faktyczne dostępy, wydatki, publikację, instalację na jego Macu i zmiany uprawnień. Nigdy nie deklaruj przeprowadzonego wdrożenia, jeśli przetestowano jedynie środowisko developerskie.

**Pierwsza sesja Codexa:** zbuduj etapy 0–2 z mock providerem. Priorytetem jest realnie działający panel z przełącznikami, trwałym zadaniem MORPHOS i gotowym stackiem Portainera. Zakończ działającym demo i instrukcją uruchomienia. Następnie rozwijaj broker, LOGOS i pozostałe etapy, nie wymagając stałego prowadzenia przez operatora.
