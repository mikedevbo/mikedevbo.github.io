---
layout: post
title:  "F# Bolero - A cóż to takiego? - routing"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób używać mechanizmu o nazwie Routing do symulowania strony, której adres widać w pasku przeglądarki internetowej."
date:   2021-04-11
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][3]
* [F# Bolero - A cóż to takiego? - HTML templates][4]
* [F# Bolero - A cóż to takiego? - view components][5]
* [F# Bolero - A cóż to takiego? - routing][6]
* ...

Wspólną cechą omawianych do tej pory przykładów jest to, że dane wyświetlane są na jednej stronie. W rzeczywistości systemy webowe zawierają wiele stron. Adres aktualnie przeglądanej strony pokazywany jest w pasku przeglądarki internetowej. W **Bolero** taki efekt można uzyskać, wykorzystując mechanizm o nazwie [Routing][1].

**Bolero** nie definiuje pojęcia strony. Mamy do dyspozycji widok, który sami musimy zaprojektować pod wymagane potrzeby. Weźmy dla przykładu aplikację, która zawiera trzy przyciski:

* Counter
* EnterValues
* ViewComponents

Po kliknięciu odpowiedniego przycisku, widok powinien pokazać jedną z funkcjonalności omawianych w poprzednich artykułach. Taki efekt możemy uzyskać, modelując kliknięty przycisk, a następnie na podstawie wartości modelu, wyświetlić prawidłowy HTML template. W ostateczności widok będzie się zmieniał, ale adres w pasku przeglądarki pozostanie ten sam. Dla uproszczenia poniższy przykład wyświetla prosty napis załadowanej funkcjonalności:

![Picture1]({{ site.url }}/assets/fsharp_bolero/bolero_routing_site_address.png)
<br />
![Picture2]({{ site.url }}/assets/fsharp_bolero/bolero_routing_counter.png)
<br />
<br />
![Picture3]({{ site.url }}/assets/fsharp_bolero/bolero_routing_site_address.png)
<br />
![Picture4]({{ site.url }}/assets/fsharp_bolero/bolero_routing_entervalues.png)
<br />
<br />
![Picture5]({{ site.url }}/assets/fsharp_bolero/bolero_routing_site_address.png)
<br />
![Picture6]({{ site.url }}/assets/fsharp_bolero/bolero_routing_viewcomponents.png)

Jeśli chcemy, aby każdy z wyświetlanych widoków posiadał swój unikalny adres, wyświetlany w pasku przeglądarki, musimy skorzystać z mechanizmu **Routing'u**.

Zaprojektujmy oraz zaprogramujmy taki efekt. Zamienimy jedną rzecz w stosunku do powyższego przykładu - zamiast wyświetlać przyciski, wyświetlimy linki. Dzięki temu zobaczymy, w jaki sposób możemy skorzystać z jednej z właściwości **Routing'u** pozwalającej na pobranie adresu URL dla wybranej strony.

Kod opisanych przykładów znajdziesz na [GitHub'e][2].

Końcowy wynik będzie wyglądał tak:

![Picture7]({{ site.url }}/assets/fsharp_bolero/bolero_routing_site_address.png)
<br />
![Picture8]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_main.png)
<br />
<br />
![Picture9]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_counter_address.png)
<br />
![Picture10]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_counter.png)
<br />
<br />
![Picture11]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_entervalues_address.png)
<br />
![Picture12]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_entervalues.png)
<br />
<br />
![Picture13]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_viewcomponents_address.png)
<br />
![Picture14]({{ site.url }}/assets/fsharp_bolero/bolero_routing_link_viewcomponents.png)
<br />
<br />

W pierwszym kroku tworzymy plik `main.html` z poniższym kodem i umieszczamy w katalogu **wwwroot**:

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

Definiujemy proste **Templates** dla poszczególnych stron:

* Counter - plik **wwwroot/counter.html**

{% highlight html %}
<span>Counter</span>
{% endhighlight %}

* EnterValues - plik **wwwroot/entervalues.html**

{% highlight html %}
<span>EnterValues</span>
{% endhighlight %}

* ViewComponents - plik **wwwroot/viewcomponents.html**

{% highlight html %}
<span>ViewComponents</span>
{% endhighlight %}

Modelujemy dostępne strony oraz ustawiamy stronę startową:

{% highlight fsharp %}
type Page =
    | [<EndPoint "/">] Main
    | [<EndPoint "/counter">] Counter
    | [<EndPoint "/entervalues">] EnterValues
    | [<EndPoint "/viewcomponents">] ViewComponents

type Model =
    {
        page: Page
    }

let initModel =
    {
        page = Main
    }
{% endhighlight %}

**Union Type** `Page` reprezentuje strony. Atrybut `EntryPoint` pozwala konstruować wzorce dla ścieżki `URL`. Przykładowo definicja `[<EndPoint "/counter/{id}">]` wymagałby podania w URL `id` licznika. W naszym przykładzie nie są wymagane żadne parametry. W takim przypadku można pominąć atrybut `EntryPoint`, ale zostawiamy go dla lepszej czytelności.

Główną stroną jest strona, która nie wyświetla nic (patrz screen powyżej).

Definiujemy **message**, który będzie ustawiał stronę:

{% highlight fsharp %}
type Message =
    | SetPage of Page
{% endhighlight %}

Definiujemy logikę funkcji `update`, która zaktualizuje stan o wybraną stronę:

{% highlight fsharp %}
let update message model =
    match message with
    | SetPage p -> { model with page = p }
{% endhighlight %}

Definiujemy **Type Templates** dla stron:

{% highlight fsharp %}
type MainTemplate = Template<"wwwroot/main.html">
type CounterTemplate = Template<"wwwroot/counter.html">
type EnterValuesTemplate = Template<"wwwroot/entervalues.html">
type ViewComponentsTemplate = Template<"wwwroot/viewcomponents.html">
{% endhighlight %}

Definiujemy funkcję `view`, która wyświetli widok w zależności od stanu modelu:

{% highlight fsharp %}
let view model dispatch =
    let main =
        MainTemplate()
            .MainLink(router.Link Main)
            .CounterLink(router.Link Counter)
            .EnterValuesLink(router.Link EnterValues)
            .ViewComponentsLink(router.Link ViewComponents)

    match model.page with
    | Main ->
        main.Body("").Elt()
    | Counter ->
        main.Body(CounterTemplate().Elt()).Elt()
    | EnterValues ->
        main.Body(EnterValuesTemplate().Elt()).Elt()
    | ViewComponents ->
        main.Body(ViewComponentsTemplate().Elt()).Elt()
{% endhighlight %}

Pomocnicza funkcja `main` zwraca reprezentację strony głównej. W miejsce `Body` wstawiamy zawartość, w zależności od tego, jaki link został kliknięty lub jaki adres został podany w przeglądarce.

W przykładzie widzimy, że link do strony generowany jest przez member `router.Link` przyjmujący w parametrze wartość modelu `Page`. Definicja value `router` wygląda tak:

{% highlight fsharp %}
let router = Router.infer SetPage (fun m -> m.page)
{% endhighlight %}

Funkcja `infer` z modułu `Router`, należącego do **Bolero**, zwraca konfigurację **Routing'u** bazując na message, który ustawia stronę oraz na modelu reprezentującym stronę.

To samo value wykorzystujemy, aby poinstruować całą aplikację o zdefiniowanym **Routing'u**, wołając `Program.withRouter router`:

{% highlight fsharp %}
type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program =
        Program.mkSimple (fun _ -> initModel) update view
        |> Program.withRouter router
{% endhighlight %}

W taki sposób można wykorzystać **Bolero Routing** do symulowania funkcjonalności przechodzenia pomiędzy stronami systemu webowego.

Mechanizm **Routing'u** zawiera właściwość **Page Models**, która pozwala zamodelować stan dla konkretnej strony. Zajmiemy się tym tematem w następnym artykule.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsbolero.io/docs/Routing "F# Bolero Routing"
[2]: https://github.com/mikedevbo/fsharp-bolero-sandbox "F# Bolero Sandbox"
[3]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[4]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[5]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}
[6]: {{ site.url }}{% link _posts/2021-04-11-fsharp-bolero-routing.md %}begin-**Anonymous**-Great post. Thank you :)-2021-05-06 00:20 UTC 
