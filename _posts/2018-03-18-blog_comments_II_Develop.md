---
layout: post
title:  "System komentarzy do bloga II - Develop"
date:   2018-03-18
---

Posty z tej serii:

* [System komentarzy do bloga I - Design]({% post_url 2018-03-11-blog_comments_I_Design %})
* [System komentarzy do bloga II - Develop]({% post_url 2018-03-18-blog_comments_II_Develop %})
* [System komentarzy do bloga III - Test]({% post_url 2018-03-30-blog_comments_III_Test %})

W [pierwszej części]({% post_url 2018-03-11-blog_comments_I_Design %}) serii widzieliśmy przykładowy projekt rozwiązania dla systemu komentarzy do bloga. W tym poście skupimy się na wybranych fragmentach implementacji poszczególnych komponentów. Całość można zobaczyć na [GitHub'e][1]. 

## Narzędzia

Większość framework'ów, bibliotek, narzędzi jest rozwijanych przez cały czas, dlatego poniższe przykłady bazują na konkretnych wersjach:

* [C#][2] ver. 6.0
* [NServiceBus][3] ver. 6.4.3
* [Nancy][4] ver. 1.4.4

## Nancy - Web Module

Jak pamiętamy, proces dodania komentarza do bloga zaczyna się od jego przesłania przez czytelnika. Pierwszym komponentem przyjmującym zgłoszenie jest komponent Web'owy. W `Nancy` żądania `HTTP` implementuje się poprzez utworzenie tzw. `NancyModule` oraz implementację metod `Get`, `Put`, `Post`, itd. Ponieważ dodanie komentarza sugeruje nam, że mamy do czynienia z jego utworzeniem (w przeciwieństwie np. do jego wyświetlenia) odpowiednim miejscem na implementację jest metoda `HTTP Post` udostępniania przez API Nancy.

{% highlight csharp %}
this.Post["/", true] = async (r, c) =>
{
    var comment = this.Bind<Comment>();

    var validationResult = await this.validator.ValidateAsync(comment)
                                     .ConfigureAwait(false);
    if (!validationResult.IsValid)
    {
        return this.Negotiate
                   .WithModel(validationResult)
                   .WithStatusCode(HttpStatusCode.BadRequest);
    }

    await this.messageSession.Send<StartAddingComment>(command =>
    {
        command.CommentId = Guid.NewGuid();
        command.UserName = comment.UserName;
        command.UserEmail = comment.UserEmail;
        command.UserWebsite = comment.UserWebsite;
        command.FileName = comment.FileName;
        command.Content = comment.Content;
    }).ConfigureAwait(false);

    return HttpStatusCode.OK;
};
{% endhighlight %}

Zgodnie z założeniami projektowymi, komponent Web'owy przyjmuje zgłoszenie z danymi oraz wysyła Message inicjujący utworzenie i obsługę komentarza. Dodatkiem do implementacji jest walidacja przesłanych danych, tak aby upewnić się, że są one poprawne i nie zaburzą dalszego przebiegu procesu.

## NServiceBus - Saga

`Saga` jest komponentem na bazie którego zostało zaprojektowane całe rozwiązanie. Jest to serce i rozum koordynujące wszystkie pozostałe kroki w procesie dodawania i obsługi komentarza do bloga. Zobaczmy przykład implementacji.

### Definicja

{% highlight csharp %}
public class HandlerCommentSaga :
    SqlSaga<CommentSagaData>,
    IAmStartedByMessages<StartAddingComment>,
    IHandleMessages<IBranchCreated>,
    IHandleMessages<ICommentAdded>,
    IHandleMessages<IPullRequestCreated>,
    IHandleTimeouts<CheckCommentResponseTimeout>,
    IHandleMessages<ICommentResponseAdded>
{
    ...
}
{% endhighlight %}

Definicję Sagi pełni klasa `HandlerCommentSaga`. Dane, jakie Saga musi przechowywać na potrzeby koordynacji procesu reprezentowane są przez klasę `CommentSagaData`. Nowa instancja Sagi tworzona jest w momencie otrzymania message'a reprezentowanego przez klasę `StartAddingComment`. W tym momencie Saga istnieje i czeka na sygnały do dalszego działania. Sygnały te reprezentowane są przez odpowiednie message'e:

* `IBranchCreated` - message o utworzeniu branch'a na GitHub'e
* `ICommentAdded` - message o dodaniu treści komentarza do odpowiedniego posta
* `IPullRequestCreated` - message o utworzeniu Pull Request'a na GitHub'e
* `ICommentResponseAdded` - message o podjęciu działania w zależności od wyniku odpowiedzi na komentarz

Message `CheckCommentResponseTimeout` pełni specjalną rolę. Jest to sygnał dla Sagi o "wybudzeniu" się po określonym czasie.

## Korelacja

Każda instancja Sagi reprezentuje obsługę oddzielnego posta co oznacza, że instancji będzie dokładnie tyle ile jeszcze nie obsłużonych komentarzy. Aby NServiceBus wiedział jaką instancję Sagi ma "podnieść", trzeba zdefiniować korelację pomiędzy poszczególnymi message'ami.

{% highlight csharp %}
protected override string CorrelationPropertyName => nameof(CommentSagaData.CommentId);

protected override void ConfigureMapping(IMessagePropertyMapper mapper)
{
    mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId);
    mapper.ConfigureMapping<IBranchCreated>(message => message.CommentId);
    mapper.ConfigureMapping<ICommentAdded>(message => message.CommentId);
    mapper.ConfigureMapping<IPullRequestCreated>(message => message.CommentId);
    mapper.ConfigureMapping<CheckCommentResponseTimeout>(message => message.CommentId);
    mapper.ConfigureMapping<ICommentResponseAdded>(message => message.CommentId);
}
{% endhighlight %}

Zarówno dane Sagi jak i każdy message zawierają property `CommentId`. NServiceBus, przetwarzając message, sprawdza czy istnieje Saga o danym `CommentId`. Jeśli tak, to "podnosi" instancję Sagi wraz z jej aktualnymi danymi. Jeśli nie to tworzy nową instancję.

## Message Handlers

Mając już zdefiniowaną i skonfigurowaną Sagę można przystąpić do implementacji jej logiki. Z definicji, Saga jako Process Manager, ma służyć do zarządzania tzw. `Long-running process`. Z tego względu jej logika "musi" być ograniczona do:

* przyjmowania oraz wysyłania message'y
* zarządzania swoim własnym stanem
* decydowania jaki kolejny krok powinien zostać podjęty w procesie

Saga "nie może" bezpośrednio (przez bazę danych) lub pośrednio (np. przez zdalne wywołanie) pobierać oraz zapisywać żadnych danych. "Nie może" mieć również dostępu do jakichkolwiek innych zasobów. Słowa "musi" i "nie może" są w cudzysłowie, bo jest to założenie na poziomie projektowym podobnie jak dla innych zdefiniowanych wzorców. Przejdźmy do implementacji poszczególnych `Message Handler'ów`.

### StartAddingComment

{% highlight csharp %}
public Task Handle(StartAddingComment message, IMessageHandlerContext context)
{
    this.Data.CommentId = message.CommentId;
    this.Data.UserName = message.UserName;
    this.Data.UserEmail = message.UserEmail;
    this.Data.UserWebsite = message.UserWebsite;
    this.Data.FileName = message.FileName;
    this.Data.Content = message.Content;

    return context.Send<CreateBranch>(command => 
        command.CommentId = this.Data.CommentId);
}
{% endhighlight %}

Pierwszym krokiem jaki wykonuje Saga jest wysłanie message'a typu command o utworzenie nowego branch'a na GitHub'e. Jak widzimy Saga nie odwołuje się bezpośrednio do `GitHub API!` Zapamiętuje dane komentarza, wysyła message i przechodzi w stan oczekiwania.

### CreateBranch

{% highlight csharp %}
public async Task Handle(CreateBranch message, IMessageHandlerContext context)
{
    var sb = new StringBuilder();
    sb.Append("c-")
      .Append(DateTime.UtcNow.ToString("yyyy-MM-dd-HH-mm-ss-fff"));
    string branchName = sb.ToString();

    await this.gitHubApi.CreateRepositoryBranch(
        this.configurationManager.UserAgent,
        this.configurationManager.AuthorizationToken,
        this.configurationManager.RepositoryName,
        this.configurationManager.MasterBranchName,
        branchName).ConfigureAwait(false);

    await context.Publish<IBranchCreated>(
        evt =>
        {
            evt.CommentId = message.CommentId;
            evt.CreatedBranchName = branchName;
        })
        .ConfigureAwait(false);
}
{% endhighlight %}

Message Handler, którego zadaniem jest utworzenie branch'a nie jest częścią Sagi, a co za tym idzie jest od niej całkowicie niezależny. Logika utworzenia nowego branch'a składa się z trzech kroków:

1. stworzenie nazwy branch'a
2. wywołanie API GitHub'a
3. opublikowanie wiadomości typu event o tym, że dla `CommentId` został utworzony branch o nazwie `branchName`

Tutaj widzimy, że Message Handler korzysta z GitHub API, natomiast nie wie nic o tym, że to akurat Saga zainicjowała utworzenie branch'a. W ten oto sposób mam dwa komponenty luźno powiązane ze sobą ([loose coupling][5]) jedynie poprzez kontrakt message'y: `CreateBranch` oraz `IBranchCreated`

### IBranchCreated

{% highlight csharp %}
public Task Handle(IBranchCreated message, IMessageHandlerContext context)
{
    this.Data.BranchName = message.CreatedBranchName;

    return context.Send<AddComment>(command =>
        {
            command.CommentId = this.Data.CommentId;
            command.UserName = this.Data.UserName;
            command.BranchName = this.Data.BranchName;
            command.FileName = this.Data.FileName;
            command.Content = this.Data.Content;
        });
}
{% endhighlight %}

Saga jest komponentem, który jest zainteresowany informacją o utworzeniu branch'a. Po otrzymaniu message'a typu event `IBranchCreated` podejmuje kolejny krok w procesie - dodanie treści posta. W tym celu zapamiętuje w swoim stanie nazwę branch'a oraz wysyła message `AddComment` o dodanie treści komentarza. Zauważmy, że dane potrzebne do dalszego procesowania pobierane są z wewnętrznego stanu Sagi.

Aby post nie był zbyt długi, w dalszej części zajmiemy się tylko `Handler'ami`, które są częścią Sagi. Całość implementacji można zobaczyć na [GitHub'e][1]

### ICommentAdded

{% highlight csharp %}
public Task Handle(ICommentAdded message, IMessageHandlerContext context)
{
   return context.Send<CreatePullRequest>(command =>
   {
        command.CommentId = this.Data.CommentId;
        command.CommentBranchName = this.Data.BranchName;
        command.BaseBranchName = this.configurationManager.MasterBranchName;
   });
}
{% endhighlight %}

Zgodnie z dalszymi krokami procesu, po dodaniu komentarza do posta, Saga wysyła Message `CreatePullRequest` o utworzenie GitHub Pull Request'a. Jak w poprzednich Handler'ach dane potrzebne do realizacji procesu pobierane są z wewnętrznego stanu Sagi.

### IPullRequestCreated

{% highlight csharp %}
public Task Handle(IPullRequestCreated message, IMessageHandlerContext context)
{
    this.Data.PullRequestLocation = message.PullRequestLocation;

    return this.SendTimeout(
        context,
        TimeSpan.FromSeconds(this.configurationManager.CommentResponseAddedSagaTimeoutInSeconds),
        this.Data.CommentId);      
}
{% endhighlight %}

Jeśli Pull Request jest już gotowy Saga przechodzi do punktu 16 na diagramie z [poprzedniego posta]({% post_url 2018-03-11-blog_comments_I_Design %}). W tym celu wysyła Message typu `Timeout` do samej siebie aby "zbudzić" się za określony czas zdefiniowany w `this.configurationManager.CommentResponseAddedSagaTimeoutInSeconds`. Wartość ta pobierana jest z konfiguracji aby można było, w razie potrzeb, skracać lub wydłużać interwał bez konieczności re-kompilacji kodu. Jak za chwilę zobaczymy `Message Timeout` użyty jest w jeszcze jednym miejscu także w tym momencie można zastosować tzw. `small tactical re-use` i "zamknąć" implementację wysyłania Timeout w metodzie `SendTimeout`. Zobaczmy jak wygląda implementacja tej metody:

{% highlight csharp %}
private Task SendTimeout(
    IMessageHandlerContext context,
    TimeSpan timeoutInterval,
    Guid commentId)
{
    return this.RequestTimeout(
        context,
        timeoutInterval,
        new CheckCommentResponseTimeout { CommentId = commentId });
}
{% endhighlight %}

Metoda `RequestTimeout` jest częścią API NServiceBus'a realizujacą funkcjonlaność "wybudzania" Sagi za określony interwał czasowy poprzez wysłanie wiadomości `CheckCommentResponseTimeout`

### CheckCommentResponseTimeout

{% highlight csharp %}
public Task Timeout(
    CheckCommentResponseTimeout state,
    IMessageHandlerContext context)
{
    return context.Send<CheckCommentResponse>(command =>
    {
        command.CommentId = this.Data.CommentId;
        command.PullRequestUri = this.Data.PullRequestLocation;
        command.Etag = this.Data.ETag;
    });
}
{% endhighlight %}

Po "zbudzeniu" Saga wysyła message `CheckCommentResponse` nakazujący sprawdzenie stanu odpowiedzi na komentarz.

### ICommentResponseAdded

{% highlight csharp %}
public async Task Handle(ICommentResponseAdded message, IMessageHandlerContext context)
{
    if (message.CommentResponse.ResponseStatus == CommentResponseStatus.Approved ||
        message.CommentResponse.ResponseStatus == CommentResponseStatus.Rejected)
    {
        await context.Send<SendEmail>(command =>
        {
            command.UserName = this.Data.UserName;
            command.UserEmail = this.Data.UserEmail;
            command.FileName = this.Data.FileName;
            command.CommentResponseStatus = message.CommentResponse.ResponseStatus;
        }).ConfigureAwait(false);

        this.MarkAsComplete();
    }
    else
    {
        this.Data.ETag = message.CommentResponse.ETag;

        await this.SendTimeout(
            context,
            TimeSpan.FromSeconds(this.configurationManager.CommentResponseAddedSagaTimeoutInSeconds),
            this.Data.CommentId).ConfigureAwait(false);
    }
}
{% endhighlight %}

Ostatnim krokiem jest podjęcie decyzji co zrobić w zależności od statusu odpowiedzi na komentarz. Zgodnie z założeniem procesu, jeśli komentarz nie został jeszcze obsłużony Saga ponownie wysyła message do samej siebie o "zbudzenie" w celu ponownego sprawdzenia statusu odpowiedzi. Jeśli jest odpowiedź na komentarz (pozytywna lub negatywna) to Saga:

1. Wysyła message o wysłania e-maila do autora komentarza. Zauważmy ponownie, Saga nie korzysta z żadnego API do wysyłania maili, to nie jest jej odpowiedzialność.
2. Kończy swoje działanie poprzez wywołanie metody `MarkAsComplete`, która kasuje instancję Sagi. Z jej punktu widzenia cały proces dodania i obsługi komentarza został zakończony.

To tyle jeśli chodzi o wybrane fragmenty implementacji. W następnym poście zobaczymy w jaki sposób można testować poszczególne elementy.

[1]: https://github.com/mikedevbo/blog-comments "blog-comments"
[2]: https://docs.microsoft.com/pl-pl/dotnet/csharp/whats-new/csharp-6 "C#"
[3]: https://particular.net/nservicebus "NServiceBus"
[4]: http://nancyfx.org/ "Nancy"
[5]: https://en.wikipedia.org/wiki/Loose_coupling "Loose coupling" 
