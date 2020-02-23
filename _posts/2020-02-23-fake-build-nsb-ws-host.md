---
layout: post
title:  "Fake it... ale nie tak jak myślisz - NServiceBus Windows Service Host"
date:   2020-02-23
---

Posty z tej serii:

* [Fake it... ale nie tak jak myślisz - NServiceBus Web Host][1]
* [Fake it... ale nie tak jak myślisz - ASP.NET Web Host][7]
* [Fake it... ale nie tak jak myślisz - NServiceBus Windows Service Host][8]
* Fake it... ale nie tak jak myślisz - Build - Run Unit tests - Publish

Po [przeniesieniu na Web Hosting ][1] komponentu odpowiedzialnego za przetwarzanie komentarzy, zastanawiałem się co zrobić z funkcjonalnością wdrażania **Endpointa NServiceBus** jako **Windows Service**. Rozważałem trzy opcje:

1. Usunięcie kodu
2. Zostawienie kodu tak, jak jest - wdrażanie za pomocą **PowerShella**
3. Przepisanie kodu używając narzędzia [Fake][2]

Wybrałem ostatni scenariusz z trzech powodów:

1. Zobaczyć, że się da
2. Poćwiczyć kodowanie w **F#**
3. Zostawić tę opcję jako awaryjną, w razie gdyby wystąpił problem z Web Hostingiem

Przejdźmy więc przez funkcjonalność, zaczynając od sekcji **Design**.

### Design

Strategia wdrażania pozostała taka sama jak poprzednio - **Blue-Green Deployment**. Po szczegóły zapraszam Cię do [jednego z moich poprzednich artykułów][3]. Schemat wdrożenia dla wersji **Blue** zawiera kroki:

1. Usuń aktualne artefakty z zasobu, z których czyta usługa Windows dla wersji **Blue**
2. Skopiuj artefakty z nową wersją do zasobu, z których czyta usługa Windows dla wersji **Blue**
3. Zatrzymaj usługę Windows z wersją **Green**
4. Wystartuj usługę Windows z wersją **Blue**:
    * Jeśli jest to pierwsze wdrożenie, utwórz usługę Windows dla wersji **Blue**

Schemat wdrożenia dla wersji **Green** jest taki sam. Zmieniają się tylko wartości - tam gdzie jest **Blue** będzie **Green**. Tam, gdzie jest **Green** będzie **Blue**.

Inną strategią, zastosowaną przy wdrażaniu [Endpointa NServiceBus jako Web Host][1], może być użycie opcji backupu. W takim przypadku wystarczy tylko jedna usługa Windows, która jest zatrzymywana na czas wdrożenia, a następnie uruchamiana w ostatnim kroku procesu.

W obydwu strategiach korzystamy z właściwości **Message-Queuing**, dzięki której zatrzymanie usługi Windows nie blokuje użytkownika przed dodawaniem komentarzy w trakcie trwania wdrożenia.

### Develop

Przejdźmy teraz przez implementację zgodną z procesem opisanym w sekcji **Design** kodując program np. w pliku **host_ws_deploy.fsx**:

{% highlight fsharp %}
// Targets
Target.create "Deploy Endpoint" (fun _ ->
    let deployEndpointPath = retrieveParam deployEndpointPathParamName
    let buildArtifactsPath = retrieveParam buildArtifactsPathParamName
    let settingsPath = retrieveParam settingsPathParamName
    let previousWindowsServiceName = retrieveParam previousWindowsServiceNameParamName
    let newWindowsServiceName = retrieveParam newWindowsServiceNameParamName
    let newWindowsServiceBinPath = retrieveParam newWindowsServiceBinPathParamName
    let newWindowsServiceDescription = retrieveParam newWindowsServiceDescriptionParamName
    
    Trace.trace ("-> Endpoint " + deployEndpointPath)

    Trace.trace ("-> Clean " + deployEndpointPath)
    Shell.cleanDir deployEndpointPath

    Trace.trace ("-> Copy " + buildArtifactsPath)
    Shell.copyDir deployEndpointPath buildArtifactsPath FileFilter.allFiles

    Trace.trace ("-> Copy " + settingsPath)
    Shell.copyDir deployEndpointPath settingsPath FileFilter.allFiles

    Trace.trace ("-> Stop Windows Service " + previousWindowsServiceName)
    try
        let sc = new ServiceController(previousWindowsServiceName);
        if sc.CanStop then sc.Stop()
    with
    | :? System.InvalidOperationException as ex ->
        Trace.trace ("-> Windows Service " + previousWindowsServiceName + "doesn't exist.")
        Trace.trace ex.Message
    
    Trace.trace ("-> Create and start Windows Service " + newWindowsServiceName)
    let sc = new ServiceController(newWindowsServiceName);
    try
        if sc.Status = ServiceControllerStatus.Stopped
        then
            Trace.trace ("-> Start Windows Service " + newWindowsServiceName)
            sc.Start()
    with
    | :? System.InvalidOperationException as ex ->
        Trace.trace ex.Message
        Trace.trace ("-> Create Windows Service " + newWindowsServiceName)
        CreateProcess.fromRawCommandLine "sc.exe" (sprintf "create %s start= demand binpath= %s " newWindowsServiceName newWindowsServiceBinPath)
        |> Proc.run
        |> ignore

        CreateProcess.fromRawCommandLine "sc.exe" (sprintf "description %s \"%s\" " newWindowsServiceName newWindowsServiceDescription)
        |> Proc.run
        |> ignore

        CreateProcess.fromRawCommandLine "sc.exe" (sprintf "failure %s reset= 3600 actions= restart/5000/restart/10000/restart/60000" newWindowsServiceName)
        |> Proc.run
        |> ignore

        Trace.trace ("-> Start Windows Service " + newWindowsServiceName)
        sc.Start()

    Trace.trace (sprintf "-> Endpoint %s deployed successfully." deployEndpointPath)
)

// start build
Target.runOrDefaultWithArguments "Deploy Endpoint"
{% endhighlight %}

W powyższym kodzie pominąłem elementy opisane w poprzednich częściach tej serii. Całość implementacji możesz zobaczyć na moim [GitHubie][4].

Interpretację poszczególnych fragmentów pozostawiam Ci jako ćwiczenie z analizy kodu. Nowym elementem niewystępującym w poprzednich artykułach tej serii, jest funkcjonalność uruchamiania zewnętrznych programów. W tym przypadku jest to program **sc.exe**, dzięki któremu możemy tworzyć usługi Windows. **Fake** udostępnia funkcję o nazwie `fromRawCommandLine` wchodzącą w skład modułu `CreateProcess`, która umożliwia zdefiniowanie programu do uruchomienia. Funkcja `run` z modułu `Proc` uruchamia zdefiniowany program.

### Test & Deploy

Uniwersalność kodu pozwala na przeprowadzanie wdrożeń na różne środowiska. Decydują o tym parametry wejściowe skryptu **FAKEa**. Tak, jak poprzednio, wdrożenie realizujemy za pomocą skryptów **PowerShell** - osobny **ps1** na każde środowisko:

{% highlight powershell %}
# fake build parameters
$deployEndpointPath = "deployEndpointPath=[path_to_windows_service_share]"
$buildArtifactsPath = "buildArtifactsPath=[path_to_artifacts]"
$settingsPath = "settingsPath=[path_to_settings]"
#idt.

#execute script
fake run host_ws_deploy.fsx -e $deployEndpointPath -e $buildArtifactsPath -e $settingsPath itd.
{% endhighlight %}

W tym krótkim artykule przeszliśmy przez możliwość wdrażania artefaktów jako **Windows Service**. Do tej pory zakładaliśmy, że artefakty istnieją, ale same z siebie nie powstaną :) W ostatniej części tej serii pokażę Ci, w jaki sposób można użyć **FAKEa** do pobrania kodu z repozytorium [Git][5], jego kompilacji, uruchomienia testów jednostkowych, a także przygotowania artefaktów do wdrożenia za pomocą komendy [dotnet publish][6].

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2020-02-09-fake-build-nsb-web-host.md %}
[2]: https://fake.build/ "FAKE Build"
[3]: {{ site.url }}{% link _posts/2018-08-19-blog_comments_IV_Deploy.md %}
[4]: https://github.com/mikedevbo/blog-comments/tree/master/src/Deployment "BlogComments Deployment"
[5]: https://git-scm.com/ "Git"
[6]: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet-publish?tabs=netcore21 "dotnet publish"
[7]: {{ site.url }}{% link _posts/2020-02-16-fake-build-web-host.md %}
[8]: {{ site.url }}{% link _posts/2020-02-23-fake-build-nsb-ws-host.md %}
