---
layout: post
title:  "F# Bolero - A cóż to takiego? - remoting"
description: "Wstęp do F# Bolero. W tym artykule przeczytasz, w jaki sposób zaimplementować Remote Service."
date:   2021-08-18
---

Posty z tej serii:

* [F# Bolero - A cóż to takiego? - wprowadzenie][8]
* [F# Bolero - A cóż to takiego? - HTML templates][9]
* [F# Bolero - A cóż to takiego? - view components][10]
* [F# Bolero - A cóż to takiego? - routing][11]
* [F# Bolero - A cóż to takiego? - routing - page models][12]
* [F# Bolero - A cóż to takiego? - routing - remoting][13]

Aplikacja webowa składa się (w dużym uproszczeniu) z dwóch części:

* Widoków
* Danych, prezentowanych na widokach

Dane dla widoków możemy dostarczyć co najmniej na trzy sposoby:

* Wołamy zewnętrzne **API** bezpośrednio z aplikacji
* Tworzymy własne **API**, które dla aplikacji będzie punktem wejścia do wołania innych **API**
* Mix dwóch powyższych

Tworząc własne **API** dla aplikacji pisanej w [Bolero][1], możemy skorzystać z mechanizmu [Remoting][2], który w mojej subiektywnej opinii posiada trzy bardzo fajne właściwości:

* Szczegóły komunikacji między klientem a serwerem ukryte są pod warstwą abstrakcji
* Ten sam sposób używania, niezależnie od tego, czy pobieramy, czy wysyłamy dane
* Dwa wynikowe stany - udało się lub nie

W szczegółach, dane serilizowane są do formatu **JSON** oraz wymieniane przez protokół **HTTP(s)** z użyciem metody **POST**.

Zobaczmy, w jaki sposób możemy zaimplementować najprostsze **API** z jedną metodą, która zwraca tekst, a także, w jaki sposób możemy wywołać zaimplementowaną metodę.

[Bolero Template][5] umożliwia wygenerowanie kodu, gdzie serwer oraz klient skonfigurowani są w modelu [hosted][4]. My zrealizujemy scenariusz, w którym klient oraz serwer hostowani są niezależnie od siebie. 

Cały kod znajdziesz na [GitHub'e][3].

Zacznijmy od utworzenia trzech projektów:

* kontrakty pomiędzy klientem, a serwerem - dll

    `dotnet new classlib -n Remoting.Contracts -lang f#`

* host serwera - ASP.NET Web API

    `dotnet new webapi -n Remoting.Server -lang f#`

* host klienta - **Bolero** minimal

    `dotnet new bolero-app -n Remoting.Client --minimal=true --server=false`

Następnie, w projekcie **Remoting.Contracts**:

* dodajemy paczkę **Bolero**

`dotnet add package Bolero`

* usuwamy plik `Library.fs`

* dodajemy plik `SimpleService.fs` z deklaracją **API**

{% highlight fsharp %}
module Remoting.Contracts

open Bolero.Remoting

type SimpleService =
    {
        GetHello: unit -> Async<string>
    }

    interface IRemoteService with
        member this.BasePath = "/simpleservice"
{% endhighlight %}

**Record** `SimpleService` zawiera pole `GetHello`, które reprezentuje metodę **API**. Wynikiem wywołania `GetHello` będzie zwrócenie przykładowego napisu. `SimpleService` dziedziczy po interfejsie `IRemoteService` z przestrzeni nazw `Bolero.Remoting`. Member `this.BasePath` wskazuje ścieżkę **API**. W ten sposób tworzymy deklarację, którą rozpozna **Bolero**. 

Do projektu **Remoting.Server**:

* dodajemy paczkę **Bolero.Server**

`dotnet add package Bolero.Server`

* dodajemy referencję do projektu **Remoting.Contracts**:

`dotnet add reference ../Remoting.Contracts`

* dodajemy plik `SimpleServiceHandler.fs` z implementacją **API**

{% highlight fsharp %}
namespace Remoting.Server

open Bolero.Remoting.Server
open Microsoft.AspNetCore.Hosting
open Remoting.Contracts
open Microsoft.Extensions.Primitives

type SimpleServiceHandler(ctx: IRemoteContext, env: IWebHostEnvironment) =
    inherit RemoteHandler<SimpleService>()

    override this.Handler =
        {
            GetHello = fun _ -> async {
                ctx.HttpContext.Response.Headers.Add("Access-Control-Allow-Origin", StringValues("*"))
                return "Hello from Simple Service!"
            }
        }
{% endhighlight %}

Klasa `SimpleServiceHandler` dziedziczy po klasie `RemoteHandler` z przestrzeni nazw `Bolero.Remoting.Server`. Pod `Handler` podstawiamy definicję metody **API**. **Bolero** umożliwia skorzystanie z dwóch zależności, dla których dostarcza implementacje: `IRemoteContext` oraz `IWebHostEnvironment`. W przykładzie pierwszą z nich wykorzystujemy, aby do odpowiedzi **HTTP(s)** dodać nagłówek `Access-Control-Allow-Origin`.

* w pliku `Startup.fs` konfigurujemy **ASP.NET Web API** włączając **Bolero Remoting** oraz definiując **CORS**

{% highlight fsharp %}
type Startup(configuration: IConfiguration) =
    member _.Configuration = configuration

    // This method gets called by the runtime. Use this method to add services to the container.
    member _.ConfigureServices(services: IServiceCollection) =
        // Add framework services.
        services.AddControllers() |> ignore
        services.AddRemoting<SimpleServiceHandler>()
                .AddCors() |> ignore

    // This method gets called by the runtime. Use this method to configure the HTTP request pipeline.
    member _.Configure(app: IApplicationBuilder, env: IWebHostEnvironment) =
        if (env.IsDevelopment()) then
            app.UseDeveloperExceptionPage() |> ignore
        app.UseRemoting()
           .UseHttpsRedirection()
           .UseRouting()
           .UseAuthorization()
           .UseEndpoints(fun endpoints ->
                endpoints.MapControllers() |> ignore
            )
           .UseCors(fun policy ->
                policy.AllowAnyOrigin()
                      .AllowAnyMethod()
                      .AllowAnyHeader() |> ignore
             ) |> ignore
{% endhighlight %}

**Remoting** włączamy poprzez wywołanie metod:

* services.AddControllers()
* app.UseRemoting()

**CORS** definiujemy poprzez wywołanie metod:

* services..AddCors()
* app.UseCors(...)

W konfiguracji [hosted][4] **CORS** nie jest potrzebny, ponieważ zarówno serwer, jak i klient uruchamiane są z tej samej domeny.

Na koniec, w projekcie **Remoting.Client**:

* dodajemy referencję do projektu **Remoting.Contracts**:

`dotnet add reference ../../../Remoting.Contracts`

* w pliku `Startup.fs`, włączamy **Bolero Remoting** , podając bazowe **URI** pod którym dostępne jest **API**

{% highlight fsharp %}
builder.Services.AddRemoting((fun httpClient ->
    httpClient.BaseAddress <- Uri("http://localhost:1234"))) |> ignore
{% endhighlight %}

* w pliku `main.fs`, realizujemy aplikację, które zawoła **API** za pomocą [Elmish Commands][6]:

Definiujemy model

{% highlight fsharp %}
type Model =
    {
        Text: string
    }
{% endhighlight %}

Ustawiamy początkowe wartości modelu

{% highlight fsharp %}
let initModel =
    {
        Text = ""
    }, Cmd.none
{% endhighlight %}

Nowością w stosunku do przykładów z poprzednich artykułów tej serii jest to, że wraz z wartościami modelu podajemy jaki **Command** ma się wykonać. W momencie inicjalizowania początkowych wartości nie wykonujemy żadnej akcji, oznaczając to przez `Cmd.none`

Definiujemy Messages

{% highlight fsharp %}
type Message =
    | GetHello
    | ShowHello of string
    | ShowError of exn
{% endhighlight %}

* **GetHello** - zawołanie **API**
* **ShowHello** - wykonanie działania, w przypadku, gdy **API** zwróci sukces
* **ShowError** - wykonanie działania, w przypadku, gdy  **API** zwróci błąd

Definiujemy funkcję update

{% highlight fsharp %}
let update simpleService message model =
    match message with
    | GetHello ->
        model, Cmd.OfAsync.either simpleService.GetHello () ShowHello ShowError
    | ShowHello text ->
        {model with Text = text}, Cmd.none
    | ShowError exn ->
        {model with Text = exn.Message}, Cmd.none
{% endhighlight %}

Oprócz standardowych parametrów `message` oraz `model`, funkcja przyjmuje parametr `simpleService`. Parametr jest typem `SimpleService` z projektu `Remoting.Contracts`. Po nadejściu Message'a `GetHello` wołamy **API**, za pomocą **Command'a** `Cmd.OfAsync.either`. W przypadku prawidłowego zakończenia wywołania zostanie zgłoszony Message `ShowHello` wpp. zostanie zgłoszony Message `ShowError`. Przy sukcesie wypełniamy model zwróconym rezultatem wpp. opisem błędu.

Definiujemy funkcję view

{% highlight fsharp %}
let view model dispatch =
    concat [
        div [] [
            button [on.click (fun _ -> dispatch GetHello)] [text "GetHello"]
        ]

        div [] [
            text model.Text
        ]
    ]
{% endhighlight %}

Widok zawiera dwa elementy:

* przycisk z akcją `on.Click`, która wysyła Message `GetHello`
* pole `text`, które wyświetla zawartość modelu

Na koniec zmieniamy inicjalizowanie głównego komponentu z `mkSimple` na `mkProgram`

{% highlight fsharp %}
type MyApp() =
    inherit ProgramComponent<Model, Message>()

    override this.Program =
        let simpleService = this.Remote<SimpleService>()
        Program.mkProgram (fun _ -> initModel) (update simpleService) view
{% endhighlight %}

W ten sposób wstrzykujemy klienta **API** do funkcji update oraz dostajemy możliwość używania [Elmish Commands][6].

## Koniec Serii

Tak, oto dotarliśmy do końca serii, która pokazuje podstawowe możliwości [Bolero][1]. Dzięki przedstawionym mechanizmom możemy tworzyć rozwiązania webowe w podejściu **SPA**, pisząc kod w języku **F#**.

**Bolero** posiada [o wiele więcej możliwości][7]. Jest to jednak temat na osobną serię artykułów.

Jeśli chodzi o mnie, to zostaję przy frameworku. Pamiętam, jak pierwszy raz przeczytałem o jego koncepcjach i doznałem tego fajnego efektu *"aha...ciekawe*...". Teraz kiedy znam **Bolero** trochę lepiej, nic się nie zmieniło :) 

Kolejnym etapem rozwoju jest wymyślenie tematu na aplikację oraz jej realizacja. Efektami prac podzielę się na blogu, którego w dalszej perspektywie będę chciał przepisać właśnie w **Bolero**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://fsbolero.io/ "F# Bolero"
[2]: https://fsbolero.io/docs/Remoting "F# Bolero Remoting"
[3]: https://github.com/mikedevbo/fsharp-bolero-sandbox "F# Bolero Sandbox"
[4]: https://docs.microsoft.com/en-us/aspnet/core/blazor/hosting-models?view=aspnetcore-5.0 "Blazor hosting"
[5]: https://github.com/fsbolero/Template "F# Bolero Template"
[6]: https://elmish.github.io/elmish/ "Elmish"
[7]: https://fsbolero.io/docs/ "F# Bolero Documentation"
[8]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[9]: {{ site.url }}{% link _posts/2021-02-07-fsharp-bolero-html-templates.md %}
[10]: {{ site.url }}{% link _posts/2021-02-28-fsharp-bolero-view-components.md %}
[11]: {{ site.url }}{% link _posts/2021-04-11-fsharp-bolero-routing.md %}
[12]: {{ site.url }}{% link _posts/2021-06-12-fsharp-bolero-routing-page-models.md %}
[13]: {{ site.url }}{% link _posts/2021-08-18-fsharp-bolero-remoting.md %}