---
layout: post
title:  "NServiceBus: Saga + Request/Response = DDD Aggregate Root"
date:   2018-09-30
---

Jak sprawdzić, czy robimy postępy w programowaniu lub projektowaniu systemów? Po siedmiu miesiącach, roku lub półtora roku patrzymy na swój kod lub projekt rozwiązania. Jeśli pierwszą myślą, jaka nam przychodzi do głowy, jest myśl w stylu **_"ale zaj... kod/projekt"_** to jest to znak, że stoimy w miejscu. Jeśli natomiast stwierdzamy coś w stylu **_"o ku$#$#, ale to rozwiązanie jest słabe, to powinno być napisane/zaprojektowane tak..."_** tzn. że się rozwijamy ;D. Oczywiście jest to stwierdzenie pół-żartem, pół-serio, ale faktem jest, że zdobywając nową wiedzę oraz nowe umiejętności, zaczynamy patrzeć na ten sam problem z innej perspektywy. Jeśli trafia się okazja rozwoju istniejącej funkcjonalności i widzimy, że pewne elementy w tej funkcjonalności można poprawić na lepsze, to warto "przemycić" taką zmianę jako część zadania rozwojowego. Jeśli podejmujemy decyzję o zmianie, dobrze jest przeprowadzać taką zmianę etapami, przez ewolucję, zamiast dokonywać rewolucji, zmieniając wszystko za jednym razem. Zobaczmy na przykładzie zmiany [systemu dodawania komentarzy na blogu]({% post_url 2018-03-11-blog_comments_I_Design %}), w jaki sposób NServiceBus oraz Messaging wspomagają podejście ewolucyjne przy dokonywaniu zmian w kodzie lub projekcie rozwiązania.

Zanim przejdziemy do przeprojektowania oraz przeprogramowania pierwotnego rozwiązania, zobaczmy, jakimi rodzajami wiadomości możemy rozmawiać w języku NServiceBusa:

* **Command** - jest to rodzaj wiadomości zlecającej wykonanie jakiejś akcji np. _"wyślij e-mail"_
* **Event** - jest to rodzaj wiadomości komunikującej o wykonaniu jakiejś akcji np. _"e-mail został wysłany"_
* **Message** - jest to rodzaj wiadomości, która nie jest żadnym z powyższych np. _"odpowiedź do nadawcy zlecającego wysłanie e-maila"_

Jeśli popatrzymy na [pierwszą implementację][1] systemu komentarzy na blogu, zobaczymy, że Saga inicjuje kolejne kroki w procesie, wysyłając wiadomości typu Command. Sama zaś nasłuchuje na wiadomości typu Event jako odpowiedź na realizację konkretnego Commanda np.

* AddComment -> ICommentAdded
* CreatePullRequest -> IPullRequestCreated
* CheckCommentResponse -> ICommentResponseAdded

Po dokładniejszej analizie można zadać sobie pytanie: Czy ktoś inny oprócz Sagi może być zainteresowany Eventami publikowanymi przez Handlery? Na obecny stan systemu odpowiedź brzmi NIE. Jeśli jednak pojawiłby się kandydat, to czy powinien dostawać Eventy bezpośrednio od Handlerów? Po dłuższym zastanowieniu możemy również stwierdzić, że NIE. To Saga koordynuje cały proces i takie Eventy powinny być publikowane tylko przez nią. Co więc zatem możemy zrobić? Ano skorzystać z możliwości NServiceBusa i zrefaktoryzować obecną implementację, wykorzystując podejście [Saga and Request/Response][2], gdzie poszczególne Handlery zamiast publikować Eventy, wysyłają odpowiedź do Sagi jako wiadomości typu Message. Zobaczmy więc, jak taki refaktoring może wyglądać na wybranym fragmencie kodu, pamiętając o podejściu ewolucyjnym zamiast rewolucyjnym.

## Krok 1 - SqlSaga -> Saga

Przy okazji większych zmian w kodzie, zawsze warto rozważyć przejście na nowsze wersje używanych bibliotek/frameworków, dzięki czemu możemy być na bieżąco z poprawkami błędów. Czasami też pojawiają się nowe funkcjonalności, z których możemy skorzystać. Tak jest np. w przypadku paczki Nugetowej **NServiceBus.Persistence.Sql**. Po przejściu na wersję **4.2** do implementacji Sagi można użyć bazowej klasy **Saga** zamiast specyficznej klasy bazowej [SqlSaga][3]. Dzięki podziałowi całej logiki na osobne Handlery zmiana ta dotyczy tylko klasy reprezentującej definicję Sagi - **HandlerCommentSaga**.

### Krok 1.1 - zmiana klasy bazowej

Kod przed zmianą:

{% highlight csharp %}
public class HandlerCommentSaga :
    SqlSaga<CommentSagaData>,
    //...
{
    //...
}
{% endhighlight %}

Kod po zmianie:

{% highlight csharp %}
public class HandlerCommentSaga :
    Saga<CommentSagaData>,
    //...
{
    //...
}
{% endhighlight %}

### Krok 1.2 - podążamy za kompilatorem

Ponieważ zmieniliśmy klasę bazową, to i definicja klasy się zmienia. Pewne elementy dochodzą, pewne wypadają. Z racji tego, że C# w swej pierwotnej naturze jest językiem ze statyczną kontrolą typów, to kompilator przeprowadzi nas przez pierwszą fazę zmian, jakich musimy dokonać:

![Picutre1]({{ site.url }}/assets/nservicebus-saga-request-response-ddd-aggregate-root/compile-time-error.png)

Zgodnie z sugestią usuwamy elementy, których nie ma w nowej klasie bazowej:

{% highlight csharp %}
protected override void ConfigureMapping(IMessagePropertyMapper mapper)
{
    //...
}
protected override string CorrelationPropertyName => nameof(CommentSagaData.CommentId);
{% endhighlight %}

Dodajemy implementację nowej wymaganej metody **ConfigureHowToFindSaga**, która zastępuje poprzednią metodę **ConfigureMapping**:

{% highlight csharp %}
protected override void ConfigureHowToFindSaga(SagaPropertyMapper<CommentSagaData> mapper)
{
    mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId);
    mapper.ConfigureMapping<IBranchCreated>(message => message.CommentId);
    mapper.ConfigureMapping<ICommentAdded>(message => message.CommentId);
    mapper.ConfigureMapping<IPullRequestCreated>(message => message.CommentId);
    mapper.ConfigureMapping<CheckCommentResponseTimeout>(message => message.CommentId);
    mapper.ConfigureMapping<ICommentResponseAdded>(message => message.CommentId);
}
{% endhighlight %}

Po tych zmianach kod ponownie kompiluje się bez błędów, ale czy to oznacza, że działa? Przekonać się o tym możemy, testując zmienioną funkcjonalność. Inicjując pierwszą Sagę, okazuje się, że dostajemy wyjątek jak poniżej. Tutaj należy powiedzieć o bardzo dobrym i bardzo przydatnym opisie błędów zwracanym przez NServiceBusa:

![Picutre1]({{ site.url }}/assets/nservicebus-saga-request-response-ddd-aggregate-root/run-time-exception.png)

Stosujemy się do wskazówek i uzupełniamy konfigurację o brakujące wywołanie metody **ToSaga**, używając do korelacji tego samego property jak poprzednio - CommentId:

{% highlight csharp %}
protected override void ConfigureHowToFindSaga(SagaPropertyMapper<CommentSagaData> mapper)
{
    mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<IBranchCreated>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<ICommentAdded>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<IPullRequestCreated>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<CheckCommentResponseTimeout>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<ICommentResponseAdded>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
}
{% endhighlight %}

Mimo że zmiana jest mała, to jest na tyle specyficzna, że warto zatrzyma się na chwilę i zastanowić nad jej wdrożeniem produkcyjnym. W ten sposób będziemy mieli pewność, że wszystko działa tak jak poprzednio przed przystąpieniem do następnego etapu refaktoryzacji.

## Krok 2 - Pub/Sub -> Request/Response

Ponownie korzystamy z podziału funkcjonalności na osobne Handlery, dzięki czemu możemy każdy z nich zrefaktoryzować osobno. W ten sposób cały proces zmiany może wyglądać tak:

* zmiana Sagi
* zmiana Handlera
* testy
* commit
* ...
* zmiana Sagi
* zmiana Handlera
* testy
* commit
* ...

Zmian można dokonać co najmniej na dwa sposoby:

* sposób nr 1
    * tworzymy nowy kod
    * wpinamy nowy kod w miejsca, gdzie używany jest stary kod
    * usuwamy stary kod
* sposób nr 2
    * zastępujemy istniejący kod nowym, przez co automatycznie stary jest usuwany
    * wpinamy nowy kod w miejsca, gdzie był używany stary kod

Podejście nr 1 jest bardziej bezpieczne od podejścia nr 2. W tym konkretnym przypadku możemy wybrać sposób nr 2, ponieważ Handlery realizują jedną konkretną rzecz, a co za tym idzie, są małe. Dodatkowo dostajemy pomoc w postaci kompilatora, który podpowie nam, gdzie trzeba dokonać zmian po usunięciu starego kodu. Zobaczmy, jak może przebiegać taka zmiana na przykładzie funkcjonalności tworzenia GitHub brancha dla nowego komentarza.

### Krok 2.1 Saga

Kod zlecający utworzenie brancha przed zmianą:

{% highlight csharp %}
public Task Handle(StartAddingComment message, IMessageHandlerContext context)
{
    this.Data.CommentId = message.CommentId;
    this.Data.UserName = message.UserName;
    this.Data.UserEmail = message.UserEmail;
    this.Data.UserWebsite = message.UserWebsite;
    this.Data.FileName = message.FileName;
    this.Data.Content = message.Content;

    return context.Send<CreateBranch>(command => command.CommentId = this.Data.CommentId);
}
{% endhighlight %}

Kod zlecający utworzenie brancha po zmianie:

{% highlight csharp %}
public Task Handle(StartAddingComment message, IMessageHandlerContext context)
{
    this.Data.CommentId = message.CommentId;
    this.Data.UserName = message.UserName;
    this.Data.UserEmail = message.UserEmail;
    this.Data.UserWebsite = message.UserWebsite;
    this.Data.FileName = message.FileName;
    this.Data.Content = message.Content;

    return context.Send(new RequestCreateBranch());
}
{% endhighlight %}

W powyższym kodzie widzimy jedną zmianę. Zamiast wysyłać wiadomość **CreateBranch**, wysyłana jest wiadomość **RequestCreateBranch**. Dodatkowo nowa wiadomość nie przekazuje wartości **CommentId**. Spowodowane jest to tym, że przy wykorzystaniu podejścia Request/Response **NServiceBus automatycznie potrafi skorelować wiadomość wychodzącą z Sagi, z tą, która jest odpowiedzią na tę wiadomość.** Dzięki temu zmniejsza się ilość jawnej konfiguracji. Wiadomość CreateBranch nie jest już potrzebna, więc można ją usunąć. W jej miejsce wchodzi wiadomość RequestCreateBranch, która jest wiadomością typu Message, w odróżnieniu do CreateBranch, która była wiadomością typu Command. Właściwość CommentId nie jest już potrzebna, więc sama definicja wiadomości również się upraszcza.

Poprzednia definicja:

{% highlight csharp %}
namespace Messages.Commands
{
    using System;

    public class CreateBranch
    {
        public Guid CommentId { get; set; }
    }
}
{% endhighlight %}

Nowa definicja:

{% highlight csharp %}
namespace Messages.Messages
{
    public class RequestCreateBranch
    {
    }
}
{% endhighlight %}

Zobaczmy teraz zmianę logiki obsługującej odpowiedź o utworzeniu brancha.

Kod przed zmianami:

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

Kod po zmianach:

{% highlight csharp %}
public Task Handle(CreateBranchResponse message, IMessageHandlerContext context)
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

W tym przypadku zmiana również jest mała. Wiadomość obsługiwana przez Handlera zmienia się z wiadomości **IBranchCreated** typu Event na wiadomość **CreateBranchResponse** typu Message. Podobnie jak poprzednio definicja wiadomości odpowiedzi upraszcza się poprzez usunięcie właściwości CommentId - NServiceBus nie tylko wie, do jakiego typu Sagi wysłać odpowiedź, ale dokładnie do jakiej instancji.

Poprzednia definicja:

{% highlight csharp %}
namespace Messages.Events
{
    using System;

    public interface IBranchCreated
    {
        Guid CommentId { get; set; }

        string CreatedBranchName { get; set; }
    }
}
{% endhighlight %}

Nowa definicja:

{% highlight csharp %}
namespace Messages.Messages
{
    public class CreateBranchResponse
    {
        public string CreatedBranchName { get; set; }
    }
}
{% endhighlight %}

Ostatnią zmianą, jeśli chodzi o Sagę, jest zmiana jej sygnatury oraz konfiguracji:

Kod przed zmianą:

{% highlight csharp %}
public class HandlerCommentSaga :
    Saga<CommentSagaData>,
    IAmStartedByMessages<StartAddingComment>,
    IHandleMessages<IBranchCreated>,
    IHandleMessages<ICommentAdded>,
    IHandleMessages<IPullRequestCreated>,
    IHandleTimeouts<CheckCommentResponseTimeout>,
    IHandleMessages<ICommentResponseAdded>
    IHandleMessages<ICommentResponseAdded>
    {
        //...
    }

protected override void ConfigureHowToFindSaga(SagaPropertyMapper<CommentSagaData> mapper)
{
    mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<IBranchCreated>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<ICommentAdded>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<IPullRequestCreated>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<CheckCommentResponseTimeout>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
}    
{% endhighlight %}    

Kod po zmianie:

{% highlight csharp %}
public class HandlerCommentSaga :
    Saga<CommentSagaData>,
    IAmStartedByMessages<StartAddingComment>,
    IHandleMessages<ICommentAdded>,
    IHandleMessages<IPullRequestCreated>,
    IHandleTimeouts<CheckCommentResponseTimeout>,
    IHandleMessages<ICommentResponseAdded>
    IHandleMessages<ICommentResponseAdded>,
    IHandleMessages<CreateBranchResponse>
    {
        //...
    }

protected override void ConfigureHowToFindSaga(SagaPropertyMapper<CommentSagaData> mapper)
{
    mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<ICommentAdded>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<IPullRequestCreated>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    mapper.ConfigureMapping<CheckCommentResponseTimeout>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
}    
{% endhighlight %}    

Ponownie zmiana jest mała. Saga przestaje nasłuchiwać na wiadomość **IBranchCreated**, w związku z tym usuwane jest dziedziczenie oraz konfiguracja korelacji:

 * **IHandleMessages&lt;IBranchCreated&gt;**
 * **mapper.ConfigureMapping&lt;IBranchCreated&gt;(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);**
 
 
 Zaczyna natomiast nasłuchiwać na wiadomość **CreateBranchResponse**, w związku z tym dodawane jest dziedziczenie **IHandleMessages&lt;CreateBranchResponse&gt;**. Podobnie jak we wcześniejszych przykładach żadna dodatkowa konfiguracja korelacji wiadomości CreateBranchResponse nie jest potrzebna. NServiceBus obsługuje to za nas.

### Krok 2.2 RequestCreateBranch Handler

Po refaktoryzacji Sagi czas na refaktoryzację Handlera obsługującego wiadomość tworzenia nowego brancha.

Kod przed zmianą:

{% highlight csharp %}
public class HandlerCreateBranch : IHandleMessages<CreateBranch>
{
    //...

    public async Task Handle(CreateBranch message, IMessageHandlerContext context)
    {
        //...

        await context.Publish<IBranchCreated>(evt =>
        {
            evt.CommentId = message.CommentId;
            evt.CreatedBranchName = branchName;
        }
        .ConfigureAwait(false);
    }
}
{% endhighlight %}

Kod po zmianie:

{% highlight csharp %}
public class RequestCreateBranchHandler : IHandleMessages<RequestCreateBranch>
{
    //...

    public async Task Handle(RequestCreateBranch message, IMessageHandlerContext context)
    {
        //...

        await context.Reply<CreateBranchResponse>(response =>
        {
            response.CreatedBranchName = branchName;
        }
        .ConfigureAwait(false);
    }
}
{% endhighlight %} 

Pierwszą zmianą jest zmiana obsługiwanej wiadomości. Wiadomość **RequestCreateBranch**, która jest wiadomością typu Message, zastępuje wiadomość **CreateBranch** będącą wiadomością typu Command. Drugą zmianą jest sposób zwracania odpowiedzi. Poprzednio publikowana była wiadomość typu Event **IBranchCreated** poprzez wywołanie **context.Publish&lt;IBranchCreated&gt;**. Po zmianie do nadawcy zwracana jest odpowiedź w postaci wiadomości typu Message **CreateBranchResponse** poprzez wywołanie **context.Reply&lt;CreateBranchResponse&gt;**. Tak jak poprzednio w definicji wiadomości nie trzeba podawać właściwości korelacji CommentId - NServiceBus obsługuje to za nas.

Poprzednia definicja:

{% highlight csharp %}
namespace Messages.Events
{
    using System;

    public interface IBranchCreated
    {
        Guid CommentId { get; set; }

        string CreatedBranchName { get; set; }
    }
}
{% endhighlight %} 

Nowa definicja:

 {% highlight csharp %}
namespace Messages.Messages
{
    public class CreateBranchResponse
    {
        public string CreatedBranchName { get; set; }
    }
} 
{% endhighlight %} 
 
### Krok 2.3 - Saga -> zmiana nazwy

W ten sam sposób można zrefaktoryzować pozostałe elementy składające się na całość funkcjonalności dodawania komentarzy na blogu. Samą Sagę można również wykorzystać do kontrolowania wielu różnych procesów. Warto wtedy potraktować ją jako **Policy**. W kontekście naszego przykładu Saga pełni rolę **Comment Policy**, co możemy wyeksponować poprzez nadanie odpowiedniej nazwy klasie definiującą Sagę. Poniżej ostateczna sygnatura oraz konfiguracja Sagi. Całość implementacji po zmianach można zobaczyć na [GitHubie][4]

{% highlight csharp %}
    public class CommentPolicy :
        Saga<CommentPolicy.CommentPolicyData>,
        IAmStartedByMessages<StartAddingComment>,
        IHandleMessages<CreateBranchResponse>,
        IHandleMessages<AddCommentResponse>,
        IHandleMessages<CreatePullRequestResponse>,
        IHandleTimeouts<CheckCommentAnswerTimeout>,
        IHandleMessages<CheckCommentAnswerResponse>
    {
        //...
    }

    protected override void ConfigureHowToFindSaga(SagaPropertyMapper<CommentPolicyData> mapper)
    {
        mapper.ConfigureMapping<StartAddingComment>(message => message.CommentId).ToSaga(sagaData => sagaData.CommentId);
    }    
{% endhighlight %}

## Podsumowanie

Dzięki podejściu [Saga and Request/Response][2] Saga otrzymuje jeszcze większe uprawnienia do kontrolowania całego Policy. Jeśli całą funkcjonalność dodawania komentarzy na blogu uznamy jako jeden z elementów większej układanki, projektowanej i realizowanej wg podejścia [Domain Driven Design][5], umieszczonej w jednym konkretnym [Bounded Context][6], to Sagę wraz z Handlerami, z którymi się komunikuje, możemy traktować jako [Aggregate][7], a samą Sagę jako **Aggregate Root**. Tym samym wszystkie komunikaty wychodzące na zewnątrz zdefiniowanego Aggregate, a także całego Bounded Context muszą\powinny być publikowane jedynie przez samą Sagę. Przykładowo takimi komunikatami mogą być wiadomości typu Event używane przez framework NServiceBus.

Jeśli masz okazję brać udział w projektowaniu rozwiązań informatycznych, zachęcam, w ramach zdobywania nowej wiedzy, do tego, abyś spróbował/-a zaprojektować rozwiązanie z użyciem Sagi, nawet jeśli na co dzień nie używasz NServiceBusa oraz Messagingu.

[1]: https://github.com/mikedevbo/blog-comments/tree/saga-pub-sub "saga-pub-sub"
[2]: https://docs.particular.net/nservicebus/sagas/ "NServiceBus Saga"
[3]: https://docs.particular.net/persistence/sql/sqlsaga "NServiceBus SqlSaga"
[4]: https://github.com/mikedevbo/blog-comments "blog-comments"
[5]: http://domainlanguage.com/wp-content/uploads/2016/05/DDD_Reference_2015-03.pdf "DDD Reference"
[6]: https://martinfowler.com/bliki/BoundedContext.html "DDD Bounded Context"
[7]: https://martinfowler.com/bliki/DDD_Aggregate.html "DDD Aggregate"

{{ site.mark_post_as_end }}

### {{ site.comments }}
