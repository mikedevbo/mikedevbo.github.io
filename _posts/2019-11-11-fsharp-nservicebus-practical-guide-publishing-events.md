---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń"
description: F# oraz NServiceBus. Czy to możliwe? Przeczytaj o tym, jak używając języka F# można zaprogramować publikowanie wiadomości typu Event.
date:   2019-11-11
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][12]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][13]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][14]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][15]
* [F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń][16]
* [F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga][17]
* [F# & NServiceBus - praktyczny przewodnik: Zakończenie][18]

Powoli zbliżamy się do końca niniejszej serii. Jednak zanim to nastąpi, przejdziemy jeszcze przez dwa zagadnienia. W tym artykule zajmiemy się kolejnym typem wiadomości obsługiwanym przez framework **NServiceBus**, a mianowicie **Zdarzeniami**. Tak jak w poprzednio, tak i teraz kontynuujemy rozbudowę **Retail E-commerce System**. Jeśli masz swój kod napisany na bazie poprzednich części, możesz go dalej rozwijać. Jeśli nie posiadasz swojego kodu, możesz pobrać przykład, który udostępniam na moim [GitHubie][1].

Przykłady bazują na kolejnej części [tutoriala][2] wprowadzającego do frameworka **NServiceBus**. Otwórz link i przejdź do sekcji **Exercise**, która zawiera diagram pokazujący końcowe rozwiązanie. Zobaczysz na nim zależności pomiędzy poszczególnymi **Endpointami**.

## Utworzenie zdarzenia

Czym jest **Zdarzenie (Event)**? Informacją o tym, że akcja została wykonana - reprezentuje przeszłość. Jest odwrotnością **Commanda**, który inicjuje wykonanie akcji - reprezentuje przyszłość. Często **Command** oraz **Event** występują w parze przy realizacji **User Story**:

* wysłanie **Commanda** np. **DoSomething** inicjuje realizację logiki biznesowej
* logika biznesowe jest realizowana
* opublikowanie **Eventa** np. **SomethingHappened** informuje o tym, że logika biznesowa została zrealizowana

Wiadomości typu **Command** powinny być nazywane w trybie rozkazującym - ***zrób coś***. Z kolei wiadomości typu **Event** powinny być nazywane w czasie przeszłym - ***coś zostało zrobione***.

**Command** zawiera jednego logicznego odbiorcę. Może być wysłany przez wielu logicznych nadawców.
<br />
**Event** zawiera jednego logicznego nadawcę. Może być odebranych przez zero lub więcej logicznych odbiorców.

Przejdź do sekcji **Events** [tutoriala][2], aby porównać, w formie tabeli, różnicę między wiadomością typu **Command**, a wiadomością typu **Event**.

Używanie wiadomości typu **Command**, wysyłanych pomiędzy różnymi **Endpointami** pozwala na budowanie systemów składających się z luźno ze sobą powiązanych komponentów. Używanie wiadomości typu  **Event** pozwala zrobić kolejny krok do przodu i jeszcze bardziej te zależności poluzować. Jak to jest możliwe? Już tłumaczę.

Weźmy za przykład system dodawania komentarzy do bloga. W [mojej przykładowej implementacji][3] wysyłanie maila do autora komentarza z informacją o tym, że dodałem odpowiedź, jest częścią procesu utworzenia oraz obsługi komentarza. Inicjowanie wysyłki maila zaimplementowane jest jako wysłanie wiadomości typu **Command**. Gdybym teraz chciał dodać nową funkcjonalność np. *"powiadom inne zainteresowane osoby o pojawieniu się odpowiedzi na komentarz"* musiałbym wysłać kolejną wiadomość typu **Command**. Wiązałoby się to z **ingerencją w istniejący kod, jego ponowne testy oraz nowe wdrożenie całego komponentu**. Podobne kroki, musiałbym wykonać implementując kolejne wymagania.

Używając wiadomości typu **Event** mogę opublikować zdarzenie informujące o tym, że pojawiła się odpowiedź na komentarz. Następnie mogę dodać dwóch niezależnych odbiorców tego zdarzenia. W jednym zakodować logikę wysyłki maila do autora komentarza. W drugim zakodować logikę wysyłki maila do wszystkich innych zainteresowanych. W ten sposób przy, kodowaniu nowych wymagań, **nie muszę ingerować w kod nadawcy**, a także w kod istniejących odbiorców. Dodaję nowego odbiorcę, subskrybuję się na publikowany przez nadawcę **Event** i wykonuję logikę zgodną z wymaganiem. Jeśli nowego odbiorcę implementuję w osobnym **Endpoincie** to nie **muszę ponownie wdrażać kodu nadawcy**.

Wrzucam na swój backlog zadanie refactoringu systemu komentarzy do bloga do takiej postaci :)

W kontekście projektowania rozwiązań na bazie [Domain Driven Design][4] mogę powiedzieć, że wynoszę funkcjonalność wysyłki maili poza [Agregat][5] odpowiedzialny za przyjmowanie oraz obsługę komentarzy. W ten sposób **Agregat** staje się jeszcze bardziej zgodny z definicją [Single Responsibility Principle][6]. Więcej o takim podejściu możesz przeczytać w jednym z moich [poprzednich artykułów][7].

Przejdźmy teraz do naszego **Retail E-commerce System** i wykorzystajmy **zdarzenia** do implementacji nowego wymagania - *"pobierz opłatę za złożone zamówienie"*. Funkcjonalność pobierania opłaty zakodujemy w nowym **Endpoincie**. Będzie ona reagowała na **zdarzenie** złożenia zamówienia. W pierwszej kolejności stwórzmy **Event** reprezentujący złożone zamówienie. W tym celu w Visual Studio:

1. dodaj do projektu **Messages** nowy pliki o nazwie **Events.fs**
2. usuń domyślnie utworzony kod
3. dodaj poniższy kod

{% highlight fsharp %}
namespace Events

open NServiceBus

type OrderPlaced(orderId: string) =
    interface IEvent
    member this.OrderId = orderId
{% endhighlight %}

Kod **F#** jest dokładnie taki sam jak przy tworzeniu wiadomości typu **Command**. Jedyną różnicą jest to, że w tym przypadku dziedziczymy po interfejsie `IEvent`, który tak samo, jak interfejs `ICommand`, należy do przestrzeni nazw `NServiceBus`.

## Publikowanie zdarzeń

Mając zdefiniowane zdarzenie, możemy je opublikować:

1. przejdź do pliku **Handlers.fs** w projekcie **Sales**
2. zastąp klasę `PlaceOrderHandler` poniższą definicją
3. doprowadź kod do stanu kompilacji dodając odpowiednie sekcje `open`

{% highlight fsharp %}
type PlaceOrderHandler() =
    static member log = LogManager.GetLogger<PlaceOrderHandler>()
    interface IHandleMessages<PlaceOrder> with
        member this.Handle(message, context) =
            PlaceOrderHandler.log.Info(sprintf "Received PlaceOrder, OrderId = %s" message.OrderId)
            let orderPlaced = new OrderPlaced(message.OrderId)
            context.Publish(orderPlaced)
{% endhighlight %}

W powyższym przykładzie zastąpiliśmy fragment kodu `Task.CompletedTask`, który kończył działanie **member**, kodem, który tworzy i publikuje zdarzenie **OrderPlaced**:

{% highlight fsharp %}
let orderPlaced = new OrderPlaced(message.OrderId)
context.Publish(orderPlaced)
{% endhighlight %}

Zdarzenie **OrderPlaced** jest obiektem, który przekazywany jest do metody `context.Publish`, wchodzącej w skład API frameworka **NServiceBus**. Metoda `context.Publish` zwraca typ `Task`, który jest zgodny z typem zwracanym przez metodę `Handle`. Dzięki temu nie musimy jawnie używać `Task.CompletedTask`.

## Przetwarzanie zdarzeń

Wysyłając wiadomość typu **Command** musimy podać adres **Endpointa** na jaki wiadomość ma zostać wysłana. Wysyłając wiadomość typu **Event** to odbiorca zdarzenia musi zainicjować chęć odbierania tego typu wiadomości. Stwórzmy nowy **Endpoint**, którego zadaniem będzie pobranie opłaty za złożone zamówienie. Nowy endpoint będzie reagował na publikowane zdarzenie **OrderPlaced**. W tym celu w **Visual Studio**:

1. dodaj do solucji **RetailDemo** nowy projekt **Add...New Project...**
2. wybierz typ projektu **Visual F#...NET Core...Console App (.NET Core)**
3. w pole **Name** wpisz nazwę projektu **Billing**
4. zainstaluj paczkę Nuget **NServiceBus**
    * sposób instalacji i aktualizacji paczek znajdziesz w [drugie części][8] niniejszej serii
    * paczki Nuget powinny być w tych samych wersjach co w projekcie **Sales**
5. zainstaluj paczkę Nuget **NServiceBus.Newtonsoft.Json**
6. uaktualnij paczkę Nuget **FSharp.Core** w projekcie **Billing**
7. dodaj referencję do projektu **Messages**
8. dodaj do pliku **Program.fs** kod konfiguracji oraz uruchamiania **Endpointa**
    * możesz wzorować się na kodzie tworzenia Endpointa **Sales** z [poprzedniego artykułu][15]
    * ustaw value **endpointName** na **Billing**
9. dodaj do projektu plik **Handlers.fs**
10. dodaj do pliku **Handlers.fs** poniższy kod
11. doprowadź kod do stanu kompilacji dodając odpowiednie sekcje **open**

{% highlight fsharp %}
module Handlers

type OrderPlacedHandler() =
    static member log = LogManager.GetLogger<OrderPlacedHandler>()
    interface IHandleMessages<OrderPlaced> with
        member this.Handle(message, context) =
            OrderPlacedHandler.log.Info(sprintf "Received OrderPlaced, OrderId = %s - Charging credit card..." message.OrderId)
            Task.CompletedTask
{% endhighlight %}

Sposób przetwarzania wiadomości typu **Event** jest dokładnie taki sam jak sposób przetwarzania wiadomości typu **Command**. Jedyną różnicą jest to, że dziedzicząc po interfejsie `IHandleMessages` podajemy w generycznym parametrze odpowiedni typ zdarzenia - `OrderPlaced`. **NServiceBus** przy starcie Endpointa **Billing** skonfiguruje subskrypcję w taki sposób, aby publikowane zdarzenie **OrderPlaced** trafiło do Handlera **OrderPlacedHandler**.

Ustaw projekt **Billing** tak, aby przy uruchamianiu startował razem z pozostałymi **Endpointami**. Instrukcję znajdziesz [na stronie dokumentacji][9]. Uruchom program i wyślij kilka wiadomości z Endpointa **ClientUI**. Sprawdź efekt przetwarzania zdarzenia **OrderPlaced** przez Endpoint **Billing**:

`INFO  Handlers+OrderPlacedHandler Received OrderPlaced, OrderId = eaa1236e-58ce-4689-b8b1-1ed526336f7f - Charging credit card...`

Dodajmy do naszego systemu kolejną funkcjonalność realizującą wysyłkę złożonego zamówienia. Proces wysyłki powinien być uruchomiony po udanym złożeniu zamówienia oraz po pomyślnym pobraniu za nie opłaty. Mamy już funkcjonalność publikacji zdarzenia, informującą o tym, że zamówienie zostało złożone. Rozszerzymy Endpoint **Billing** o funkcjonalność publikowania zdarzenia o pobraniu opłaty. Proces wysyłki zamówienia jest dobrym kandydatem do zakodowania w osobnym **Endpointcie** reagującym na zdarzenia złożenia zamówienia oraz pobrania opłaty. Cała implementacja sprowadza się do wykorzystania wiedzy z poprzednich artykułów, dlatego jest to dobre ćwiczenie dla Ciebie w celu jej utrwalenia. Wykonaj kroki:

1. dodaj do projektu **Messages** nowe zdarzenie o nazwie **OrderBilled** zawierające member **OrderId**
2. opublikuj zdarzenie  **OrderBilled** w Handlerze **OrderPlacedHandler** należącym do Endpointa **Billing**
3. stwórz nowy Endpoint o nazwie **Shipping**
4. dodaj obsługę zdarzeń **OrderPlacedHandler** oraz **OrderBilledHandler** w Endpoincie **Shipping**

Możesz wspomóc się przykładami dostępnymi na moim [GitHubie][1].

Po uruchomieniu całości powinny wystartować cztery Endpointy. Po wysłaniu wiadomości z Endpointa **ClientUI** w oknie Endpointa **Shipping** powinno pojawić się:

{% highlight txt %}
Handlers+OrderPlacedHandler Received OrderPlaced, OrderId = 12034d63-cc9f-4739-b09a-59ae77ab71a2
- Should we ship now?
Handlers+OrderBilledHandler Received OrderBilled, OrderId = 12034d63-cc9f-4739-b09a-59ae77ab71a2
- Should we ship now?
{% endhighlight %}

Dzięki temu, że logiczny Endpoint **Sales** publikuje zdarzenia, nie trzeba było dokonywać w nim żadnym zmian, w trakcie kodowania Endpointa **Shipping**. Endpoint **Sales** nawet nie wie i nie musi wiedzieć, że istnieją logiczne Endpointy **Billing** oraz **Shipping**. To jest prawdziwa moc zdarzeń w kontekście luzowania zależności pomiędzy poszczególnymi komponentami. Na poziomie fizycznym dostarczaniem wiadomości do odpowiednich Endpointów zajmuje się **NServiceBus**. 

Projektując rozwiązania bazujące na Messagingu oraz Queueingu musimy brać pod uwagę ważną kwestię. **Odbiorca może otrzymywać wiadomości w różnej kolejności, niekoniecznie w tej samej, w jakiej zostały wysłane przez nadawcę**.

Nasze wymaganie odnośnie wysyłki mówi o tym, że produkty możemy wysłać tylko wtedy, kiedy zamówienie zostało złożone oraz została pobrana za nie opłata. Z perspektywy **NServiceBusa** możemy zainicjować wysyłkę tylko wtedy, kiedy otrzymamy dwa zdarzenia: **OrderPlaced** oraz **OrderBilled**. Każde z nich może przyjść w dowolnej kolejności. Jak sobie poradzić z koordynacją takiego zachowania? O tym dowiesz się w następnym artykule, w którym przejdziemy przez właściwość **NServiceBusa** zwaną **Saga**.

Tutorial ze strony frameworka **NServiceBus** zawiera [część][10] dotyczącą obsługi różnego rodzaju sytuacji awaryjnych. Opisana koncepcja oraz sposób implementacji był dla mnie jednym z głównych powodów, dla których zainteresowałem się frameworkiem i zostałem z nim aż do dzisiaj :) Sytuacje, w których takie zachowanie może być pomocne, opisałem w artykule [Wytwarzanie oprogramowania IV - NServiceBus - framework, który zmienia zasady gry][11]. W tej serii pominę ten wątek, natomiast zachęcam Cię do przejścia przez [tutorial][10] i wykonanie opisanych w nim ćwiczeń. Dzięki temu zapoznasz się z koncepcją, a także potrenujesz język **F#**.

[1]: https://github.com/mikedevbo/fsharp-nservicebus "fsharp-nservicebus"
[2]: https://docs.particular.net/tutorials/nservicebus-step-by-step/4-publishing-events/ "publishing events"
[3]: https://github.com/mikedevbo/blog-comments "blog-comments"
[4]: http://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf "DDD Reference"
[5]: https://martinfowler.com/bliki/DDD_Aggregate.html "DDD Aggregate"
[6]: https://en.wikipedia.org/wiki/Single_responsibility_principle "Single Responsibility Principle"
[7]: {{ site.url }}{% link _posts/2018-09-30-nservicebus-saga-request-response-ddd-aggregate-root.md %}
[8]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[9]: https://docs.microsoft.com/en-us/visualstudio/ide/how-to-set-multiple-startup-projects?view=vs-2017&redirectedfrom=MSDN "how to set multiple startup projects"
[10]: https://docs.particular.net/tutorials/nservicebus-step-by-step/5-retrying-errors/ "Retrying Errors"
[11]: {{ site.url }}{% link _posts/2017-04-29-wytwarzanie_oprogramowania_IV.md %}
[12]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[13]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[14]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[15]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}
[16]: {{ site.url }}{% link _posts/2019-11-11-fsharp-nservicebus-practical-guide-publishing-events.md %}
[17]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[18]: {{ site.url }}{% link _posts/2019-12-22-fsharp-nservicebus-practical-guide-ending.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}