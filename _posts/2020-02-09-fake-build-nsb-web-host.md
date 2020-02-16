---
layout: post
title:  "Fake it... ale nie tak jak myślisz - NServiceBus Web Host"
date:   2020-02-09
---

Posty z tej serii:

* [Fake it... ale nie tak jak myślisz - NServiceBus Web Host][17]
* [Fake it... ale nie tak jak myślisz - ASP.NET Web Host][18]
* Fake it... ale nie tak jak myślisz - NServiceBus Windows Service Host

Jakie jest twoje pierwsze skojarzenie z frazę ***Fake it*** w kontekście wytwarzania oprogramowania? U mnie jest to**/**było Fakeowanie zależności w celu [jednostkowego przetestowania][1] konkretnego kawałka funkcjonalności. Przykłady zależności? Warstwa dostępu do bazy danych, zależność do zewnętrznego serwisu lub do systemu plików. W skrócie, każda zależność do zasobu, który trzeba przygotować przed uruchomieniem testu. Fakeując taki zasób, symulujemy jego istnienie, wykorzystując do tego pamięć operacyjną maszyny na której uruchamiany jest test. Z takim podejściem związana jest [cała filozofia][2], która definiuje pojęcia, nazwy, sposób zastosowania, itd. W wersji uproszczonej, chodzi o możliwość jednostkowego testowania logiki biznesowej w oderwaniu od zewnętrznych zależności. A czy istnieją biblioteki, wspomagające w realizacji takiego podejścia? Tak? A jakie? A proszę bardzo:

* [NSubstitute][3]
* [FakeItEasy][4]

Od momentu, kiedy natrafiłem na narzędzie [FAKE][5] fraza ***Fake it*** nabrała dla mnie innego znaczenia, a mianowicie, przygotowanie napisanego kodu do wdrożenia oraz jego wdrożenie:

***FAKE - A DSL for build tasks and more The power of F# - anywhere - anytime*** ([https://fake.build/][5])

**FAKE** pozwala pisać skrypty wdrożeniowe w języku **F#**. Samo narzędzie również napisane jest w języku **F#**.

Artykuł ten jest pierwszym z serii, w której pokażę Ci, w jaki sposób wykorzystałem **FAKEa** do buildowania oraz wdrażania [systemu komentarzy][6] używanego na niniejszym blogu. Do tej pory używałem skryptów napisanych w **PowerShellu**. O szczegółach implementacji możesz przeczytać w [jednym z moich poprzednich artykułów][7].

Zacznijmy od pewnej nowości. Właścicielem agregatu **CommentPolicy** jest **Endpoint** o nazwie **BlogComments**. Framework **NServiceBus** umożliwia hostowanie **Endpointów** [na wiele sposobów][8]. Jednym z nich jest możliwość wykorzystania któregoś z [frameworków webowych][9]. Dzięki temu możemy wysyłać wiadomości z aplikacji webowej za pomocą **Endpointa** typu **Send-only**. Jeśli używamy technologii kolejkowej opartej o model centralnego [Message Brokera][10], to nic nie stoi na przeszkodzie, aby w procesie aplikacji webowej hostować pełny **Endpoint**, który oprócz wysyłania wiadomości potrafi je również przetwarzać. W wersji [2.1.0-beta.1][11] systemu komentarzy na blogu przeszedłem z hostingu **Windows Service** na hosting **Web Application**.

W dalszej części artykułu przejdziemy przez funkcjonalności wdrażania aplikacji webowej, której jedynym zadaniem jest hostowanie **Endpointa NServiceBus**.

### Design

Wdrożenie nowej wersji **Endpointa** składa się z minimum czterech operacji:

1. Zatrzymanie uruchomionego **Endpointa**
2. Usunięcie artefaktów z aktualnymi funkcjonalnościami
3. Wgranie artefaktów z nowymi funkcjonalnościami
4. Wystartowanie **Endpointa**

Powyższe kroki pokrywają się 1 do 1 z krokami wdrażania nowych wersji hostowanych jako **Windows Service**, w których mechanizm wdrażający ma uprawnienia do zatrzymywania oraz uruchamiania usług Windows. Jeśli mechanizm nie ma takich uprawnień to hmm...nie spotkałem się z tego typu rozwiązaniami.

Jeśli chodzi o aplikacje webowe, mając odpowiednie uprawnienia na serwerze webowym, możemy wykonać takie same kroki. Jedna ważna uwaga. Pamiętaj, że wdrażamy **Endpoint NServiceBusa**, co oznacza, że używamy mechanizmu kolejkowego, co z kolei oznacza, że zatrzymanie aplikacji webowej `nie wstrzymuje` użytkowników przed dalszym korzystaniem z systemu. Wszystkie wygenerowane dane w trakcie wdrażania nowej wersji dodawane są do kolejki i zostaną przetworzone, jak tylko pojawi się nowa wersja aplikacji. Jest to odwrotne zachowanie w stosunku do aplikacji webowych przetwarzających żądania **HTTP**, gdzie zatrzymanie aplikacji powoduje `niemożliwość` korzystania z systemu. W takich przypadkach można np. stosować podejście **Blue-Green-Deployment**. Więcej o takim podejściu przeczytasz [we wcześniej wspomnianym artykule][7].

No dobrze, a co jeśli mechanizm wdrażający nie ma uprawnień zatrzymywania oraz startowania aplikacji na serwerze webowym? W takim przypadku trzeba przedefiniować pojęcia **Startu** i **Stopu**. Wystartować aplikację możemy poprzez wywołanie dowolnego żądania **HTTP**. Jeśli używamy np. frameworka [ASP.NET][12] to wystarczy, że mamy jeden **Controller** z jedną metodą typu **GET**, którą zawołamy w ostatnim kroku wdrożeniowym. Sposób zatrzymywania aplikacji zależny jest od dostawcy. Może to być np. wgranie na serwer pliku z odpowiednim rozszerzeniem, a następnie wywołanie żądania **HTTP**.

Do powyższego schematu możemy dodawać elementy opcjonalne np. backup aktualnej wersji, wysyłanie powiadomień o pojawieniu się nowej wersji itp. Schemat, którego ja używam, przy wdrażaniu nowej wersji systemu komentarzy na blogu zawiera kroki:

1. Zatrzymać uruchomiony **Endpoint**
2. Zrobić backup aktualnej wersji
3. Usunąć artefakty z aktualnymi funkcjonalnościami
4. Wgrać artefakty z nowymi funkcjonalnościami
5. Wystartować **Endpoint**

### Develop

Pisząc w języku **F#** mamy możliwość uruchamiania kodu w tzw. [trybie interaktywnym][13]. Oznacza to, że nie musimy tworzyć projektów ***.csproj**. W **Visual Studio** kod możemy pisać oraz uruchamiać w oknie **F# Interactive**. Możemy też dodać implementację do pliku z rozszerzeniem **.fsx**, a następnie uruchomić program z wiersza poleceń za pomocą narzędzia **fsi.exe**.

**FAKE** umożliwia uruchamianie kodu napisanego w plikach **.fsx**. Zobaczmy, w jaki sposób możemy zakodować prototyp skryptu wdrożeniowego np. w pliku o nazwie **host_ftp_deploy.fsx**:

{% highlight fsharp %}
#r "paket:
nuget Fake.Core.Target //"
#load "./.fake/host_ftp_deploy.fsx/intellisense.fsx"

open Fake.Core

// Targets
Target.create "Stop Endpoint" (fun _ ->
  Trace.trace "..."
)

Target.create "Backup Endpoint" (fun _ ->
  Trace.trace "..."
)

Target.create "Deploy Endpoint" (fun _ ->
  Trace.trace "..."
)

open Fake.Core.TargetOperators

// Dependencies
"Stop Endpoint"
  ==> "Backup Endpoint"
  ==> "Deploy Endpoint"

// Start build
Target.runOrDefaultWithArguments "Deploy Endpoint"

{% endhighlight %}

Kroki do uruchomienia definiuje się w tzw. **Targetach**. Następnie za pomocą operatora `==>` definiuje się kolejność wykonywania poszczególnych Targetów. Ostatnim krokiem jest uruchomienie Targetu rozpoczynającego proces wdrażania. W naszym przykładzie uruchamiamy Target `Deploy Endpoint`. **FAKE** rozpozna zależność do Targetu `Backup Endpoint`, a następnie do Targetu `Stop Endpoint`, co spowoduje uruchamianie Targetów w kolejności:

* Stop Endpoint
* Backup Endpoint
* Deploy Endpoint

Po uruchomieniu skryptu dzieją się ciekawe rzeczy. Kod napisany jest w języku **F#**, także mamy pełne wsparcie kompilatora! Jeśli implementacja jest niepoprawna, to nic się nie uruchomi. **FAKE** zwróci błąd kompilacji. Przykładowo, jeśli zastąpisz, któreś z wywołań `Trace.trace` na `Trace.trac`, to uruchamiając skrypt, otrzymasz komunikat:

{% highlight text %}
Script is not valid: C:\...\host_ftp_deploy.fsx (9,8)-(9,12):
Error FS0039: The value, constructor, namespace or type 'trac' is not defined.
Maybe you want one of the following:
           trace
           tracef
           tracefn
           traceTag
           traceFAKE
{% endhighlight %}

Drugą pomocną rzeczą jest pełne wsparcie **Intellisense**! Otwierając skrypt w **Visual Studio** zobaczysz kolorowanie składki, podpowiedzi dostępnych funkcji, błędy kompilacji itd.

Trzecim elementem jest wsparcie dla [Nugeta][16]. Pierwsze uruchomienie skryptu spowoduje pobranie zależności zdefiniowanych na początku pliku.

Moją ulubioną cechą w programowaniu funkcyjnym jest możliwość dzielenia kodu na mniejsze kawałki, poprzez definiowanie kolejnych funkcji. W programowaniu obiektowym tymi kawałkami są klasy, natomiast mam takie odczucie, że proces decyzyjny w przypadku funkcji jest prostszy niż w przypadku klas. Funkcje mogą być zarówno proste, jak i złożone. Sama funkcja jest oddzielnym elementem. W przypadku klas trzeba się trochę bardziej nagimnastykować, aby dobrze zdefiniować jej dobrą odpowiedzialność, dobrać odpowiednie pola, metody, itp. W przypadku **FAKEa** możemy określić, że pojedynczą odpowiedzialnością jest sam skrypt wdrożeniowy. W naszym przypadku jest to wdrożenie **Endpointa** hostowanego w aplikacji webowej. Poszczególne **Targety** są funkcjami, które wyznaczają kroki wdrożeniowe. Każdy krok może być w całości zakodowany w **Targecie** lub podzielony na mniejsze elementy - funkcje.

Przejdźmy teraz do szczegółów. Skrypt powinien być uniwersalny tak, aby można go było wykorzystać do wdrażania na różne środowiska. Pierwszym krokiem jest wydzielenie kodu pobierającego parametry wdrożeniowe:

{% highlight fsharp %}
let winSCPExecutablePathParamName = "winSCPExecutablePath"
let ftpHostNameParamName = "ftpHostName"
let ftpUserNameParamName = "ftpUserName"
let ftpPasswordParamName = "ftpPassword"
// idt.
{% endhighlight %}

**FAKE** posiada wbudowane moduły oraz funkcje, których możemy używać. Jedna z nich umożliwia odczytanie parametrów wejściowych skryptu wewnątrz **Targetu** - `Environment.environVarOrFail`. Opakujmy wbudowaną funkcję naszą funkcją, której będziemy używać we wszystkich **Targetach**:

{% highlight fsharp %}
let retrieveParam paramName =
    Environment.environVarOrFail paramName
{% endhighlight %}

**FAKE** w piątej wersji nie posiada modułu pozwalającego wykonywać operacje na protokole **FTP**. Z tego względu musimy taką funkcjonalność, zakodować sami używając np. [WinSCP][14]. Trzy podstawowe kroki wykonania żądania **FTP** to:

1. Otwórz sesję
2. Wykonaj operację
3. Zamknij sesję

Idealna funkcjonalność do zamknięcia w osobnej funkcji:

{% highlight fsharp %}
let makeFtpAction action =
    let winSCPExecutablePath = retrieveParam winSCPExecutablePathParamName
    let ftpHostName = retrieveParam ftpHostNameParamName
    let ftpUserName = retrieveParam ftpUserNameParamName
    let ftpPassword = retrieveParam ftpPasswordParamName
    let ftpTlsHostCertificateFingerprint = retrieveParam ftpTlsHostCertificateFingerprintParamName
    
    let sessionOptions = new SessionOptions()
    sessionOptions.Protocol <- Protocol.Ftp
    sessionOptions.FtpSecure <- FtpSecure.Explicit
    sessionOptions.TlsHostCertificateFingerprint <- ftpTlsHostCertificateFingerprint
    sessionOptions.HostName <- ftpHostName
    sessionOptions.UserName <- ftpUserName
    sessionOptions.Password <- ftpPassword

    use session = new Session()
    session.ExecutablePath <- winSCPExecutablePath
    session.Open(sessionOptions)
    action session
{% endhighlight %}

A teraz dwa magiczne elementy języka **F#**:

1. Słowo kluczowe `use` użyte przy tworzeniu obiektu `Session` sprawi, że po zakończeniu cyklu życia obiektu automatycznie zostanie zawołana metoda `Dispose`.

2. Parametr `action` funkcji `makeFtpAction` jest również funkcją, która jako parametr bierze obiekt `Session` i zwraca generyczną wartość. Oznacza to, że do funkcji `makeFtpAction` możemy przekazać dowolną operację zgodną z sygnaturą funkcji `action`.

Zobaczmy, jak można wykorzystać funkcję `makeFtpAction` w kroku zatrzymywania **Endpointa**:

{% highlight fsharp %}
Target.create "Stop Endpoint" (fun _ ->
    let ftpEndpointPath = retrieveParam ftpEndpointPathParamName
    let ftpOfflineHtm = retrieveParam ftpOfflineHtmParamName
    let ftpOnlineHtm = retrieveParam ftpOnlineHtmParamName
    let endpointUrl = retrieveParam endpointUrlParamName
    
    Trace.trace ("-> Endpoint " + ftpEndpointPath)

    let offline = sprintf @"%s/%s" ftpEndpointPath ftpOfflineHtm
    let online = sprintf @"%s/%s" ftpEndpointPath ftpOnlineHtm

    let getEndpointState =
        let isDirectoryEmpty = makeFtpAction (fun ftp -> ftp.EnumerateRemoteFiles(ftpEndpointPath, null, EnumerationOptions.None) |> Seq.isEmpty)
        match isDirectoryEmpty with
        | true -> NotExists
        | false ->
            let isStopped = makeFtpAction (fun ftp -> ftp.FileExists(online))
            match isStopped with
            | true -> Stopped
            | false -> Running

    let stopEndpoint =
        makeFtpAction (fun ftp -> ftp.MoveFile(offline, online))
        try
            Trace.trace ("-> Call URL " + endpointUrl)
            Http.get "" "" endpointUrl |> ignore
        with
        | :? System.Net.Http.HttpRequestException as ex ->
            Trace.trace ex.Message

            let isStatusCorrect = ex.Message.Contains("503");
            match isStatusCorrect with
            | true -> ()
            | false -> reraise()

    match getEndpointState with
    | NotExists -> Trace.trace (sprintf "-> Endpoint %s is not exists yet." ftpEndpointPath)
    | Stopped -> Trace.trace (sprintf "-> Endpoint %s is already stopped." ftpEndpointPath)
    | Running ->
        Trace.trace ("-> Stop Endpoint " + ftpEndpointPath)
        stopEndpoint
        Trace.trace (sprintf "-> Endpoint %s stopped successfully." ftpEndpointPath)
)
{% endhighlight %}

Przejdźmy przez powyższy kod, zaczynając analizę od końca. Najpierw sprawdzamy stan **Endpointa** poprzez wywołanie zakodowanej lokalnej funkcji `getEndpointState`. Jeśli **Endpoint** nie istnieje lub jest już zatrzymany, to nie robimy nic. Jeśli jest uruchomiony, to wywołujemy zakodowaną lokalną funkcję `stopEndpoint`, która zatrzyma **Endpoint**. W lokalnych funkcjach używany wcześniej zdefiniowanej funkcji `makeFtpAction`. Na przykład:

* `makeFtpAction (fun ftp -> ftp.EnumerateRemoteFiles(ftpEndpointPath, null, EnumerationOptions.None`
    * pobranie listy plików ze zdalnego zasobu
* `makeFtpAction (fun ftp -> ftp.FileExists(online))`
    * sprawdzenie, czy plik istnieje na zdalnym zasobie
* `makeFtpAction (fun ftp -> ftp.MoveFile(offline, online))`
    * zmiana nazwy pliku na zdalnym zasobie

Do podejmowania decyzji o kolejnych krokach programu, bazujących na wartościach zwracanych przez funkcje, używamy konstrukcji języka **F#** zwanej **Pattern Matching**.

A jak może wyglądać implementacja backupu aktualnej wersji?

{% highlight fsharp %}
Target.create "Backup Endpoint" (fun _ ->
    let localEndpointBackupPath = retrieveParam localEndpointBackupPathParameName
    let ftpEndpointPath = retrieveParam ftpEndpointPathParamName
    let ftpEndpointBackupPath = retrieveParam ftpEndpointBackupPathParamName

    Trace.trace ("-> Endpoint " + ftpEndpointPath)

    Trace.trace ("-> Clean " + localEndpointBackupPath)
    Shell.cleanDir localEndpointBackupPath

    Trace.trace ("-> Download files from " + ftpEndpointPath)
    makeFtpAction (fun ftp -> ftp.GetFiles(ftpEndpointPath, localEndpointBackupPath).Check())

    Trace.trace ("-> Remove files from " + ftpEndpointBackupPath)
    makeFtpAction (fun ftp -> ftp.RemoveFiles(ftpEndpointBackupPath + "/*").Check())

    Trace.trace ("-> Upload files to " + ftpEndpointBackupPath)
    makeFtpAction (fun ftp -> ftp.PutFiles(localEndpointBackupPath, ftpEndpointBackupPath).Check())

    Trace.trace (sprintf "-> Endpoint %s backuped successfully." ftpEndpointPath)
)
{% endhighlight %}

Funkcjonalność zakodowana jest wprost, zgodnie z algorytmem:

1. Wyczyść lokalny zasób
2. Pobierz do lokalnego zasobu aktualną wersję z zasobu zdalnego
3. Wyczyść zdalny zasób przeznaczony na backup
4. Wgraj na zdalny zasób wersję z zasobu lokalnego

Do operacji na zdalnym zasobie ponownie używamy funkcji `makeFtpAction`

Ostatnim krokiem w procesie Deploy'u jest wgranie nowej wersji na zdalny zasób:

{% highlight fsharp %}
Target.create "Deploy Endpoint" (fun _ ->
    let ftpEndpointPath = retrieveParam ftpEndpointPathParamName
    let deployArtifactsPath = retrieveParam deployArtifactsPathParamName
    let buildArtifactsPath = retrieveParam buildArtifactsPathParamName
    let settingsPath = retrieveParam settingsPathParamName
    let nservicebusPath = retrieveParam nservicebusPathParamName
    let endpointUrl = retrieveParam endpointUrlParamName

    let removeRemoteItem item =
        let isItemExists = makeFtpAction (fun ftp -> ftp.FileExists(item))
        if isItemExists then makeFtpAction (fun ftp -> ftp.RemoveFiles(item).Check())
    
    Trace.trace (sprintf "-> Endpoint %s." ftpEndpointPath)

    Trace.trace ("-> Clean " + deployArtifactsPath)
    Shell.cleanDir deployArtifactsPath

    Trace.trace ("-> Copy " + buildArtifactsPath)
    Shell.copyDir deployArtifactsPath buildArtifactsPath FileFilter.allFiles

    Trace.trace ("-> Copy " + settingsPath)
    Shell.copyDir deployArtifactsPath settingsPath FileFilter.allFiles

    Trace.trace ("-> Copy " + nservicebusPath)
    Shell.copyDir deployArtifactsPath nservicebusPath FileFilter.allFiles

    Trace.trace ("-> Clean " + ftpEndpointPath)
    // protects against starting Endpoint after removing offline htm
    removeRemoteItem(ftpEndpointPath + "/web.config")
    makeFtpAction (fun ftp -> ftp.RemoveFiles(ftpEndpointPath + "/*").Check())

    Trace.trace ("-> Upload files to  " + ftpEndpointPath)
    makeFtpAction (fun ftp -> ftp.PutFiles(deployArtifactsPath, ftpEndpointPath).Check())

    Trace.trace ("-> Call URL " + endpointUrl)
    Http.get "" "" endpointUrl |> ignore

    Trace.trace (sprintf "-> Endpoint %s deployed successfully." ftpEndpointPath)
)
{% endhighlight %}

Ponownie kod odzwierciedla poszczególne kroki algorytmu:

1. Wyczyszczenie lokalnego zasobu
2. Przegranie do lokalnego zasobu skompilowanych artefaktów wraz z plikami konfiguracji
3. Wyczyszczenie zasobu zdalnego
4. Wgranie nowej wersji z zasobu lokalnego na zasób zdalny
5. Wystartowanie **Endpointa** poprzez wywołanie żądania **HTTP**

Jeśli w którymkolwiek **Targecie** wystąpi wyjątek, którego nie obsłużyliśmy, dostaniemy pełną informację w kliencie uruchamiającym skrypt np. oknie konsoli.

Implementację całej funkcjonalności możesz podejrzeć na moim [GitHubie][15].

### Test & Deploy

Skrypty możemy uruchamiać poleceniem `fake run nazwa_pliku.fsx`. Do przekazywania parametrów, które można odczytać w **Targetach** służy parametr `-e`. Przykład wywołania:

`fake run host_ftp_deploy.fsx -e p1 -e p2`

W przypadku gdy liczba parametrów jest spora, możemy skorzystać z narzędzia **PowerShell** i napisać skrypt **.ps1**, który uruchomi skrypt napisany w **FAKE**:

{% highlight powershell %}
#fake build parameters
$winSCPExecutablePath = "winSCPExecutablePath=C:\deploy\blog-comments\winscp\WinSCPnet.dll"
$ftpHostName = "ftpHostName=[host_name_or_host_ip_address]"
$ftpUserName = "ftpUserName=[ftp_user_name]"
$ftpPassword = "ftpPassword=[ftp_user_password]"
#idt.

#execute script
fake run host_ftp_deploy.fsx -e $winSCPExecutablePath -e $ftpHostName -e $ftpUserName -e $ftpPassword itd.
{% endhighlight %}

Konwencja nazewnictwa parametrów rozpoznawanych przez **FAKE** to `key=value`.

Dzięki parametrom możemy użyć tego samego skryptu przy wdrożeniach na różne środowiska, zarówno testowe, jak i produkcyjne. Całość sprowadza się do utworzenia osobnego skryptu **PowerShell** z odpowiednimi parametrami dla każdego środowiska. Ja posiadam dwa takie skrypty:

* **run_deploy_test.ps1**
* **run_deploy_production.ps1**

Kiedy natrafiłem na **FAKEa**, od razu zainteresowałem się jego możliwościami. Pisanie kodu dla funkcjonalności oraz kodu wdrażającego w tym samym języku z użyciem tych samych narzędzi developerskich mocno upraszcza cały proces wytwarzania oprogramowania. Podejście funkcyjne, kompilacja kodu, Intellisense, obsługa **Nugeta** itp. znacząco zwiększa komfort pracy nad skryptem. Dodatkowym bonusem jest frajda z pisania kodu w **F#**, a następnie obserwowanie jak napisany kod działa i robi to, co ma robić :)

W następnym artykule przejdziemy przez funkcjonalność wdrażania aplikacji webowej obsługującej żądania **HTTP**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: https://en.wikipedia.org/wiki/Unit_testing "Unit Testing"
[2]: http://xunitpatterns.com/Mocks,%20Fakes,%20Stubs%20and%20Dummies.html "Mocks Fakes Stubs Dummies"
[3]: https://nsubstitute.github.io/ "NSubstitute"
[4]: https://fakeiteasy.github.io/ "FakeItEasy"
[5]: https://fake.build/ "FAKE Build"
[6]: https://github.com/mikedevbo/blog-comments "BlogComments"
[7]: {{ site.url }}{% link _posts/2018-08-19-blog_comments_IV_Deploy.md %}
[8]: https://docs.particular.net/nservicebus/hosting/ "NServiceBus Hosting"
[9]: https://docs.particular.net/nservicebus/hosting/web-application "NServiceBus Hosting Web"
[10]: https://docs.particular.net/transports/selecting "NServiceBus Selecting Transport"
[11]: https://github.com/mikedevbo/blog-comments/releases/tag/2.1.0-beta.1 "BlogComments Beta"
[12]: https://dotnet.microsoft.com/apps/aspnet "ASP.NET"
[13]: https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/ "F# Interactive"
[14]: https://winscp.net/eng/docs/library "WinSCP"
[15]: https://github.com/mikedevbo/blog-comments/tree/master/src/Deployment "BlogComments Deployment"
[16]: https://www.nuget.org/ "Nuget"
[17]: {{ site.url }}{% link _posts/2020-02-09-fake-build-nsb-web-host.md %}
[18]: {{ site.url }}{% link _posts/2020-02-16-fake-build-web-host.md %}