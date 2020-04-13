---
layout: post
title:  "ADSD - NServiceBus - C# - F# - kwartet doskonały"
description: Minęły trzy miesiące od czasu, kiedy zdefiniowałem i upubliczniłem plany, związane z kierunkiem rozwoju technologicznego, w którym podążam. Jak na razie nic się nie zmieniło. Z każdym kolejnym tygodniem umacniam się w przekonaniu, że jest to słuszna droga. W tym artykule podzielę się z Tobą garstką historii oraz obecnymi wydarzeniami, które związane są z czterema fundamentami, na których bazuje wszystko inne.
date:   2020-04-13
---

Minęły trzy miesiące od czasu, kiedy zdefiniowałem i upubliczniłem [plany ][1] związane z kierunkiem rozwoju technologicznego, w którym podążam. Jak na razie nic się nie zmieniło. Z każdym kolejnym tygodniem umacniam się w przekonaniu, że jest to słuszna droga. W tym artykule podzielę się z Tobą garstką historii oraz obecnymi wydarzeniami, które związane są z czterema fundamentami, na których bazuje wszystko inne:

* kursem **A**dvanced **D**istributed **S**ystem **D**esign
* frameworkiem **NServiceBus**
* językiem **C#**
* językiem **F#**

## ADSD - Advanced Distributed System Design

Jest to kurs, wokół którego buduję, a dokładniej mówiąc, poszerzam swoją wiedzę na temat projektowania oraz realizacji systemów rozproszonych. Dlaczego poszerzam? W 2015 roku widziałem nagrany materiał. Wcześniej bazowałem na informacjach z [bloga][2] oraz publicznych [nagrań][3] twórcy kursu - **Udi'ego Dahan'na**. Kurs porządkuje całą wiedzę w jedną spójną całość, a także zawiera dodatkowe kluczowe elementy. Materiał jest bardzo obszerny i bardzo merytoryczny.

W obecnej sytuacji, w której się znaleźliśmy, mamy możliwość obejrzenia nagrań z całego kursu za tzw. **free**. Po zarejestrowaniu się na stronie dostajemy dostęp na określony limitowany czas. Więcej szczegółów znajdziesz [w tym wpisie][4].

Zachęcam Cię do przejścia przez kurs. Znajdziesz tam dużo wartości. Jedno tylko małe ostrzeżenie. Po zapoznaniu się materiałem, świat IT nie wygląda już tak samo ;)

## NServiceBus

Tak jak pewne uniwersalne elementy architektonicznie lub projektowe pozostają aktualne przez lata, tak schodząc na poziom konkretnych narzędzi, sytuacja zupełnie się odwraca. Języki, biblioteki, frameworki, technologie cały czas się zmieniają lub...przestają być rozwijane. Taka jest naturalna kolej rzeczy. Pamiętam, jak w 2014 roku uruchamialiśmy pierwszy podsystem zrealizowany z użyciem **NServiceBus'a** w wersji **4**. Potem przyszła kolej na **NServiceBus v5**. Kolejne projekty, kolejne systemy, tym razem zrealizowane z użyciem **NServiceBus v6**. Wszystkie produkty, które powstały, miały jedną wspólną cechę - były realizowane w wykorzystaniem [transportu MSMQ][5], który był pierwszym systemem kolejkowy, którego używał **NServiceBus**. Na chwilę obecną wygląda na to, że [czas MSMQ dobiegł końca][6]. W takiej sytuacji **NServiceBus** pokazuje swoją siłę, ponieważ jest on abstrakcją na różne [systemy kolejkowe][7]. Obecna wersja **NServiceBus v7** współpracuje zarówno z rozwiązaniami [On-premises][8], jak i z rozwiązaniami chmurowymi. Dzięki takim możliwością możemy w miarę bezbolesny sposób przesiąść się z jednego transportu na inny, bez konieczności radykalnej zmiany kodu wysyłającego oraz przetwarzającego wiadomości.

Jeśli chcesz szybko zapoznać się z możliwościami, jakie daje **NServiceBus** polecam Ci przejść przez [tutorial ze strony Frameworka][10]. Więcej informacji na temat całej **Platformy Particular** znajdziesz w tzw. [Learning Path][11].

## C#

Obecnie numer jeden, jeśli chodzi o język programowania na platformie **.NET**. Pierwotnie stworzony dla programowania w stylu [Object-Oriented Programming][12]. Od wersji **3.0** umożliwia również programowanie w stylu [Functional Programming][13]. Od wersji **4.0** dopuszcza programowanie z pominięciem statycznego typowania za pomocą słowa kluczowego [dynamic][14]. **C#** towarzyszy mi nieprzerwanie od momentu, kiedy napisałem w nim pierwszą linijkę kodu, aż do dziś. Dzięki językowi szlifowałem i nadal szlifuję programistyczne umiejętności. W parze z programowaniem rozwiązań zawsze idą elementy związane z ich projektowaniem. W moim przypadku, w początkowej fazie, były to elementy dookoła architektury 3-warstwowej z klasycznym podziałem na:

* **UI** - User Interface Layer
* **BLL** - Business Logic Layer
* **DAL** - Data Access Layer

Po czasie doszła czwarta warstwa **API** co przekształciło się w projektowanie oraz programowanie w stylu architektonicznym **RPC** (Remote Procedure Call). Wynikiem tego powstały rozwiązania posiadające w warstwie **BLL** np. klasę `ItemManager`, która zawierała kilkadziesiąt metod związanych z logiką biznesową. Podobnie było na warstwie **DAL** np. `ItemDatabaseMapper` z wieloma operacjami dotyczącymi operacji na bazie danych. Takie podejście umożliwia współdzielenie kodu w łatwy sposób, co przekłada się z tworzenie zależności, co z kolei wiąże się **utrudnionym wprowadzaniem zmian**. Z powodu architektury warstwowej nigdy nie miałem konieczność tworzenia modelu z prawdziwego zdarzenia, ponieważ potrzebne dane przerzucane były pomiędzy poszczególnymi warstwami w jakiejkolwiek, jak najprostszej postaci.

Plusem architektury warstwowej jest jej niska bariera wejścia. Daje możliwość łatwego nauczania języka **C#**. Pokazuje wartość z używania interfejsów jako zależności pomiędzy klasami znajdującymi się na każdej warstwie. Można na jej bazie uczyć **Unit Testów**, **Mock'owania obiektów** itp.

Natrafienie na materiały dookoła frameworka **NServiceBus**, a także zapoznanie się kursem **ADSD** mówiąc delikatnie, zburzyły mój pogląd na wytwarzanie oprogramowania, oczywiście na plus :) Z podejścia warstwowego przeszedłem na projektowanie oraz programowanie komponentowe. Koncepcja oraz implementacja [Sagi][15] pokazała mi, w jaki sposób powinien wyglądać i czym powinien charakteryzować się prawdziwy zorientowany obiektowo komponent:

* odpowiedzialny za całą realizację konkretnego kawałka funkcjonalności
* posiadający swój własny persystentny stan, do którego tylko on sam ma dostęp
* komunikujący się z innymi komponentami w najbardziej jak to jest możliwe luźny sposób - **messaging**

**C#** umożliwia naukę programowania w podejściu zorientowanym obiektowo oraz w podejściu funkcyjnym. W pewnych zastosowaniach pozwala uprościć kod, poprzez zastosowanie typu `dynamic`. Na stronie [dokumentacji][16] znajdziesz wiele materiałów dotyczących różnych zagadnień związanych z językiem.

## F#

Na koniec element, którym zajmuję sie dopiero od paru miesięcy. **F#**, język pozwalający w pełni wykorzystać możliwości programowania funkcyjnego, który zawiera również elementy związane z programowaniem obiektowym, ale uwaga, nie mylić z programowaniem zorientowanym obiektowo. Tak, jak w języku **C#** programując zorientowanie obiektowo, możemy wspomagać się konstrukcjami funkcyjnymi np. [Lambda Expressions][17], tak w języku **F#** możemy programować funkcyjne, wspomagając się konstrukcjami obiektowymi np. [Classes][18]. Zagłębiając się coraz bardziej w świat języka **F#**, odnajduję w nim koncepcje, których nie widziałem w świecie języka **C#**, jak np. **kompilowany SQL**, typy **Option**, **Record** oraz **Union**, operator **Pipe** itd. W moim przypadku samo podejście funkcyjne jest bardziej naturalne niż podejście zorientowane obiektowo, dlatego chętniej zapoznaję się z kolejnymi konstrukcjami i elementami związanymi z językiem **F#**.

Jeśli chcesz zobaczyć w jaki sposób język **F#** współpracuje z frameworkiem **NServiceBus** oraz jakie motywy stały za podjęciem próby połączenia tych dwóch elementów zapraszam Cię do mojego [praktycznego przewodnika][19].

## Podsumowanie

**ADSD**, **NServiceBus**, **C#**, **F#**, to cztery fundamenty, na których można budować rozwiązania, zarówno te proste, jak i te bardziej złożone. Aktualnie sprawdzam i testuję różne techniki, które pozwalają wykorzystać właściwości wszystkich czterech elementów do tworzenia końcowych produktów. Wnioskami i przemyśleniami będę dzielił się na blogu. Tymczasem, udanego oglądania materiałów z kursu [ADSD][4], a także nabywania oraz podnoszenia umiejętności w dziedzinach, którymi się interesujesz.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2020-01-12-what-was-what-is-what-will-be.md %}
[2]: http://udidahan.com/ "Udi Dahan Website"
[3]: https://skillsmatter.com/members/udidahan#skillscasts "Udi Dahan Skillcasts"
[4]: https://discuss.particular.net/t/free-for-a-limited-time-advanced-distributed-systems-design-videos/1708 "free for a limited time advanced distributed systems design videos"
[5]: https://docs.particular.net/transports/msmq/ "MSMQ Transport"
[6]: https://particular.net/blog/msmq-is-dead "MSMQ is dead"
[7]: https://docs.particular.net/transports/ "NServieBus Transports"
[8]: https://en.wikipedia.org/wiki/On-premises_software "On premises software"
[9]: https://docs.particular.net/persistence/ "NServiceBus Persistence"
[10]: https://docs.particular.net/tutorials/nservicebus-step-by-step/ "NServiceBus tutorial step by step"
[11]: https://particular.net/learn/getting-started "Particular Platform learning path"
[12]: https://en.wikipedia.org/wiki/Object-oriented_programming "Object-oriented programming"
[13]: https://en.wikipedia.org/wiki/Functional_programming "Functional programming"
[14]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/using-type-dynamic "C# using type dynamic"
[15]: https://docs.particular.net/tutorials/nservicebus-sagas/ "NServiceBus Saga"
[16]: https://docs.microsoft.com/en-us/dotnet/csharp/ "C# documentation"
[17]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/statements-expressions-operators/lambda-expressions "C# lambda expressions"
[18]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/classes "F# Classes"
[19]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}