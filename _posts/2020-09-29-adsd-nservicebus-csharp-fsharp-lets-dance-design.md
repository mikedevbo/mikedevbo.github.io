---
layout: post
title:  "ADSD - NServiceBus - C# - F# - let's dance - Design"
description: Koniec lata, początek jesieni. Jak patrzę wstecz na daty publikacji artykułów, to widzę, że okres jesienno-zimowy sprzyja mi najbardziej w pisaniu oraz publikowaniu nowych treści. Zobaczymy, jak będzie w tym sezonie :) Zapraszam Cię do przeczytania mini serii, w której pokażę Ci, jak kwartet doskonały radzi sobie w warunkach praktycznych. Przejdziemy przez zmiany, jakich dokonałem w systemie komentarzy, którego używam na tym blogu. W tej części skupimy się na koncepcji oraz projekcie rozwiązania. W następnej części przejdziemy przez wybrane elementy implementacji.
date:   2020-09-29
---

Posty z tej serii:

* [ADSD - NServiceBus - C# - F# - let's dance - Design][15]
* [ADSD - NServiceBus - C# - F# - let's dance - Develop][16]

Koniec lata, początek jesieni. Jak patrzę wstecz na daty publikacji artykułów, to widzę, że okres jesienno-zimowy sprzyja mi najbardziej w pisaniu oraz publikowaniu nowych treści. Zobaczymy, jak będzie w tym sezonie :) Zapraszam Cię do przeczytania mini serii, w której pokażę Ci, jak [kwartet doskonały][1] radzi sobie w warunkach praktycznych. Przejdziemy przez zmiany, jakich dokonałem w [systemie komentarzy][2], którego używam na tym blogu. W tej części skupimy się na koncepcji oraz projekcie rozwiązania. W następnej części przejdziemy przez wybrane elementy implementacji.

Design [pierwszej wersji][2] bazował na koncepcji [Sagi][3], która pełniła rolę koordynatora pomiędzy przyjęciem komentarza od użytkownika, a jego weryfikacją oraz odpowiedzią przez autora. Komunikacja pomiędzy Sagą, a poszczególnymi Message Handlerami odbywała się za pomocą wiadomości typu **Event**. W [następnej wersji][4] Saga otrzymała jeszcze większe uprawnienia koordynacji poprzez zastosowanie komunikacji typu **Request/Response**. Takie podejście idealnie nadaje się w scenariuszach integracji między różnymi systemami. Saga lub zbiór Sag, przyjmują dane z jednego systemu, a następnie koordynują ich przepływ do innych systemów. W stylu architektonicznym **SOA** (Service Oriented Architecture) wg interpretacji [ADSD][5], a ściśle mówiąc autora kursu, taka funkcjonalność leży w odpowiedzialności specyficznej, typowo technicznej Usługi, zwanej **IT/OPS**. Więcej informacji na temat rodzaju takiej Usługi znajdziesz [tu][6] i [tu][7].

**Usługa** jest pojęciem logicznym. W ramach Usługi zgrupowane są określone funkcjonalności, dane, procesy, itd. Kod należący do danej Usługi jest technicznym rozwiązaniem realizującym tzw. **Business Capability**. Dane biznesowe **nie mogą** być duplikowane pomiędzy Usługami, a komunikacja między nimi **musi** odbywać się w najbardziej możliwie luźny sposób np. **Messaging** z wykorzystaniem wiadomości typu **Event**. Wewnątrz Usługi możemy stosować różne inne style architektoniczne np. **Domain Model**, **CQRS** itd. Wyodrębnienie Usług nie jest prostym zadaniem. Nadanie nazw Usługom jest jeszcze trudniejsze.

Kiedy przeprojektowywałem system komentarzy na blogu, próbowałem podzielić całość na Usługi, ale okazało się, że jest to zbyt mała funkcjonalność na tego rodzaju podział. Skuteczniejszym podejściem było wyodrębnienie **Business Policies** wraz z danymi, potrzebnymi do ich realizacji, zdefiniowanie rodzaju komunikacji między nimi i dopiero na końcu logiczne pogrupowanie w Usługi. Poniższy diagram przedstawia końcowe rozwiązanie:

[![Picture1]({{ site.url }}/assets/adsd_nservicebus_csharp_fsharp_lets_dance_I/SolutionDesign.svg)][12]

**Comment Registration** odpowiada za zarejestrowanie komentarza. Technicznie chodzi o utworzenie **Pull Requesta** na **GitHubie**. Zajmuje się tym **GitHub Pull Request Creation**. Za sprawdzenie, czy pojawiła się odpowiedź na komentarz, odpowiada **Comment Answer**. Technicznie chodzi o sprawdzenie, czy Pull Request został obsłużony. Zajmuje się tym **GitHub Pull Request Verification**. Za poinformowanie czytelnika o ukazaniu się odpowiedzi na komentarz odpowiada **Comment Answer Notification**. Przyjmowaniem danych wejściowych zajmuje się **Comment Controller**, który jako jedyny jest komponentem Webowym. Jest jeszcze **Comment Taking**. Wrócimy do niego za chwilę.

W poprzedniej wersji systemu wszystkie te elementy były realizowane przez jedno Policy, zwane **Comment**. Dzięki podziałowi poszczególne funkcjonalności są od siebie bardziej niezależne, co ułatwi wprowadzanie ewentualnych zmian. Każde Policy **operuje na własnym zestawie danych**. Komunikacja pomiędzy Policies odbywa się **w maksymalnie luźny sposób** - za pomocą Messagingu. **Biznesowe Policies**, po zakończeniu realizacji swojej części, publikują wiadomości typu **Event**. **Techniczne Policies** przyjmują Message typu Request, a zwracają odpowiedź, jako Message typu Response. Dane biznesowe **nie są duplikowane pomiędzy Policies**. Jedynym łącznikiem pomiędzy nimi są stabilne elementy, które po utworzeniu nie ulegają zmianom np. identyfikatory, daty, URLe, itp. 

W tym miejscu możemy wrócić do pojęcia Usługi. Poprzednio całe rozwiązanie należało do Usługi **IT/OPS**. Obecnie jej częścią są **Comment Controller**, **GitHub Pull Request Creation** oraz **GitHub PullRequest Verification**, ponieważ wykonują typowo techniczne zadania. **Comment Registration**, **Comment Answer** oraz **Comment Answer Notification** wykonują zadania biznesowe.

W jaki sposób podzielić Business Policies na Usługi? Nie ma jednoznacznej metody. Najprościej byłoby umieścić wszystkie trzy komponenty w jednej Usłudze, jednak intuicyjnie obsługa komentarza jest czymś innym niż powiadomienia do użytkownika. W najprostszej wersji jest to wysłanie maila z tekstem. W bardziej złożonym scenariuszu mail mógłby być rozpoczęciem programu lojalnościowego, gdzie byłyby przyznawane punkty za merytoryczne komentarze. Idąc dalej, doszlibyśmy do wniosku, że wysyłka maila jest tylko technicznym aspektem, a początek Policy związanego z przyznawaniem punktów czytelnikowi zaczyna się w momencie akceptacji jego pierwszego komentarza. W związku z tym umieściłem **Comment Registration** oraz **Comment Answer** w jednej Usłudze, a **Comment Answer Notification** w drugiej, osobnej Usłudze.

W jaki sposób nazwać zdefiniowane Usługi? Ponownie kusi, aby zrobić to wprost, czyli pierwszą z nich nazwać **Comment Service**, a drugą **Notification Service**. W przypadku tej drugiej przeanalizowaliśmy, że powiadomienia mogą być częścią czegoś większego. W przypadku tej pierwszej możemy stwierdzić, że przyjmowanie komentarzy to tylko jeden z kanałów wejściowych. Innymi kanałami mogą być ankiety, emaile itd. Wszystkie te elementy mogą należeć do jednej Usługi odpowiedzialnej za interakcję z użytkownikami. W związku z tym najlepiej nazywać usługi kolorami i tak pierwszą nazwałem **Green Service**, a drugą **Orange Service**. Z czasem, kiedy funkcjonalności w ramach Usług stabilizują się, można próbować nadawać im konkretniejsze nazwy.

Wróćmy do komponentu **Comment Taking**. Pełni on funkcję tzw. kleju pomiędzy Usługami. Dane, które obsługuje, jako wiadomości typu **Command**, pochodzą z różnych Usług. Jego odpowiedzialnością jest wyodrębnienie (z wiadomości) danych z poszczególnych Usług i wysłanie ich w odpowiednie miejsca. Elementem korelującym dane z różnych Usług jest **CommentId**. Dzięki temu **Event** publikowany przez **Comment Answer** może zawierać minimalną ilość informacji - tylko **CommentId**. Więcej na temat takiego sposobu projektowania **Eventów** znajdziesz w artykule [Putting your events on a diet][13] oraz [prezentacji][14] o tym samym tytule. **Comment Taking** wykonuje typowo techniczne zadanie, dlatego idealnie pasuje, aby umieścić go w Usłudze **IT/OPS**.

Przy projektowaniu rozwiązania warto mieć na uwadze, że z czasem pewne elementy mogą migrować pomiędzy Usługami. W takim przypadku idealnie, jeśli przenoszona funkcjonalność jest maksymalnie niezależna i można ją przenieść bez konieczności "zabierania" za sobą innych funkcjonalności. Dobrze zdefiniowane Policy umożliwia taką migrację.

Tak wygląda projekt rozwiązania systemu komentarzy na blogu wg idei zawartych w krusie [ADSD][5] - **Advanced Distributed System Design**. Pytanie, jakie możesz sobie zadać to *"OK, ale w jaki sposób można to zaimplementować?"*. W tym momencie do gry wchodzi drugi z wielkiej czwórki, czyli framework [NServiceBus][8]. **Business Policy** możemy przełożyć jeden do jeden jako implementacja [Sagi][3]. Komunikacja za pomocą **Messagingu**? Jest to podstawa działania **NServiceBusa**, który obsługuje [trzy rodzaje komunikatów][9]: **Command**, **Event** oraz **Message**. A co z Usługami? Pamiętajmy, że Usługa jest pojęciem **logicznym**. Nie ma nic wspólnego z technologią. Pozwala grupować poszczególne elementy w logiczne całości, stawiając pomiędzy nimi granice odpowiedzialności.

W następnym artykule pokażę Ci, w jaki sposób połączyć kod języka [C#][10] z kodem języka [F#][11], aby zrealizować zaprojektowaną funkcjonalność.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2020-04-13-adsd-nservicebus-csharp-fsharp-perfect-quartet.md %}
[2]: {{ site.url }}{% link _posts/2018-03-11-blog_comments_I_Design.md %}
[3]: https://docs.particular.net/nservicebus/sagas/ "NServiceBus Saga"
[4]: {{ site.url }}{% link _posts/2018-09-30-nservicebus-saga-request-response-ddd-aggregate-root.md %}
[5]: https://learn.particular.net/courses/adsd-online "Advanced Distributed System Design"
[6]: https://udidahan.com/2012/12/10/service-oriented-api-implementations/ "service-oriented-api-implementations"
[7]: https://udidahan.com/2014/03/31/on-that-microservices-thing/ "on-that-microservices-thing"
[8]: https://docs.particular.net/nservicebus/ "NServiceBus"
[9]: https://docs.particular.net/nservicebus/messaging/messages-events-commands "messages commands events"
[10]: https://docs.microsoft.com/en-us/dotnet/csharp/ "C# language"
[11]: https://fsharp.org/ "F# language"
[12]: {{ site.url }}/assets/adsd_nservicebus_csharp_fsharp_lets_dance_I/SolutionDesign.svg
[13]: https://particular.net/blog/putting-your-events-on-a-diet "putting-your-events-on-a-diet"
[14]: https://skillsmatter.com/skillscasts/2990-events-diet "2990-events-diet"
[15]: {{ site.url }}{% link _posts/2020-09-29-adsd-nservicebus-csharp-fsharp-lets-dance-design.md %}
[16]: {{ site.url }}{% link _posts/2020-10-20-adsd-nservicebus-csharp-fsharp-lets-dance-develop.md %}