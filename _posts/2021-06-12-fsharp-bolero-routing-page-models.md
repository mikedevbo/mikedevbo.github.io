---
layout: post
title:  "F# Bolero - A cóż to takiego? - routing - page models"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób używać Page Models do specyfikowania modelu danych dla konkretnej strony."
date:   2021-06-12
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][5]
* [F# Bolero - A cóż to takiego? - HTML templates][6]
* [F# Bolero - A cóż to takiego? - view components][7]
* [F# Bolero - A cóż to takiego? - routing][8]
* [F# Bolero - A cóż to takiego? - routing - page models][9]
* [F# Bolero - A cóż to takiego? - remoting][10]

[Bolero][1] przetrzymuje stan, przez cały cykl życia aplikacji. Każda strona, symulowana za pomocą [Routingu][2], ma dostęp do aktualnego stanu, co jest bardzo fajną właściwością. Poniższe nagranie pokazuje, jak to działa. Obserwuj dane, wprowadzane na każdej stronie:

<video width="470" height="242" controls style="border-style: solid; border-width: thin;">
  <source src="{{ site.url }}/assets/fsharp_bolero/bolero_routing_nopagemodel.webm" type="video/webm">
Your browser does not support the video tag.
</video>

Korzystając z właściwości [Page Models][3], możemy zaprogramować zachowanie, w którym, przy każdym wyświetlaniu strony, stan będzie ustawiany domyślnymi wartościami. Poniższe nagranie pokazuje, jak to działa. Ponownie, obserwuj dane, wprowadzane na każdej stronie:

<video width="470" height="242" controls style="border-style: solid; border-width: thin;">
  <source src="{{ site.url }}/assets/fsharp_bolero/bolero_routing_pagemodel.webm" type="video/webm">
Your browser does not support the video tag.
</video>

Zobaczmy, w jaki sposób możemy zrealizować powyższą funkcjonalność. Cały kod znajdziesz na [GitHub'e][4].

Tworzymy plik `main.html` z poniższym kodem i umieszczamy w katalogu **wwwroot**:

{% highlight html %}
<div>Choose:</div>
<a href="${MainLink}">Main</a>&nbsp;&nbsp;
<a href="${CounterLink}">Counter</a>&nbsp;&nbsp;
<a href="${EnterValuesLink}">EnterValues</a>&nbsp;&nbsp;
<a href="${ViewComponentsLink}">ViewComponents</a>&nbsp;&nbsp;
<br />
${Body}
{% endhighlight %}

**Holes** zdefiniowane w atrybutach `href` reprezentują linki do poszczególnych stron. `$Body` reprezentuje zawartość aktualnie wyświetlanej strony.

Definiujemy **Templates** dla poszczególnych stron:

* Counter - plik **wwwroot/counter.html**

{% highlight html %}
<span>Counter</span>
<div>
    <button onclick="${Decrement}">-</button>
    <span>${Value}</span>
    <button onclick="${Increment}">+</button>
</div>
{% endhighlight %}

* EnterValues - plik **wwwroot/entervalues.html**

{% highlight html %}
<span>EnterValues</span>
<div>
    <label><input type="text" bind="${EnterValue}"/></label>
    <span>${EnterValue}</span>
</div>
{% endhighlight %}

* ViewComponents - plik **wwwroot/viewcomponents.html**

{% highlight html %}
<span>ViewComponents</span>
<div>
    <label><input type="text" bind="${ViewComponentValue}"/></label>
    <span>${ViewComponentValue}</span>
</div>
{% endhighlight %}

Definiujemy model oraz stan początkowy dla poszczególnych stron:

{% highlight fsharp %}
type CounterModel =
    {
        value: int
    }

type EnterValuesModel =
    {
        value: string
    }

type ViewComponentsModel =
    {
        value: string
    }

let defaultModel = function
    | Main -> ()
    | Counter model -> Router.definePageModel model { value = 0 }
    | EnterValues model -> Router.definePageModel model { value = "" }
    | ViewComponents model -> Router.definePageModel model { value = "" }    
{% endhighlight %}

Do zdefiniowania początkowego stanu wykorzystujemy funkcję `Router.definePageModel`

Modelujemy dostępne strony z odpowiednim **Page Model**:

{% highlight fsharp %}
type Page =
    | [<EndPoint "/">] Main
    | [<EndPoint "/counter">] Counter of PageModel<CounterModel>
    | [<EndPoint "/entervalues">] EnterValues of PageModel<EnterValuesModel>
    | [<EndPoint "/viewcomponents">] ViewComponents of PageModel<ViewComponentsModel>
{% endhighlight %}

Definiujemy główny model oraz ustawiamy stan początkowy dla całej aplikacji:

{% highlight fsharp %}
type Model =
    {
        page: Page
    }

let initModel =
    {
        page = Main
    }
{% endhighlight %}

Stan poszczególnych stron został przeniesiony do **Page Models**. Główny model aplikacji zawiera informację o dostępnych stronach.

Definiujemy **Message** dla poszczególnych akcji:

{% highlight fsharp %}
type Message =
    | SetPage of Page
    | Increment
    | Decrement
    | EnterValue of string
    | EnterViewComponentValue of string
{% endhighlight %}

Definiujemy **Routing**:

{% highlight fsharp %}
let router = Router.inferWithModel SetPage (fun m -> m.page) defaultModel
{% endhighlight %}

W zwykłym **Routingu** używamy funkcji `Router.infer`. W **Routingu** z modelem używamy funkcji `Router.inferWithModel`. Trzeci parametr funkcji przyjmuje definicję domyślnych wartości dla poszczególnych stron.

Definiujemy funkcję `update`:

{% highlight fsharp %}
let update message model =
    match message with
    | SetPage p -> { model with page = p }
    | Increment ->
        match model.page with
        | Counter pageModel -> { model with page = Counter { Model = { pageModel.Model with value = pageModel.Model.value + 1 } } }
        | _ -> model
    | Decrement ->
        match model.page with
        | Counter pageModel -> { model with page = Counter { Model = { pageModel.Model with value = pageModel.Model.value - 1 } } }
        | _ -> model
    | EnterValue value ->
        match model.page with
        | EnterValues pageModel -> { model with page = EnterValues { Model = { pageModel.Model with value = value } } }
        | _ -> model
    | EnterViewComponentValue value ->
        match model.page with
        | ViewComponents pageModel -> { model with page = ViewComponents { Model = { pageModel.Model with value = value } } }
        | _ -> model
{% endhighlight %}

W naszym przykładzie każdy **Message** może być wykonany na jednej, konkretnej stronie. W związku z tym, w pierwszej kolejności rozpoznajemy, czy jesteśmy na właściwej stronie - `match model.page with` - następnie wykonujemy logikę działania.

Parametr `pageModel` reprezentuje aktualny stan na stronie. **Routing** z **Page Models** zapewnia resetowanie stanu do wartości domyślnych, przy przechodzeniu pomiędzy stronami.

Definiujemy **View Component**:

{% highlight fsharp %}
type InputComponent() =
    inherit ElmishComponent<string, string>()

    override this.ShouldRender(oldModel, newModel) =
        oldModel <> newModel

    override this.View model dispatch =
        ViewComponentsTemplate()
            .ViewComponentValue(model, fun value -> dispatch value)
            .Elt()
{% endhighlight %}

Definiujemy funkcję **view**:

{% highlight fsharp %}
let view model dispatch =
    let main =
        MainTemplate()
            .MainLink(router.Link Main)
            .CounterLink(router.Link (Counter Router.noModel))
            .EnterValuesLink(router.Link (EnterValues Router.noModel))
            .ViewComponentsLink(router.Link (ViewComponents Router.noModel))

    let counter (pageModel: CounterModel) =
        CounterTemplate()
            .Value(string pageModel.value)
            .Increment(fun _ -> dispatch Increment)
            .Decrement(fun _ -> dispatch Decrement)

    let enterValues (pageModel: EnterValuesModel) =
        EnterValuesTemplate()
            .EnterValue(pageModel.value, fun value -> dispatch (EnterValue value))

    let viewComponents (pageModel: ViewComponentsModel) =
        ecomp<InputComponent, _, _> [] pageModel.value (fun value -> dispatch (EnterViewComponentValue value))

    match model.page with
    | Main ->
        main.Body("").Elt()
    | Counter pageModel ->
        main.Body((counter pageModel.Model).Elt()).Elt()
    | EnterValues pageModel ->
        main.Body((enterValues pageModel.Model).Elt()).Elt()
    | ViewComponents pageModel->
        main.Body((viewComponents pageModel.Model)).Elt()
{% endhighlight %}

Podobnie, jak w przypadku funkcji `update`, w pierwszej kolejności musimy rozpoznać, która strona ma być wyświetlona - `match model.page with`. Pomocnicze funkcje `counter`, `enterValues` oraz `viewComponents` renderują odpowiedni **HTML Template**. Każda funkcja przyjmuje w parametrze model, należący do danej strony. Wartości modelu wstawiane są w odpowiednie **Holes**.

Na koniec definiujemy start aplikacji:

{% highlight fsharp %}
type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program =
        Program.mkSimple (fun _ -> initModel) update view
        |> Program.withRouter router
{% endhighlight %}

To tyle, jeśli chodzi o programowanie w **Bolero** z użyciem **Page Models**.

Poznane do tej pory właściwości, pozwalają programować tzw. **Frontend**. Okazuje się, że **Bolero** posiada dodatkową cechę zwaną **Remoting**, która pozwala kodować punkt wejścia do **Backendu**. Przyjrzymy się temu w następnym artykule.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsbolero.io/ "F# Bolero"
[2]: https://fsbolero.io/docs/Routing "F# Bolero Routing"
[3]: https://fsbolero.io/docs/Routing#page-models "F# Bolero Routing Page Model"
[4]: https://github.com/mikedevbo/fsharp-bolero-sandbox "F# Bolero Sandbox"
[5]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[6]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[7]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}
[8]: {{ site.url }}{% link _posts/2021-04-11-fsharp-bolero-routing.md %}
[9]: {{ site.url }}{% link _posts/2021-06-12-fsharp-bolero-routing-page-models.md %}
[10]: {{ site.url }}{% link _posts/2021-08-18-fsharp-bolero-remoting.md %}