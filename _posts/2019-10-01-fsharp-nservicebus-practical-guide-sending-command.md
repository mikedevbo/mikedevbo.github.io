---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy"
date:   2019-10-01
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][1]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][2]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][14]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][15]
* [F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń][16]
* [F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga][17]
* [F# & NServiceBus - praktyczny przewodnik: Zakończenie][18]

Z [poprzedniego artykułu][2] wiesz, w jaki sposób skonfigurować oraz wystartować **Endpoint NServiceBusa**. Utworzony kod będzie Ci potrzebny do kontynuowania ćwiczeń z tego artykułu, w którym przejdziemy przez utworzenie, wysyłanie oraz przetworzenie **wiadomości**.

Źródłem, na którym bazują przykłady, jest [druga część][3] tutoriala wprowadzającego do **NServiceBusa**, który znajduje się na stronie [dokumentacji technicznej][4] frameworka.

## Utworzenie wiadomości

**Wiadomości** służą do przesyłania informacji pomiędzy **Endpointami**. W kodzie wiadomość definiowana jest przez **klasę**. Poszczególne informacje wiadomości definiowane są przez odpowiednie **member** klasy. Wiadomość jest kontraktem pomiędzy **Endpointami**. Powinna być jak najprostsza. Nie powinna zawierać żadnej logiki. Można ją traktować tak samo, jak wzorzec [Data Transfer Object][5].

W celu zminimalizowania zależności pomiędzy **Endpointami** wiadomości powinny znajdować się w osobnym projekcie, który z kolei posiada zależności tylko do bibliotek systemowych oraz paczki Nuget **NServiceBusa**.

**NServiceBus** definiuje trzy typy wiadomości:

* **Command** - wysyłając Command rozpoczynamy jakąś akcję
* **Event** - publikując Event oznajmiamy, że jakaś akcja została zrealizowana
* **Message** - wysyłając Message odpowiadamy nadawcy, który wysłał wiadomość rozpoczynającą akcję

W tym artykule skupiamy się na wiadomościach typu **Command**. Więcej informacji o typach wiadomości znajdziesz na [stronie dokumentacji][6].

Tak jak pisałem w [pierwszym artykule][1], kodując rozwiązanie w języku obiektowym, dobrą praktyką jest umieszczanie **klasy** reprezentującej wiadomość w osobnym pliku. Przy większych systemach wynikiem takiego podejścia jest bardzo duża ilość plików, zawierających bardzo małą ilość kodu. W **F#** klasa jest jednym z wielu typów. Nie ma konieczności trzymania się zasady klasa per osobny plik. Plik możemy potraktować jako kontener na logicznie powiązane ze sobą typy oraz funkcje, operujące na tych typach.

Po tym krótkim wstępie przejdźmy do stworzenia nowej wiadomości. W tym celu w **Visual Studio**:

1. dodaj do solucji **RetailDemo** nowy projekt **Add...New Project...**
2. wybierz typ projektu **Visual F#...NET Core...Class Library (.NET Core)**
3. w pole **Name** wpisz nazwę projektu **Messages**
4. zainstaluj paczkę Nuget **NServiceBus**
5. zmień nazwę pliku **Library.fs** na **Commands.fs**
6. usuń domyślnie utworzony kod
7. dodaj poniższy kod

{% highlight fsharp %}
namespace Commands

open NServiceBus

type PlaceOrder(orderId: string) =
    interface ICommand
    member this.OrderId = orderId
{% endhighlight %}

Przejdźmy po kolei przez utworzone konstrukcje:

* `namespace Commands` - definicja przestrzeni nazw
* `open NServiceBus` - zaimportowanie elementów z przestrzeni nazw **NServiceBus**
* `type PlaceOrder(orderId: string) =` - deklaracja oraz definicja klasy reprezentującej wiadomość o nazwie **PlaceOrder**
    * `PlaceOrder(orderId: string)` - definiuje nazwę oraz konstruktor klasy, który zawiera parametr **orderId**
    * `interface ICommand` - dziedziczenie po interfejsie `ICommand`, należącego do przestrzeni nazw **NServiceBus**
    * `member this.OrderId = orderId` - deklaracja oraz definicja informacji wysyłanej wraz z wiadomością

W **F#** typy definiowane są przez słowo kluczowe `type`. Wcięcia określają początek i koniec definicji typu. Dziedziczenie po interfejsie odbywa się przy pomocy słowa kluczowego `interface`. Pola, properties, metody, ... definiuje się za pomocą słowa kluczowego `member`. Domyślnym modyfikatorem dostępu każdego `member` jest `private`. Sama klasa posiada domyślny modyfikator dostępu `public`.

W powyższym przykładzie wiadomość o nazwie **PlaceOrder** zawiera prywatne **member** o nazwie **OrderId**, któremu przypisywane jest **Value** o nazwie **orderId**, pochodzące z konstruktora klasy. Konstruktor zawiera doprecyzowanie typu, ponieważ chcemy, aby id zamówienia zawsze miało wartości typu `string`.

Klasa **PlaceOrder** dziedziczy po interfejsie `ICommand` należącego do przestrzeni nazw `NServiceBus`. W ten sposób oznaczamy, że jest to wiadomości typu **Command**. Dzięki temu framework będzie umiał ją wysłać oraz przetworzyć.

## Przetwarzanie wiadomości

Mając zdefiniowaną wiadomość, możemy utworzyć funkcjonalność, która będzie ją przetwarzać. W języku **NServiceBusa** taka funkcjonalność nazwa się **Message Handler**. W celu utworzenia **Handlera** dla wiadomości **PlaceOrder**:

1. dodaj do projektu **ClientUI** referencję do wcześniej utworzonego projektu **Messages**
1. dodaj do projektu **ClientUI** nowy plik źródłowy **F#** oraz nadaj mu nazwę **Handlers.fs**
2. usuń domyślnie utworzony kod
3. dodaj poniższy kod

{% highlight fsharp %}
module Handlers

open NServiceBus
open NServiceBus.Logging
open System.Threading.Tasks
open Commands

type PlaceOrderHandler() =
    static member log = LogManager.GetLogger<PlaceOrderHandler>()
    interface IHandleMessages<PlaceOrder> with
        member this.Handle(message, context) =
            PlaceOrderHandler.log.Info(sprintf "Received PlaceOrder, OrderId = %s" message.OrderId)
            Task.CompletedTask

{% endhighlight %}

**PlaceOrderHandler** jest klasą, która dziedziczy po interfejsie `IHandleMessages<T>`, należącym do przestrzeni nazw **NServiceBus**, gdzie **T** jest typem wiadomości do przetworzenia. W przykładzie jest to nasz Command **PlaceOrder**. Jeśli interfejs, po którym dziedziczy klasa, zawiera jakiekolwiek **member**, musi ono być jawnie zaimplementowane. Służy do tego słowo kluczowe `with`, wstawiane zaraz za nazwą dziedziczonego interfejsu.

Metoda `Handle` zawiera implementację logiki, którą wykona **NServiceBus**, po otrzymaniu Commanda **PlaceOrder**. Zobacz, że zarówno parametry metody `Handle`, jak i jej zwracany typ, nie muszą być doprecyzowane. Kompilator ma wystarczającą ilość informacji, aby wydedukować, że parametr **message** jest typu `PlaceOrder`, a parametr **context** jest typu `IMessageHandlerContext`. Sama metoda zwraca zaś wartość typu `Task`.

Aby się o tym przekonać, najedź kursorem w **Visual Studio** na nazwę metody **Handle** i sprawdź wyświetloną informację.

**F#** obsługuje statyczne member. Przykładem tego jest **log**, reprezentujący mechanizm logowania wbudowany w **NServiceBusa**. Do elementów statycznych możemy odwoływać się poprzez ich pełną nazwę `PlaceOrderHandler.log`. Ostatnią instrukcją member `Handle` jest zwrócenie typu zgodnego z sygnaturą.

Powyższy przykład zawiera bardzo prostą logikę przetwarzania wiadomości, wypisującą tekst za pomocą mechanizmu logowania.

Zwróć uwagę na pierwszą linijkę kodu w plikach **Commands.fs** oraz **Handlers.fs**. W pierwszym przypadku tworzymy `namespace`, a w drugim `module`. W **F#** **namespace** pozwala na grupowanie typów. Nie pozwala na grupowanie funkcji. Do tego służy **module**, który pozwala grupować zarówno typy jak funkcje. W ten sposób możemy grupować ze sobą logicznie powiązane elementy. Jeśli plik zawiera tylko definicje typów, lepiej jest używać **namespace**. Jeśli definiujemy funkcje, używamy **module**.

W powyższym przykładzie ponownie skorzystaliśmy z tego, że w **F#**, klasy nie muszą znajdować się w osobnych plikach. Logicznie powiązane ze sobą **Message Handlery**, mogą być zdefiniowane w jednym pliku. Dzięki wykorzystaniu **module**, mamy możliwość definiowania funkcji, których możemy używać w dowolnym handlerze należącym do modułu.

Ćwiczenie dla Ciebie. Utwórz nową funkcję w module **Handlers** zwracającą tekst, a następnie użyj tej funkcji do wyświetlenia tekstu przez member **log**, klasy **PlaceOrderHandler**.

W tym momencie **NServiceBus** posiada pełną informację. W momencie, kiedy wiadomość **PlaceOrder** zostanie wysłana, framework automatycznie utworzy nową instancję klasy **PlaceOrderHandler** i wykona logikę, zaimplementowaną w member **Handle**.

Zanim przejdziemy dalej, przeanalizujmy jeszcze jedną pewną właściwość. W tutorialu z [dokumentacji NServiceBusa][3] opisany jest przypadek grupowania **Handlerów**, poprzez definiowanie kolejnych metod w klasie:

{% highlight csharp %}
public class DoSomething { }
public class DoSomethingElse { }

public class DoSomethingHandler :
    IHandleMessages<DoSomething>,
    IHandleMessages<DoSomethingElse>
{
    public Task Handle(DoSomething message, IMessageHandlerContext context)
    {
        Console.WriteLine("Received DoSomething");
        return Task.CompletedTask;
    }

    public Task Handle(DoSomethingElse message, IMessageHandlerContext context)
    {
        Console.WriteLine("Received DoSomethingElse");
        return Task.CompletedTask;
    }
}
{% endhighlight %}

Jeśli spróbujemy zrobić to samo w **F#**:

{% highlight fsharp %}
type DoSomething() = class end
type DoSomethingElse() = class end

type DoSomethingHandler() =
    interface IHandleMessages<DoSomething> with
        member this.Handle(message, context) =
            printfn "Received DoSomething"
            Task.CompletedTask
    interface IHandleMessages<DoSomethingElse> with
        member this.Handle(message, context) =
            printfn "Received DoSomethingElse"
            Task.CompletedTask
{% endhighlight %}

Kod nie skompiluje się, a kompilator zwróci przyczynę:

`This type implements the same interface at different generic instantiations 'IHandleMessages<DoSomethingElse>' and 'IHandleMessages<DoSomething>'. This is not permitted in this version of F#.`

**F#** w wersji **4.7.0** nie obsługuje takiej konstrukcji. Ponieważ **F#** rozwijany jest na zasadach [Open Source][7], możemy śledzić jego rozwój. Dzięki temu dowiadujemy się, że istnieje zatwierdzone [RFC][8], którego celem jest dodanie powyższej konstrukcji, w którejś z przyszłych wersji języka.

Ponieważ logicznie ze sobą powiązane **Handlery** możemy implementować w konkretnym **module**, to brak powyższej właściwości językowej nie utrudnia nam konstruowania kodu. Będzie to miało większe znaczenie przy definiowaniu funkcjonalności zarządzającej bardziej skomplikowanym procesem biznesowym. Wrócimy do tego zagadnienia w ostatnim artykule wchodzącym w skład tej serii.

## Wysłanie wiadomości

Teraz możemy dopisać ostatnią funkcjonalność, która wyśle wiadomość do przetworzenia. W tym celu:

1. w projekcie **ClientUI**, dodaj do pliku **Program.fs** poniższy kod, tak, aby znajdował się przed funkcją **main**
2. doprowadź kod do stanu kompilacji, dodając brakujące sekcje **open**

{% highlight fsharp %}
let RunLoop (endpointInstance: IEndpointInstance) = 
    
    let log = LogManager.GetLogger("ClientUI.Program")
    let mutable continueLooping = true

    while continueLooping do
        log.Info("Press 'P' to place an order, or 'Q' to quit.")
        let key = Console.ReadKey()
        printfn ""

        match key.Key with
        | ConsoleKey.P ->
            // Instantiate the command
            let command = new PlaceOrder(Guid.NewGuid().ToString())
            
            // Send the command to the local endpoint
            log.Info(sprintf "Sending PlaceOrder command, OrderId = %s" command.OrderId);
            async {
                do! endpointInstance.SendLocal(command) |> Async.AwaitTask
            } |> Async.RunSynchronously
        
        | ConsoleKey.Q ->
            continueLooping <- false
        
        | _ ->
            log.Info("Unknown input. Please try again.")
{% endhighlight %}

Mechanizm polega na tym, że po naciśnięciu klawisza **P**, tworzony jest **Command** `PlaceOrder` z unikatowym **orderId**. Następnie, poprzez asynchroniczne wywołanie metody `SendLocal`, Command wysyłany jest na Endpoint **ClientUI**. Metoda `SendLocal` należy do obiektu `endpointInstance`, który reprezentowany jest przez interfejs `IEndpointInstance`, należący do API **NServiceBusa**. Obiekt `endpointInstance` przekazywany jest w parametrze funkcji `RunLoop`. Po naciśnięciu klawisza **Q** następuje wyjście z funkcji. Po naciśnięciu innego przycisku następuje powrót do ponownego wprowadzenia znaku.

Rozpoznajesz elementy języka **F#**, których używaliśmy w poprzednich przykładach? Są to między innymi: tworzenie obiektów, tworzenie loga **NServiceBusa**, wywołanie asynchroniczne, ... Nowością są dwie konstrukcje: pętla **while** oraz [Pattern Matching][9] z wykorzystaniem słów kluczowych `match...with`.

Pętla `while` działa tak samo jak w języku **C#**. Dopóki spełniony jest warunek pętli, wykonywany będzie kod zawarty w ciele pętli. W naszym przykładzie warunek pętli reprezentuje `mutable value` o nazwie **continueLooping**. W momencie wybrania znaku **Q** warunek pętli zmienia się na wartość **false** i następuje wyjście z pętli.

**Pattern Matching**, w dużym skrócie, polega na dopasowaniu wartości wejściowej do odpowiedniego wzorca, a następnie wykonanie kodu zdefiniowanego w dopasowaniu. W naszym przykładzie wejście to wybrany znak na klawiaturze. Za pomocą wyrażania `match` następuje próba dopasowania wejścia do odpowiednio bloku kodu. Bloki definiuje się za pomocą `|`. Pierwsze dopasowanie wygrywa. Jeśli wybranym znakiem jest **P**, następuje wysłanie wiadomości. Jeśli wybranym znakiem jest **Q** następuje wyjście z pętli. We wszystkich innych przypadkach, wypisywana jest informacja o nieznanym wejściu. Dowolne dopasowanie realizujemy za pomocą `_`.

Aby wykorzystać stworzoną funkcję `RunLoop`, zastąp w funkcji `main` fragment kodu z:

{% highlight fsharp %}
printf "Press Enter to exit..."
Console.ReadLine() |> ignore
{% endhighlight %}

na:

{% highlight fsharp %}
RunLoop endpointInstance
{% endhighlight %}

W tym momencie wszystko jest gotowe do uruchomienia i przetestowania. Naciśnij w **Visual Studio** klawisz **F5**, a następnie klawisz **P**, aby wysłać wiadomość.

Czy wszystko działa tak jak powinno?

Na pierwszy rzut oka tak, ale przyjrzyj się, co jest wyświetlane na ekranie, kiedy wiadomość jest przetwarzana. Porównaj wynik działania z zakodowaną logiką w klasie `PlaceOrderHandler`.

## Immutable Messages

Brak wypisywania id zamówienia spowodowane jest tym, że member `OrderId` klasy `PlaceOrder` jest **immutable**. Jego wartość można ustawić jedynie w konstruktorze klasy. **NServiceBus** wysyłając wiadomość, serializuje dane do zdefiniowanego formatu. Pobierając wiadomość do przetworzenia deserializuje ją do odpowiedniej klasy. Domyślny serializator nie wspiera operacji dla modyfikatora dostępu `private`, a taki jest tworzony dla `OrderId` w momencie utworzenia obiektu `PlaceOrder`. Więcej o **immutable messages** możesz przeczytać w materiałach:

* [support-for-immutable-messages][10]
* [immutable-messages ][11]
* [immutability-in-message-types][12]

Ponieważ **immutability** jest jednym z podstawowych założeń języka **F#**, użyjmy serializatora wspierającego modyfikator `private`. W tym celu zainstaluj paczkę Nuget:

`Install-Package NServiceBus.Newtonsoft.Json -Version 2.2.0 -ProjectName ClientUI`

Następnie, w funkcji `main`, zaraz za utworzeniem transportu, dodaj poniższy wpis:

`endpointConfiguration.UseSerialization<NewtonsoftSerializer>() |> ignore`

Bonusem tej operacji jest to, że **NServiceBus** będzie serializował dane do formatu **JSON**.

Teraz uruchom ponownie program i sprawdź, czy wszystko działa, tak jak powinno:

{% highlight txt %}
ClientUI.Program Press 'P' to place an order, or 'Q' to quit.
p
ClientUI.Program Sending PlaceOrder command, OrderId = 16e6cfc8-1db9-4d47-b32d-844b77758c66
ClientUI.Program Press 'P' to place an order, or 'Q' to quit.
Handlers+PlaceOrderHandler Received PlaceOrder, OrderId = 16e6cfc8-1db9-4d47-b32d-844b77758c66
p
ClientUI.Program Sending PlaceOrder command, OrderId = 292456e4-690a-4503-ba7a-646fa8d68995
ClientUI.Program Press 'P' to place an order, or 'Q' to quit.
Handlers+PlaceOrderHandler Received PlaceOrder, OrderId = 292456e4-690a-4503-ba7a-646fa8d68995
{% endhighlight %}

Na ten moment znasz już podstawowe elementy frameworka **NServiceBus**, które umożliwiają tworzenie, wysyłanie oraz przetwarzanie wiadomości. Wiesz również, jakie są podstawowe konstrukcje języka **F#**, pozwalające na ich zakodowanie.

Ja również wyciągam z tej serii coś dla siebie. Mianowicie...zaczynam *"zaprzyjaźniać"* się z językiem **F#** :) Ucząc się nowych konstrukcji, kodując je w praktyce, powracając do nich po czasie i analizując, zauważam, że wszystko układa się w jedną całość. Dorzucając do tego kodowanie przykładów **NServiceBusa**, wyzwala to u mnie efekt **"Wow. To działa!"** :)

Całe rozwiązanie znajdziesz na [GitHubie][13].

W następnym artykule do naszych funkcjonalności dołożymy nowy **Endpoint**, a następnie wyślemy do niego wiadomość do przetworzenia.

[1]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[2]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[3]: https://docs.particular.net/tutorials/nservicebus-step-by-step/2-sending-a-command/ "nservicebus-sending-a-command"
[4]: https://docs.particular.net/tutorials/nservicebus-step-by-step/ "nservicebus-sending-a-command"
[5]: https://martinfowler.com/eaaCatalog/dataTransferObject.html "data transfer objects"
[6]: https://docs.particular.net/nservicebus/messaging/messages-events-commands "messages-events-commands"
[7]: https://fsharp.org/ "fsharp"
[8]: https://github.com/fsharp/fslang-design/blob/master/RFCs/FS-1031-Allow%20implementing%20the%20same%20interface%20at%20different%20generic%20instantiations%20in%20the%20same%20type.md "F# RFC FS-1031"
[9]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching "pattern matching"
[10]: https://discuss.particular.net/t/support-for-immutable-messages/1270 "support-for-immutable-messages"
[11]: https://docs.particular.net/nservicebus/messaging/immutable-messages "immutable-messages"
[12]: https://jimmybogard.com/immutability-in-message-types/ "immutability-in-message=types"
[13]: https://github.com/mikedevbo/fsharp-nservicebus "fsharp-nservicebus"
[14]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[15]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}
[16]: {{ site.url }}{% link _posts/2019-11-11-fsharp-nservicebus-practical-guide-publishing-events.md %}
[17]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[18]: {{ site.url }}{% link _posts/2019-12-22-fsharp-nservicebus-practical-guide-ending.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}