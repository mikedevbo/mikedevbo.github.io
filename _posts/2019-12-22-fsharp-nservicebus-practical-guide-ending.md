---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Zakończenie"
date:   2019-12-22
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][7]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][8]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][9]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][10]
* [F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń][11]
* [F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga][12]
* [F# & NServiceBus - praktyczny przewodnik: Zakończenie][13]

Kiedy siadasz do napisania kawałka funkcjonalności, czy zadajesz sobie pytanie, w jaki sposób ułożyć kod, aby daną funkcjonalność zrealizować? Pisząc w paradygmacie ...

Po trzech miesiącach zapoznawania się z językiem **F#** powyższe pytanie nabrało dla mnie innego wymiaru. Rozważania na temat ilości klas, ich wielkości, powiązań między nimi, zostały zastąpione przez rozważania na temat ilości modułów, liczbie funkcji, ich rozmiarze oraz złożoności.

Wykorzystanie frameworka **NServiceBus** jako bazowego narzędzia do konstruowania rozwiązań mocno upraszcza proces decyzyjny:

* definicje wiadomości trzymać w **namespace**:
    * Commands
    * Events
    * Messages
* logicznie powiązane ze sobą **Message Handlery**, trzymać w osobnych **module**
* implementację **Sagi** trzymać w osobnym **module**

Dzięki takiemu podejściu możemy większą uwagę przeznaczyć na poszukiwanie odpowiedzi na pytania:

* używać wiadomości typu **Command** czy **Event**?
* jak wiele **Endpointów** będzie potrzebnych?
* wykorzystać standardowe **Message Handlery** czy  **Sagę**?
* jakich konstrukcji językowych oraz struktur danych użyć do zakodowania logiki biznesowej?
* ...

Odpowiedzi zależne są od kontekstu realizowanych funkcjonalności.

Moje piąte podejście do zapoznania się z programowaniem funkcyjnym okazało się podejściem najbardziej skutecznym. Dzięki wykorzystaniu języka **F#** miałem okazję przejść razem z tobą zarówno przez konstrukcje funkcyjne, jak i obiektowe. Zobaczyliśmy też, że kod napisany w języku **C#** może być z powodzeniem używany w kodzie języka **F#**.

Przypomnijmy sobie główne elementy poznane w tej serii, zarówno te dotyczące języka **F#**, jak i frameworka **NServiceBus**:

* **F#**:
    * **open** - importowanie elementów z przestrzeni nazw
    * **[&lt;EntryPoint&gt;]** - atrybut określający funkcję wejścia programu
    * **let main argv** - definiowanie function value
    * **wcięcia** - jako element oddzielania poszczególnych bloków kodu
    * **wnioskowanie typów przez kompilator** - brak konieczności jawnego podawania typów
    * **immutability** - niezmienność zdefiniowanych elementów
    * **no return** - ostatnia instrukcja kodu jako zwracany wynik przez funkcje
    * **Install-Package** - możliwość wykorzystania paczek Nuget
    * **let endpointName = "ClientUI"** - definiowanie simple value
    * **mutable** - jawne określanie możliwości zmiany wartości zdefiniowanych elementów
    * **let endpointConfiguration = new EndpointConfiguration(endpointName)** - tworzenie obiektów
    * **let transport = endpointConfiguration.UseTransport&lt;LearningTransport&gt;()** - wsparcie dla typów generycznych
    * **async** - wsparcie programowania asynchronicznego
    * **Async.AwaitTask** - wsparcie dla programowania asynchronicznego z języka C#
    * **Console.ReadLine()** - możliwość wykorzystywania bibliotek platformy .NET
    * **&#124;&gt;** - pipe operator
    * **namespace** - tworzenie przestrzeni nazw
    * **type PlaceOrder(orderId: string) = ...** - tworzenie klas z wykorzystaniem konstruktora
    * **interface ICommand** - dziedziczenie po interfejsach
    * **member this.OrderId = orderId** - tworzenie immutable pól, metod, property, ...
    * **module Handlers** - tworzenie modułów
    * **static member log = ...** - tworzenie statycznych elementów w klasie
    * **interface IHandleMessages&lt;PlaceOrder&gt; with** - implementacja metod zdefiniowanych w odziedziczonych interfejsach
    * **while continueLooping do** - pętle do sterowania programem
    * **match key.Key with** - pattern matching do sterowania programem
    * **member val OrderId = "" with get, set** - jawne określenie możliwości zmiany wartości member klasy
    * **inherit Saga&lt;ShippingPolicyData&gt;()** - dziedziczenie po klasach abstrakcyjnych
    * **override this.ConfigureHowToFindSaga** - implementacja abstrakcyjnych metod
    * **fun message -> message.OrderId** - wyrażenia lambda

* **NServiceBus**
    * **new EndpointConfiguration(endpointName)** - konfiguracja Endpointa
    * **endpointConfiguration.UseTransport&lt;LearningTransport&lt;()** - definiowanie transportów
    * **Endpoint.Start(endpointConfiguration)** - startowanie Endpointa
    * **endpointInstance.Stop()** - zatrzymywanie Endpointa
    * **ICommand** - definiowanie wiadomości typu Command
    * **PlaceOrderHandler** - definiowanie Message Handlerów
    * **this.Handle(message, context) = ...** - definiowanie logiki przetwarzania wiadomości
    * **endpointInstance.SendLocal(command)** - wysyłanie wiadomości z Endpointa do samego siebie
    * **endpointConfiguration.UseSerialization&lt;NewtonsoftSerializer&gt;()** - definiowanie serializatora wiadomości
    * **endpointInstance.Send(command)** - wysyłanie wiadomości z jednego Endpointa na inny Endpoint
    * **routing.RouteToEndpoint(typeof&lt;PlaceOrder&lt;, "Sales")** - definiowanie rutingu wiadomości
    * **IEvent** - definiowanie wiadomości typu Event
    * **context.Publish(orderPlaced)** - publikowanie Eventów
    * **type ShippingPolicyData() = ...** - definiowanie danych Sagi
    * **type ShippingPolicy() = ...** - definiowanie logiki Sagi
    * **override this.ConfigureHowToFindSaga** - mapowanie danych wiadomości na dane Sagi
    * **this.MarkAsComplete()** - oznaczanie instancji Sagi jako zakończonej
    * **endpointConfiguration.UsePersistence&lt;LearningPersistence&gt;()** - definiowanie persistence

Połączenie języka **F#** z frameworkiem **NServiceBus** jest możliwe i daje fajne rezultaty. W kolejnym kroku powinniśmy wybrać któryś z produkcyjnych [transportów][1] oraz produkcyjnych [persistence][2] i sprawdzić, czy wszystko zadziała tak, jak powinno. Jeśli tak, to moglibyśmy zacząć używać takiego stosu technologicznego. Przed ostateczną decyzją trzeba wziąć pod uwagę dwa elementy:

* brak wsparcia na poziomie języka **F#** dziedziczenia po tym samym interfejsie różniącym się tylko generycznym parametrem, co ma wpływ na sposób kodowania **Sagi**.
* **NServiceBus** jest frameworkiem komercyjnym, napisanym w języku **C#**, z pełnym wsparciem zespołu, który go rozwija

W [poprzednim artykule][3] przeszliśmy przez rozwiązanie pierwszego elementu. Jeśli chodzi o drugi element to przed wysłaniem zgłoszenia na linię wsparcia lub zadaniem pytania na [grupie dyskusyjnej][6] może zajść konieczność przetłumaczenia kodu z języka **F#** na język **C#**, co może być czynnością dość czasochłonną.

Sytuacja jednak nie jest do końca stracona. Wręcz przeciwnie, otwiera się ciekawa perspektywa na zastosowanie zasady **Single Responsibility Principle (SRP)**. W [pierwszym artykule][4] tej serii opisałem w jaki sposób podejście **impure-pure-impure** wpasowuje się w rozwiązania bazujące na **Messagingu** i **Queueingu**:

* message pobierany jest z kolejki - impure
* realizowana jest logika - pure
* message przesyłany jest na kolejkę - impure

Jak widzieliśmy kod napisany w **C#** możemy wykorzystywać w kodzie **F#**. Trzymając się pewnych zasad opisanych w [dokumentacji][5], kod napisany w **F#** możemy również wykorzystać w kodzie **C#**. Dokładając do tego framework **NServiceBus** oraz zasadę **SRP** możemy zdefiniować stos technologiczny w sposób:

* message pobierany jest z kolejki - impure
    * **C#**
    * **NServiceBus**
* realizowana jest logika - pure
    * **F#**
* message przesyłany jest na kolejkę - impure
    * **C#**
    * **NServiceBus**

W powyższym schemacie ukryte są operacje na zewnętrznych źródłach danych takich jak baza danych, web api, pliki itp. Są to elementy, które należą do grupy **impure**. Do ich realizacji możemy również użyć języka **F#**.

W ten sposób rozgraniczamy odpowiedzialność:

* **C#** do Messagingu i Queueingu
* **F#** do pozostałych elementów

I tak oto doszliśmy do końca niniejszej serii. Mam nadzieję, że była dla Ciebie wartościowa. Jeśli masz jakieś pytania lub przemyślenia odnośnie poznanego materiału, podziel się nimi w komentarzach.

[1]: https://docs.particular.net/transports/ "nservicebus transports"
[2]: https://docs.particular.net/persistence/ "persistence"
[3]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[4]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[5]: https://docs.microsoft.com/en-us/dotnet/fsharp/style-guide/component-design-guidelines "fsharp component-design-guidelines"
[6]: https://discuss.particular.net/ "particular discussion group"
[7]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[8]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[9]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[10]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}
[11]: {{ site.url }}{% link _posts/2019-11-11-fsharp-nservicebus-practical-guide-publishing-events.md %}
[12]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[13]: {{ site.url }}{% link _posts/2019-12-22-fsharp-nservicebus-practical-guide-ending.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}