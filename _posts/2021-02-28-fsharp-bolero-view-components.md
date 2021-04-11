---
layout: post
title:  "F# Bolero - A cóż to takiego? - view components"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób używać View Components do optymalizacji renderowania widoku."
date:   2021-02-28
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][5]
* [F# Bolero - A cóż to takiego? - HTML templates][4]
* [F# Bolero - A cóż to takiego? - view components][6]
* [F# Bolero - A cóż to takiego? - routing][7]
* ...

W procesie rozwijania oprogramowania czasami trzeba zatrzymać dorabianie nowych funkcjonalności, ponieważ istotne stają się kwestie wydajnościowe. System działał w akceptowalny sposób i nagle zaczął działać wolniej. W takim przypadku poszukiwane są miejsca, które sprawiają problemy. Zdarza się, że rozwiązanie dostarcza technologia, której używamy.

[Bolero][1] posiada mechanizm [View Components][2] pozwalający optymalizować renderowanie widoku. Przy każdej aktualizacji **Bolero** porównuje cały widok i odświeża zmienione elementy. Porównywane są również te elementy, które nie uległy zmianie. Zajmuje to pewną jednostkę obliczeniową. Mechanizm **View Components** pozwala definiować, kiedy poszczególne elementy mają być porównane i odświeżone. W ten sposób skracany jest czas renderowania całego widoku.

Zobaczmy, jak to działa na przykładzie. Tym razem zacznijmy od efektu końcowego:

![Picture1]({{ site.url }}/assets/fsharp_bolero/bolero_viewcomponents.png)

Pierwszy trzy elementy **Nickname**, **Website** oraz **Sport** renderowane są zawsze, niezależnie od tego, w którym polu tekstowym wprowadzono zmianę. Aktualizacja nastąpiła w tej samej sekundzie i prawie w tej same milisekundzie- element **Last Update**.

Kolejne trzy elementy **Nickname**, **Website** oraz **Sport** zrealizowane są z użyciem **View Components**. Każdy z elementów renderuje się tylko wtedy, kiedy wprowadzono zmianę w przypisanym do niego polu tekstowym. Aktualizacje nastąpiły w różnych sekundach i milisekundach - element **Last Update**.

Zobaczymy, w jaki sposób można zaprogramować **View Components**. Cały kod znajdziesz na [GitHub'e][3].

Tworzymy plik `vieComponents.html` z poniższym kodem i umieszczamy w katalogu wwwroot:

{% highlight html %}
<h1>No Components</h1>
${NickNameInput}
${WebsiteInput}
${SportInput}
<br />
<h1>View Components</h1>
${NickNameInputComp}
${WebsiteInputComp}
${SportInputComp}

<template id="InputValue">
    <div>
        <label>${Label}: <input type="text" bind="${Value}"/></label>
        <label>Last Update: ${ValueRenderDate}</label>
    </div>
</template>
{% endhighlight %}

Wykorzystujemy poznane [w poprzedniej część][4] **HTML Templates**.

Definiujemy konwencję nazewniczą wg, której wszystkie elementy z końcówką `Comp` reprezentują części związane z **View Components**.

W pliku `Main.fs` programujemy logikę działania:

Model:

{% highlight fsharp %}
type Model =
    {
        Nickname: string
        Website: string
        Sport: string
        NicknameComp: string
        WebsiteComp: string
        SportComp: string
    }
{% endhighlight %}

Wszystkie wartości typu `string`.

Init Model:

{% highlight fsharp %}
let initModel =
    {
        Nickname = ""
        Website = ""
        Sport = ""
        NicknameComp = ""
        WebsiteComp = ""
        SportComp = ""
    }
{% endhighlight %}

Na początku nic nie jest wprowadzone/wyświetlane.

Message:

{% highlight fsharp %}
type Message =
    | SetNickname of string
    | SetWebsite of string
    | SetSport of string
    | SetNicknameComp of string
    | SetWebsiteComp of string
    | SetSportComp of string
{% endhighlight %}

Każdy element aktualizowany jest osobno.

Update:

{% highlight fsharp %}
let update message model =
    match message with
    | SetNickname value -> {model with Nickname = value}
    | SetWebsite value -> {model with Website = value}
    | SetSport value -> {model with Sport = value}
    | SetNicknameComp value -> {model with NicknameComp = value}
    | SetWebsiteComp value -> {model with WebsiteComp = value}
    | SetSportComp value -> {model with SportComp = value}
{% endhighlight %}

Aktualizacja poszczególnych elementów modelu.

Template + elementy pomocnicze:

{% highlight fsharp %}
type ViewComponents = Template<"wwwroot/viewComponents.html">

let NicknameLabel = "Nickname"
let WebsiteLabel = "Website"
let SportLabel = "Sport"

let viewInput (label:string) value setValue =
    ViewComponents.InputValue()
        .Label(label)
        .Value(value, fun vp -> setValue vp)
        .ValueRenderDate(DateTime.Now.ToString("ss.ffff"))
        .Elt()
{% endhighlight %}

`ViewComponents` reprezentuje **Template**. Definiujemy pomocne **labels** oraz funkcję `viewInput`, która renderuje poszczególne elementy widoku. Konkretne wartości przekazywane są w parametrach funkcji. Pokazywany jest również punkt w czasie (w sekundach i milisekundach), w którym nastąpiło renderowanie widoku.

To jest cała istota programowania funkcyjnego - dziel elementy na małe funkcje i składaj je w jedną działającą całość :)

Definicja **View Components**:

{% highlight fsharp %}
type InputComponentModel = { label: string; value: string }

type InputComponent() =
    inherit ElmishComponent<InputComponentModel, string>()

    override this.ShouldRender(oldModel, newModel) =
        oldModel.value <> newModel.value

    override this.View model dispatch =
        viewInput model.label model.value dispatch
{% endhighlight %}

**View Component** zazwyczaj przyjmuje tylko część z całego modelu. W naszym przypadku model reprezentowany jest przez `InputComponentModel`, który zawiera **label** oraz **value**.

**View Component** tworzymy przez dziedziczenie po klasie `ElmishComponent` podając w parametrach generycznych **model type** oraz **message type**. W naszym przypadku są to `InputComponentModel` oraz `string`. Sam komponent nazywamy `InputComponent`.

W metodzie `ShouldRender` określamy, kiedy komponent powinien odświeżyć swój widok. W naszym przypadku jest to moment, kiedy następuje zmiana wartości `value`.

Metoda `View` odświeża widok. W naszym przypadku wykorzystujemy wcześniej utworzoną funkcję `viewInput`.

Renderowanie całego widoku:

{% highlight fsharp %}
let view model dispatch =
    ViewComponents()
        .NickNameInput(viewInput NicknameLabel model.Nickname (fun value -> dispatch (SetNickname value)))
        .WebsiteInput(viewInput WebsiteLabel model.Website (fun value -> dispatch (SetWebsite value)))
        .SportInput(viewInput SportLabel model.Sport (fun value -> dispatch (SetSport value)))
        .NickNameInputComp(ecomp<InputComponent, _, _> [] {label = NicknameLabel; value = model.NicknameComp} (fun value -> dispatch (SetNicknameComp value)))
        .WebsiteInputComp(ecomp<InputComponent, _, _> [] {label = WebsiteLabel; value = model.WebsiteComp} (fun value -> dispatch (SetWebsiteComp value)))
        .SportInputComp(ecomp<InputComponent, _, _> [] {label = SportLabel; value = model.SportComp} (fun value -> dispatch (SetSportComp value)))
        .Elt()
{% endhighlight %}

Instancję `InputComponent` tworzymy, wykorzystując, wbudowaną w **Bolero**, funkcję `ecomp`. W parametrach generycznych podajemy typ komponentu, a w parametrach funkcji model oraz funkcję `dispatch`, która zostanie wywołana w momencie wykonania akcji na komponencie.

W ten sposób możemy decydować, czy dany element widoku powinien być brany pod uwagę w momencie renderowania całości.

To tyle, jeśli chodzi o programowanie w **Bolero** z użyciem **View Components**. W następnym artykule sprawdzimy, jak działa mechanizm o nazwie **Routing**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsbolero.io "F# Bolero"
[2]: https://fsbolero.io/docs/Elmish#view-components "View Components"
[3]: https://github.com/mikedevbo/fsharp-bolero-sandbox "F# Bolero Sandbox"
[4]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[5]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[6]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}
[7]: {{ site.url }}{% link _posts/2021-04-11-fsharp-bolero-routing.md %}