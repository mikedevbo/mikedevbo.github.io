---
layout: post
title:  "System komentarzy do bloga III - Test"
date:   2018-03-30
---

Posty z tej serii:

* [System komentarzy do bloga I - Design]({% post_url 2018-03-11-blog_comments_I_Design %})
* [System komentarzy do bloga II - Develop]({% post_url 2018-03-18-blog_comments_II_Develop %})
* [System komentarzy do bloga III - Test]({% post_url 2018-03-30-blog_comments_III_Test %})
* [System komentarzy do bloga IV - Deploy]({% post_url 2018-04-27-blog_comments_IV_Deploy %})

Jest to trzecia część cyklu na temat systemu komentarzy do bloga. W dwóch poprzednich częściach widzieliśmy projekt rozwiązania oraz jego poszczególne elementy implementacji. W tej części zobaczymy wybrane fragmenty implementacji testów.

## Narzędzia

Poniższe przykłady bazują na wersjach:

* [C#][1] ver. 7.1
* [NServiceBus][2] ver. 6.4.3
* [NServiceBus.Testing][3] ver. 6.0.3
* [NUnit][4] ver. 3.9.0
* [NSubstitute][5] ver. 3.1.0

W procesie wytwarzania oprogramowania istnieje wiele różnych rodzajów [testów][6]. Jedną z wielu wartości, jakie daje pisanie testów, jest podniesienie jakości tworzonych funkcjonalności. W tej części skupimy się testach jednostkowych oraz integracyjnych.

{% highlight csharp %}
{% endhighlight %}

## Unit Tests

To co jest bardzo fajne w NServiceBus'e to jego bardzo rozbudowane możliwości. Jedną z takich możliwości jest testowanie jednostkowe zaimplementowanych `Message Handler'ów` przy pomocy `NServiceBus.Testing`. Zobaczmy przykłady:

{% highlight csharp %}
[Test]
public async Task Handle_StartAddingComment_SendCreateBranchWithProperData()
{
    // Arrange
    var message = new StartAddingComment { CommentId = this.id };
    var saga = this.GetHandler();
    var context = this.GetContext();

    // Act
    await saga.Handle(message, context).ConfigureAwait(false);

    // Assert
    var sentMessage = this.GetSentMessage<CreateBranch>(context);
    Assert.IsNotNull(sentMessage);
    Assert.True(sentMessage.CommentId == this.id);
}
{% endhighlight %}

Rozłóżmy powyższy kod na czynniki pierwsze. Nazwa metody reprezentującej test stworzona jest wg konwencji  `X_Y_Z` gdzie:

* **X** oznacza nazwę metody, który jest testowana - `Handle`
* **Y** oznacza scenariusz, jaki jest testowany - `StartAddingComment`
* **Z** oznacza oczekiwany rezultat testu - `SendCreateBranchWithProperData`

Składając wszystko razem, można wywnioskować, że testowana jest metoda `Handle` w scenariuszu dodania komentarza, a oczekiwanym rezultatem jest wysłanie message'a `CreateBranch` z poprawnymi danymi. Ciało metody składa się z trzech sekcji:

* **Arrange** - w tej sekcji inicjowane są wszystkie elementy potrzebne do wykonania testu
* **Act** - w tej sekcji wywoływana jest metoda, której dotyczy test
* **Assert** - w tej sekcji sprawdzany jest rezultat wywołania metody z sekcji `Act`

Nie ma jednej uniwersalnej metody analizowania testu. Patrząc na całość, widać, że najprostszą sekcją jest sekcja `Act`, gdzie od razu możemy zobaczyć, co jest testowane. Potem można przejść do sekcji `Assert`, aby przeanalizować, jaki jest spodziewany rezultat, a na końcu sprawdzić, w jaki sposób tworzone są elementy wymagane do wykonania testu.

Z racji tego, że testów jest zawsze więcej niż jeden, pewne powtarzające się elementy kodu można uwspólnić tak, aby możne je było wykorzystać w pozostałych testach. W przykładzie są trzy takie elementy:

* **var saga = this.GetHandler();** - metoda `GetHandler` tworzy Message Handler'a, którego metody będą testowane
* **var context = this.GetContext();** - metoda `GetContext` tworzy context NServiceBus'a tak, aby można było uruchomić test bez konieczności odwoływania się do zewnętrznym źródeł danych np. Queue Transport'u
* **var sentMessage = this.GetSentMessage&lt;CreateBranch&gt;(context);** - metoda `GetSentMessage` zwraca message, który powinien zostać wysłany przez metodę `Handle` 

Patrząc jeszcze raz na całość widać, że oczekiwanym rezultatem jest to, aby metoda `Handle` wysłała message typu `CreateBranch` z poprawną wartością `CommentId`

Zobaczmy implementację poszczególnych pomocniczych metod.

{% highlight csharp %}
private HandlerCommentSaga GetHandler()
{
    this.configurationManager = Substitute.For<IConfigurationManager>();

    return new HandlerCommentSaga(this.configurationManager)
    {
        Data = new CommentSagaData
        {
            CommentId = this.id,
            UserName = @"test",
            UserEmail = @"test@test.com",
            UserWebsite = @"test.com",
            FileName = @"test.txt",
            Content = @"test comment",
            BranchName = @"testBranch"
        }
    };
}
{% endhighlight %}

Ponieważ message handler `HandlerCommentSaga` posiada zależność do `IConfigurationManager`, w pierwszej kolejności przy pomocy biblioteki `NSubstitute` tworzona jest fake'owa instancja  konfiguratora tak, aby nie odwoływać się do prawdziwej implementacji. Następnie zwracany jest Saga handler z przykładowymi danymi

{% highlight csharp %}
private TestableMessageHandlerContext GetContext()
{
    return new TestableMessageHandlerContext();
}
{% endhighlight %}

Jest to proste opakowanie klasy `TestableMessageHandlerContext`, która jest częścią `NServiceBus.Testing`

{% highlight csharp %}
private TSentMessage GetSentMessage<TSentMessage>(
    TestableMessageHandlerContext context) where TSentMessage : class
{
    return context.SentMessages[0].Message as TSentMessage;
}
{% endhighlight %}

Jest to ponowne opakowanie funkcjonalności udostępnianej przez `NServiceBus.Testing`, która zwraca message wysłany przez testowaną metodę. Opakowanie jest trochę bardziej złożone, ponieważ używa parametru generycznego tak, aby kod wołający metodę mógł decydować, jakiego typu message'a się spodziewa.

Zobaczmy inny, trochę bardziej złożony, scenariusz testowy.

{% highlight csharp %}
[TestCase(false, false, CommentResponseStatus.Rejected)]
[TestCase(false, true, CommentResponseStatus.Approved)]
[TestCase(true, false, CommentResponseStatus.NotAddded)]
[TestCase(true, true, CommentResponseStatus.NotAddded)]
public async Task GetCommentResponseStatus_Input_ExpectedResult(
    bool isPullRequestOpen,
    bool isPullRequestMerged,
    CommentResponseStatus expectedResult)
{
    // Arrange
    const string etagResult = "1234";
    
    Func<Task<(bool result, string etag)>> f1 = () => 
        Task.Run(() => (isPullRequestOpen, etagResult));
    
    Func<Task<bool>> f2 = () => 
        Task.Run(() => isPullRequestMerged);
    
    var handler = this.GetHandler();

    // Act
    var result = await handler.GetCommentResponseStatus(f1, f2)
        .ConfigureAwait(false);

    // Assert
    Assert.That(result.ResponseStatus, Is.EqualTo(expectedResult));
    Assert.That(result.ETag, Is.EqualTo(etagResult));
}
{% endhighlight %}

W implementacji tego testu został użyty feature biblioteki `NUnit` o nazwie [TestCase-Attribute][7]. Pozwala on w jednej testowej metodzie zakodować wiele scenariuszy. Kod dla każdego scenariusza jest taki sam, a jedyną różnicą są parametry wejściowe oraz oczekiwany rezultat testowanej metody. Inną możliwością jest napisanie czterech osobnych testów dla każdego scenariusza. Patrząc na nazwę metody można wywnioskować, że sprawdzany jest status odpowiedzi na komentarz w zależności od stanu `Pull Request'a`. Testowana metoda `GetCommentResponseStatus` przyjmuje dwa parametry:

* **Func&lt;Task&lt;(bool result, string etag)&gt;&gt;** - parametr ten jest funkcją zwracającą informację tak lub nie, na pytanie czy `Pull Request` jest nadal otwarty. Rezultat, jaki ma być zwrócony przez tę funkcję, sterowany jest poprzez parametr metody `isPullRequestOpen`.

* **Func&lt;Task&lt;bool&gt;&gt;** - parametr ten jest również funkcją, która zwraca informację o tym, czy `Pull Request` został z'merge'owany do gałęzi `master`. Rezultat, jaki ma być zwrócony przez tę funkcję, sterowany jest prze parametr `isPullRequestMerged`.

Wynik zwracany przez testowaną metodę `GetCommentResponseStatus` porównywany jest to oczekiwanego rezultatu, który również jest sterowany przez parametr wejściowy `expectedResult`. Scenariusze do przetestowania opisane są w atrybutach metody testującej.

## Integration Tests

Do pisania testów integracyjnych można wykorzystać tę samą technikę co przy pisaniu testów jednostkowych. Zasadniczą różnicą, w stosunku do testów jednostkowych jest to, że testy integracyjne odwołują się do prawdziwych źródeł danych. W projektach integracyjnych zazwyczaj występuje więcej niż jedno zewnętrzne źródło, z którym trzeba się zintegrować, co sprawia, że bardzo trudno jest przygotować tak dane oraz asercje, aby testy wykonywały się automatycznie. Przykładowo, w tym projekcie takim zewnętrznymi źródłami są GitHub oraz dostawca, za pomocą którego można wysyłać maile. Mimo tego testy integracyjne są bardzo pomocne, jeśli chcemy przetestować jeden konkretny kawałek rozwiązania, przygotowując dane oraz sprawdzając wyniki manualnie. Nie trzeba wtedy uruchamiać całości, a jedynie wybrany fragment. Oczywiście zawsze warto mieć końcowe testy integracyjne tak, aby nawet ręcznie przejść przez cały proces i sprawdzić czy wszystko działa zgodnie z oczekiwaniami. Zobaczmy przykłady:

{% highlight csharp %}
[Test]
public async Task CreatePullRequest_Execute_ProperResult()
{
    // Arrange
    const string branchName = "c-12";
    var api = this.GetGitHubApi();

    // Act
    var result = await api.CreatePullRequest(
        this.configurationManager.UserAgent,
        this.configurationManager.AuthorizationToken,
        this.configurationManager.RepositoryName,
        branchName,
        "master").ConfigureAwait(false);

    // Assert
    Assert.NotNull(result);
    Console.WriteLine(result);
}
{% endhighlight %}

W tym teście testowana jest implementacja metody `CreatePullRequest`, która tworzy `Pull Request'a` na `GitHub'e`. Metoda ta jako rezultat zwraca `URL` do utworzonego `Pull Request'a`. Rezultat ten jest sprawdzany w sekcji `Assert`. Dodatkowo zwrócony rezultat wypisywany jest na standardowe wyjście. Aby rzeczywiście przekonać się czy metoda działa zgodnie z oczekiwaniami, trzeba zalogować się na konto `GitHub'a` i sprawdzić, czy `Pull Request` faktycznie został utworzony.

Na koniec zobaczmy, jak za pomocą testu integracyjnego można zasymulować końcowego klienta, który będzie inicjował proces dodawania komentarza.

{% highlight csharp %}
[Test]
public async Task Post_ForCommentData_NoException()
{
    // Arrange
    HttpClient client = new HttpClient
    {
        BaseAddress = new Uri("http://someTestUrl/")
    };

    client.DefaultRequestHeaders
          .Accept
          .Add(new MediaTypeWithQualityHeaderValue("application/json"));

    var comment = new Comment
    {
        UserName = "testUser",
        UserEmail = "testUser@test.com",
        UserWebsite = "testUser.com",
        FileName = @"test.txt",
        Content = @"new comment",
    };

    var serializer = new JavaScriptSerializer();
    var json = serializer.Serialize(comment);
    var stringContent = new StringContent(
        json,
        Encoding.UTF8,
        "application/json");

    // Act
    HttpResponseMessage response = 
        await client.PostAsync("comment", stringContent).ConfigureAwait(false);

    // Assert
    try
    {
        response.EnsureSuccessStatusCode();
    }
    catch (Exception ex)
    {
        Console.WriteLine(ex);
        Assert.True(false);
        return;
    }

    Assert.True(true);
}
{% endhighlight %}

Tym razem zacznijmy analizę od początku. W sekcji `Arrange` tworzony jest klient `HTTP`, za pomocą którego zostanie wysłane żądanie typu `POST` oraz przykładowy komentarz (aby test zadziałał wymagane są rzeczywiste dane). W sekcji `Act` wysyłane jest żądanie utworzenia komentarza. W sekcji `Assert` sprawdzany jest wynik żądania. Każdy inny rezultat niż sukces traktowany jest jako błąd, który wypisywany jest na standardowe wyjście.

Tych parę przykładów pokazuje, w jaki sposób można pisać testy. W następnym poście zobaczymy możliwości wdrażania wytworzonych komponentów.

[1]: https://docs.microsoft.com/en-us/dotnet/csharp/whats-new/csharp-7-1 "CSharp-7.1"
[2]: https://particular.net/nservicebus "NServiceBus"
[3]: https://docs.particular.net/nservicebus/testing/ "NServiceBus.Testing"
[4]: http://nunit.org/ "NUnit"
[5]: http://nsubstitute.github.io/ "NSubstitute"
[6]: https://en.wikipedia.org/wiki/Software_testing "Software_testing"
[7]: https://github.com/nunit/docs/wiki/TestCase-Attribute "TestCase-Attribute"