---
layout: post
title:  "Fake it... ale nie tak jak myślisz - ASP.NET Web Host"
date:   2020-02-16
---

Posty z tej serii:

* [Fake it... ale nie tak jak myślisz - NServiceBus Web Host][5]
* [Fake it... ale nie tak jak myślisz - ASP.NET Web Host][9]
* [Fake it... ale nie tak jak myślisz - NServiceBus Windows Service Host][10]
* Fake it... ale nie tak jak myślisz - Build - Run Unit tests - Publish

W poprzednim artykule przeszliśmy przez funkcjonalność wdrażania aplikacji webowej, której jedynym zadaniem jest hostowanie **Endpointa NServiceBus**. W kontekście [Systemu komentarzy na blogu][1] wdrażany **Endpoint** jest właścicielem komponentu przetwarzającego dodane komentarze. System posiada też drugi komponent, który przyjmuje komentarze od użytkowników, a następnie wysyła je do wcześniej wspomnianego **Endpointa**. Komponent ten hostowany jest w osobnej aplikacji webowej napisanej we frameworku [ASP.NET][2] w połączeniu z frameworkiem [NancyFx][3]. Dodanie komentarza odbywa się poprzez obsługę żądania **HTTP**. **ASP.NET** przyjmuje żądanie, przekierowuje je do modułu **NancyFx**, który wysyła **Message** do **Endpointa** zawierającego obsługę komentarza.

W dalszej części artykułu przejdziemy przez funkcjonalności wdrażania aplikacji webowej, której zadaniem jest obsługa żądań **HTTP**.

### Design

W odróżnieniu od aplikacji webowej hostującej **Endpointa NServiceBus** nie możemy tak po prostu wyłączyć aplikacji webowej obsługującej żądania **HTTP**. Wdrożenie trwa jakiś czas. Polega na usuwaniu aktualnych artefaktów oraz wgrywaniu nowych. W kontekście systemu komentarzy na blogu, w trakcie wdrożenia użytkownik nie mógłby dodawać komentarzy. Jedną ze strategi poradzenia sobie z taką sytuacją jest zastosowanie strategii **Blue-Green-Deployment**. Do zapoznania się ze szczegółami zapraszam Cię do przeczytania [jednego z moich wcześniejszych artykułów][4].

Schemat, którego ja używam, zawiera kroki:

1. Przygotuj lokalnie artefakty do wdrożenia
2. Usuń artefakty:
    * Jeśli aktualnie działającą wersją jest **Blue**, to usuń ze zdalnej lokalizacji artefakty wersji **Green**
    * Jeśli aktualnie działającą wersją jest **Green**, to usuń ze zdalnej lokalizacji artefakty wersji **Blue**
3. Wgraj artefakty z nową wersją:
    * Jeśli aktualnie działającą wersją jest **Blue**, to wgraj nową wersję na zasób zdalny dla wersji **Green**
    * Jeśli aktualnie działającą wersją jest **Green**, to wgraj nową wersję na zasób zdalny dla wersji **Blue**
4. Wywołaj żądanie **HTTP** używając adresu URL dla wgranej wersji
5. Ustaw przekierowanie żądań **HTTP** dla głównego URL na URL wgranej nowej wersji
6. Wywołaj żądanie **HTTP** używając głównego adresu URL

### Develop

Z [poprzedniego artykułu][5] wiesz już, w jaki sposób można pisać skrypty wdrożeniowe za pomocą narzędzia [FAKE][6]. Przejdźmy przez implementację zgodną z procesem opisanym w sekcji **Design** kodując program np. w pliku **web_ftp_deploy.fsx**:

{% highlight fsharp %}
// Targets
Target.create "Deploy Web" (fun _ ->
    let ftpWebPath = retrieveParam ftpWebPathParamName
    let deployArtifactsPath = retrieveParam deployArtifactsPathParamName
    let buildArtifactsPath = retrieveParam buildArtifactsPathParamName
    let settingsPath = retrieveParam settingsPathParamName
    let nservicebusPath = retrieveParam nservicebusPathParamName
    let webUrlRedirectValue = retrieveParam webUrlRedirectValueParamName
    let webUrlRedirect = retrieveParam webUrlRedirectParamName
    let mainWebConfigFilePath = retrieveParam mainWebConfigFilePathParamName
    let ftpMainWebConfigFilePath = retrieveParam ftpMainWebConfigFilePathParamName
    let webUrlMain = retrieveParam webUrlMainParamName
    
    Trace.trace ("-> Web " + ftpWebPath)

    Trace.trace ("-> Clean " + deployArtifactsPath)
    Shell.cleanDir deployArtifactsPath

    Trace.trace ("-> Copy " + buildArtifactsPath)
    Shell.copyDir deployArtifactsPath buildArtifactsPath FileFilter.allFiles

    Trace.trace ("-> Copy " + settingsPath)
    Shell.copyDir deployArtifactsPath settingsPath FileFilter.allFiles

    Trace.trace ("-> Copy " + nservicebusPath)
    Shell.copyDir deployArtifactsPath nservicebusPath FileFilter.allFiles

    Trace.trace ("-> Clean " + ftpWebPath)
    makeFtpAction (fun ftp -> ftp.RemoveFiles(ftpWebPath + "/*").Check())

    Trace.trace ("-> Upload files to " + ftpWebPath)
    makeFtpAction (fun ftp -> ftp.PutFiles(deployArtifactsPath + "\*", ftpWebPath + "/").Check())

    Trace.trace ("-> Call URL " + webUrlRedirect)
    try
        Http.get "" "" webUrlRedirect |> ignore
    with
    | :? System.Net.Http.HttpRequestException as ex ->
        Trace.trace ex.Message

        let isStatusCorrect = ex.Message.Contains("404");
        match isStatusCorrect with
        | true -> ()
        | false -> reraise()

    Trace.trace ("-> Set " + mainWebConfigFilePath)
    let webConfig = loadDoc(mainWebConfigFilePath)
    let webConfigWithChangedUrl = replaceXPathAttribute "//action[@type = 'Redirect']" "url" webUrlRedirectValue webConfig
    saveDoc mainWebConfigFilePath webConfigWithChangedUrl

    Trace.trace ("-> Upload file to " + ftpMainWebConfigFilePath)
    makeFtpAction (fun ftp -> ftp.PutFiles(mainWebConfigFilePath, ftpMainWebConfigFilePath).Check())

    Trace.trace ("-> Call URL " + webUrlMain)
    Http.get "" "" webUrlMain |> ignore

    Trace.trace (sprintf "-> Web %s deployed successfully." ftpWebPath)
)

// start build
Target.runOrDefaultWithArguments "Deploy Web"
{% endhighlight %}

W powyższym fragmencie kodu pominąłem elementy, które opisałem w [poprzednim artykule][5]. Całość implementacji możesz zobaczyć na moim [GitHubie][7].

Podobnie jak poprzednio, tak i teraz chcemy, aby skrypt był uniwersalny. Dzięki temu będziemy mogli go wykorzystać przy wdrożeniach na różne środowiska. W pierwszej kolejności, za pomocą lokalnie zdefiniowanej funkcji `retrieveParam`, pobieramy parametry podawane przy uruchamianiu skryptu. Następnie przygotowujemy artefakty do wdrożenia za pomocą dostarczanych przez **FAKEa** funkcji `Shell.cleanDir` oraz `Shell.copyDir`. Kolejnym krokiem jest wyczyszczenie zdalnego zasobu oraz wgranie artefaktów. Wykorzystujemy do tego lokalnie zdefiniowaną funkcję `makeFtpAction`. Potem sprawdzamy, czy wgrana aplikacja webowa działa, odpytując ją żądaniem **HTTP** za pomocą funkcji `Http.get`, która pochodzi z przestrzeni nazw `Fake.Net`. Podany URL jest adresem wgranej aplikacji **Blue** lub **Green**. Jeśli wszystko działa, to za pomocą funkcji pochodzących z przestrzeni nazwa `Fake.Core.Xml`, ustawiamy w pliku **web.config** przekierowanie głównego adresu URL na adres wgranej aplikacji. Ostatnim krokiem jest sprawdzenie głównego adresu URL poprzez wywołanie żądania **HTTP**.

Analizując powyższą implementację, mogą nasunąć Ci się dwa pytania:

1. Dlaczego kod nie jest podzielony na mniejsze funkcje, w których każda realizuje konkretny krok procesu?
2. Może warto przenieść lokalne funkcje `retrieveParam` oraz `makeFtpAction` do osobnego pliku skryptowego, a następnie wykorzystać tę samą implementację w plikach **host_ftp_deploy.fsx** oraz **web_ftp_deploy.fsx**? - **F#** [umożliwia][8] takie podejście. W skrócie - uwspólnić funkcjonalność i wprowadzić do niej zależność w poszczególnych skryptach.

Odpowiedź na pierwsze pytanie brzmi: *Jak najbardziej można tak zrobić*, natomiast jak popatrzysz na kod, to możesz zauważyć, że czyta się go wprost od góry do dołu, w taki sam sposób, w jaki opisany jest algorytm z sekcji **Design**. Kolejne wywołania funkcji `Trace.trace` oddzielają od siebie poszczególne kroki. W tym przypadku podział na mniejsze funkcje byłby tylko *"cukrem syntaktycznym"*.

Jak zapewne się domyślasz, odpowiedź na drugie pytanie brzmi: *Jak najbardziej można tak zrobić*. Pytanie *Co powinno być uwspólnione, a co powinno być rozdzielone?* jest klasycznym pytaniem w kontekście wytwarzania oprogramowania. Temat dotyczy zależności, ich definiowania, zarządzania nimi itp. Decyzje dotyczące zależności mają bardzo duży wpływ na utrzymanie oraz rozwój oprogramowania, zarówno ten początkowy, jak i późniejszy. Nie ma jednoznacznego podejścia, a przynajmniej ja się z nim nie spotkałem, w jaki sposób decydować co powinno być wspólne, a co powinno być rozdzielone. Odnosząc się do skryptów wdrożeniowych opisywanych w tej serii, założenie jest takie, aby były one maksymalnie niezależne od siebie, ponieważ realizują inne zadania. W związku z tym kod funkcji `retrieveParam` oraz `makeFtpAction` jest kopiowany do poszczególnych plików **.fsx**. Jakie są tego konsekwencje?

* Plusy:
    * Jeśli w przyszłości zmieni się koncepcja implementacji konkretnego skryptu, to zmiana będzie konieczna tylko w tym skrypcie. Nie trzeba będzie myśleć o tym, czy zmiana jest w kodzie wspólnym, a jak tak, to jaki będzie miała wpływ na działanie pozostałych skryptów.
    * Jeśli zmiana będzie dotyczyć skopiowanego kodu, to będzie można zmieniać skrypty jeden po drugim, testować i wdrażać niezależnie.
* Minusy
    * Jeśli okaże się, że skopiowany kod zawiera błędy, to trzeba będzie je poprawić we wszystkich skopiowanych miejscach.
    * Trzeba mieć wiedzę, w których miejscach jest ten sam, skopiowany kod.
    * Jeśli zmiana będzie dotyczyć skopiowanego kodu, to trzeba będzie tę zmianę wprowadzić we wszystkich skopiowanych miejscach

Minusy są dla mnie do zaakceptowania, stąd decyzja o rozdzieleniu funkcji `retrieveParam` oraz `makeFtpAction`. Krótkie ćwiczenie dla Ciebie. Przy założeniu, że kod dla tych dwóch funkcji byłby wspólny, wypisz plusy i minusy takiego podejścia. Czy widzisz jakiś związek z powyższymi Plusami i Minusami?

### Test & Deploy

Tak jak poprzednio, skrypt testujemy za pomocą **PowerShella**, pisząc osobny **.ps1** na każde środowisko wdrożeniowe. O danym środowisku decydują parametry wejściowe dla skryptu **FAKEa** np.

{% highlight powershell %}
#fake build parameters
$winSCPExecutablePath = "winSCPExecutablePath=C:\deploy\blog-comments\winscp\WinSCPnet.dll"
$ftpHostName = "ftpHostName=[host_name_or_host_ip_address]"
$ftpUserName = "ftpUserName=[ftp_user_name]"
$ftpPassword = "ftpPassword=[ftp_user_password]"
#idt.

#execute script
fake run web_ftp_deploy.fsx -e $winSCPExecutablePath -e $ftpHostName -e $ftpUserName -e $ftpPassword itd.
{% endhighlight %}

To tyle, jeśli chodzi o funkcjonalność wdrażania aplikacji webowej obsługującej żądania **HTTP**. W następnym artykule pokaże Ci przykład skryptu, który umożliwia wdrażanie **Endpointa NServiceBus** jako **Windows Service**.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2018-03-11-blog_comments_I_Design.md %}
[2]: https://dotnet.microsoft.com/apps/aspnet "ASP.NET"
[3]: http://nancyfx.org/ "NancyFX"
[4]: {{ site.url }}{% link _posts/2018-08-19-blog_comments_IV_Deploy.md %}
[5]: {{ site.url }}{% link _posts/2020-02-09-fake-build-nsb-web-host.md %}
[6]: https://fake.build/ "FAKE Build"
[7]: https://github.com/mikedevbo/blog-comments/tree/master/src/Deployment "BlogComments Deployment"
[8]: https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/fsharp-interactive/ "FSharp Interactive"
[9]: {{ site.url }}{% link _posts/2020-02-16-fake-build-web-host.md %}
[10]: {{ site.url }}{% link _posts/2020-02-23-fake-build-nsb-ws-host.md %}
