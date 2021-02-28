---
layout: post
title:  "F# Bolero - A cóż to takiego? - wprowadzenie"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób można napisać najprostszy program w Bolero."
date:   2021-01-23
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][11]
* [F# Bolero - A cóż to takiego? - HTML templates][13]
* [F# Bolero - A cóż to takiego? - view components][14]
* ...

Programowanie Webowe. Sposób wytwarzania oprogramowania, dzięki któremu produkt może zostać uruchomiony w przeglądarce internetowej. Zagadnienie jest bardzo obszerne. Możliwości realizacji można dzielić na różne kategorie, np. ze względu na **miejsce** generowania wyniku:

* Server Side - generowanie zawartości po stronie serwera, a następnie przesłanie do klienta - przeglądarki internetowej
* Client Side - generowanie zawartości po stronie klienta - przeglądarce internetowej

Inny podział to **sposób** generowania wyniku:

* Statyczny - pobieranie gotowej, wcześniej przygotowanej zawartości
* Dynamiczny - generowanie zawartości na żądanie, w momencie jej pobierania

Kategorie można łączyć ze sobą:

* Server Side / Statyczny - pierwsze strony internetowe
* Server Side / Dynamiczny - wyświetlanie strony internetowej w zależności od zapisanych danych
* Client Side / Statyczny - cache przeglądarki internetowej
* Client Side / Dynamiczny - początki języka **JavaScript**

Na bazie różnych kategorii powstało wiele podejść oraz technologii, które pozwalają realizować końcowe produkty.

Świat **F#** [oferuje wiele][1] z programowania webowego. Moją uwagę przykuło [Bolero][2].

![Picture1]({{ site.url }}/assets/fsharp_bolero/bolero.jpeg)

Jakbym miał zobrazować, na czym to polega to, zrobiłbym to tak:

![Picture2]({{ site.url }}/assets/fsharp_bolero/bolero_arch.svg)

**Bolero** umożliwia pisanie aplikacji webowych w języku **F#**. Kod konstruuje się w podejściu [Elmish][3] - implementacji architektury [Model-View-Update][4], zapoczątkowanej wraz z pojawieniem się języka [Elm][5]. Wynikiem kompilacji jest program, który za pomocą [Blazor][6] oraz [.NET][7] uruchamiany jest w środowisku [WebAssembly][8] - alternatywy dla środowiska **JavaScript**.

## Simple Program

Najszybszy sposób zapoznania się z możliwościami **Bolero**, to wygenerowanie przykładowego projektu, postępując wg wskazówek [ze strony][2]. Utworzony program zawiera implementację głównych feature'ów. Kod, generowany jest na podstawie [Template'u][9], przygotowanego przez [Loïc Denuzière][10] - **autora** Bolero.

**Template** zawiera parametry, którymi można konfigurować końcowy efekt. Najprostszy program można wygenerować, postępując wg kroków:

* zainstalować **Template**

{% highlight text %}
dotnet new -i Bolero.Templates
{% endhighlight %}

* utworzyć projekt

{% highlight text %}
dotnet new bolero-app -o Simple --minimal=true --server=false
{% endhighlight %}

* uruchomić program

{% highlight text %}
cd Simple
dotnet run -p src/Simple.Client
{% endhighlight %}

* otworzyć stronę wpisując w przeglądarkę adres wypisany na konsoli np.

{% highlight text %}
http://localhost:5000
{% endhighlight %}

Najważniejsze elementy wygenerowanego programu to:

* `Main.fs` - plik z głównym kodem
* `Startup.fs` - plik z konfiguracją
* `wwwroot` - katalog zawierający statyczne elementy

Główy kod programu wygląda tak:

{% highlight fsharp %}
module Simple.Client.Main

open Elmish
open Bolero
open Bolero.Html

type Model =
    {
        x: string
    }

let initModel =
    {
        x = ""
    }

type Message =
    | Ping

let update message model =
    match message with
    | Ping -> model

let view model dispatch =
    text "Hello, world!"

type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program =
        Program.mkSimple (fun _ -> initModel) update view

{% endhighlight %}

Przejdźmy przez poszczególne elementy:

#### Klasa MyApp

Komponent ładowany przy starcie. Plik `Startup.fs` zawiera konfigurację uruchomienia:

{% highlight fsharp %}
builder.RootComponents.Add<Main.MyApp>("#main")
{% endhighlight %}

Klasa `MyApp` dziedziczy po klasie `ProgramComponent`, która w parametrach generycznych przyjmuje `Model` oraz `Message`. Uruchomienie następuje w momencie wywołania funkcji `mkSimple`, która w parametrach przyjmuje opisane poniżej wartości.

#### Funkcja view

Renderuje widok, czyli to wszystko, co użytkownik widzi na ekranie. Posiada dostęp do aktualnego stanu programu w postaci parametru `model`. Drugi parametr `dispatch` pozwala wysyłać **Message'e** do programu, które umożliwiają kodowanie zachowania.

####  Funkcja update

Uruchamiana w momencie obsługi **Message'a**. Parametr `message` określa jego rodzaj. Stan aktualizuje się, zmieniając wartości w parametrze `model`.

#### Discriminated Union Message

Definiuje rodzaje **Message'y**, jakie można zgłaszać np. sygnalizacja naciśnięcia odpowiedniego przycisku na stronie.

#### Value initModel

Początkowy stan programu, inicjalizowany przez `mkSimple`.

#### Record Model

Model reprezentujący aktualny stan programu.

Powyższy przykład zawiera najprostszą logikę działania - wypisanie tekstu na ekranie. Nie wykorzystuje on wszystkich wymienionych właściwości. Działanie kończy się na funkcji `view`. Zmieńmy to implementując prosty licznik.

## Simple Counter

Definiujemy `Model` tak, aby przechowywał aktualną wartość licznika

{% highlight fsharp %}
type Model =
    {
        value: int
    }
{% endhighlight %}

Ustawiamy początkową wartość

{% highlight fsharp %}
let initModel =
    {
        value = 0
    }
{% endhighlight %}    

Definiujemy **Message'e**

* `Increment` -  przesuwa licznik do przodu
* `Decrement` - przesuwa licznik do tyłu
* `Reset` - ustawia początkową wartość

{% highlight fsharp %}
type Message =
    | Increment
    | Decrement
    | Reset
{% endhighlight %}

Aktualizujemy funkcję `update`, ustawiając odpowiednią wartość licznika w zależności od **Message'a**

{% highlight fsharp %}
let update message model =
    match message with
    | Increment -> { model with value = model.value + 1 }
    | Decrement -> {model with value = model.value - 1 }
    | Reset -> { model with value = 0 }
{% endhighlight %}

Aktualizujemy funkcję `view` tak, aby wyświetliła interfejs użytkownika zawierający:

* pokazanie aktualnej wartości licznika
* przyciski przesunięcia licznika w dwie strony
* zresetowanie licznika

Dodatkowo funkcja zgłasza odpowiedni **Message** w zależności od tego, który przycisk zostanie kliknięty.

Poniższy kod pokazuje jedną z właściwości **Bolero** - konstruowanie UI w kodzie **F#**:

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

Uruchamiamy program

{% highlight text %}
dotnet run -p src/Simple.Client
{% endhighlight %}

Sprawdzamy efekt w przeglądarce internetowej

![Picture3]({{ site.url }}/assets/fsharp_bolero/bolero_counter.png)

Zgodnie z implementacją przycisk `+` zwiększa wartość licznika, a przycisk `-` zmniejsza. Przycisk `Reset` ustawia początkową wartość.

Istnieje jeszcze jeden, bardziej znany, sposób konstruowania UI - za pomocą standardowych znaczników **HTML**. Strona napisana w **HTML** reprezentowana jest w kodzie **Bolero** w postaci tzw. **Templates**. 

## Początek Serii

Spędziłem wiele miesięcy w poszukiwaniu sposobu programowania webowego, który najbardziej by mi odpowiadał. Taki sposób znalazłem w świecie **F#**. Obecnie zapoznaję się z różnymi możliwościami **Bolero**. Wyniki będę umieszczał na [GitHub'e][12], a poszczególne elementy opisywał w serii, której pierwszy artykuł właśnie czytasz. Zachęcam Cię do własnych eksperymentów. Może taki sposób pisania kodu będzie odpowiadał również Tobie.

W następnym artykule zobaczymy, w jaki sposób używać wyżej wymienionych **Templates**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsharp.org/guides/web/ "F# web"
[2]: https://fsbolero.io/ "F# Bolero"
[3]: https://elmish.github.io/elmish/ "Elmish"
[4]: https://guide.elm-lang.org/architecture/ "Elm architecture"
[5]: https://elm-lang.org/ "Elm lang"
[6]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor "Blazor"
[7]: https://dotnet.microsoft.com/ "dotnet"
[8]: https://webassembly.org/ "WebAssembly"
[9]: https://github.com/fsbolero/Template "F# Bolero template"
[10]: https://github.com/Tarmil "Tarmil"
[11]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[12]: https://github.com/mikedevbo/fsharp-bolero-sandbox "fsharp-bolero-sandbox"
[13]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[14]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}