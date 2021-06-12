---
layout: post
title:  "F# Bolero - A cóż to takiego? - HTML templates"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób używać HTML Templates do konstruowania UI."
date:   2021-02-07
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][4]
* [F# Bolero - A cóż to takiego? - HTML templates][7]
* [F# Bolero - A cóż to takiego? - view components][8]
* [F# Bolero - A cóż to takiego? - routing][9]
* [F# Bolero - A cóż to takiego? - routing - page models][10]
* ...

Konstruowanie elementów UI [w kodzie F#][1]. Czy jest to dobre podejście? Kiedy pierwszy raz zobaczyłem taką możliwość, pomyślałem - *ciekawe, ale nie jestem do końca przekonany*. Z jednej strony mamy wsparcie kompilatora, który pilnuje poprawności wyniku. Z drugiej strony, każda, nawet najmniejsza zmiana wymaga re-kompilacji całości. Z trzeciej strony, dla osób, które piszą **backend**, re-kompilacja po zmianie to norma. Z czwartej strony, nie da się podzielić pracy tak, aby projektowaniem UI oraz przygotowaniem prototypu zajmowały się osoby od projektowania, a programowaniem całości zajmowały się osoby od programowania. Wszystko jest w kodzie **F#**, więc wszystko trzeba zaprogramować. Na chwilę obecną nadal podchodzę z pewną rezerwą do takiego podejścia, ale nie skreślam całkowicie.

**Bolero** umożliwia także alternatywny sposób konstruowania elementów UI - za pomocą tzw. [HTML Templates][2]. Do realizacji używamy standardowych znaczników **HTML**, w których możemy zawrzeć tzw. **Holes**. Służą one jako łącznik pomiędzy kodem UI napisanym w **HTML**, a logiką napisaną w **F#**. Dzięki temu możemy wyświetlać dynamiczne elementy na stronie oraz pobierać informacje, wprowadzane w formularzach. Nazywa się to **bindowaniem** danych. Możemy również wydzielać powtarzalne części za pomocą tzw. **Nested Templates**.

Zobaczmy, na przykładach, jak to wszystko działa.

Całą implementację znajdziesz na [GitHub'e][3].

## Simple Counter

W [poprzedniej części][4] stworzyliśmy prosty licznik, przesuwający wartość w dwie strony. Zamienimy kod renderujący widok tak, aby używał **HTML Templates**.

Tworzymy plik `counter.html` z poniższym kodem i umieszczamy w katalogu `wwwroot`:

{% highlight html %}
<div>
    <button onclick="${Decrement}">-</button>
    <span>${Value}</span>
    <button onclick="${Increment}">+</button>
</div>
<div>
    <button onclick="${Reset}">Reset</button>
</div>
{% endhighlight %}

Między znacznikami **HTML** znajdują się **Holes** `${Decrement}`, `${Value}`, `${Increment}` oraz `${Reset}` pod które podstawimy wartości w kodzie **F#**.

W pliku `Main.fs` definiujemy typ, który będzie reprezentował **Template**:

{% highlight fsharp %}
type Counter = Template<"wwwroot/counter.html">
{% endhighlight %}

Zamieniamy definicję funkcji `view` z:

{% highlight fsharp %}
let view model dispatch =
    concat [
        div [] [
            button [on.click (fun _ -> dispatch Decrement)] [text "-"]
            text (string model.value)
            button [on.click (fun _ -> dispatch Increment)] [text "+"]
        ]

        div [] [
            button [on.click (fun _ -> dispatch Reset)] [text "Reset"]
        ]
    ]
{% endhighlight %}

na:

{% highlight fsharp %}
let view model dispatch =
    Counter()
        .Decrement(fun _ -> dispatch Decrement)
        .Value(string model.value)
        .Increment(fun _ -> dispatch Increment)
        .Reset(fun _ -> dispatch Reset)
        .Elt()
{% endhighlight %}

Konstruktor `Counter()` inicjalizuje **Template**. **Bolero** rozpoznaje **Holes** zdefiniowane w pliku `counter.html`. Dla każdego tworzy odpowiednik w postaci metody z taką samą nazwą oraz pasującą sygnaturą. W ten sposób możemy zakodować zachowanie programu:

* `Decrement`, `Increment` oraz `Reset` reprezentują kliknięcie odpowiedniego przycisku na stronie
* `Value` reprezentuje wyświetlaną wartość licznika
* `Elt` jest wewnętrzną metodą **Bolero**, która konwertuje całą definicję do końcowego typu

Wszystko jest silnie typowane. Mamy pełne wsparcie kompilatora oraz podpowiadanie składni. W [JetBrains Rider][5] wygląda to tak:

![Picture1]({{ site.url }}/assets/fsharp_bolero/bolero_temp_code_comp.png)

Stan licznika wyświetlamy, wołając metodę `Value`. Do parametru metody przekazujemy wartość `model.value`, skonwertowaną z typu `int` - taki przechowujemy w modelu, na typ `string` - takiego spodziewa się metoda. Służy do tego operator o nazwie `string`.

Zachowanie kodujemy, wykorzystując funkcję `dispatch`, której podajemy odpowiedni `Message` reprezentujący konkretną akcję.

A jak działa **bindowanie**? Sprawdźmy to, dodając do licznika funkcjonalność ustawiania liczby, która będzie wskazywać, o ile licznik ma się przesuwać.

Rozszerzamy `counter.html` o możliwość wprowadzenia wartości:

{% highlight html %}
<div>
    <label>
        Step: <input type="text"  bind="${Step}" onkeydown="${KeyDown}" />
        <span>Key: ${Key} - Key Code: ${KeyCode}</span>
    </label>
</div>
<!-- whole code you can see on GitHub
// ... -->
{% endhighlight %}

Atrybut `bind` łączy wartość znacznika `input` z **Hole** `${Step}`. Przykład zawiera dodatkową funkcjonalność - wyświetlenie informacji o naciśniętym klawiszu - **Holes** `${KeyDown}`, `${Key}` oraz `${KeyCode}`

Rozszerzamy model dodając elementy `step`, `key` oraz `keyCode`:

{% highlight fsharp %}
type Model =
    {
        // whole code you can see on GitHub
        // ...
        step: int
        key: string
        keyCode: string        
    }
{% endhighlight %}

Ustawiamy wartości początkowe `initModel`:

{% highlight fsharp %}
let initModel =
    {
        // whole code you can see on GitHub
        // ...
        step = 1
        key = ""
        keyCode = ""
    }
{% endhighlight %}

Dodajemy **Message'e**:

{% highlight fsharp %}
type Message =
    // whole code you can see on GitHub
    // ...
    | SetStep of int
    | KeyDown of string * string
{% endhighlight %}

`SetStep` - wysyłany w momencie wprowadzania nowej wartości przesunięcia licznika - wartość podawana w parametrze typu `int`

`KeyDown` - wysyłany w momencie naciśnięcia klawisza na klawiaturze - klawisz oraz jego kod podawane w parametrze typu `string * string` - **Tuple**

Aktualizujemy funkcję `update`:

{% highlight fsharp %}
let update message model =
    match message with
    | Increment -> { model with value = model.value + model.step }
    | Decrement -> { model with value = model.value - model.step }
    | Reset -> { model with value = 0; step = 1 }
    | SetStep step -> { model with step = step }
    | KeyDown (key, keyCode) -> { model with key = key; keyCode = keyCode }
{% endhighlight %}

Licznik przesuwany jest o wartość, zapamiętaną w `model.step`. `Reset` ustawia `step` na wartość początkową. `SetStep` ustawia wartość przesunięcia, a `KeyDown` informacje o naciśniętym klawiszu.

Rozszerzamy funkcję `view`:

{% highlight fsharp %}
let view model dispatch =
    // whole code you can see on GitHub
    // ...
    .Step(string model.step, fun inputStep ->
        match System.Int32.TryParse inputStep with
        | true, step -> dispatch (SetStep (step))
        | false, _ -> ())
    .Key(model.key)
    .KeyCode(model.keyCode)
    .KeyDown(fun key -> dispatch (KeyDown (key.Key, key.Code)))
    .Elt()
{% endhighlight %}

Metoda `Step` wyśle `Message` aktualizujący wartość przesunięcia tylko wtedy, kiedy wprowadzony napis jest liczbą. W tym miejscu powinna być **pełna walidacja**, ale dla uproszczenia pomijamy ten element. Metody `Key` oraz `KeyCode` wyświetlają informacje o klawiszu. Metoda `KeyDown` wysyła `Message` aktualizujący informacje o naciśniętym klawiszu. Tak jak w poprzednim przykładzie, tak i tu, wszystko jest silnie typowane i wspierane przez kompilator **F#**.

Ostatecznie otrzymujemy program:

![Picture2]({{ site.url }}/assets/fsharp_bolero/bolero_counter_templates.png)

Zobaczmy kolejny przykład.

## Enter Values

W sytuacji, kiedy pewne elementy zawierają tę samą strukturę kodu **HTML**, ale różnią się wartościami, warto stworzyć dla nich re-używalny komponent. W **Bolero** możemy to zrobić, używając **Nested Templates**. Zobaczmy, jak to działa na przykładzie programu wyświetlającego wprowadzane napisy.

Definiujemy **Template**:

{% highlight html %}
<div>
    <label>Enter value and press ENTER: <input type="text" bind-onchange="${NewValue}"/></label>
    <div><span>Entered Values:</span></div>
    ${EnteredValues}
    <div><button onclick="${ClearValues}">Clear</button></div>
</div>

<template id="ShowValue">
    <div>
        <span>${Value}</span>
    </div>
</template>
{% endhighlight %}

Zasada działania jest taka sama jak w przykładzie z licznikiem - **HTML** + **Holes**:

* `${NewValue}` - reprezentuje wprowadzaną wartość
* `${EnteredValues}` - reprezentuje listę wprowadzonych wartości
* `${ClearValues}` - reprezentuje naciśnięcie przycisku czyszczącego wprowadzone wartości

Nowością jest `<template id="ShowValue">` - definicja **Nested Template**. Template nie jest wyświetlony na ekranie, można go użyć w kodzie **F#**. **Hole** `${Value}` reprezentuje pojedynczą wyświetlaną wartość.

Druga nowość to zastąpienie atrybutu `bind` przez `bind-onchange`. Dzięki temu `Message` wysyłany jest po naciśnięciu klawisza `ENTER`, a nie po każdorazowym wprowadzeniu znaku.

Pozostaje zakodować logikę w **F#** wg standardowego schematu.

Model:

{% highlight fsharp %}
type Model =
    {
        enteredValues: string list
    }
{% endhighlight %}

Wartości trzymamy jako listę napisów.

Init:

{% highlight fsharp %}
let initModel =
    {
        enteredValues = []
    }
{% endhighlight %}

Message:

{% highlight fsharp %}
type Message =
    | NewValue of string
    | ClearValues
{% endhighlight %}

Zgłaszamy wprowadzenie nowej wartości lub wyczyszczenie wprowadzonych danych.

Template:

{% highlight fsharp %}
type EnterValuesTemplate = Template<"wwwroot/enterValues.html">
{% endhighlight %}

Ładowany z pliku `enterValues.html`

Update:

{% highlight fsharp %}
let update message model =
    match message with
    | NewValue value -> { model with enteredValues = model.enteredValues @ [value] }
    | ClearValues -> { model with enteredValues = [] }
{% endhighlight %}

Operator `@` pozwala połączyć dwie listy w jedną.

View:

{% highlight fsharp %}
let view model dispatch =
    EnterValuesTemplate()
        .EnteredValues(forEach model.enteredValues (fun value -> EnterValuesTemplate.ShowValue().Value(value).Elt()))
        .NewValue("", fun value -> dispatch (NewValue value))
        .ClearValues(fun _ -> dispatch ClearValues)
        .Elt()
{% endhighlight %}

Do **Nested Template** odwołujemy się po identyfikatorze zdefiniowanym w pliku **.html**, wg schematu - `Template.NestedTemplate()`. W przykładzie jest to `EnterValuesTemplate.ShowValue()`. 
Wartości z listy, podajemy jako parametr do metody `EnteredValues`. Każdy element wyświetlany jest jako osobny **Nested Template**.

Ostatecznie otrzymujemy program:

![Picture3]({{ site.url }}/assets/fsharp_bolero/bolero_enter_values.png)

## Hot Reloading

Wspólną cechą framework'ów webowych generujących wynik po stronie klienta, jest automatyczne odświeżanie widoku, w momencie zapisu zmienionego kodu. Nazywa się to **Hot Reloading**. **Bolero** również posiada taką możliwości. Po zapisaniu zmian w pliku ***.html**, następuje przeładowanie i wynik dostępny jest na stronie. Oczywiście, jeśli zmiany wprowadzamy w kodzie **F#**, to dalej trzeba dokonać kompilacji.

Dla mnie, jako **backend'owca**, kompilacja kodu po zmianach to standardowa, wręcz naturalna rzecz. Obecnie, gdy działam z **Bolero** nie używam mechanizmu **Hot Reloading**. Być może to się zmieni, kiedy będę pisał więcej kodu UI, zwłaszcza na etapie prototypowania interfejsów.

Generowany kod za pomocą `dotnet new bolero-app -o MyApp` zawiera konfigurację **Hot Reloading**. Szczegóły, jak włączyć tę funkcjonalność we własnym projekcie znajdują się [na stronie dokumentacji][6].

Tak wygląda programowanie w **Bolero** z użyciem **HTML Templates**. W następnym artykule zajmiemy się mechanizmem **View Components**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsbolero.io/docs/HTML "F# Bolero HTML"
[2]: https://fsbolero.io/docs/Templating "F# Bolero Templates"
[3]: https://github.com/mikedevbo/fsharp-bolero-sandbox "F# Bolero Sandbox"
[4]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[5]: https://www.jetbrains.com/rider/ "Jetbrains Rider"
[6]: https://fsbolero.io/docs/Templating#hot-reloading "F# Bolero Hot Reloading"
[7]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[8]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}
[9]: {{ site.url }}{% link _posts/2021-04-11-fsharp-bolero-routing.md %}
[10]: {{ site.url }}{% link _posts/2021-06-12-fsharp-bolero-routing-page-models.md %}