# Security Knowledge Base

Notatki po polsku do powtórek z bezpieczeństwa aplikacji i DevSecOps. Definicje, przykłady i pytania rekrutacyjne, połączone z praktyką w projektach własnych.

## Spis treści

- [1. OWASP](#1-owasp)
- [2. SAST, SCA i DAST](#2-sast-sca-i-dast)
- [3. Jenkins i GitHub Actions](#3-jenkins-i-github-actions)
- [4. Trivy](#4-trivy)
- [5. SBOM](#5-sbom)
- [6. Security gate i ochrona brancha](#6-security-gate-i-ochrona-brancha)
- [7. Zarządzanie podatnościami](#7-zarządzanie-podatnościami)
- [8. Priorytety: CVSS, EPSS, KEV i reachability](#8-priorytety-cvss-epss-kev-i-reachability)
- [9. Hardening i supply chain](#9-hardening-i-supply-chain)
- [10. Praktyczny przykład z projektu](#10-praktyczny-przykład-z-projektu)
- [11. Szybka powtórka przed rozmową](#11-szybka-powtórka-przed-rozmową)
- [Materiały źródłowe](#materiały-źródłowe)

## 1. OWASP

**OWASP — Open Worldwide Application Security Project** — to organizacja tworząca otwarte materiały, standardy i narzędzia dotyczące bezpieczeństwa aplikacji.

| Nazwa | Co oznacza |
| --- | --- |
| OWASP Top 10 | Zestawienie najważniejszych kategorii ryzyk dla aplikacji webowych; punkt wyjścia do nauki, a nie pełna lista kontroli bezpieczeństwa |
| OWASP Dependency-Check | Narzędzie SCA do identyfikacji zależności ze znanymi podatnościami |
| OWASP Juice Shop | Celowo podatna aplikacja do ćwiczeń w kontrolowanym środowisku |

**ZAP**, znany wcześniej jako OWASP ZAP, to narzędzie do testowania działających aplikacji webowych — DAST. Dokumentacja i aktualny projekt: [zaproxy.org](https://www.zaproxy.org/).

**Do zapamiętania:** OWASP to organizacja i zbiór projektów. OWASP Top 10 nie jest skanerem.

## 2. SAST, SCA i DAST

| Metoda | Co analizuje | Czy aplikacja musi działać? | Przykład |
| --- | --- | --- | --- |
| **SAST** — Static Application Security Testing | Kod źródłowy | Nie | Niebezpieczne użycie poleceń systemowych lub budowanie SQL z danych użytkownika |
| **SCA** — Software Composition Analysis | Zależności, także pośrednie | Nie | Biblioteka w wersji objętej znanym CVE |
| **DAST** — Dynamic Application Security Testing | Działającą aplikację przez jej interfejsy | Tak | Testy odpowiedzi aplikacji na próby SQL injection lub XSS |

**SAST → mój kod. SCA → biblioteki. DAST → działająca aplikacja.**

Przykładowe narzędzia:

- SAST: Bandit dla Pythona, CodeQL, Semgrep.
- SCA: pip-audit, OWASP Dependency-Check, funkcje analizy zależności Trivy.
- DAST: ZAP.

Dodatkowe kontrole:

| Kontrola | Cel |
| --- | --- |
| Secret scanning | Wykrywanie ujawnionych kluczy, tokenów i haseł |
| Container scanning | Sprawdzanie pakietów systemowych i bibliotek w obrazie |
| IaC scanning | Wykrywanie błędów konfiguracji infrastruktury, np. w manifestach Kubernetes |
| Testy aplikacji | Sprawdzenie zachowania aplikacji; uzupełniają skanowanie bezpieczeństwa |

Skaner może zgłosić **false positive**, czyli wynik, który po weryfikacji nie stanowi zgłoszonego problemu. Może też czegoś nie wykryć — brak alertów nie dowodzi braku podatności.

## 3. Jenkins i GitHub Actions

**Jenkins i GitHub Actions uruchamiają kontrole. Skanery wykonują analizę.**

W Jenkinsie definiujemy etapy np. w `Jenkinsfile`, a w GitHub Actions — workflowy YAML w `.github/workflows/`.

Przykładowy przepływ:

1. Pobranie kodu.
2. Instalacja zależności i testy.
3. SAST, SCA i skan sekretów.
4. Analiza konfiguracji oraz budowanie obrazu.
5. Skan obrazu i wygenerowanie SBOM.
6. Zapis raportów i decyzja końcowego security gate.

### Gotowe mechanizmy GitHuba

| Mechanizm | Rola |
| --- | --- |
| GitHub Advisory Database | Informacje o znanych podatnościach |
| Dependabot alerts | Alerty o podatnych zależnościach |
| Dependabot security updates | PR-y z aktualizacjami naprawiającymi wspierane podatne zależności |
| Dependabot version updates | Zaplanowane PR-y aktualizujące wersje według konfiguracji |
| Code scanning / CodeQL | Analiza kodu i prezentacja wyników |
| Secret scanning | Wykrywanie wspieranych typów sekretów |
| GitHub Actions | Uruchamianie testów, skanerów i własnych reguł |

Dostępność funkcji zależy od rodzaju repozytorium, planu i ustawień. Sam plik konfiguracji Dependabota nie jest dowodem, że wszystkie alerty i security updates są włączone.

## 4. Trivy

**Trivy to skaner bezpieczeństwa rozwijany przez Aqua Security.**

Może sprawdzać m.in.:

- podatności w pakietach systemowych i zależnościach aplikacji;
- sekrety w plikach;
- błędy konfiguracji infrastruktury;
- komponenty obrazu do wygenerowania SBOM.

Nie należy utożsamiać skanowania zależności przez Trivy z analizą logiki kodu SAST. W naszym projekcie SAST wykonuje Bandit.

### Przykładowe polecenia

Skan obrazu, który zwraca błąd przy wykryciu HIGH lub CRITICAL:

```bash
trivy image --scanners vuln --severity HIGH,CRITICAL --exit-code 1 myapp:local
```

Skan Dockerfile i konfiguracji Kubernetes w katalogu:

```bash
trivy config --severity HIGH,CRITICAL --exit-code 1 .
```

Skan sekretów w aktualnych plikach, bez wypisywania wartości do raportu:

```bash
trivy fs --scanners secret --skip-dirs .git --exit-code 1 --format json --output /dev/null .
```

To ostatnie polecenie przekazuje wynik przez kod wyjścia, ale nie zachowuje szczegółów wykrycia. Do diagnozy używamy lokalnego, chronionego raportu. Skan aktualnych plików nie oznacza przeskanowania całej historii Git. Wykryty prawdziwy sekret należy unieważnić lub obrócić; samo usunięcie go z pliku nie wystarcza.

**Ważne:** sam wynik skanera nie zawsze blokuje pipeline. Trzeba ustawić właściwy kod wyjścia i obsłużyć go w CI. Błąd skanera również nie może zostać uznany za sukces.

## 5. SBOM

**SBOM — Software Bill of Materials — to spis komponentów oprogramowania**, podobny do listy składników produktu.

Może zawierać nazwy komponentów, wersje, identyfikatory, zależności i dostępne informacje o licencjach. Kompletność zależy od generatora oraz analizowanego artefaktu.

| Artefakt | Na jakie pytanie odpowiada? |
| --- | --- |
| SBOM | Z czego składa się aplikacja lub obraz? |
| Raport podatności | Które rozpoznane komponenty mają znane problemy według użytych baz? |

Przykład: po ogłoszeniu podatności konkretnej biblioteki sprawdzamy, które obrazy zawierają jej podatną wersję.

**CycloneDX** i **SPDX** to formaty używane do wymiany SBOM.

Generowanie SBOM obrazu w CycloneDX:

```bash
trivy image --format cyclonedx --output sbom.cdx.json myapp:local
```

W Trivy samo `--format cyclonedx` generuje inwentarz bez włączania skanowania podatności. Jeśli chcemy dołączyć wyniki podatności:

```bash
trivy image --scanners vuln --format cyclonedx --output sbom-with-vulnerabilities.cdx.json myapp:local
```

SBOM najlepiej powiązać z konkretnym digestem obrazu, commitem i datą. Nie zastępuje skanowania, podpisu obrazu ani potwierdzenia jego pochodzenia.

## 6. Security gate i ochrona brancha

**Security gate to reguła decydująca, czy zmiana może przejść dalej na podstawie wyników kontroli.**

Przykładowa polityka:

- testy muszą przejść;
- brak wykrytych sekretów;
- brak podatności kontenera HIGH/CRITICAL;
- brak błędów IaC HIGH/CRITICAL;
- wszystkie wymagane skany muszą się rzeczywiście zakończyć.

**Czerwony check sam w sobie nie musi blokować merge'a.** Potrzebne są reguły brancha lub ruleset wymagające konkretnego checka przed scaleniem.

Końcowy gate powinien odrzucać nie tylko wykryte problemy, ale też nieudane, anulowane lub pominięte wymagane kontrole. Nie należy ukrywać ich przez `continue-on-error` lub `|| true`.

Zielony wynik oznacza spełnienie skonfigurowanych reguł w danym przebiegu. Nie jest certyfikatem pełnego bezpieczeństwa.

## 7. Zarządzanie podatnościami

**Vulnerability management to cały proces obsługi podatności. Uruchomienie skanera jest jednym z etapów.**

1. **Zbierz wyniki** — z kodu, zależności, kontenerów i infrastruktury.
2. **Usuń duplikaty** — np. według CVE/reguły, komponentu, wersji i lokalizacji.
3. **Zweryfikuj zgłoszenia** — sprawdź rzeczywiście używany artefakt i kontekst.
4. **Ustal priorytet** — uwzględnij możliwość wykorzystania i skutki.
5. **Przypisz właściciela i termin** — zgłoszenie musi mieć osobę odpowiedzialną.
6. **Napraw albo ogranicz ryzyko** — aktualizacja, zmiana konfiguracji, usunięcie komponentu lub inna uzasadniona decyzja.
7. **Sprawdź ponownie** — zachowaj dowód, że problem został usunięty.

| Status | Znaczenie |
| --- | --- |
| Open / in progress | Problem czeka na obsługę lub jest analizowany |
| Fixed | Naprawa została potwierdzona ponowną weryfikacją |
| Contained | Ograniczono ekspozycję lub możliwość wykorzystania; podatność może pozostać |
| Risk accepted | Uprawniona osoba zaakceptowała ryzyko z uzasadnieniem i terminem ponownej oceny |
| False positive | Weryfikacja wykazała błędne zgłoszenie |

Przykładowy wpis: CVE, komponent i wersja, artefakt, severity, kontekst ryzyka, właściciel, termin, status, decyzja oraz linki do wyników przed i po naprawie.

**DefectDojo** może agregować wyniki wielu narzędzi i wspierać triage. Nie zastępuje oceny ryzyka.

Raport dla CISO powinien pokazywać np. otwarte i przeterminowane istotne problemy, nowe i naprawione zgłoszenia, trend, pokrycie skanowaniem oraz wygasające akceptacje ryzyka. Liczba alertów bez kontekstu może być myląca.

## 8. Priorytety: CVSS, EPSS, KEV i reachability

| Pojęcie | Odpowiedź na pytanie |
| --- | --- |
| **CVE** — Common Vulnerabilities and Exposures | Jaki identyfikator ma podatność? CVE nie jest oceną ryzyka |
| **CVSS** — Common Vulnerability Scoring System | Jakie są techniczne cechy i dotkliwość podatności? Wynik wymaga interpretacji z uwzględnieniem użytych metryk |
| **EPSS** — Exploit Prediction Scoring System | Jakie jest szacowane prawdopodobieństwo wykorzystania CVE w rzeczywistych atakach w ciągu kolejnych 30 dni? |
| **CISA KEV** — Known Exploited Vulnerabilities | Czy istnieją dowody wykorzystania tej podatności w rzeczywistych atakach? |
| **Reachability** | Czy podatna funkcja lub ścieżka jest osiągalna w naszym zastosowaniu? |
| **Exposure** | Kto może dotrzeć do podatnego komponentu, np. przez Internet? |
| **Business impact** | Co utracimy, jeśli dojdzie do skutecznego ataku? |

**Severity nie jest tym samym co priorytet naprawy.**

Przykład: podatność HIGH w publicznym, osiągalnym komponencie, aktywnie wykorzystywana przez atakujących, może wymagać szybszej reakcji niż CRITICAL w nieużywanej funkcji odizolowanego środowiska. Taka decyzja wymaga dowodów.

Niski EPSS ani brak CVE w KEV nie dowodzą bezpieczeństwa. Brak informacji o poprawce także nie oznacza braku ryzyka.

## 9. Hardening i supply chain

**Hardening** to ograniczanie powierzchni ataku i zbędnych uprawnień.

Przykłady dla kontenerów i Kubernetes:

- uruchamianie aplikacji jako użytkownik non-root;
- `allowPrivilegeEscalation: false`;
- usunięcie zbędnych Linux capabilities;
- system plików tylko do odczytu, z kontrolowanymi miejscami do zapisu;
- profil seccomp `RuntimeDefault`;
- limity zasobów i możliwie małe uprawnienia RBAC;
- mały, utrzymywany obraz uruchomieniowy bez zbędnych narzędzi.

**Multi-stage build** oddziela budowanie od uruchamiania: narzędzia potrzebne do zbudowania aplikacji nie muszą trafiać do gotowego obrazu.

**Supply chain security** dotyczy m.in. zależności, narzędzi CI, artefaktów i ich pochodzenia:

| Mechanizm | Zastosowanie |
| --- | --- |
| SBOM | Inwentarz komponentów |
| Przypięcie akcji do SHA | Odwołanie do konkretnej wersji kodu akcji |
| Lockfile i hashe | Lepsza powtarzalność i kontrola zależności |
| Sigstore | Podpisywanie i weryfikacja artefaktów oraz attestations |
| SLSA | Ramy oceny zabezpieczeń procesu budowania i provenance |

Sam SBOM nie oznacza wdrożenia SLSA, a podpis nie dowodzi, że obraz nie ma podatności.

## 10. Praktyczny przykład z projektu

Projekt: [Azure Kubernetes Agentic Ops](https://github.com/JeanneBM/azure-kubernetes-agentic-ops).

Dodana warstwa bezpieczeństwa CI:

| Obszar | Implementacja |
| --- | --- |
| Testy | pytest |
| SAST | Bandit |
| Zależności | pip-audit |
| Sekrety, obraz i IaC | Trivy |
| Inwentarz | SBOM CycloneDX |
| Aktualizacje | Konfiguracja Dependabota |
| Decyzja CI | Końcowy Security gate |
| Obsługa zgłoszeń | Rejestr podatności i instrukcja triage |

### Co wykazała weryfikacja

1. Skanowanie wykryło podatności obrazu bazowego Debiana i narzędzi budowania.
2. Narzędzia budowania zostały oddzielone od obrazu uruchomieniowego.
3. Obraz bazowy został zmieniony na aktualizowany Alpine.
4. Dodano test importów zależności i odpowiedzi `/healthz` w kontenerze, z zamockowanymi adapterami chmurowymi.
5. Ponowne CI przeszło wszystkie kontrole, w tym skan obrazu i końcowy gate.

Dowody:

- [PR z naprawą obrazu i testem uruchomieniowym](https://github.com/JeanneBM/azure-kubernetes-agentic-ops/pull/5)
- [Zielony przebieg CI z 28.10.2026](https://github.com/JeanneBM/azure-kubernetes-agentic-ops/actions/runs/40658799239)

W tym przebiegu skan nie wykrył podatności obrazu HIGH/CRITICAL według zastosowanych reguł i bazy. To historyczny wynik konkretnego artefaktu. Alpine nie jest z definicji bezpieczniejsze w każdym zastosowaniu; zmiana bazy wymaga sprawdzenia kompatybilności.

Skanowanie i testy CI nie wymagały wdrożenia do Azure. Test z mockami nie zastępuje testu integracyjnego na AKS. Projekt nie stanowi wdrożenia enterprise-scale vulnerability management ani automatycznego enrichmentu EPSS/KEV.

### Jak uczciwie opowiedzieć o tym na rozmowie

> W projekcie własnym wdrożyłam kontrole bezpieczeństwa w GitHub Actions: SAST, analizę zależności, skanowanie sekretów, obrazu kontenera i konfiguracji Kubernetes. Dodałam SBOM, końcowy security gate i rejestr podatności. Zweryfikowałam wykrycia, zmieniłam obraz uruchomieniowy i potwierdziłam naprawę ponownym skanem oraz testem działania kontenera.

Jeżeli pytanie dotyczy doświadczenia produkcyjnego lub skali przedsiębiorstwa, należy osobno wskazać rzeczywiste doświadczenie zawodowe. Projekt własny nie jest wdrożeniem produkcyjnym u pracodawcy.

## 11. Szybka powtórka przed rozmową

| Pytanie | Krótka odpowiedź |
| --- | --- |
| Czy Jenkins sam skanuje kod? | Uruchamia i integruje skanery w pipeline |
| Czym różnią się SAST, SCA i DAST? | Kod, zależności, działająca aplikacja |
| Do czego służy Trivy? | M.in. do skanowania podatności komponentów, sekretów i konfiguracji oraz generowania SBOM |
| Co to jest SBOM? | Inwentarz komponentów; nie jest sam w sobie raportem podatności |
| Czy CRITICAL zawsze naprawiamy najpierw? | Priorytet uwzględnia też wykorzystanie, osiągalność, ekspozycję i wpływ biznesowy |
| Co robi security gate? | Egzekwuje skonfigurowane warunki przejścia kontroli |
| Czy czerwony workflow blokuje merge? | Dopiero wymagany check w regułach repozytorium zapewnia taką blokadę |
| Co zrobić z tysiącem alertów? | Deduplikować, zweryfikować, ustalić ryzyko, właścicieli i terminy, następnie śledzić naprawy |
| Co zrobić z podatnością bez poprawki? | Ocenić ryzyko, rozważyć usunięcie/zastąpienie komponentu lub ograniczenie ekspozycji; ewentualną akceptację udokumentować |
| Czy zielone CI oznacza pełne bezpieczeństwo? | Oznacza spełnienie określonych kontroli w konkretnym przebiegu |

## Pozostałe działy repozytorium

- [SAST](SAST/README.md)
- [Hardening](hardening/README.md)
- [Azure](azure/README.md)
- [AI agents / AIDEAS](ai-agents/AIDEAS/README.md)
- [EITCA-IS](EITCA-IS/README.md)

## Materiały źródłowe

- [OWASP](https://owasp.org/) i [OWASP Top 10](https://owasp.org/www-project-top-ten/)
- [OWASP Dependency-Check](https://owasp.org/www-project-dependency-check/)
- [ZAP](https://www.zaproxy.org/)
- [Trivy — dokumentacja](https://trivy.dev/docs/latest/)
- [Trivy — generowanie SBOM](https://trivy.dev/docs/latest/guide/supply-chain/sbom/)
- [Bandit](https://bandit.readthedocs.io/)
- [pip-audit](https://github.com/pypa/pip-audit)
- [GitHub — bezpieczeństwo kodu](https://docs.github.com/en/code-security)
- [GitHub — chronione branche](https://docs.github.com/en/repositories/configuring-branches-and-merges-in-your-repository/managing-protected-branches/about-protected-branches)
- [FIRST — CVSS](https://www.first.org/cvss/)
- [FIRST — EPSS](https://www.first.org/epss/)
- [CISA — KEV](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [CycloneDX](https://cyclonedx.org/) i [SPDX](https://spdx.dev/)
- [DefectDojo](https://docs.defectdojo.com/), [SLSA](https://slsa.dev/) i [Sigstore](https://www.sigstore.dev/)
