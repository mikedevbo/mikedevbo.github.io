---
layout: post
title:  "F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa"
date:   2019-09-16
---

Posty z tej serii:

* [F# & NServiceBus - praktyczny przewodnik: Wprowadzenie][1]
* [F# & NServiceBus - praktyczny przewodnik: Konfiguracja Endpointa][6]
* [F# & NServiceBus - praktyczny przewodnik: Wysłanie komendy][8]
* [F# & NServiceBus - praktyczny przewodnik: Komunikacja wielu Endpointów][9]
* [F# & NServiceBus - praktyczny przewodnik: Publikowanie zdarzeń][10]
* F# & NServiceBus - praktyczny przewodnik: Zarządzanie procesem biznesowym - Saga

[W poprzednim artykule][1] opisałem motywy stojące za powstaniem niniejszej serii. Jeśli chcesz się dowiedzieć dlaczego **F#** oraz dlaczego **NServiceBus**, zachęcam Cię do jego przeczytania.

Cała seria bazuje na bardzo dobrym [Tutorialu][2] wprowadzającym do **NServiceBusa**, który pokazuje i wyjaśnia główne koncepcje stojące na frameworkiem. Tutorial zawiera przykłady napisane w języku **C#**. W całej serii posłużę się tym samym zagadnieniem i pokażę Ci, w jaki sposób zbudować **Retail E-commerce System**, używając do tego języka **F#**. Dzięki temu będziesz mieć możliwość odniesienia się do źródła.

W przypadku gdy coś okaże się niejasne, będziesz mieć pytania lub spostrzeżenia, podziel się nimi w komentarzu. Jeśli znasz język **F#** i stwierdzisz, że to o czym piszę, jest niepoprawne, również podziel się tym w komentarzu.

W tym artykule pokażę Ci, w jaki sposób utworzyć, skonfigurować oraz wystartować nowy **Endpoint NServiceBusa**. Do napisania i uruchomienia kodu na własnej maszynie, będziesz potrzebować:

* Visual Studio 2017
* .NET Core 2.2
* F# 4.7.0
* NServiceBus 7.1.10

## Stworzenie solucji

W pierwszej kolejności stwórzmy nową solucję oraz projekt w **Visual Studio**. W tym celu:

1. wybierz z menu **File...New...Project**
2. następnie **Visual F#...NET Core...Console App (.NET Core)**
3. zaznacz opcję **Create directory for solution**
4. w pole **Name** wpisz nazwę projektu **ClientUI**
5. w pole **Solution name** wpisz nazwę solucji **RetailDemo**

Po utworzeniu projektu otworzy się plik **Program.fs** zawierający implementację funkcjonalności, wypisującej tekst na standardowe wyjście:

{% highlight fsharp %}
// Learn more about F# at http:fsharp.org
open System

[<EntryPoint>]
let main argv =
    printfn "Hello World from F#!"
    0 // return an integer exit code 
{% endhighlight %}

Przejdźmy po kolei przez utworzone konstrukcje:

* **// Learn more about...** - jednowierszowy komentarz
* **open System** - zaimportowanie elementów z przestrzeni nazw `System` wchodzącej w skład standardowej biblioteki `.NET Core`
* **[&lt;EntryPoint&gt;]** - atrybut określający tzw. funkcję wejścia, która ma być uruchomiona przy starcie programu
* **let main argv = ...** - deklaracja oraz definicja funkcji
    * **let** - słowo kluczowe języka **F#** wiążące nazwę z funkcją
    * **main** - nazwa funkcji wejścia
    * **argv** - parametry funkcji wejścia
    * **=** - element definicji funkcji
        * **printf "..."** - funkcja ze standardowej biblioteki **F#** wypisująca tekst na standardowe wyjście 
        * **0** - zwrócenie wartości oznaczającej prawidłowe zakończenie programu

Tak mało kodu, a tak dużo pojęć :) W skrócie:

* słowo kluczowe **let** wiąże nazwę z funkcją, tworząc tzw. **Function Value** - w skrócie **Function**
* parametry funkcji definiowane są po spacji, bez użycia nawiasów **()**
* wcięcia określają kolejne bloki programu
* ostatnia linijka w ciele funkcji oznacza zwracaną wartość - brak konieczności używania słowa **return**
* kompilator automatycznie wnioskuje typ zwracany przez funkcję, a także typy jej parametrów wejściowych

Przykład porównujący język **F#** do języka **C#** możesz znaleźć w artykule [Comparing F# with C#: A simple sum][3].

## Konfiguracja Endpointa

Przykłady z tej serii bazują na **F#** w wersji 4.7. W przypadku gdy **Visual Studio** utworzyło projekt ze starszą wersją, należy dokonać aktualizacji. W tym celu w **Package Manager Console** uruchom komendę:

`Get-Package -ProjectName ClientUI`

Jeśli paczka **FSharp.Core** jest niższa niż wersja 4.7, uruchom komendę:

`Update-Package FSharp.Core -Version 4.7.0 -ProjectName ClientUI`

Jeśli paczka jest większa niż wersja 4.7, jest duża szansa, że przykłady zadziałają poprawnie.

Przejdźmy teraz do konfiguracji **Endpointa**. Zainstaluj paczkę **NServiceBus**, uruchamiając komendę:

`Install-Package NServiceBus -Version 7.1.10 -ProjectName ClientUI`

Podobnie jak dla języka **F#**, jest duża szansa, że przykłady zadziałają poprawnie w wyższej wersji **NServiceBusa**. Jeśli chcesz pobrać wersję najnowszą, pomiń parametr **-Version 7.1.10**

Zobaczmy, jak wygląda ostateczne rozwiązanie, a następnie przejdziemy przez kolejne jego elementy:

{% highlight fsharp %}
open System
open NServiceBus

[<EntryPoint>]
let main argv =
    
    let endpointName = "ClientUI"
    Console.Title <- endpointName

    let endpointConfiguration = new EndpointConfiguration(endpointName)
    let transport = endpointConfiguration.UseTransport<LearningTransport>()    
    
    async {

        let! endpointInstance = Endpoint.Start(endpointConfiguration) |> Async.AwaitTask
            
        printf "Press Enter to exit..."
        Console.ReadLine() |> ignore

        do! endpointInstance.Stop() |> Async.AwaitTask

    } |> Async.RunSynchronously

    0 // return an integer exit code
{% endhighlight %}

#### Nazwa Endpointa

{% highlight fsharp %}
let endpointName = "ClientUI"
Console.Title <- endpointName
{% endhighlight %}

Dwie linijki kodu, ale jakże ciekawe. Używając słowa kluczowego **let**, wiążemy nazwę **endpointName** z napisem **ClientUI**. W ten sposób **endpointName** staje się tzw. **Simple Value** - w skrócie **Value**. Tak jak w przypadku definiowania funkcji, tak i tutaj, podawanie typu nie jest konieczne. Kompilator sam wydedukuje, że jest to **Value** typu **string**. Możesz to sprawdzić, najeżdżając w **Visual Studio** kursorem na nazwę **endpointName**.

Na pierwszy rzut oka wydaje się, że jest to przypisanie wartości do zmiennej. Tak nie jest, ponieważ **endpointName** nie jest zmienną, a wartością, której po utworzeniu nie można zmienić! Jest to właściwość zwana **immutable**. Jest ona charakterystyczna dla programowania funkcyjnego.

Aby przekonać się, że **endpointName** nie jest zmienną, napisz w następnej linijce **Visual Studio** fragment kodu:

`endpointName = "UIClient"`

Efekt? Kod się skompiluje, ale dodany fragment nie zmieni zachowania programu. W takiej sytuacji **Visual Studio** podpowiada przyczynę takiego zachowania. W tym kontekście **F#** traktuje `=` jako operator porównania dwóch elementów. Wynikiem jest porównanie wartości, która związana jest z nazwą **endpointName**, z napisem **UIClient**. Z rezultatem takiego porównania nic nie jest robione, więc w podpowiedzi widzimy sugestię, czy aby na pewno chodziło nam o takie zachowanie.

W **F#** operatorem przypisania jest `<-`:

`Console.Title <- endpointName`

W tym przypadku pod property **Title**, klasy **Console**, podstawiamy wartość reprezentowaną przez wcześniej zdefiniowane **Value** o nazwie **endpointName**. Tak jak pisałem w [poprzednim artykule][1], **F#** jest językiem funkcyjnym, ale wspiera elementy programowania obiektowego. To jest właśnie jeden z tych elementów. A co się stanie, jak pod **endpointName** będziemy chcieli podstawić inną wartość?

`endpointName <- "UIClient"`

W tym przypadku kod nie skompiluje się! Dostajemy informację, że **endpointName** jest **immutable**.

Jeśli z jakichś powodów potrzebujemy zmieniać raz zdefiniowane **Value**, to możemy użyć słowa kluczowego **mutable**:

{% highlight fsharp %}
let mutable endpointName = "ClientUI"
endpointName <- "UIClient"
{% endhighlight %}

Unikamy takich konstrukcji, jak tylko się da, ponieważ jest to jeden z tych elementów, który popycha nas w kierunku programowania zorientowanego obiektowo, zamiast programowania funkcyjnego.

#### Konfiguracja Endpointa

{% highlight fsharp %}
let endpointConfiguration = new EndpointConfiguration(endpointName)
let transport = endpointConfiguration.UseTransport<LearningTransport>()
{% endhighlight %}

W tym miejscu przechodzimy do konfiguracji **Endpointa**. Tworzymy obiekt **EndpointConfiguration** należący do przestrzeni nazw **NServiceBus**, podając w konstruktorze jego nazwę. W przykładzie jest to wcześniej zdefiniowane **Value** o nazwie **endpointName**. Podobnie jak w przypadku definiowania funkcji oraz Value, do utworzenia obiektu związanego z nazwą **endpointConfiguration** używamy słowa kluczowego **let**. Do utworzenia obiektu używamy słowa kluczowego **new**.

Ćwiczenie dla Ciebie. Spróbuj podstawić pod **endpointConfiguration** nowy obiekty, używając operatora `<-`. Czy rozpoznajesz schemat działania?

Na utworzonym obiekcie, w ten sam sposób jak w językach obiektowych, możemy wywoływać metody za pomocą symbolu `.`

`let transport = endpointConfiguration.UseTransport<LearningTransport>()`

**F#** w pełni wspiera typy generyczne. W powyższym przykładzie, wywołując metodę **UseTransport** obiektu **endpointConfiguration**, konfigurujemy **Endpoint** w taki sposób, aby do przesyłania wiadomości używał transportu **LearningTransport**. Jednocześnie wiążemy zwracany przez metodę obiekt typu **TransportExtensions&lt;LearningTransport&gt;** z nazwą **transport**. Więcej o transportach **NServiceBusa** możesz przeczytać w [Dokumentacji][4].

Powyższy fragment kodu jest kolejnym przykładem wykorzystania programowania obiektowego w języku **F#**. Pokazuje też, że kod napisany na platformę **.NET** w innym języku niż **F#**, może być z powodzeniem używany (API **NServiceBusa** napisane jest w języku **C#**). Bez wsparcia konstrukcji obiektowych, zapewne nie byłoby to możliwe.

## Wystartowanie Endpointa

{% highlight fsharp %}
async {

    let! endpointInstance = Endpoint.Start(endpointConfiguration) |> Async.AwaitTask
        
    printf "Press Enter to exit..."
    Console.ReadLine() |> ignore

    do! endpointInstance.Stop() |> Async.AwaitTask

} |> Async.RunSynchronously
{% endhighlight %}

Mając skonfigurowanego **Endpointa**, możemy go wystartować. API **NServiceBusa** wspiera model programowania asynchronicznego **async/await** wprowadzonego w języku **C#** 5.0. **F#** również wspiera programowanie asynchroniczne. Przejdźmy przez kolejne elementy powyższego kodu:

{% highlight fsharp %}
async {

    // ...

} |> Async.RunSynchronously
{% endhighlight %}

Kod asynchroniczny zapisujemy w specjalnym bloku `async {..}`. Do uruchomienia takiego kodu używamy statycznego **member** **RunSynchronously** klasy **Async**. Tutaj ciekawostka nazewnicza. W **C#** klasa może zawierać takie elementy, jak: pole, property, metoda, ... W **F#** wszystkie te elementy nazywają się **member**.

W przykładzie widzimy tzw. **pipe operator** - `|>`. Pozwala on na bardziej czytelny zapis kodu, który wynik działania jednej funkcji przekazuje jako argument do drugiej funkcji. Przykładowo, mając zdefiniowane funkcje f1, f2 i f3, zamiast pisać:

{% highlight fsharp %}
let r1 = f1 x
let r2 = f2 r1
let result = f3 r2
{% endhighlight %}

możemy napisać:

`let result = x |> f1 |> f2 |> f3`

W przypadku uruchomienia kodu asynchronicznego bez operatora `|>`  musielibyśmy napisać trochę mniej czytelny kod:

{% highlight fsharp %}
Async.RunSynchronously(async { 
    ...
})
{% endhighlight %}

**Endpoint NServiceBusa** startujemy poprzez wywołanie statycznej metody **Start** należącej do klasy **Endpoint**. W parametrze metody przekazujemy wcześniej utworzoną konfigurację:

`let! endpointInstance = Endpoint.Start(endpointConfiguration) |> Async.AwaitTask`

Obiekt zwracany przez metodę **Start** wiążemy z nazwą **endpointInstance**, używając do tego słowa kluczowego **let!**, które służy do wiązania wyniku działania operacji asynchronicznej - uwaga, nie mylić ze słowem kluczowym **let**. Dzięki member **AwaitTask** klasy **Async** dostajemy wynik działania operacji asynchronicznej.

W tym momencie **Endpoint** jest wystartowany i gotowy do wysyłania lub procesowania wiadomości. Aby uniknąć natychmiastowego zamknięcia okna konsoli, programujemy funkcjonalność oczekiwania na wciśnięcie klawisza **Enter**:

{% highlight fsharp %}
printf "Press Enter to exit..."
Console.ReadLine() |> ignore
{% endhighlight %}

Ćwiczenie dla Ciebie. Wpisz samo wywołanie metody **Console.ReadLine()**. Efekt? **Visual Studio** podpowiada, że z wartością zwracaną przez metodę **ReadLine()** nic nie jest robione. Jeśli taka jest intencja, a w naszym przykładzie tak jest, warto zapisać ją w sposób jawny. Służy do tego specjalna funkcja wbudowana w bibliotekę **F#** o nazwie **ignore**.

W przykładzie ponownie wykorzystany jest **pipe operator**. Propozycja kolejnego ćwiczenia. Spróbuj zapisać to samo wywołanie, ale bez użycia operatora **pipe**.

Ostatnią rzeczą, którą musimy zrobić, jest zatrzymanie **Endpointa** w momencie zamykania programu:

`do! endpointInstance.Stop() |> Async.AwaitTask`

Asynchroniczna metoda API **NServiceBusa** o nazwie **Stop** nie zwraca żadnego rezultatu. W takim przypadku wykorzystujemy **do!**, aby wykonać operację asynchroniczną.

W tym momencie całość gotowa jest do uruchomienia. Naciśnij w **Visual Studio** klawisz **F5**. Zobaczysz okno konsoli z informacją, że **Endpoint** wystartował:

[![Picutre1][5]][5]

To tyle, jeśli chodzi o konfigurację **Endpointa NServiceBusa**. Całe rozwiązanie znajdziesz na [GitHubie][7].

W następnym artykule pokażę Ci, w jaki sposób stworzyć nowy message, a także jak wysłać oraz przetworzyć message na danym **Endpointcie**.

[1]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[2]: https://docs.particular.net/tutorials/nservicebus-step-by-step/ "nservicebus-step-by-step"
[3]: https://fsharpforfunandprofit.com/posts/fvsc-sum-of-squares/ "fvsc-sum-of-squares"
[4]: https://docs.particular.net/transports/ "nservicebus transports"
[5]: {{ site.url }}/assets/fsharp/fsharp_nsb_start_endpoint.png
[6]: {{ site.url }}{% link _posts/2019-09-16-fsharp-nservicebus-practical-guide-endpoint-configuration.md %}
[7]: https://github.com/mikedevbo/fsharp-nservicebus "fsharp-nservicebus"
[8]: {{ site.url }}{% link _posts/2019-10-01-fsharp-nservicebus-practical-guide-sending-command.md %}
[9]: {{ site.url }}{% link _posts/2019-10-13-fsharp-nservicebus-practical-guide-multiple-endpoints.md %}
[10]: {{ site.url }}{% link _posts/2019-11-11-fsharp-nservicebus-practical-guide-publishing-events.md %}

{{ site.mark_post_as_end }}

### {{ site.comments }}