---
layout: post
title:  "Fake it... ale nie tak jak myślisz - Build - Run Unit Tests - Publish"
date:   2020-03-01
---

Posty z tej serii:

* [Fake it... ale nie tak jak myślisz - NServiceBus Web Host][10]
* [Fake it... ale nie tak jak myślisz - ASP.NET Web Host][4]
* [Fake it... ale nie tak jak myślisz - NServiceBus Windows Service Host][11]
* [Fake it... ale nie tak jak myślisz - Build - Run Unit Tests - Publish][12]

Witam Cię w ostatniej części serii, w której opisuję, w jaki sposób zrealizowałem mechanizmy wdrażania nowych wersji [Systemu komentarzy na blogu][1], wykorzystując do tego narzędzie [FAKE][2]. Trzy poprzednie artykuły dotyczyły wgrywania artefaktów ze zmianami. W tym artykule przejdziemy przez funkcjonalności przygotowania kodu do wdrożenia. 

### Design

Cały proces składa się z kroków:

1. Pobranie źródeł z systemu kontroli wersji
2. Kompilacja kodu
3. Uruchomienie testów jednostkowych
4. Przygotowanie do wdrożenia Endpointa **BlogComments**, za pomocą komendy [dotnet publish][3]
5. Przygotowanie do wdrożenia komponentu **Web**, za pomocą tej samej komendy `dotnet publish`

W tym momencie natrafiamy na klasyczny element do rozstrzygnięcia - w jaki sposób podzielić kod. Co powinno być razem, a co osobno. Pisałem o tym trochę w [drugiej][4] części. Strategia, która w takich sytuacjach pomaga mi podjąć decyzję, zwłaszcza wtedy, kiedy od razu nie widać rozwiązania, polega na rozważaniu dwóch skrajnych podejść:

1. Maksymalnie wszystko uwspólnić
2. Maksymalnie wszystko rozdzielić

W ten sposób zwiększam swoją szansę na zauważenie plusów oraz minusów. Następnie bazując na tej wiedzy, podejmuję decyzję.

W pierwszym przypadku mógłbym zakodować wszystko w jednym **Targecie** w jednym pliku **.fsx**. Plus byłby taki, że miałbym wszystko w jednym miejscu i od razu wiedziałbym, w którym miejscu wprowadzać ewentualne zmiany. Minusem mogłoby być to, że z każdą zmianą musiałbym testować cały proces od początku do końca.

W drugim przypadku mógłbym umieścić każdy krok w osobnym **Targecie**, a każdy Target w osobnym pliku **.fsx**. Plusem byłoby to, że mógłbym wprowadzać zmiany niezależnie dla każdego kroku. Minus byłby taki, że miałbym kod w różnych miejscach.

Jeśli udało Ci się zrealizować ćwiczenie opisane w [drugiej][4] części, to zauważysz, że mamy tutaj taką samą sytuację, a mianowicie plusy pierwszego podejścia stają się automatycznie minusami podejścia drugiego i odwrotnie.

Jeśli żadne z podejść mi nie pasuję, to przechodzę do drugiej iteracji, patrząc, co mogłoby być razem, a co osobno itd.

Ostatecznie wyszło mi, że najbardziej pasującą opcją jest zakodowanie poniższych kroków w jednym pliku **.fsx**:

1. Target o nazwie **Compile Code**:
    * Pobranie źródeł z systemu kontroli wersji
    * Kompilacja kodu
2. Target o nazwie **Run Unit Tests**:
    * Uruchomienie testów jednostkowych

Krok przygotowania do wdrożenia za pomocą komendy `dotnet publish` pasuje, aby był w osobnym pliku **.fsx** jako jeden Target o nazwie **Publish Artifacts**, który w parametrze przyjmuję ścieżkę do projektu, który ma zostać opublikowany.

### Develop

Mając zaprojektowane rozwiązanie, możemy przejść do implementacji. Na początek kompilacja kodu oraz testy jednostkowe np. w pliku o nazwie **build.fsx**:

{% highlight fsharp %}
// Targets
Target.create "Compile Code" (fun _ ->
    let buildArtifactsWorkingDirectoryPath = retrieveParam buildArtifactsWorkingDirectoryPathParamName
    let gitRepositoryUrl = retrieveParam gitRepositoryUrlParamName
    let buildArtifactsSubDirectoryName = retrieveParam buildArtifactsSubDirectoryNameParamName
    let solutionRelativePath = retrieveParam solutionRelativePathParamName
    
    let buildArtifactsPath = buildArtifactsWorkingDirectoryPath + "\\" + buildArtifactsSubDirectoryName
    let slnPath = buildArtifactsPath + "\\" + solutionRelativePath

    Trace.trace ("-> Solution " + slnPath)
    
    Trace.trace ("-> Clean " + buildArtifactsPath)
    Shell.cleanDir buildArtifactsPath

    Trace.trace ("-> Clone " + gitRepositoryUrl)
    Repository.clone buildArtifactsWorkingDirectoryPath gitRepositoryUrl buildArtifactsSubDirectoryName
    
    Trace.trace ("-> Build " + slnPath)
    DotNet.build (fun p -> { p with Configuration = DotNet.BuildConfiguration.Release }) slnPath
    
    Trace.trace "-> Code built successfully."
)

Target.create "Run Unit Tests" (fun _ ->
    let buildArtifactsWorkingDirectoryPath = retrieveParam buildArtifactsWorkingDirectoryPathParamName
    let buildArtifactsSubDirectoryName = retrieveParam buildArtifactsSubDirectoryNameParamName
    let solutionRelativePath = retrieveParam solutionRelativePathParamName

    let buildArtifactsPath = buildArtifactsWorkingDirectoryPath + "\\" + buildArtifactsSubDirectoryName
    let slnPath = buildArtifactsPath + "\\" + solutionRelativePath

    Trace.trace ("-> Solution " + slnPath)

    DotNet.test (fun p -> 
        { p with
            NoBuild = true
            Configuration = DotNet.BuildConfiguration.Release
        }) slnPath

    Trace.trace "-> Unit Tests run successfully."
)

//// Dependencies
open Fake.Core.TargetOperators

"Compile Code"
    ==> "Run Unit Tests"

// start build
Target.runOrDefaultWithArguments "Run Unit Tests"
{% endhighlight %}

Standardowo w powyższym kodzie pominąłem elementy opisane w poprzednich częściach tej serii. Całość implementacji możesz zobaczyć na moim [GitHubie][5].

Elementem startowym jest uruchomienie testów jednostkowych, które zależne są od sukcesu kompilacji kodu. Kod pobierany jest z [Gita][6] za pomocą funkcji `clone` należącej do modułu `Repository`, który udostępniany jest przez **FAKEa**. Następnie uruchamiana jest kompilacja za pomocą funkcji `build` należącej do modułu `DotNet`, który również dostarcza **FAKE**.

Kiedy dotarłem do momentu kompilacji kodu, wpadła mi nowa wiedza z języka **F#**. W [poprzedniej][7] implementacji, zakodowanej w **PowerShellu**, jawnie podawałem konfiguracją **Release**. W implementacji z wykorzystaniem funkcji `DotNet.build` chciałem uzyskać taki sam efekt. Sygnatura funkcji w piątej wersji **FAKEa** wygląda tak:

{% highlight fsharp %}
val build:setParams:(DotNet.BuildOptions -> DotNet.BuildOptions) -> project:string -> unit
{% endhighlight %}

Co oznacza, że funkcja przyjmuje dwa parametry:

* Pierwszy parametr jest funkcją, która przyjmuje parametr typu `BuildOptions` i zwraca wartość typu `BuildOptions`
* Drugi parametr to ścieżka do projektu/solucji do kompilacji

Funkcja nie zwraca żadnej wartości.

Pierwsza myśl, jaka przyszła mi do głowy, to podejście obiektowe. Podstawię pod zmienną `BuildOptions.Configuration` nową wartość udostępnianą przez **FAKEa** - `DotNet.BuildConfiguration.Release`. Efekt? Błąd kompilacji z informacją:

`This field is not mutable`

No tak. W **F#** domyślnie wszystkie już utworzone wartości są niezmienne. Poza tym `BuildOptions` nie jest klasą, tylko typem [Record][8], który nie występuję np. w wersji ósmej języka **C#**. Research w internecie naprowadził mnie na nową konstrukcję języka **F#** - [Copy and Update Record Expressions][9]. Połączenie tej konstrukcji z możliwością tworzenia funkcji anonimowych okazało się tym, czego potrzebowałem. Fragment kodu:

{% highlight fsharp %}
DotNet.build (fun p -> { p with Configuration = DotNet.BuildConfiguration.Release }) slnPath
{% endhighlight %}

oznacza utworzenie anonimowej funkcji z parametrem `p`, który jest typu `BuildOptions`, z ciałem funkcji, w której za pomocą słowa kluczowego `with` podmieniany jest **Record field** o nazwie `Configuration` na wartość `DotNet.BuildConfiguration.Release`. Drugim parametrem funkcji `DotNet.build` jest ścieżka do pliku **.sln**. 

Uruchomienie testów jednostkowych odbywa się poprzez wywołanie funkcji `DotNet.test`, gdzie `test` jest nazwą funkcji, a `DotNet` modułem udostępnianym przez **FAKEa**. Wywołanie funkcji odbywa się za pomocą tej samej konstrukcji, którą opisałem w powyższym akapicie.

Przejdźmy teraz przez implementację publikacji, umieszczając kod np. w pliku **publish.fsx**:

{% highlight fsharp %}
// Targets
Target.create "Publish Artifacts" (fun _ ->
    let projectPath = retrieveParam projectPathParamName
    let rid = retrieveParam ridParamName
    
    Trace.trace ("-> Project " + projectPath)
        
    Trace.trace ("-> Publish " + projectPath)
    DotNet.publish (fun p -> 
        { p with
            Configuration = DotNet.BuildConfiguration.Release
            NoBuild = true
            Runtime = if rid = "empty" then p.Runtime else Some rid
        }) projectPath
    
    Trace.trace "-> Code published successfully."
)

//// Dependencies

// start build
Target.runOrDefaultWithArguments "Publish Artifacts"
{% endhighlight %}

Zgodnie z założeniami projektowymi, jeden **Target** odpowiada za publikację, która odbywa się poprzez wywołaniem funkcji `publish` należącej do modułu `DotNet`. Funkcja przyjmuje konfigurację pod postacią rekordu `BuildOptions` oraz ścieżkę do projektu, który ma zostać opublikowany. Tym razem w opcjach podmieniamy trzy wartości:

1. `Configuration` - tak, jak poprzednio, ustawiamy wersję **Release**
2. `NoBuild` - mamy już zbuildowany kod, więc nie trzeba powtarzać tej operacji
3. `Runtime` - jeśli chcemy opublikować kod pod konkretne środowisko np. **Windows**

### Test & Deploy

Stworzyliśmy dwa skrypty **FAKEa**. Do przygotowania do wdrożenia całego **Systemu komentarzy** potrzebne są trzy skrypty **PowerShell**. Pierwszy np. **run_build.ps1** uruchamia kompilację oraz testy jednostkowe:

{% highlight powershell %}
# fake build parameters
$buildArtifactsWorkingDirectoryPath = "buildArtifactsWorkingDirectoryPath=[path_to_sln_directory]"
$gitRepositoryUrl = "gitRepositoryUrl=[path_to_git_repository]"
$buildArtifactsSubDirectoryName = "buildArtifactsSubDirectoryName=build_artifacts"
$solutionRelativePath = "solutionRelativePath=src"

#execute script
fake run build.fsx -e $buildArtifactsWorkingDirectoryPath -e $gitRepositoryUrl -e $buildArtifactsSubDirectoryName -e $solutionRelativePath
{% endhighlight %}

Drugi np. **run_publish_blogcomments.ps1** publikuje kod Endpointa **BlogComments**:

{% highlight powershell %}
# fake build parameters
$projectPath = "projectPath=[path_to_endpoint_project]"
$rid = "rid=empty"

#execute script
fake run publish.fsx -e $projectPath -e $rid
{% endhighlight %}

Trzeci np. **run_publish_web.ps1** publikuje kod komponentu **Web**:

{% highlight powershell %}
# fake build parameters
$projectPath = "projectPath=[path_to_web_project]"
$rid = "rid=empty"

#execute script
fake run publish.fsx -e $projectPath -e $rid
{% endhighlight %}

Kod uruchamiany jest tak samo, niezależnie od tego, czy artefakty mają być wdrażane na środowisko testowe, czy produkcyjne.

W ten oto sposób dotarliśmy do końca serii, w której podzieliłem się z Tobą moimi doświadczeniami przy realizacji funkcjonalności **CI/CD** z wykorzystaniem narzędzia [FAKE][2]. Samo narzędzie posiada o wiele więcej możliwości. Jeśli chcesz sprawdzić w praktyce, jak działa mechanizm, wdrożony za pomocą skryptów, z którymi zapoznałeś/zapoznałaś się w tej serii, zostaw komentarz pod tym artykułem. Jestem ciekaw, jakich narzędzi używasz do realizacji procesów **CI/CD** dla funkcjonalności, którymi się zajmujesz.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2018-03-11-blog_comments_I_Design.md %}
[2]: https://fake.build/ "FAKE Build"
[3]: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21 "dotnet publish"
[4]: {{ site.url }}{% link _posts/2020-02-16-fake-build-web-host.md %}
[5]: https://github.com/mikedevbo/blog-comments/tree/master/src/Deployment "BlogComments Deployment"
[6]: https://git-scm.com/ "Git"
[7]: {{ site.url }}{% link _posts/2018-08-19-blog_comments_IV_Deploy.md %}
[8]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/records "FSharp Records"
[9]: https://docs.microsoft.com/en-us/dotnet/fsharp/language-reference/copy-and-update-record-expressions "Copy and update record expressions"
[10]: {{ site.url }}{% link _posts/2020-02-09-fake-build-nsb-web-host.md %}
[11]: {{ site.url }}{% link _posts/2020-02-23-fake-build-nsb-ws-host.md %}
[12]: {{ site.url }}{% link _posts/2020-03-01-fake-build-build-test-publish.md %}


