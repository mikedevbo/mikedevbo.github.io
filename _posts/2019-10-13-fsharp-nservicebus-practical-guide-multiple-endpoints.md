---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów"
date:   2019-10-13
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][6]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][7]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][8]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][9]
* F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń
* F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga

Witam Cię w czwartej części przewodnika, w którym pokazuję, w jaki sposób zaprogramować funkcjonalności dla **Retail E-commerce System** wykorzystując do tego framework **NServiceBus** oraz język **F#**.

Każdy kolejny artykuł bazuje na przykładach zakodowanych w poprzednich częściach. Jeśli jesteś tu pierwszy raz, zachęcam Cię do przejścia przez pierwszy trzy artykuły. Jeśli kontynuujesz serię, zapewne masz swój kod, który chcesz dalej rozwijać. W każdej chwili możesz pobrać końcowe rozwiązania, które udostępniam na [GitHubie][1].

W tym artykule pokażę Ci, jak skomunikować ze sobą dwa **Endpointy**.

Źródłem, na którym bazują przykłady, jest [kolejna część][2] tutoriala wprowadzającego do **NServiceBusa**.

## Konfiguracja nowego Endpointa

Myślę, że jest to dobry momenty, aby powiedzieć trochę więcej o tym, czym tak właściwie jest **NServiceBus Endpoint**.

**Endpoint** jest logicznym bytem, zawierającym swoją nazwę, którego zadaniem jest obsługa wiadomości. Zakodowany **Message Handler** umieszczamy w konkretnym **Endpointcie**. Od tej pory **Endpoint** staje się właścicielem **Message Handlera**.

Logiczny **Endpoint** może mieć uruchomionych wiele swoich fizycznych instancji zwanych **Endpoint Instance**. Każda instancja posiada jedną **Input Queue**, z której pobiera wiadomości do przetworzenia.

Po co tworzyć wiele logicznych **Endpointów**? Powody są co najmniej trzy:

1. **Loose coupling** - odseparowanie od siebie logicznych komponentów składających się na całość rozwiązania
2. **Scaling** - każdy logiczny **Endpoint** posiada co najmniej jeden fizyczny **Endpoint Instance**, który przetwarza wiadomości niezależnie od pozostałych instancji
3. **Deploy** - każdy fizyczny **Endpoint Instance** może być hostowany w osobnym procesie oraz wdrażany niezależnie od pozostałych instancji

Po co tworzyć wiele fizycznych **Endpoint Instance** w ramach jednego logicznego **Endpointa**?

1. **Scaling** - skrócenie czasu przetwarzania większej ilość wiadomości

Nasz kodowany przykład zawiera jeden logiczny **Endpoint** o nazwie **ClientUI**. Przy jego uruchamianiu tworzony jest jeden fizyczny **Endpoint Instance** powiązany z jedną **Input Queue**. **ClientUI** zawiera zarówno logikę wysyłania zamówienia **PlaceOrder**, jak i jego przetwarzania - **PlaceOrderHandler**.

Patrząc na dotychczasowe rozwiązanie z punktu wiedzenia **Design** możemy stwierdzić, że wysyłanie wiadomości oraz jej przetwarzanie to są dwie niezależne od siebie operacje. Z punktu widzenia **Deploy** chcielibyśmy, aby zastopowanie wysyłania **PlaceOrder** nie miało wpływu na przetwarzanie już wysłanych zamówień. Podobnie w drugą stronę. To, że zatrzymuję przetwarzanie **PlaceOrder** nie powinno blokować możliwości jego wysyłania.

Stwórzmy więc nowy **Endpoint** i zróbmy refactoring istniejącego kodu. W tym celu w **Visual Studio**:

1. dodaj do solucji **RetailDemo** nowy projekt **Add...New Project...**
2. wybierz typ projektu **Visual F#...NET Core...Console App (.NET Core)**
3. w pole **Name** wpisz nazwę projektu **Sales**
4. zainstaluj paczkę Nuget **NServiceBus**
    * sposób instalacji i aktualizacji paczek znajdziesz w [drugie części][3] niniejszej serii
    * paczki Nuget powinny być w tych samych wersjach co w projekcie **ClientUI**
5. zainstaluj paczkę Nuget **NServiceBus.Newtonsoft.Json**
6. uaktualnij paczkę Nuget **FSharp.Core** w projektach **Messages** oraz **Sales**
7. dodaj referencję do projektu **Messages**
8. usuń z pliku **Program.fs** domyślnie utworzony kod
9. dodaj poniższy kod

{% highlight fsharp %}
open System
open NServiceBus

[<EntryPoint>]
let main argv =
    
    let endpointName = "Sales"
    Console.Title <- endpointName

    let endpointConfiguration = new EndpointConfiguration(endpointName)
    let transport = endpointConfiguration.UseTransport<LearningTransport>()

    endpointConfiguration.UseSerialization<NewtonsoftSerializer>() |> ignore;
    
    async {

        let! endpointInstance = Endpoint.Start(endpointConfiguration) |> Async.AwaitTask
            
        printf "Press Enter to exit..."
        Console.ReadLine() |> ignore

        do! endpointInstance.Stop() |> Async.AwaitTask

    } |> Async.RunSynchronously

    0 // return an integer exit code
{% endhighlight %}    

Kod **F#** jest taki sam jak przy tworzeniu **Endpointa** ***ClientUI***. Jedyną różnicą jest to, że nowemu **Endpointowi** nadajemy nazwę **Sales**:

`let endpointName = "Sales"`

Dodajmy teraz logikę przetwarzania **PlaceOrderHandler** do nowego **Endpointa**:

1. przenieś (nie kopiuj) plik **Handlers.fs** z projektu **ClientUI** do projektu **Sales**.

Przetestuj czy wszystko działa, uruchamiając nowo stworzony **Endpoint** ***Sales***:

1. ustaw na nowym projekcie opcję **Set as StartUp Project**
2. naciśnij klawisz **F5**

**Endpoint** skonfiguruje się i zgłosi gotowość do działania.

## Wysłanie wiadomości

Obecnie do wysyłania wiadomości używamy metody `SendLocal`. Dzięki temu wiadomość wysyłana z **Endpointa** ***ClientUI*** trafia na **Endpoint** ***ClientUI***. Jest to funkcjonalność w stylu *wyślij wiadomość do samego siebie*:

`do! endpointInstance.SendLocal(command) |> Async.AwaitTask`

Do wysyłania wiadomości na inny **Endpoint** służy metoda `Send`:

1. zamień w funkcji **RunLoop** powyższy kod na:

    `do! endpointInstance.Send(command) |> Async.AwaitTask`

2. ustaw opcję **Set as StartUp Project** na projekcie **ClientUI**. 
3. uruchom program i wyślij wiadomość naciskając klawisz **P**.

Efekt? **NServiceBus** zgłasza wyjątek z informacją, że nie wie, gdzie ma wysłać wiadomość **PlaceOrder**:

`Exception: No destination specified for message: Commands.PlaceOrder`

## Routing

Aby skomunikować ze sobą dwa **Endpointy** musimy ustawić między nimi minimum dwie zależności:

1. rodzaj wiadomości tak, aby wysyłający **Endpoint** wiedział jaką wiadomość ma wysłać
2. adres docelowego **Endpointa** tak, aby wysyłający **Endpoint** wiedział gdzie ma wysłać wiadomość

W naszym przykładzie pierwszą zależność rozwiązujemy poprzez współdzielenie referencji do projektu **Messages**. Drugą zależność rozwiążemy, wskazując **Endpointowi** ***ClientUI*** adres do **Endpointa** ***Sales***:

1. dodaj do pliku **Program.fs** w projekcie **ClientUI** poniższy kod, zaraz za utworzeniem transportu:

{% highlight fsharp %}
let routing = transport.Routing()
routing.RouteToEndpoint(typeof<PlaceOrder>, "Sales")
{% endhighlight %}

Metoda ``transport.Routing()`` zwraca obiekt `RoutingSettings<T>`, gdzie **T** jest typem używanego transportu. W naszym przykładzie jest to `LearningTransport`. Zwrócony obiekt zawiera metodę `RouteToEndpoint`, która pozwala definiować docelowy adres, na który ma zostać wysłana wiadomość. W naszym przykładzie konfigurujemy **NServiceBusa** w ten sposób, aby wiadomość **PlaceOrder** wysłał do **Endpointa** o nazwie **Sales**.

Jeśli mielibyśmy więcej typów wiadomości obsługiwanych przez **Sales** to, zamiast dla każdego typu definiować osobny **routing** skorzystalibyśmy z przeciążonej metody `RouteToEndpoint` i do **Sales** wysłali wszystkie wiadomości należące do projektu **Messages**:

`routing.RouteToEndpoint(typeof<PlaceOrder>.Assembly, "Sales")`

Metoda `RouteToEndpoint` posiada również możliwość **routingu** wiadomości należących do tego samego **Assembly**, ale posiadających różne **namespace**:

{% highlight fsharp %}
routing.RouteToEndpoint(
    typeof<DoSomething>.Assembly,
    "Specific.Namespace",
    "SomeEndpoint")
{% endhighlight %}

<br />

W naszym przykładzie definiujemy **routing** na poziomie logicznym określając, że wiadomość **PlaceOrder** ma zostać wysłana do logicznego **Endpointa** ***Sales***. **NServiceBus** pozwala też na definiowanie **routingu** na poziomie fizycznym, określającym do jakiego fizycznego **Endpoint Instance** wiadomość ma zostać wysłana. Jest to element skalowania. Jeśli chodzi o szczegóły, to myślę, że jest to dobry temat na którąś z kolejnych serii artykułów :)

Uruchom teraz projekt **ClientUI** i wyślij kilka wiadomości **PlaceOrder**. **NServiceBus** nie zwraca już wyjątku, natomiast widać, że wiadomości nie są już przetwarzane przez **ClientUI**.

Ustaw teraz **Set as StartUp Project** na projekcie **Sales** i uruchom program.

Efekt? Wszystkie wcześniej wysłane wiadomości przetworzyły się z sukcesem, mimo że w trakcie ich wysyłania **Sales** nie był uruchomiony!.

Jest to jedna z wielu mocy frameworka **NServiceBus**. Jeśli chcesz się dowiedzieć, w czym taka funkcjonalność może być pomocna, zachęcam Cię do przeczytania jednego z moich [wcześniejszych artykułów][4].

Aby ułatwić sobie zadanie jednoczesnego uruchamiania wielu projektów ustaw w **Visual Studio** opcję **Multiple Startup Projects** dla projektów **ClientUI** oraz **Sales**. Instrukcję znajdziesz [na stronie dokumentacji][5].

Jeśli chcesz ponownie poczuć, na czym polega moc komunikacji między **Endpointami** to:

1. otwórz [tutorial z dokumentacji technicznej][2] frameworka **NServiceBus**
2. przejdź do sekcji **Running the solution**
3. wykonaj opisane ćwiczenie na swoim kodzie napisanym w **F#**

<br />

W artykule przeszliśmy przez komunikację dwóch **Endpointów**. W ten sam sposób możemy komunikować ze sobą ich większą liczbę. Dzięki temu mamy możliwość budowania większych systemów składających się z luźno ze sobą powiązanych komponentów. Tym zagadnieniem zajmiemy się w następnym artykule, dokładając do naszego przykładu dodaktowy logiczny **Endpoint**. Pokażę Ci też, w jaki sposób komunikować ze sobą **Endpointy** używając do tego **zdarzeń**, które są kolejnym typem wiadomości obsługiwanych przez framework **NServiceBus**.

Zakodowany przykład z tego artykułu, jak i z poprzednich części, znajdziesz na moim [GitHubie][1].

[1]: https://github.com/mikedevbo/fsharp-nservicebus "fsharp-nservicebus"
[2]: https://docs.particular.net/tutorials/nservicebus-step-by-step/3-multiple-endpoints/ "multiple endpoints"
[3]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[4]: {{ site.url }}{% link _posts/2017-04-29-wytwarzanie_oprogramowania_IV.md %}
[5]: https://docs.microsoft.com/en-us/visualstudio/ide/how-to-set-multiple-startup-projects?view=vs-2017&redirectedfrom=MSDN "how to set multiple startup projects"
[6]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[7]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[8]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[9]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}