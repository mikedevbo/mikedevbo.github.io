---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga"
description: F# oraz NServiceBus. Czy to możliwe? Przeczytaj o tym, w jaki sposób zaimplementować Sagę pisząc kod w języku F#.
date:   2019-12-15
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][11]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][12]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][13]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][14]
* [F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń][15]
* [F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga][16]
* [F# & NServiceBus - praktyczny przewodnik: Zakończenie][17]

Witam Cię w szóstej części przewodnika, w którym pokazuję, w jaki sposób zaprogramować funkcjonalności dla **Retail E-commerce System** wykorzystując do tego framework **NServiceBus** oraz język **F#**.

W tym artykule rozszerzymy nasz dotychczas napisany system o mechanizm koordynujący przepływ wiadomości. Jeśli masz swój kod napisany na bazie poprzednich części, możesz go dalej rozwijać. Jeśli nie posiadasz swojego kodu, możesz pobrać przykład, który udostępniam na moim [GitHubie][1].

Opisane przykłady bazują na tutorialu wprowadzającym do [NServiceBus Saga][2], znajdującym się na oficjalnej stronie dokumentacji frameworka.

## NServiceBus Saga

Przypomnijmy sobie, co do tej pory udało nam się zaimplementować:

* Endpoint **ClientUI** zleca realizację zamówienia, wysyłając wiadomość typu Command **PlaceOrder**
* Endpoint **Sales** przetwarza Command **PlaceOrder**, a następnie publikuje wiadomość typu Event **OrderPlaced**
* Endpoint **Billing** przetwarzania Event **OrderPlaced**, a następnie publikuje Event **OrderBilled**
* Endpoint **Shipping** przetwarza Eventy **OrderPlaced** oraz **OrderBilled**, ale nie jest w stanie zdecydować, czy po przetworzeniu któregoś z Eventów, powinien zlecić wysyłkę zamówienia. Jest to związane z tym, że Eventy mogą nadejść w dowolnej kolejności.

Standardowy **NServiceBus Message Handler** nie przechowuje żadnego stanu przetwarzania. Jak więc możemy poradzić sobie z koordynacją Eventów **OrderPlaced** oraz **OrderBilled**? Odpowiedzią jest [NServiceBus Saga][3].

**Saga** jest implementacją jednego ze wzorców [Enterprise Integration Patterns][4] o nazwie [Process Manager][5]. Reaguje na wiadomości w taki sam sposób jak standardowy **Message Handler**. Wartością dodaną jest to, że przechowuje swój stan zapisywany w zdefiniowanym [Persistence][6]. Implementując logikę **Sagi** możemy dodawać do stanu własne dane. W trakcie przetwarzania wiadomości, **Saga** udostępnia nam dane z ostatnio zapisanego stanu.

Wykorzystując **Sagę** możemy skoordynować przetwarzanie Eventów **OrderPlaced** oraz **OrderBilled**, zlecając wysyłkę tylko wtedy, kiedy oba Eventy zostaną przetworzone.

W sekcji **Exercise** [tutoriala][2] znajdziesz diagram przedstawiający koncepcję, którą zakodujemy.

## Wysyłka zamówienia

Zanim utworzymy **Sagę**, dodajmy funkcjonalność wysyłki zamówienia. W tym celu:

* dodaj do pliku **Commands.fs**, znajdującym się w projekcie **Messages**, definicję wiadomości typu **Command**

{% highlight fsharp %}
type ShipOrder(orderId: string) =
    interface ICommand
    member this.OrderId = orderId
{% endhighlight %}

* dodaj do pliku **Handlers.fs**, znajdującym się w projekcie **Shipping**, przetwarzanie wiadomości **ShipOrder**

{% highlight fsharp %}
type ShipOrderHandler() =
    static member log = LogManager.GetLogger<ShipOrderHandler>()
    interface IHandleMessages<ShipOrder> with
        member this.Handle(message, context) =
            ShipOrderHandler.log.Info(sprintf "Order %s - Successfully shipped." message.OrderId)
            Task.CompletedTask
{% endhighlight %}     

* uzupełnij brakujące elementy tak, aby kod się skompilował

## Utworzenie Sagi

Jaka jest jedna z najtrudniejszych decyzji, jaką trzeba podjąć w programowaniu? Odpowiedź - wymyślenie nazwy dla kodowanej konstrukcji :) W naszym przypadku jest to wybranie nazwy dla **Sagi**. Możemy przeprowadzić wnioskowanie i stwierdzić, że skoro **Saga** ma skoordynować zlecanie wysyłki złożonego oraz opłaconego zamówienia, to najlepiej nazwać ją **ShippingSaga**. Taka nazwa jest OK, ale możemy pójść krok dalej. Jeśli nałożymy na naszą funkcjonalność kontekst biznesowy, to możemy zastanowić się, który kodowany element możemy nazwać procesem biznesowym. Koordynacja Eventów **OrderPlaced** oraz **OrderBilled**? Realizacja wysyłki **ShipOrder**? A może jeszcze coś innego?

Patrząc na cały realizowany **Retail E-commerce System** możemy powiedzieć, że składa się on z kilku procesów biznesowych:
* przyjęcie zamówienia
* realizacja płatności za złożone zamówienie
* wysyłka zamówienia

Przyjmując taki punkt widzenia, możemy stwierdzić, że sama koordynacja Eventów jest jednak czymś innym. Pełni rolę tzw. **Policy**. Koordynuje i łączy ze sobą wiele różnych procesów biznesowych. W naszym przypadku są to trzy procesy wymienione powyżej.

Przyjmijmy taką interpretację i nazwijmy naszą Sagę **ShippingPolicy**.

Zakodujmy teraz **Sagę** i przejdźmy przez jej kolejne elementy. W tym celu:

1. dodaj plik **ShippingPolicy.fs** do projektu **Shipping**
2. dodaj do pliku poniższy kod

{% highlight fsharp %}
module ShippingPolicy

open NServiceBus
open Events
open NServiceBus.Logging
open System.Threading.Tasks
open Commands

type ShippingPolicyData() =
    inherit ContainSagaData()
    member val OrderId = "" with get, set
    member val IsOrderPlaced = false with get, set
    member val IsOrderBilled = false with get, set

let canShipOrder (policyData: ShippingPolicyData) =
    policyData.IsOrderPlaced && policyData.IsOrderBilled

let shipOrder orderId (context: IMessageHandlerContext) =
    let shipOrder = new ShipOrder(orderId)
    context.SendLocal(shipOrder)

type ShippingPolicy() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj)
                  .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
    
    static member log = LogManager.GetLogger<ShippingPolicy>()
    
    interface IAmStartedByMessages<OrderPlaced> with
        member this.Handle(message, context) = 
            ShippingPolicy.log.Info("OrderPlaced message received.")
            
            this.Data.IsOrderPlaced <- true
            
            let canShipOrder = canShipOrder this.Data
            match canShipOrder with
            | true ->
                this.MarkAsComplete()
                shipOrder this.Data.OrderId context
            | false ->
                Task.CompletedTask
{% endhighlight %}

#### Saga Data

{% highlight fsharp %}
type ShippingPolicyData() =
    inherit ContainSagaData()
    member val OrderId = "" with get,set
    member val IsOrderPlaced = false with get,set
    member val IsOrderBilled = false with get,set
{% endhighlight %}

Stan sagi definiujemy tworząc klasę z danymi do przechowania. Klasa **ShippingPolicyData** dziedziczy po klasie **ContainSagaData**, która należy do przestrzeni nazw **NServiceBus**. Aby zlecić wysyłkę konkretnego zamówienia, musimy znać jego identyfikator **OrderId** oraz wiedzieć, czy otrzymaliśmy Eventy **OrderPlaced** oraz **OrderBilled**. Informację o przyjściu pierwszego Eventa zapisujemy w member **IsOrderPlaced**. Informację o przyjściu drugiego Eventa zapisujemy w member **IsOrderBilled**. Ponieważ mamy dwa osobne member na przechowanie każdej z informacji, kolejność nadejścia Eventów może być dowolna.

W artykule [jak wysłać wiadomość typu command][7] opisałem bardzo ważną właściwość języka **F#** zwaną **Immutability**. W przypadku danych dla **Sagi** nie możemy z tego skorzystać. Dziedziczenie po klasie **ContainSagaData** wymusza, aby klasa dziedzicząca zawierała bezparametrowy konstruktor. Jak w takim razie podstawiać wartości pod zdefiniowane member? Odpowiedź - używając member, które są widoczne na zewnątrz klasy. **F#** umożliwia konstruowanie publicznych **Auto Property** przeznaczonych zarówno do odczytu, jak i zapisu za pomocą słów `val`, `with`, `get` oraz `set`. Przykład dla **OrderId**:

`member val OrderId = "" with get,set`

Co prawda **F#** umożliwia definiowanie wielu konstruktorów w tej samej klasie za pomocą słowa kluczowego `new`, dzięki czemu możemy dodać wymagany przez kompilator bezparametrowy konstruktor:

{% highlight fsharp %}
type ShippingPolicyData(orderId: string, isOrderPlaced:bool, isOrderBilled:bool) =
    inherit ContainSagaData()
    member this.OrderId = orderId
    member this.IsOrderPlaced = isOrderBilled
    member this.IsOrderBilled = isOrderBilled
    new() =
        ShippingPolicyData("", false, false)
{% endhighlight %}

Natomiast w uruchomionym programie, w momencie konstruowania obiektu **Sagi**, **NServiceBus** zgłosi wyjątek o braku możliwości ustawienia wartości poza konstruktorem:

`System.ArgumentException: Property set method not found.`

Użycie **Auto Property** pokazuje, że możemy wykorzystywać konstrukcje obiektowe jeżyka **F#** tam, gdzie jest to niezbędne.

#### Funkcje pomocnicze

{% highlight fsharp %}
let canShipOrder (policyData: ShippingPolicyData) =
    policyData.IsOrderPlaced && policyData.IsOrderBilled

let shipOrder orderId (context: IMessageHandlerContext) =
    let shipOrder = new ShipOrder(orderId)
    context.SendLocal(shipOrder)
{% endhighlight %}

W momencie otrzymania dowolnego ze Eventów **OrderPlaced** lub **OrderBilled** wykonujemy logikę biznesową sprawdzającą, czy można zlecić wysyłkę zamówienia. Funkcja **canShipOrder** przyjmuje w parametrze klasę reprezentującą dane **Sagi**, po czym sprawdza, czy obydwa Eventy zostały już przetworzone. Jeśli tak, zwraca wartość ``true``. Jeśli nie, zwraca wartość ``false``.

Zlecenie wysyłki zamówienia polega na wysłaniu wiadomości **ShipOrder** za pomocą funkcji **shipOrder**.

#### Definicja Sagi

{% highlight fsharp %}
type ShippingPolicy() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj)
                  .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
    
    static member log = LogManager.GetLogger<ShippingPolicy>()
    
    interface IAmStartedByMessages<OrderPlaced> with
        member this.Handle(message, context) = 
            ShippingPolicy.log.Info("OrderPlaced message received.")
            
            this.Data.IsOrderPlaced <- true
            
            let canShipOrder = canShipOrder this.Data
            match canShipOrder with
            | true ->
                this.MarkAsComplete()
                shipOrder this.Data.OrderId context
            | false ->
                Task.CompletedTask
{% endhighlight %}

Na pierwszy rzut oka kod definicji **Sagi** może wyglądać na skomplikowany. Zaraz zobaczysz, że wcale taki nie jest.

**Sagę** tworzymy przez zdefiniowanie klasy, dziedziczącej po klasie **Saga** należącej do przestrzeni nazw **NServiceBus**. W naszym przykładzie jest to klasa o nazwie **ShippingPolicy**. Bazowa klasa **Saga** w generycznym parametrze przyjmuje typ reprezentujący dane **Sagi**. W naszym przykładzie jest to wcześniej zdefiniowana klasa **ShippingPolicyData**.

Przejdźmy po kolei przez implementację klasy **ShippingPolicy**.

#### Mapowanie wiadomości na Sagę

{% highlight fsharp %}
inherit Saga<ShippingPolicyData>()
    override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
        mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj)
              .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
{% endhighlight %}

Dziedzicząc po klasie `Saga<ShippingPolicyData>` kompilator wymusza na nas zaimplementowanie abstrakcyjnej metody **ConfigureHowToFindSaga**. Metoda w parametrze przyjmuje klasę `SagaPropertyMapper<ShippingPolicyData>`, której używamy do mapowania wiadomości obsługiwanych przez **Sagę** na jej dane.

Do czego jest to potrzebne?

**Saga** może występować w trzech trybach:

* nie rozpoczęta - nie posiada zapisanego stanu
* w trakcie trwania - posiada zapisany stan
* zakończona - zapisany stan zostaje usunięty

**NServiceBus** definiuje pojęcie **Instancji Sagi**. W momencie otrzymania wiadomości framework musi zdecydować, czy stworzyć nową instancję czy też utworzyć już istniejącą. Do podjęcia decyzji potrzebuje informacji, które member jednoznacznie identyfikuje jej konkretną instancję.

W naszym przykładzie każde nowe zamówienie powinno być reprezentowane przez osobną instancję **Sagi**. Naturalnym elementem identyfikacji jest **OrderId**. W momencie nadejścia któregoś z Eventów **OrderPlaced** lub **OrderBilled** **NServiceBus** poszuka w zdefiniowanym **Persistence** zapisanego stanu **Sagi** na podstawie wartości **OrderId**. Jeśli nic nie znajdzie, utworzy nową instancję, a po zakończeniu przetwarzania wiadomości zapisze nowy stan. Jeśli znajdzie wpis, stworzy instancję, wypełniając ją znalezionymi danymi, a po zakończeniu przetwarzania wiadomości zapisze stan ze zaktualizowanymi wartościami. 

W implementacji metody **ConfigureHowToFindSaga** użyliśmy nowych konstrukcji języka **F#**. Przejdźmy po kolei przez poszczególne elementy:

{% highlight fsharp %}
override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) = ...
{% endhighlight %}

Metody abstrakcyjne implementujemy, używając słowa kluczowego `override`.

{% highlight fsharp %}
mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj)
      .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
{% endhighlight %}

Metoda **ConfigureMapping** obiektu **SagaPropertyMapper** w parametrze przyjmuje **Expression**, który w swoim parametrze generycznym przyjmuje **Lambdę** typu **Func**, która jako swój parametr wejściowy przyjmuje typ wiadomości, a zwraca wartość typu **object** - `Expression<Func<TMessage, object>>`. W naszym przykładzie parametrem wejściowym jest **OrderPlaced**, a zwracanym rezultatem **OrderId**.

W **F#** wyrażenia **Lambda** (funkcje anonimowe, których można używać bez konieczności tworzenia nazwanego function value) definiuje się za pomocą słowa kluczowego `fun` oraz operatora `->`. W naszym przykładzie `fun message -> message.OrderId :> obj` oznacza **Lambdę** zgodną z typem `Func<OrderPlaced, object>`, gdzie **message** reprezentuje **OrderPlaced** natomiast **message.OrderId** zwracaną wartość.

Zwracana wartość musi być zgodna z typem `object`. **OrderId** jest wartością typu `string` w związku z tym musimy jawnie rzutować typ `string` na typ `object`. Bez tego kompilator zgłosi nam błąd o niezgodności typów. Rzutowanie w **F#** realizujemy za pomocą operatora `:>`. Standardowa biblioteka **F#** definiuje typ `object` jako `obj`.

W ten sposób mamy zdefiniowane mapowanie wiadomości.

Drugim krokiem jest mapowanie danych **Sagi**.

Metoda **ConfigureMapping** zwraca obiekt typu **ToSagaExpression**, który zawiera metodę **ToSaga**. Metoda w parametrze przyjmuje `Expression<Func<TSagaData, object>>`. Parametrem wejściowym Lambdy **Func** jest typ reprezentujący dane **Sagi**. Zwracana wartość jest typem `object`. W naszym przykładzie definiujemy mapowanie przez `fun sagaData -> sagaData.OrderId :> obj`. Konstrukcja **F#** jest taka sama jak dla metody **ConfigureMapping**. W tym przypadku **sagaData** reprezentuje dane **Sagi** natomiast **sagaData.OrderId** zwracaną wartość.

#### Startowanie Sagi

{% highlight fsharp %}
interface IAmStartedByMessages<OrderPlaced> with
    member this.Handle(message, context) = 
        ShippingPolicy.log.Info("OrderPlaced message received.")
        
        this.Data.IsOrderPlaced <- true
        
        let canShipOrder = canShipOrder this.Data
        match canShipOrder with
        | true ->
            this.MarkAsComplete()
            shipOrder this.Data.OrderId context
        | false ->
            Task.CompletedTask
{% endhighlight %}

**Sagę** startujemy, dziedzicząc po interfejsie `IAmStartedByMessages`, należącym do przestrzeni nazw **NServiceBus**, podając w generycznym parametrze typ wiadomości, który ma ją wystartować. W naszym przykładzie jest to Event **OrderPlaced**. Interfejs `IAmStartedByMessages` zawiera metodę **Handle**, której sygnatura jest taka sama jak przypadku interfejsu `IHandleMessages`.

W implementacji logiki **Sagi** wykorzystujemy te same konstrukcje języka **F#** co w poprzednich artykułach.

Informację o przetworzeniu Eventa **OrderPlaced** zapisujemy w danych **Sagi**:

`this.Data.IsOrderPlaced <- true`

Następnie sprawdzamy, czy możemy zlecić wysyłkę zamówienia za pomocą wcześniej zdefiniowanej funkcji pomocniczej **canShipOrder**. Wynik zwrócony przez funkcję sprawdzamy, używając konstrukcji [Pattern Matching][8]. Jeśli Event **OrderBilled** został przetworzony wcześniej, wywołujemy pomocniczą funkcję **shipOrder**, a następnie za pomocą metody **this.MarkAsComplete** należącej do klasy `Saga`, oznaczamy instancję **Sagi** jako zakończoną.

W przykładzie kolejność wywołań jest odwrotna, aby zachować zgodność zwracanego typu przez funkcję **shipOrder** ze zwracanym typem metody **Handle**. Z punktu widzenia **NServiceBusa** nie ma to znaczenia, ponieważ całą operację wykonuje w transakcji.

Wywołując metodę **this.MarkAsComplete** instruujemy framework, że może usunąć instancję **Sagi** ze swojego persistence.

Jeśli Event **OrderBilled** jeszcze nie nadszedł, to nie robimy nic. **NServiceBus** zakończy **Sagę** aktualizując jej stan.

Ciekawostką jest to, że nie musimy jawnie zapisywać wartości dla **OrderId**. **NServiceBus** sam wywnioskuje, na podstawie konfiguracji mapowania, że wartość ta jest potrzebna i automatycznie zapisze ją w danych **Sagi**.

#### Obsługa kolejnych zdarzeń

Pytanie, jakie może Ci się teraz nasunąć to ***"Gdzie jest implementacja przetwarzania Eventa **OrderBilled**?*** Jeśli popatrzysz na przykład zakodowany w języku **C#** na stronie [tutoriala][2] zobaczysz, że logika przetwarzania tego Eventa zakodowana jest jako kolejne dziedziczenie po interfejsie **IAmStartedByMessages**. Dzięki temu klasa reprezentująca **Sagę** staje się pełnym komponentem odpowiedzialnym za realizację **Shipping Policy**.

{% highlight csharp %}
public class ShippingPolicy : Saga<ShippingPolicyData>,
    IAmStartedByMessages<OrderPlaced>, // I can start the saga!
    IAmStartedByMessages<OrderBilled>  // I can start the saga too!
{% endhighlight %}

W języku **F#** nie możemy zrobić tego samego. W [trzeciej części serii][7] wspomniałem jaki jest tego powód. **F#** w wersji **4.7.0** nie wspiera dziedziczenia po tym samym interfejsie, różniącym się tylko generycznym parametrem:

{% highlight fsharp %}
type ShippingPolicy() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj)
                  .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
    
    interface IAmStartedByMessages<OrderPlaced> with
        member this.Handle(message, context) = 
            ShippingPolicy.log.Info("OrderPlaced message received.")
            //// logic
            Task.CompletedTask

    interface IAmStartedByMessages<OrderBilled> with
        member this.Handle(message, context) = 
            ShippingPolicy.log.Info("OrderBilled message received.")
            //// logic
            Task.CompletedTask
{% endhighlight %}

W takim przypadku dostajemy błąd kompilacji z informacją:

`This type implements the same interface at different generic instantiations 'IAmStartedByMessages<OrderBilled>' and 'IAmStartedByMessages<OrderPlaced>'. This is not permitted in this version of F#.`

Na liście wymagań **F#** istnieje zatwierdzone [RFC][9], którego celem jest dodanie powyższej konstrukcji, w którejś z przyszłych wersji języka.

W tej sytuacji musimy stworzyć osobną klasę przetwarzającą Event **OrderBilled**:

* dodaj do pliku **ShippingPolicy.fs** w projekcie **Shipping** poniższy kod, zaraz za definicją klasy **ShippingPolicy**:

{% highlight fsharp %}
type ShippingPolicy2() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderBilled>(fun message -> message.OrderId :> obj)
                  .ToSaga(fun sagaData -> sagaData.OrderId :> obj)
            
    static member log = LogManager.GetLogger<ShippingPolicy2>()
    
    interface IAmStartedByMessages<OrderBilled> with
        member this.Handle(message, context) = 
            ShippingPolicy2.log.Info("OrderBilled message received.")
            
            this.Data.IsOrderBilled <- true
            
            let canShipOrder = canShipOrder this.Data
            match canShipOrder with
            | true ->
                this.MarkAsComplete()
                shipOrder this.Data.OrderId context
            | false ->
                Task.CompletedTask
{% endhighlight %}

Idea implementacji jest taka sama jak w przypadku przetwarzania Eventa **OrderPlaced**. Różnice to:

* w metodzie **ConfigureHowToFindSaga** oraz interfejsie **IAmStartedByMessages** używamy Eventa **OrderBilled**
* zapamiętujemy przetworzenie Eventa w przeznaczonym do tego member danych **Sagi** - `this.Data.IsOrderBilled <- true`

W ten sposób zarówno Event **OrderPlaced** jak i **OrderBilled** mogą stworzyć nową instancję **Sagi**. Jeśli którykolwiek z Eventów przyjdzie jako drugi, to **NServiceBus** wykryje zapisany stan i stworzy instancję z aktualnymi danymi. Dzięki temu zlecenie wysyłki zamówienia wykona się tylko wtedy, kiedy obydwa Eventy zostaną przetworzone, bez względu na to, w jakiej nadejdą kolejności.

Zmieńmy jeszcze nazwy klas na bardziej opisowe. Zrefaktoryzuj kod zamieniając:

* nazwę klasy **ShippingPolicy** na **ShippingPolicyOrderPlaced**
* nazwę klasy **ShippingPolicy2** na **ShippingPolicyOrderBilled**

## Saga Persistence

Ostatnim elementem implementacji jest zdefiniowanie **Persistence**, w którym **Saga** będzie przechowywać swój stan.

Dodaj do pliku **Program.cs**, w projekcie **Shipping**, poniższy kod, zaraz za utworzeniem transportu:

`endpointConfiguration.UsePersistence<LearningPersistence>() |> ignore`

Podobnie jak transport `LearningTransport`, persistence `LearningPersistence` przeznaczony jest do prototypowania i testowania idei rozwiązania. Nie są to elementy do użytku na środowisku produkcyjnym. Więcej o dostępnych **Persistence** możesz przeczytać w [dokumentacji][6].

## Usunięcie zastąpionego kodu

Usuńmy jeszcze kod, który zastąpiliśmy implementacją **Sagi**. W tym celu:

* przejdź do pliku **Handlers.fs** w projekcie **Shipping**
* usuń Handlery **OrderPlacedHandler** oraz **OrderBilledHandler**

## Uruchomienie

W tym momencie całość jest gotowa do uruchomienia:

*  w **Visual Studio** naciśnij klawisz **F5**
* po wystartowaniu wszystkich czterech **Endpointów**, wyślij wiadomość z **ClientUI** wciskając klawisz **P**
* **Shipping** powinien zalogować przetworzenie Eventów przez **Sagę** oraz wysyłkę zamówienia:

{% highlight txt %}
INFO  ShippingPolicy+ShippingPolicyOrderPlaced OrderPlaced message received.
INFO  ShippingPolicy+ShippingPolicyOrderBilled OrderBilled message received.
INFO  Handlers+ShipOrderHandler Order 7453805a-4da6-462c-81c6-e19ee0614303 - Successfully shipped.
{% endhighlight %}

W momencie, kiedy Event **OrderBilled** przyjdzie jako pierwszy, **Saga** zaczeka ze zleceniem wysyłki, dopóki nie otrzyma Eventa **OrderPlaced**.

Jeśli nie widzisz u siebie takiego efektu, możesz wspomóc się przykładem, który udostępniam na moim [GitHubie][1].

**Saga** odgrywa bardzo ważną rolę w projektowaniu rozwiązań bazujących na **Messagingu** oraz **Queueingu**. Jest niezbędnym narzędziem w funkcjonalnościach w których wiadomości przesyłane są w sposób nieliniowy. Pomaga w koordynacji procesów biznesowych, składających się w jedno większe biznesowe **Policy**. **NServiceBus** "zdejmuje" z nas konieczność implementacji szczegółów infrastruktury, takich jak zarządzanie stanem, jego spójnością, tworzeniem oraz usuwaniem instancji **Sagi**.

**Saga** zawiera dużo innych możliwości wykraczających poza zakres tego artykułu jak np. "wybudzanie" po zdefiniowanym czasie. **NServiceBus** umożliwia także testowanie logiki **Sagi** za pomocą [Unit Testów][10].

Język **F#** umożliwia dziedziczenie po klasach abstrakcyjnych oraz implementację abstrakcyjnych metod należących do dziedziczonej klasy. Jest to kolejny element wsparcia programowania obiektowego w języku funkcyjnym. **Wyrażenia lambda** pozwalają na tworzenie funkcji "w locie" bez konieczności definiowania nazwanych **function values**. W kontekście implementacji **Sagi** i jej podstawowego założenia implementacji **Message Handlerów** w jednej klasie brak wsparcia dziedziczenia po tym samym interfejsie różniącym się tylko parametrem generycznym jest zauważalnym ograniczeniem.

W tym miejscu planowałem podsumować całą serię, natomiast postanowiłem zrobić to w osobnym artykule. Trzymając się aktualnego wątku, mogę stwierdzić, że **Saga** przedstawiająca połączenie frameworka **NServiceBus** z językiem **F#** jeszcze trwa i zakończy się w ostatniej części tej serii :)

[1]: https://github.com/mikedevbo/fsharp-nservicebus "fsharp-nservicebus"
[2]: https://docs.particular.net/tutorials/nservicebus-sagas/1-getting-started/ "saga getting started"
[3]: https://docs.particular.net/nservicebus/sagas/ "sagas"
[4]: https://www.enterpriseintegrationpatterns.com/ "enterprise integration patterns"
[5]: https://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html "process manager"
[6]: https://docs.particular.net/persistence/ "persistence"
[7]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[8]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/pattern-matching "pattern matching"
[9]: https://github.com/fsharp/fslang-design/blob/master/RFCs/FS-1031-Allow%20implementing%20the%20same%20interface%20at%20different%20generic%20instantiations%20in%20the%20same%20type.md "F# RFC FS-1031"
[10]: https://docs.particular.net/nservicebus/testing/ "NServiceBus testing"
[11]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[12]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[13]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[14]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}
[15]: {{ site.url }}{% link _posts/2019-11-11-fsharp-nservicebus-practical-guide-publishing-events.md %}
[16]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[17]: {{ site.url }}{% link _posts/2019-12-22-fsharp-nservicebus-practical-guide-ending.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}