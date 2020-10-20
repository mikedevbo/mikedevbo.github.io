---
layout: post
title:  "ADSD - NServiceBus - C# - F# - let's dance - Develop"
description: W ostatniej części praktycznego przewodnika pokazującego, w jaki sposób można połączyć język **F#** z frameworkiem **NServiceBus** zdefiniowałem pojedynczą odpowiedzialność...
date:   2020-10-20
---

Posty z tej serii:

* [ADSD - NServiceBus - C# - F# - let's dance - Design][2]
* [ADSD - NServiceBus - C# - F# - let's dance - Develop][17]

W [ostatniej części][1] praktycznego przewodnika pokazującego, w jaki sposób można połączyć język **F#** z frameworkiem **NServiceBus** zdefiniowałem pojedynczą odpowiedzialność:

* C# do Messagingu i Queueingu
* F# do pozostałych elementów

Z taką koncepcją rozpoczynałem [przebudowę systemu komentarzy][2], którego używam na tym blogu. W ten sposób po stronie **C#** zostały:

* konfiguracja Endpointa NServiceBus
* implementacja komponentów NServiceBus
* hosting Endpointa NServiceBus
* implementacja komponentu Webowego
* hosting komponentu Webowego
* część testów integracyjnych

Na stronę **F#** przeszły:

* implementacja GitHub API
* definicje kontraktów wiadomości NServiceBus
* logika systemu
* część testów integracyjnych
* wszystkie testy jednostkowe

Przejdźmy przez realizację wybranych elementów.

## F# - NServiceBus - kontrakty

Policy **Comment Registration** inicjalizowane jest przez wiadomość typu **Command** o nazwie **RegisterComment**. W **NServiceBus** wiadomość definiuje się poprzez klasę. Język **F#** umożliwia definiowanie klas, które są rozpoznawane przez język **C#**. Dzięki temu kontrakty mogą być definiowane w pierwszym języku, a używane w drugim języku.

Definicja **RegisterComment** wygląda tak:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.CommentRegistration.Commands

    open System

    type RegisterComment(
                        commentId: Guid,
                        userName: string,
                        userWebsite: string,
                        userComment: string,
                        articleFileName: string,
                        commentAddedDate: DateTime
                    ) =
        member this.CommentId = commentId
        member this.UserName = userName
        member this.UserWebsite = userWebsite
        member this.UserComment = userComment
        member this.ArticleFileName = articleFileName
        member this.CommentAddedDate = commentAddedDate
{% endhighlight %}

<br />

**Comment Registration** kończąc realizację swojej odpowiedzialności, publikuje wiadomość typu **Event** o nazwie **CommentRegistered**:

{% highlight fsharp %}
namespace Bc.Contracts.Externals.Endpoint.CommentRegistration.Events

    open System

    type CommentRegistered(commentId: Guid, commentUri: string) =
        member this.CommentId = commentId
        member this.CommentUri = commentUri
{% endhighlight %}

<br />

A tak wyglądają kontrakty dla **Comment Answer Notification**:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Commands

    open System

    type RegisterCommentNotification(commentId: Guid, userEmail: string, articleFileName: string) =
        member this.CommentId = commentId
        member this.UserEmail = userEmail
        member this.ArticleFileName = articleFileName
        
    type NotifyAboutCommentAnswer(commentId: Guid, isApproved: bool) =
        member this.CommentId = commentId
        member this.IsApproved = isApproved
{% endhighlight %}

Dlaczego dwie wiadomości typu **Command** skoro [diagram][2] pokazuje jedno połączenie? Wrócimy do tego w późniejszej części artykułu.

<br />

Zobaczmy, w jaki sposób można definiować wiadomość typu **Message**. Za przykład weźmy Policy **GitHub Pull Request Creation**:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Messages

    open System

    type RequestCreateGitHubPullRequest(
                                           commentId: Guid,
                                           fileName: string,
                                           content: string,
                                           addedDate: DateTime
                                       ) =
        member this.CommentId = commentId
        member this.FileName = fileName
        member this.Content = content
        member this.AddedDate = addedDate
{% endhighlight %}

<br />

Definicje kontraktów dla pozostałych Policies wyglądają analogicznie. Przyglądnij się końcówką nazw **namespace** dla każdego z typów wiadomości. Jest to zdefiniowana [konwencja][16], która instruuje **NServiceBusa** o typie wiadomości do przetworzenia.

## F# - Logika - interfejsy

Interfejs jest łącznikiem pomiędzy kodem napisanym w **C#** a kodem napisanym w **F#**. Interfejsy mogą być zdefiniowane w którymkolwiek języku. Postanowiłem umieścić je po stronie **F#**.

Interfejs logiki dla **Comment Registration** wygląda tak:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.CommentRegistration.Logic

    open System

    type ICommentRegistrationPolicyLogic =
        abstract member FormatUserName: userName: string -> userWebsite: string -> string
        abstract member FormatUserComment: userName: string -> userComment: string -> commentAddedDate: DateTime -> string
{% endhighlight %}

Metoda `FormatUserName` formatuje nazwę użytkownika. Metoda `FormatUserComment` formatuje przesłany komentarz. W takiej postaci dane pojawiają się na GitHubie.

<br />

Zobaczmy definicję interfejsu logiki dla **Comment Answer Notification**:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Logic

    type ICommentAnswerNotificationPolicyLogic =
        abstract member From: string
        abstract member Subject: string
        abstract member GetBody: articleFileName: string -> string
        abstract member IsSendNotification: isCommentApproved: bool -> userEmail: string -> bool
{% endhighlight %}        

Główną odpowiedzialnością logiki jest zwrócenie informacji o tym, czy powiadomienie powinno być wysłane - metoda `IsSendNotification`. Pozostałe elementy zwracają dane potrzebne do wysłania komunikatu.

<br />

Na koniec przyjrzyjmy się definicji interfejsu logiki dla **GitHub Pull Request Creation**:

{% highlight fsharp %}
namespace Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Logic

    open System
    open System.Threading.Tasks

    type IGitHubPullRequestCreationPolicyLogic =
        abstract member CreateBranch: creationDate: DateTime -> Task<string>
        abstract member UpdateFile: branchName: string -> fileName: string -> content: string -> Task
        abstract member CreatePullRequest: branchName: string -> Task<string>
{% endhighlight %}

Celem logiki jest zgłoszenie **Pull Requesta** na **GitHubie**. Całość podzielona jest na trzy osobne kroki:

* utworzenie brancha
* aktualizacja pliku
* utworzenie Pull Requesta.

<br />

Definicje interfejsów dla pozostałych Policies wyglądają analogicznie. Ponownie, przyjrzyj się końcówce w nazwach **namespace**. W ten sposób pogrupowane są kontrakty związane z interfejsami logiki.

## F# - Logika - implementacja

Implementacja interfejsów wygląda tak samo, jak w języku **C#**. Trzeba utworzyć klasę, która dziedziczy po interfejsie, a następnie zaimplementować funkcjonalność zgodnie z jego sygnaturą. Zobaczmy szczegóły na trzech, znanych nam już przykładach. Na początek logika dla **Comment Registration**:

{% highlight fsharp %}
namespace Bc.Logic.Endpoint.CommentRegistration

open System
open Bc.Contracts.Internals.Endpoint.CommentRegistration.Logic

type CommentRegistrationPolicyLogic() =
    interface ICommentRegistrationPolicyLogic with
        member this.FormatUserName userName userWebsite =
            match userWebsite with
            | null -> sprintf "**%s**" userName
            | "" -> sprintf "**%s**" userName
            | _ -> sprintf "[%s](%s)" userName userWebsite


        member this.FormatUserComment userName userComment commentAddedDate =
            sprintf "begin-%s-%s-%s UTC %s" userName userComment (commentAddedDate.ToString("yyyy-MM-dd HH:mm")) Environment.NewLine
{% endhighlight %}

**ICommentRegistrationPolicyLogic** jest pomostem pomiędzy językiem **C#**, a językiem **F#**, dlatego implementacja **CommentRegistrationPolicyLogic** musi uwzględniać specyfikę pierwszego z nich, jak np. obsługa wartości **null**.

<br />

Logika dla **Comment Answer Notification** wygląda tak:

{% highlight fsharp %}
namespace Bc.Logic.Endpoint.CommentAnswerNotification

open System
open System.Linq
open System.Configuration
open Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Logic

module private ConfigurationProvider =
    let SmtpFrom = ConfigurationManager.AppSettings.["SmtpFrom"]

    let BlogDomainName = ConfigurationManager.AppSettings.["BlogDomainName"]

module GetBody =
    let execute (fileName: String) (blogDomainName: string) =

        // ...

        result

type CommentAnswerNotificationPolicyLogic() =
    interface ICommentAnswerNotificationPolicyLogic with
        member this.From = ConfigurationProvider.SmtpFrom

        ////TODO: move to resource file
        member this.Subject = "Dodano odpowiedź do komentarza."
        member this.GetBody fileName = GetBody.execute fileName ConfigurationProvider.BlogDomainName
        member this.IsSendNotification isCommentApproved userEmail =
            not(System.String.IsNullOrEmpty(userEmail)) && isCommentApproved
{% endhighlight %}

Implementując logikę, możemy wykorzystywać dobrodziejstwa języka **F#** jak np. dzielenie kodu na moduły. W powyższym przykładzie pominąłem szczegóły implementacji metody `GetBody.execute`.

<br />

Na koniec zobaczmy kod dla **GitHub Pull Request Creation**:

{% highlight fsharp %}
namespace Bc.Logic.Endpoint.GitHubPullRequestCreation

open System.Threading.Tasks
open Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Logic
open GitHubApi

type GitHubPullRequestCreationPolicyLogic() =
        interface IGitHubPullRequestCreationPolicyLogic with
            member this.CreateBranch creationDate =
                async {
                    let branchName = sprintf "c-%s" (creationDate.ToString("yyyy-MM-dd-HH-mm-ss-fff"))
                    do! GitHubApi.CreateRepositoryBranch.execute
                            GitHubConfigurationProvider.userAgent
                            GitHubConfigurationProvider.authorizationToken
                            GitHubConfigurationProvider.repositoryName
                            GitHubConfigurationProvider.masterBranchName
                            branchName
                    return branchName
                } |> Async.StartAsTask

            member this.UpdateFile branchName fileName content =
                async {
                   // ...
                } |> Async.StartAsTask :> Task

            member this.CreatePullRequest branchName  =
                async {
                    // ...
                    return pullRequestUri
                } |> Async.StartAsTask
{% endhighlight %}

Ponownie, dla lepszej czytelności, pominąłem kod dla metod `this.UpdateFile` oraz `this.CreatePullRequest`. Ich implementacja wygląda analogicznie jak dla metody `this.CreateBranch`, która konstruuje nazwę, a następnie tworzy brancha za pomocą metody `GitHubApi.CreateRepositoryBranch.execute`.

## F# - Testy

Zanim przejdziemy do szczegółów implementacji w języku **C#**, zobaczmy, jak może wyglądać kod testu jednostkowego:

{% highlight fsharp %}
module Bc.Logic.Endpoint.Tests.CommentRegistration

open System
open Bc.Contracts.Internals.Endpoint.CommentRegistration.Logic
open Bc.Logic.Endpoint.CommentRegistration
open NUnit.Framework

module CommentRegistrationPolicyLogicTests =

    let getLogic () =
        CommentRegistrationPolicyLogic()

    [<TestCase("user", null, "**user**")>]
    [<TestCase("user", "", "**user**")>]
    [<TestCase("user", "webSite", "[user](webSite)")>]
    let FormatUserName_Execute_ProperResult(userName, userWebsite, expectedResult) =

        // Arrange
        let logic = getLogic () :> ICommentRegistrationPolicyLogic

        // Act
        let result = logic.FormatUserName userName userWebsite

        // Assert
        Assert.That(result, Is.EqualTo(expectedResult))
{% endhighlight %}

Zasada działania jest taka sama jak dla testów napisanych w **C#**.

## C# - NServiceBus - Konfiguracja Endpointa

Kod **F#** dołącza się do kodu **C#** w taki sam sposób, jak łączenie kodu **C#** <-> **C#**. Do projektu **.csproj** dołącza się projekt **.fsproj**. Interfejsy oraz klasy napisane w **F#** są rozpoznawane przez **C#**. Zobaczmy, jak to działa na przykładzie konfigurowania zależności w kontenerze udostępnianym przez **NServiceBus**:

{% highlight fsharp %}
// dependency injection
endpoint.RegisterComponents(reg =>
{
    if (configurationProvider.IsUseFakes)
    {
        reg.ConfigureComponent<GitHubPullRequestVerificationPolicyLogicFake>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<GitHubPullRequestCreationPolicyLogicFake>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentAnswerPolicyLogicFake>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentAnswerNotificationPolicyLogicFake>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentRegistrationPolicyLogicFake>(DependencyLifecycle.InstancePerCall);
    }
    else
    {
        reg.ConfigureComponent<GitHubPullRequestVerificationPolicyLogic>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<GitHubPullRequestCreationPolicyLogic>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentAnswerPolicyLogic>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentAnswerNotificationPolicyLogic>(DependencyLifecycle.InstancePerCall);
        reg.ConfigureComponent<CommentRegistrationPolicyLogic>(DependencyLifecycle.InstancePerCall);
    }
});
{% endhighlight %}

Zmienna `endpoint` służy do konfiguracji różnych właściwości **NServiceBusa**. Metoda `RegisterComponents` pozwala wstrzykiwać zależności, które można wykorzystywać w **Message Handlerach**. Wstrzykiwane klasy zakodowane są w języku **F#**.

## C# - NServiceBus - Policy - Saga

W [poprzedniej części][2] wspomniałem, że zaprojektowane **Policy** zgodnie z podejściem **ADSD** najlepiej implementuje się jako [Sagę][3] **NServiceBusa**. Zobaczmy więc, jak taka implementacja może wyglądać. Na początek **Comment Registration Policy**:

{% highlight fsharp %}
using System;
using System.Threading.Tasks;
using Bc.Contracts.Externals.Endpoint.CommentRegistration.Events;
using Bc.Contracts.Internals.Endpoint.CommentRegistration.Commands;
using Bc.Contracts.Internals.Endpoint.CommentRegistration.Logic;
using Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Messages;
using NServiceBus;

namespace Bc.Endpoint
{
    public class CommentRegistrationPolicy :
        Saga<CommentRegistrationPolicy.PolicyData>,
        IAmStartedByMessages<RegisterComment>,
        IHandleMessages<ResponseCreateGitHubPullRequest>
    {
        private readonly ICommentRegistrationPolicyLogic logic;

        public CommentRegistrationPolicy(ICommentRegistrationPolicyLogic logic)
        {
            this.logic = logic;
        }

        public Task Handle(RegisterComment message, IMessageHandlerContext context)
        {
            this.Data.UserName = message.UserName;
            this.Data.UserWebsite = message.UserWebsite;
            this.Data.UserComment = message.UserComment;
            this.Data.ArticleFileName = message.ArticleFileName;
            this.Data.CommentAddedDate = message.CommentAddedDate;

            var formatUserName = this.logic.FormatUserName(this.Data.UserName, this.Data.UserWebsite);
            var formatUserComment =
                this.logic.FormatUserComment(formatUserName, this.Data.UserComment, this.Data.CommentAddedDate);

            return context.Send(new RequestCreateGitHubPullRequest(
                message.CommentId,
                this.Data.ArticleFileName,
                formatUserComment,
                this.Data.CommentAddedDate));
        }

        public Task Handle(ResponseCreateGitHubPullRequest message, IMessageHandlerContext context)
        {
            this.MarkAsComplete();
            return context.Publish(new CommentRegistered(this.Data.CommentId, message.PullRequestUri));
        }

        protected override void ConfigureHowToFindSaga(SagaPropertyMapper<PolicyData> mapper)
        {
            mapper.ConfigureMapping<RegisterComment>(message => message.CommentId)
                  .ToSaga(data => data.CommentId);

            mapper.ConfigureMapping<ResponseCreateGitHubPullRequest>(message => message.CommentId)
                .ToSaga(data => data.CommentId);
        }

        public class PolicyData : ContainSagaData
        {
            public Guid CommentId { get; set; }

            public string UserName { get; set; }

            public string UserWebsite { get; set; }

            public string UserComment { get; set; }

            public string ArticleFileName { get; set; }

            public DateTime CommentAddedDate { get; set; }
        }
    }
}
{% endhighlight %}

Flow `CommentRegistrationPolicy` wygląda następująco:

* wiadomość `RegisterComment` tworzy nową instancję Sagi
* handler wiadomości wykonuje logikę biznesową, a następnie wysyła żądanie utworzenia GitHub Pull Requesta
* Saga, po otrzymaniu odpowiedzi, publikuje zdarzenie `CommentRegistered`, rozgłaszając zarejestrowanie komentarza

Interfejs logiki oraz kontrakty używane są w taki sam sposób, jakby były napisane w **C#**, mimo, że zostały zakodowane w **F#**.

<br />

A tak wygląda przetwarzanie wiadomości dla Policy **Comment Answer Notification**:

{% highlight fsharp %}
using System;
using System.Threading.Tasks;
using Bc.Contracts.Externals.Endpoint.CommentAnswer.Events;
using Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Commands;
using Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Logic;
using NServiceBus;
using NServiceBus.Mailer;

namespace Bc.Endpoint
{
    public class CommentAnswerNotificationEventSubscribingPolicy :
        IHandleMessages<CommentApproved>,
        IHandleMessages<CommentRejected>
    {
        public Task Handle(CommentApproved message, IMessageHandlerContext context)
        {
            return context.Send(new NotifyAboutCommentAnswer(message.CommentId, true));
        }

        public Task Handle(CommentRejected message, IMessageHandlerContext context)
        {
            return context.Send(new NotifyAboutCommentAnswer(message.CommentId, false));
        }
    }

    public class CommentAnswerNotificationPolicy :
        Saga<CommentAnswerNotificationPolicy.PolicyData>,
        IAmStartedByMessages<RegisterCommentNotification>,
        IAmStartedByMessages<NotifyAboutCommentAnswer>
    {
        private readonly ICommentAnswerNotificationPolicyLogic logic;

        public CommentAnswerNotificationPolicy(ICommentAnswerNotificationPolicyLogic logic)
        {
            this.logic = logic;
        }

        public Task Handle(RegisterCommentNotification message, IMessageHandlerContext context)
        {
            this.Data.UserEmail = message.UserEmail;
            this.Data.ArticleFileName = message.ArticleFileName;
            this.Data.IsNotificationRegistered = true;

            return this.SendNotification(context);
        }

        public Task Handle(NotifyAboutCommentAnswer message, IMessageHandlerContext context)
        {
            this.Data.IsCommentApproved = message.IsApproved;
            this.Data.IsNotificationReadyToSend = true;

            return this.SendNotification(context);
        }

        private Task SendNotification(IMessageHandlerContext context)
        {
            if (!this.Data.IsNotificationRegistered || !this.Data.IsNotificationReadyToSend)
            {
                return Task.CompletedTask;
            }

            this.MarkAsComplete();

            if (!this.logic.IsSendNotification(this.Data.IsCommentApproved, this.Data.UserEmail))
            {
                return Task.CompletedTask;
            }

            var mail = new Mail
            {
                From = this.logic.From,
                To = this.Data.UserEmail,
                Subject = this.logic.Subject,
                Body = this.logic.GetBody(this.Data.ArticleFileName)
            };

            return context.SendMail(mail);
        }

        protected override void ConfigureHowToFindSaga(SagaPropertyMapper<PolicyData> mapper)
        {
            mapper.ConfigureMapping<RegisterCommentNotification>(message => message.CommentId)
                  .ToSaga(data => data.CommentId);

            mapper.ConfigureMapping<NotifyAboutCommentAnswer>(message => message.CommentId)
                  .ToSaga(data => data.CommentId);
        }

        public class PolicyData : ContainSagaData
        {
            public Guid CommentId { get; set; }

            public string UserEmail { get; set; }

            public string ArticleFileName { get; set; }

            public bool IsCommentApproved { get; set; }

            public bool IsNotificationRegistered { get; set; }

            public bool IsNotificationReadyToSend { get; set; }
        }
    }
}
{% endhighlight %}

Wróćmy do tematu dwóch wiadomości typu **Command** dla powyższego Policy. Saga może być wystartowana przez jedną z wiadomości:

* `RegisterCommentNotification`
* `NotifyAboutCommentAnswer`

Pierwszą wiadomość wysyła **Comment Taking** w momencie rejestrowania komentarza. Drugą wiadomość wysyła Policy **Comment Answer Notification Event Subscribing**. Jeśli popatrzysz na diagram [z poprzedniej części][2], to nie znajdziesz tam ostatniego elementu. Dlaczego? Bardzo ważną zasadą **ADSD** jest <u>rozdzielenie</u> logicznej odpowiedzialności od fizycznej implementacji, a także fizycznego Deploymentu.

Logicznie, **Comment Answer Notification** nasłuchuje na Event od **Comment Answer**.

Fizycznie, **NServiceBus** desarilizuje pobraną wiadomość z kolejki, a następnie uruchamiana <u>wszystkie</u> Handlery, które obsługują rozpoznany typ wiadomości np. **CommentApproved**. Jeśli w jednym z nich wystąpi błąd, mechanizm [re-try][4] ponowi całe przetwarzanie, co spowoduje, że każdy z Handlerów wykona się ponownie. Jeśli nasze Handlery zostały zaprojektowane jako osobne elementy i mają się wykonywać niezależnie, to w tym, przypadku mamy zależność, której nie chcemy.

Rozwiązaniem jest minimalizowanie powstałej zależności, stąd klasa  `CommentAnswerNotificationEventSubscribingPolicy` pełni rolę pośrednika, którego jedynym zadaniem jest nasłuchiwać na odpowiednie Eventy a następnie przekierowywać dane do `CommentAnswerNotificationPolicy`.

Jeśli w projekcie pojawi się inny komponent, który również będzie nasłuchiwał na Event **CommentApproved**, jego implementacja będzie wyglądać tak samo.

Kolejną różnicą pomiędzy logicznym projektem, a fizyczną realizacją jest to, że **Comment Answer** wysyła dwa osobne zdarzenia na poinformowanie o tym, czy komentarz został zaakceptowany, czy odrzucony:

* `CommentApproved`
* `CommentRejected`

Jest to szczegół implementacyjny, którego również nie chcemy mieć na wspomnianym diagramie. Innym możliwym rozwiązaniem byłoby wysyłanie jednego typu wiadomości ze statusem odpowiedzi.

Na koniec zobaczmy implementację **GitHub Pull Request Creation**:

{% highlight fsharp %}
using System;
using System.Threading.Tasks;
using Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Logic;
using Bc.Contracts.Internals.Endpoint.GitHubPullRequestCreation.Messages;
using NServiceBus;

namespace Bc.Endpoint
{
    public class GitHubPullRequestCreationPolicy :
        Saga<GitHubPullRequestCreationPolicy.PolicyData>,
        IAmStartedByMessages<RequestCreateGitHubPullRequest>,
        IHandleMessages<ResponseCreateBranch>,
        IHandleMessages<ResponseUpdateFile>,
        IHandleMessages<ResponseCreatePullRequest>
    {
        public Task Handle(RequestCreateGitHubPullRequest message, IMessageHandlerContext context)
        {
            this.Data.FileName = message.FileName;
            this.Data.Content = message.Content;
            this.Data.AddedDate = message.AddedDate;

            return context.Send(new RequestCreateBranch(this.Data.AddedDate));
        }

        public Task Handle(ResponseCreateBranch message, IMessageHandlerContext context)
        {
            this.Data.BranchName = message.BranchName;
            return context.Send(new RequestUpdateFile(this.Data.BranchName, this.Data.FileName, this.Data.Content));
        }

        public Task Handle(ResponseUpdateFile message, IMessageHandlerContext context)
        {
            return context.Send(new RequestCreatePullRequest(this.Data.BranchName));
        }

        public Task Handle(ResponseCreatePullRequest message, IMessageHandlerContext context)
        {
            this.MarkAsComplete();
            return this.ReplyToOriginator(
                context,
                new ResponseCreateGitHubPullRequest(this.Data.CommentId, message.PullRequestUri));
        }

        protected override void ConfigureHowToFindSaga(SagaPropertyMapper<PolicyData> mapper)
        {
            mapper.ConfigureMapping<RequestCreateGitHubPullRequest>(message => message.CommentId)
                  .ToSaga(data => data.CommentId);
        }

        public class PolicyData : ContainSagaData
        {
            public Guid CommentId { get; set; }

            public string FileName { get; set; }

            public string Content { get; set; }

            public DateTime AddedDate { get; set; }

            public string BranchName { get; set; }
        }
    }

    public class GitHubPullRequestCreationPolicyHandlers :
        IHandleMessages<RequestCreateBranch>,
        IHandleMessages<RequestUpdateFile>,
        IHandleMessages<RequestCreatePullRequest>
    {
        private readonly IGitHubPullRequestCreationPolicyLogic logic;

        public GitHubPullRequestCreationPolicyHandlers(IGitHubPullRequestCreationPolicyLogic logic)
        {
            this.logic = logic;
        }

        public async Task Handle(RequestCreateBranch message, IMessageHandlerContext context)
        {
            var branchName = await this.logic.CreateBranch(message.AddedDate).ConfigureAwait(false);
            await context.Reply(new ResponseCreateBranch(branchName)).ConfigureAwait(false);
        }

        public async Task Handle(RequestUpdateFile message, IMessageHandlerContext context)
        {
            await this.logic.UpdateFile(message.BranchName, message.FileName, message.Content).ConfigureAwait(false);
            await context.Reply(new ResponseUpdateFile()).ConfigureAwait(false);
        }

        public async Task Handle(RequestCreatePullRequest message, IMessageHandlerContext context)
        {
            var pullRequestUri = await this.logic.CreatePullRequest(message.BranchName).ConfigureAwait(false);
            await context.Reply(new ResponseCreatePullRequest(pullRequestUri)).ConfigureAwait(false);
        }
    }
}
{% endhighlight %}

Myślę, że teraz już widzisz podział odpowiedzialności pomiędzy przetwarzaniem wiadomości a realizacją logiki biznesowej.

## C# - NServiceBus - IT/OPS

Zobaczmy na implementację **Comment Taking**. Jest to punkt rozdzielający dane, przetwarzane przez różne usługi:

{% highlight fsharp %}
using System.Threading.Tasks;
using Bc.Contracts.Internals.Endpoint.CommentAnswerNotification.Commands;
using Bc.Contracts.Internals.Endpoint.CommentRegistration.Commands;
using Bc.Contracts.Internals.Endpoint.CommentTaking.Commands;
using NServiceBus;

namespace Bc.Endpoint
{
    public class CommentTakingPolicy : IHandleMessages<TakeComment>
    {
        public async Task Handle(TakeComment message, IMessageHandlerContext context)
        {
            await context.Send(new RegisterComment(
                message.CommentId,
                message.UserName,
                message.UserWebsite,
                message.UserComment,
                message.ArticleFileName,
                message.CommentAddedDate)).ConfigureAwait(false);
            
            await context.Send(new RegisterCommentNotification(
                message.CommentId,
                message.UserEmail,
                message.ArticleFileName)).ConfigureAwait(false);
        }
    }
}
{% endhighlight %}

Skomplikowane? Raczej nie :) Standardowe wysłanie wiadomości typu **Command** w odpowiednie miejsca z odpowiednimi danymi. Elementem łączącym wszystkie dane jest **CommentId**.

## Pożegnanie z NancyFx

Od samego początku komponentem webowym, przyjmującym komentarz od użytkownika był komponent o nazwie **Comment Module**, zrealizowany za pomocą frameworka **NancyFX**. Framework ten [przestał być utrzymywany][5] dlatego naturalną decyzją było przejście na coś innego. Wybór nie powinien Cię zaskoczyć. Zdecydowałem się na [ASP.NET][6], tym bardziej że **NServiceBus** wspiera konfigurowanie **Endpointa** za pomocą tzw. [Generic Host][7]. Przemianowałem też nazwę komponentu na **Comment Controller**.

## Test & Deploy

W tych dwóch kwestiach w zasadzie nic się nie zmieniło. [Techniki testowania][8] pozostały te same: 

* automatyczne **testy jednostkowe**
* manualne **testy integracyjne**.

Część kodu dla testów powędrowała na stronę języka **F#**.

Jeśli chodzi o wdrażanie, to bez zarzutu sprawdza się narzędzie [Fake][9].

## Podsumowanie

Doszliśmy do końca mini serii, w której miałeś/miałaś okazję zobaczyć, w jaki sposób można zaprojektować rozwiązanie wg. idei zawartych w krusie [Advanced Distributed System Design][10] (ADSD), a także sposób, w jaki można taki projekt zrealizować używając frameworka [NServiceBus][11] oraz języków [C#][14] i [F#][15].

Całość implementacji znajdziesz na moim [GitHubie][12]. Zachowałem poprzednie rozwiązania w postaci [branchy][13], aby móc analizować zmiany, jakie zachodziły w każdym kolejnym kroku rozwoju systemu.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2019-12-22-fsharp-nservicebus-practical-guide-ending.md %}
[2]: {{ site.url }}{% link _posts/2020-09-29-adsd-nservicebus-csharp-fsharp-lets-dance-design.md %}
[3]: https://docs.particular.net/nservicebus/sagas/ "NServiceBus Saga"
[4]: https://docs.particular.net/nservicebus/recoverability/ "NServiceBus recoverability"
[5]: https://github.com/NancyFx/Nancy "NancyFx"
[6]: https://dotnet.microsoft.com/apps/aspnet "ASP.NET"
[7]: https://docs.particular.net/nservicebus/hosting/extensions-hosting "NServiceBus extensions hosting"
[8]: {{ site.url }}{% link _posts/2018-03-30-blog_comments_III_Test.md %}
[9]: {{ site.url }}{% link _posts/2020-02-09-fake-build-nsb-web-host.md %}
[10]: https://learn.particular.net/courses/adsd-online "Advanced Distributed System Design"
[11]: https://particular.net/nservicebus "NServiceBus"
[12]: https://github.com/mikedevbo/blog-comments "mikedevbo blog-comments"
[13]: https://github.com/mikedevbo/blog-comments/branches "mikedevbo blog-comments branches"
[14]: https://docs.microsoft.com/en-us/dotnet/csharp/ "C# language"
[15]: https://fsharp.org/ "F# language"
[16]: https://docs.particular.net/nservicebus/messaging/conventions "NServiceBus message conventions"
[17]: {{ site.url }}{% link _posts/2020-10-20-adsd-nservicebus-csharp-fsharp-lets-dance-develop.md %} 