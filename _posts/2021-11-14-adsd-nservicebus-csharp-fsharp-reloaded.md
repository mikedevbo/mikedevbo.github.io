---
layout: post
title:  "ADSD - NServiceBus - C# - F# - Reloaded"
description: No i stało się! System komentarzy, na którym ćwiczę różne koncepcje związane z wytwarzaniem oprogramowania, doczekał się nowej wersji. Do tej pory bazował na wielkiej czwórce. Najnowsza wersja zrealizowana jest z wykorzystaniem trójki z nich.
date:   2021-11-14
---

No i stało się! **System komentarzy**, na którym ćwiczę [różne koncepcje][1] związane z wytwarzaniem oprogramowania, doczekał się [nowej wersji][2]. Do tej pory bazował na [wielkiej czwórce][3]. Najnowsza wersja zrealizowana jest z wykorzystaniem trójki z nich:

* [ADSD][4]
* [NServiceBus][5]
* [F#][6]

Dlaczego tak? Cała historia zaczyna się [w tym miejscu][7] i przewija się przez wszystkie późniejsze artykuły, aczkolwiek początki sięgają [wpisu nr jeden][8].

## Host

Proces realizacji całego przedsięwzięcia składał się z trzech części:

* **ADSD** - utrzymanie [dotychczasowej][9] architektury rozwiązania
* **F#** - zamiana kodu **C#** na **F# 5**, który [upraszcza][10] implementację **Sagi**
* **NServiceBus** - wykorzystanie nowości wprowadzonych w [wersji 7.4][11]

Zacząłem od podniesienia **F#** do wersji 5, **NServiceBus'a** do wersji 7.5.0 oraz pozostałych paczek Nuget do ich najnowszych wersji. Wszystko ładnie kompilowało się i działało.

Kolejnym krokiem było przerobienie projektów typu **Host**. Najpierw dla **Web**, a potem dla **NServiceBus Endpoint**. Po zmianie plików z ***.csproj*** na ***.fsproj*** projekty przestały się kompilować. Zamiana **C#** na **F#** była pierwszym testem w stylu:

*"Czy faktycznie znam już F# na tyle dobrze, żeby móc zrealizować tego typu zadanie?"*

Proces przepisywania kodu dostarczył mi niezłego fun'u :) Do realizacji używałem dwóch narzędzi:

* edytora tekstu bez żadnych dodatków - [Visual Studio Code][12]
* [kompilatora][13] języka **F#**

Po paru przebiegach coraz lepiej rozumiałem mowę kompilatora. Szybszy efekt:

*"...aha...no tak..."*

Przykłady wprowadzanych zmian:

* inne słowa kluczowe opisujące konstrukcje językowe np. klasy
* ważna kolejność ułożenia plików w projekcie - z góry na dół - elementy re-używalne muszą być "wyżej" niż elementy, które z nich korzystają
* zasięg określany przez wcięcia, a nie przez nawiasy {}
* średniki nie są potrzebne
* ...

## Policy

W tym momencie przyszła kolej na zmianę kluczowych elementów systemu - **Policies**, w większości zaimplementowanych, jako **NServiceBus Saga**. Realizując tę część, doświadczyłem tego, co naprawdę kryje się pod stwierdzeniem:

*"To zależy"*

W poprzedniej wersji systemu potrzebne były dwa projekty. Jeden na kod **NServiceBus'a** napisany w **C#**, drugi na kod **Logiki** napisanej w **F#**. Logicznie **Saga** oraz **Logika** stanowiły jedność. Fizycznie, były rozproszone po różnych projektach. Wszystko zgodnie z zasadami **ADSD**. W momencie, kiedy **Saga** przechodziła na **F#**, można było przenieść kod logiki w to samo miejsce, gdzie kod Sagi i odseparować te dwa elementy modułem. Dzięki temu zniknęła konieczność używania interoperability więc interfejs oraz klasa realizująca logikę przestały być potrzebne. Logiką stały się zwykłe **pure functions**, których nie trzeba nigdzie wstrzykiwać. Można ich używać tam, gdzie są potrzebne. Struktura kodu uprościła się, a zasady **ADSD** zostały zachowane.

W tym kontekście, odpowiedź na pytanie *"Czy potrzebny jest osobny projekt na logikę?"*, brzmiała *"To zależy"*. Od czego? Od tego, czy mieszany jest kod **C#** z kodem **F#**. Jeśli nie, to osobny projekt jest zbędny.

### Saga Mapping

W trakcie zmiany **C#** na **F#**, skorzystałem z jednej z nowości wprowadzonych w **NServiceBus 7.4** - [sposobie mapowania Sagi z Message'ami][14].

Pierwotna metoda mapowania wygląda tak (przykład w **C#**):

{% highlight csharp %}
protected override void ConfigureHowToFindSaga(SagaPropertyMapper<PolicyData> mapper)
{
    mapper.ConfigureMapping<RegisterComment>(message => message.CommentId)
          .ToSaga(data => data.CommentId);

    mapper.ConfigureMapping<ResponseCreateGitHubPullRequest>(message => message.CommentId)
          .ToSaga(data => data.CommentId);
}
{% endhighlight %}

Najpierw podaje się Message do mapowania - `ConfigureMapping`, a potem property Sagi - `ToSaga`.

Nowa metoda pozwala to zrobić tak (przykład w **F#**):

{% highlight fsharp %}
override this.ConfigureHowToFindSaga(mapper: SagaPropertyMapper<PolicyData>) =
    mapper.MapSaga(fun saga -> saga.CommentId :> obj)
          .ToMessage<RegisterComment>(fun message -> message.CommentId :> obj)
          .ToMessage<ResponseCreateGitHubPullRequest>(fun message -> message.CommentId :> obj) |> ignore
{% endhighlight %}

Najpierw podaje się property Sagi - `MapSaga`, a potem wymienia się Message'e do zmapowania - `ToMessage`.

Niewielka z pozoru zmiana, mocno upraszcza analizę kodu.

### SQL Persistence

W całym procesie zmiany najbardziej obawiałem się wystąpienia niestandardowej sytuacji, która zablokuje zrealizowanie całego przedsięwzięcia. Taka sytuacja zdarzyła się, gdy testowałem zmiany z użyciem [SQL Persistence][15]. W trakcie kompilacji pojawiał się komunikat:

`error : SqlPersistenceTask: Unable to determine Saga correlation property because an unexpected method call was detected in the ConfigureHowToFindSaga method.`

SQL Persistence próbuje wywnioskować, które property z **Saga Data** pełni rolę correlation. 

Ta sama struktura kodu działała w **C#**, ale nie w **F#**. Przeglądając [source code][16] oraz dokumentację okazało się, że wnioskowanie polega na przeglądaniu kodu **IL**, wygenerowanego z metody `ConfigureHowToFindSaga`. **F#** generuje dodatkowe wpisy w **IL** stąd przy próbie kompilacji pojawiał się powyższy komunikat

Wielokrotnie już pisałem o sile oraz możliwościach **NServiceBus'a** i tak też było w tym przypadku. Szybki wgląd w dokumentację i okazało się, że można zmienić domyślnie zachowanie, wskazując jawnie, który element ma pełnić rolę koordynatora. Służy do tego atrybut `[SqlSaga]`. Użycie w **F#** wygląda tak: 

{% highlight fsharp %}
[<SqlSaga(nameof Unchecked.defaultof<PolicyData>.CommentId)>]
type CommentRegistrationPolicy() =
    ...
{% endhighlight %}

Dokumentacja wskazuje o stosowaniu tej konstrukcji w sytuacjach niestandardowych. Połączenie **NServiceBus'a** z **F#** można uznać za taką sytuację ;)

## Endpoint Common

Następnym elementem do zmiany był **Endpoint Common**, którego intencją jest współdzielenie podstawowej konfiguracji, pomiędzy wszystkimi Endpoint'ami.

### Multiple Message Conventions

Ponownie, w trakcie zmiany **C#** na **F#**, skorzystałem z jednej z nowości **NServiceBus'a** - [Multiple Message Conventions][17].

Po przepisaniu kodu jeden do jeden, konfiguracja konwencji wyglądała tak:

{% highlight fsharp %}
let conventions = endpointConfiguration.Conventions()
conventions.DefiningCommandsAs(fun t -> t.Namespace <> null && t.Namespace.EndsWith("Commands") || t.IsAssignableFrom(typeof<ICommand>)) |> ignore
conventions.DefiningEventsAs(fun t -> t.Namespace <> null && t.Namespace.EndsWith("Events") || t.IsAssignableFrom(typeof<IEvent>)) |> ignore
conventions.DefiningMessagesAs(fun t -> t.Namespace <> null && t.Namespace.EndsWith("Messages") || t.IsAssignableFrom(typeof<IMessage>) || t = typeof<NServiceBus.Mailer.MailMessage>) |> ignore
{% endhighlight %}

Ze względu na to, że klasa `NServiceBus.Mailer.MailMessage` korzysta z domyślnej konwencji **NServiceBusa**, trzeba było ją zachować, używając dodatkowego warunku `||`.

Nowy sposób pozwala dodawać własne konwencje do już istniejących:

{% highlight fsharp %}
let conventions = endpointConfiguration.Conventions()
conventions.Add(
    { new IMessageConvention with
        member this.Name = "Type name suffix"
        member this.IsCommandType(t) = t.Namespace <> null && t.Namespace.EndsWith("Commands")
        member this.IsEventType(t) = t.Namespace <> null && t.Namespace.EndsWith("Events")
        member this.IsMessageType(t) = t.Namespace <> null && t.Namespace.EndsWith("Messages")}) |> ignore
{% endhighlight %}

Metoda `convention.Add` przyjmuje w parametrze implementację interfejsu `IMessageConvention`. 

Dzięki temu zniknęła konieczność jawnego definiowania konwencji dla `NServiceBus.Mailer.MailMessage` - uproszczenie.

### Object Expressions

**F#** posiada bardzo fajną konstrukcję - [Object Expressions][18]. Dzięki niej można zakodować implementację interfejsu w miejscu jego użycia. Przydatne w sytuacjach, w których interfejs wykorzystywany jest tylko w jednym fragmencie kodu. Konfigurowanie konwencji dla **Messge'y** idealnie wpasowuje się w ten scenariusz, dlatego skorzystałem z tej konstrukcji przy ich definiowaniu.

## Testy

Ostatnim, dużym fragmentem do dostosowania były testy jednostkowe oraz integracyjne. W wersji **C#** wszystkie zależności były dostarczane przez interfejsy, dzięki czemu w testach można było wstrzykiwać fake'owe implementacje. **F#** obsługuje interfejsy i mogłem pójść tą samą ścieżką, ale chciałem sprawdzić, czy można osiągnąć podobny efekt, korzystając z konstrukcji funkcyjnych :)

* wartości dostarczane przez jeden z dostępnych typów danych: simple, tuple, record, ...
* zachowania dostarczane przez funkcje

Efekt? Policy to klasa. Jak klasa, to można w niej definiować konstruktory. **NServiceBus**, przy tworzeniu instancji Sagi wywołuje konstruktor bezparametrowy. Test może wołać dowolny konstruktor. Rozwiązaniem jest użycie dwóch konstruktów.

Przykład dla wartości:

{% highlight fsharp %}
type CommentAnswerNotificationPolicy(smtpFrom: string, blogDomainName: string) =
    ...
    new() = CommentAnswerNotificationPolicy(
                ConfigurationProvider.smtpFrom,
                ConfigurationProvider.blogDomainName)
{% endhighlight %}

Główny konstruktor przyjmuje dwa parametry typu `string`. Drugi konstruktor woła pierwszy, dostarczając wartości pobrane z konfiguracji.

Testy:

{% highlight fsharp %}
module PolicyTests =
    ...
    let getPolicy smtpFrom blogDomainName data =
            CommentAnswerNotificationPolicy(smtpFrom, blogDomainName, Data=data)
    ...
    [<Test>]
    let Handle_RegisterCommentNotification_FillProperPolicyData () =

        // Arrange
        ...
        let policyData = PolicyData()
        let policy = getPolicy "test@test.com" "sampleBlogDomainName" policyData
        ...
{% endhighlight %}

Funkcja `getPolicy` przyjmuje parametry `smtpFrom`, `blogDomainName` oraz `data`. Przekazuje je do konstruktora `CommentAnswerNotificationPolicy`. Parametr `data` podstawiany jest pod publiczne property `Data`. Test woła `getPolicy`, podstawiając odpowiednie wartości.

Przykład dla zachowania:

{% highlight fsharp %}
type GitHubPullRequestVerificationPolicy (checkPullRequestStatus: string -> string -> Async<ResponseCheckPullRequestStatus>) =
    new() = GitHubPullRequestVerificationPolicy(GitHub.checkPullRequestStatus)
{% endhighlight %}

Główny konstruktor przyjmuje funkcję, która zawiera dwa parametry typu `string` i zwraca, w sposób asynchroniczny, wartość `ResponseCheckPullRequestStatus`. Drugi konstruktor woła pierwszy, dostarczając implementację funkcji zgodną z oczekiwaną sygnaturą.

Testy:

{% highlight fsharp %}
module PolicyTests =
    ...
    let getPolicy() = GitHubPullRequestVerificationPolicy(fun _ _ -> async { return ResponseCheckPullRequestStatus(PullRequestStatus.Open, "ETag_123") })
    ...
    [<Test>]
    let Handle_RequestCheckPullRequestStatus_ProperResult () =

        // Arrange
        ...
        let policy = getPolicy () :> IHandleMessages<RequestCheckPullRequestStatus>
        ...
{% endhighlight %}

Funkcja `getPolicy` zwraca obiekt `GitHubPullRequestVerificationPolicy`. W parametrze konstruktora przekazuje funkcję lambda, symulującą zachowanie, które polega na zwróceniu statusu odpowiedzi na komentarz. Test woła `getPolicy` i rzutuje zwrócony wynik na odpowiedni typ.

Pozostałe zmiany dotyczyły usprawnień pracy z solucją oraz jej czyszczenie z nieużywanego kodu. Całą implementację wraz z historią zmian możesz podejrzeć na [GitHub'e][19].

## Podsumowanie

**F# 5** pozwala programować **NServiceBus Saga**, w taki sam sposób, jak **C#**. **NServiceBus 7.4** wprowadza usprawnienia, które ułatwiają realizację końcowego produktu. Z wielkiej czwórki [ADSD][4], [NServiceBus][5], [C#][20], [F#][6] zrobiły się dwie, równoważne wielkie trójki:

* **ADSD**, **NServiceBus**, **C#**
* **ADSD**, **NServiceBus**, **F#**

Od tego momentu, jeśli dodajesz komentarz na tym blogu, to całość realizowana jest z wykorzystaniem ostatniej z nich :)

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2021-01-06-what-was-what-is-what-will-be.md %}
[2]: https://github.com/mikedevbo/blog-comments/releases "GitHub Releases"
[3]: {{ site.url }}{% link _posts/2020-04-13-adsd-nservicebus-csharp-fsharp-perfect-quartet.md %}
[4]: https://particular.net/adsd "ADSD course"
[5]: https://particular.net/nservicebus "NServiceBus framework"
[6]: https://fsharp.org "Fsharp language"
[7]: {{ site.url }}{% link _posts/2019-09-01-fsharp-nservicebus-practical-guide-introduction.md %}
[8]: {{ site.url }}{% link _posts/2016-11-26-wytwarzanie_oprogramowania_I.md %}
[9]: {{ site.url }}{% link _posts/2020-09-29-adsd-nservicebus-csharp-fsharp-lets-dance-design.md %}
[10]: {{ site.url }}{% link _posts/2020-11-08-fsharp-5-nservicebus-saga-perfect-match.md %}
[11]: https://particular.net/blog/whats-new-nservicebus-7-4 "NServiceBus 7.4"
[12]: https://code.visualstudio.com "Visual Studio Code"
[13]: https://docs.microsoft.com/en-us/dotnet/fsharp "Fsharp documentation"
[14]: https://particular.net/blog/whats-new-nservicebus-7-4#finding-sagas "NServiceBus Finding Saga"
[15]: https://docs.particular.net/persistence/sql/ "SQL Persistence"
[16]: https://github.com/Particular/NServiceBus.Persistence.Sql "SQL Persistence Source Code"
[17]: https://particular.net/blog/whats-new-nservicebus-7-4#multiple-message-conventions "NServiceBus Multiple Message Convention"
[18]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/object-expressions "F# Object Expressions"
[19]: https://github.com/mikedevbo/blog-comments "Blog Comment Source Code"
[20]: https://docs.microsoft.com/en-us/dotnet/csharp/ "C# language"
