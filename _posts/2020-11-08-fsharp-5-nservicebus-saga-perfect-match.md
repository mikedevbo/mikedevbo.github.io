---
layout: post
title:  "F# 5 - NServiceBus Saga - idealne dopasowanie"
description: Sprawdź, w jaki sposób wykorzystać nowe właściwości języka F# do implementacji NServiceBus Saga.
date:   2020-11-08
---

Wydanie **5'tej** wersji frameworka .NET [zbliża się wielkimi krokami][1].  W tym samym czasie [pojawi się nowa wersja][2] **F#**, która również będzie oznaczona numerem **5**. Jedna z nowych właściwości języka to:

[Allow implementing the same interface at different generic instantiations][3]

Jest to kluczowa zmiana w kontekście implementacji **Sagi** frameworka **NServiceBus**.

Kod **Sagi** w 4'tej wersji **F#** trzeba rozbijać na osobne klasy, które implementują interfejs `IAmStartedByMessages` lub `IHandleMessages` z konkretnym typem generycznym. Szczegóły takiego sposobu implementacji opisałem w [szóstej części][4] praktycznego przewodnika pokazującego, w jaki sposób można połączyć framework **NServiceBus** oraz język **F#**.

Kod **Sagi** w 5'tej wersji **F#** można napisać w taki sam sposób jak kod napisany w **C#**, przedstawiony [w dokumentacji][5] frameworka. Dzięki temu implementacja upraszcza się i jest spójna w obydwu językach.

Porównajmy kod przed i po wprowadzeniu zmian.

## NServiceBus Saga - F# 4

Kod dostępny jest [na moim koncie GitHub][6]

{% highlight fsharp %}
module ShippingPolicy

open NServiceBus
open Events
open NServiceBus.Logging
open System.Threading.Tasks
open Commands

type ShippingPolicyData() =
    inherit ContainSagaData()
    member val OrderId = "" with get,set
    member val IsOrderPlaced = false with get,set
    member val IsOrderBilled = false with get,set

let canShipOrder (policyData: ShippingPolicyData) =
    policyData.IsOrderPlaced && policyData.IsOrderBilled

let shipOrder orderId (context: IMessageHandlerContext) =
    let shipOrder = new ShipOrder(orderId)
    context.SendLocal(shipOrder)

type ShippingPolicyOrderPlaced() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj).ToSaga(fun sagaData -> sagaData.OrderId :> obj)
    
    static member log = LogManager.GetLogger<ShippingPolicyOrderPlaced>()
    
    interface IAmStartedByMessages<OrderPlaced> with
        member this.Handle(message, context) = 
            ShippingPolicyOrderPlaced.log.Info("OrderPlaced message received.")
            
            this.Data.IsOrderPlaced <- true
            
            let canShipOrder = canShipOrder this.Data
            match canShipOrder with
            | true ->
                this.MarkAsComplete()
                shipOrder this.Data.OrderId context
            | false ->
                Task.CompletedTask

type ShippingPolicyOrderBilled() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderBilled>(fun message -> message.OrderId :> obj).ToSaga(fun sagaData -> sagaData.OrderId :> obj)
            
    static member log = LogManager.GetLogger<ShippingPolicyOrderBilled>()
    
    interface IAmStartedByMessages<OrderBilled> with
        member this.Handle(message, context) = 
            ShippingPolicyOrderBilled.log.Info("OrderBilled message received.")
            
            this.Data.IsOrderBilled <- true
            
            let canShipOrder = canShipOrder this.Data
            match canShipOrder with
            | true ->
                this.MarkAsComplete()
                shipOrder this.Data.OrderId context
            | false ->
                Task.CompletedTask
{% endhighlight %}

Szczegółowy opis znajdziesz we wspomnianym już [praktycznym przewodniku][4]. Zwróć uwagę na konieczność utworzenia dwóch klas, które składają się na implementację **Sagi NServiceBus**:

* `ShippingPolicyOrderPlaced` - przetwarza wiadomość `OrderPlaced`
* `ShippingPolicyOrderBilled` - przetwarza wiadomość `OrderBilled`

Dodatkowo istnieją dwie osobne funkcje, które wykorzystywane są w obydwu klasach:

* `canShipOrder` - zwraca informację, czy zamówienie może zostać wysłane
* `shipOrder` - wysyła wiadomość, która inicjuje Policy wysyłki zamówienia

A jak może wyglądać implementacja tej samej funkcjonalności w nowej wersji języka **F#**? Zobaczmy.

## NServiceBus Saga - F# 5

Kod dostępny jest [na moim koncie GitHub][7]

{% highlight fsharp %}
module ShippingPolicy

open NServiceBus
open Events
open NServiceBus.Logging
open System.Threading.Tasks
open Commands

type ShippingPolicyData() =
    inherit ContainSagaData()
    member val OrderId = "" with get,set
    member val IsOrderPlaced = false with get,set
    member val IsOrderBilled = false with get,set

type ShippingPolicy() =
    inherit Saga<ShippingPolicyData>()
        override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<ShippingPolicyData>) =
            mapper.ConfigureMapping<OrderPlaced>(fun message -> message.OrderId :> obj).ToSaga(fun sagaData -> sagaData.OrderId :> obj)
            mapper.ConfigureMapping<OrderBilled>(fun message -> message.OrderId :> obj).ToSaga(fun sagaData -> sagaData.OrderId :> obj)

    static member log = LogManager.GetLogger<ShippingPolicy>()

    interface IAmStartedByMessages<OrderPlaced> with
        member this.Handle(message, context) =
            ShippingPolicy.log.Info("OrderPlaced message received.")

            this.Data.IsOrderPlaced <- true
            this.ProcessOrder(context)

    interface IAmStartedByMessages<OrderBilled> with
        member this.Handle(message, context) =
            ShippingPolicy.log.Info("OrderBilled message received.")

            this.Data.IsOrderBilled <- true
            this.ProcessOrder(context)

    member this.ProcessOrder(context:IMessageHandlerContext) =
        match this.Data.IsOrderPlaced && this.Data.IsOrderBilled with
        | true ->
            this.MarkAsComplete()
            let shipOrder = new ShipOrder(this.Data.OrderId)
            context.SendLocal(shipOrder)
        | false ->
            Task.CompletedTask
{% endhighlight %}

Do implementacji **NServiceBus Saga**, z wykorzystaniem nowych możliwości języka **F#**, wystarczy jedna klasa, dziedzicząca dwa razy po tym samym interfejsie `IAmStartedByMessages`, który różni się tylko parametrem generycznym:

* `IAmStartedByMessages<OrderPlaced>` - przetwarzanie wiadomości typu `OrderPlaced`
* `IAmStartedByMessages<OrderBilled>` - przetwarzanie wiadomości typu `OrderBilled`

Ponieważ całość może znajdować się w jednej klasie, możemy uprościć kod, przenosząc logikę działania funkcji `canShipOrder` oraz `shipOrder` do metody `ProcessOrder`, która znajduje się w samej klasie.

**F# 5** wprowadza wiele innych przydatnych nowości, natomiast z punktu widzenia implementacji **Sagi** zmiana opisana w tym artykule jest najistotniejsza. Jeśli **NServiceBus Saga** jest [ważną częścią][8] zaprojektowanych rozwiązań, to możliwość dziedziczenia po tym samym interfejsie różniącym się parametrem generycznym zachęca do kodowania całych rozwiązań w języku **F#**. Daje to ciekawą perspektywę na przyszłość :)

Tymczasem udanego oglądania materiałów z nadchodzącej [konferencji][1]!

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://devblogs.microsoft.com/dotnet/net-5-0-launches-at-net-conf-november-10-12/ "net 5 launches at net conf november 10-12"
[2]: https://devblogs.microsoft.com/dotnet/f-5-update-for-august/ "f# 5 update for august/"
[3]: https://devblogs.microsoft.com/dotnet/f-5-update-for-august/#interfaces-can-be-implemented-at-different-generic-instantiations "interfaces can be implemented at different generic instantiations"
[4]: {{ site.url }}{% link _posts/2019-12-15-fsharp-nservicebus-practical-guide-configure-and-use-saga.md %}
[5]: https://docs.particular.net/tutorials/nservicebus-sagas/1-getting-started/ "NServiceBus Saga getting started"
[6]: https://github.com/mikedevbo/fsharp-nservicebus/tree/fsharp-4 "mikedevbo F# 4 NServiceBus"
[7]: https://github.com/mikedevbo/fsharp-nservicebus "mikedevbo F# 5 NServiceBus"
[8]: {{ site.url }}{% link _posts/2020-09-29-adsd-nservicebus-csharp-fsharp-lets-dance-design.md %}