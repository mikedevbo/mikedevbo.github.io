---
layout: post
title:  "System komentarzy do bloga IV - Deploy"
date:   2018-05-27
---

W czwartej i ostatniej części serii pokazującej w jaki sposób można wykorzystać Messaging and Queueing do realizacji systemu komentarzy do bloga, zobaczymy implementację wybranych elementów pozwalających wdrożyć stworzone funkcjonalności.

## Proces

Podobnie jak przy projektowaniu lub programowaniu, sposobów na wdrożenie jest wiele. Narzędzi wspomagających wdrożenie również jest wiele. Wszystko zależy od fizycznej architektury rozwiązania, ilości komponentów do wdrożenia, docelowego miejsca gdzie komponenty będą hostowane, idt. Niezależnie od tego jakie rozwiązanie się wybierze, wszystkie kroki związane z wdrożeniem powinny być w maksymalnie możliwy sosób zautomatyzowane, tak aby zminimalizować możliwość popełnienia błędu. Jest to jedna z zasad tzw. [Continuous Integration software development practice][1]. Proces wdrożenia systemu komentarzy do bloga może wyglądać np. tak:

![Picutre1]({{ site.url }}/assets/blog_comments_deploy/deploy_process.png)

Patrząc na powyższy diagram możemy wyodrębnić dwa rodzaje automatyzacji:

* Dla każdego kroku w procesie istnieje automatyczny mechanizm realizujący ten krok:
    * Compile Code
    * Run Unit Tests
    * Deploy Nancy Host (Test and Production)
    * Deploy NServiceBus Host (Test and Production)

* Każdy kolejny krok uruchamia się automatycznie zaraz po udanym zakończeniu kroku poprzedniego:
    * Build
    * Test
    * Production

Druga część automatyzacji zależy od tego czy będziemy trzymać się zasad zdefiniowanych wg tzw. **Continuous Delivery** czy też zasad zdefiniowanych wg. tzw. **Continuous Deployment**. Więcej szczegółów o różnicach między tymi podejściami można przeczytać [w tym artykule][2].

O ile pierwsza część automatyzacji jest raczej stała o tyle druga część może się zmieniać w zależności od tego w którą stronę dany produkt będzie ewaluował. W dalszej części artykułu skupimy się na części pierwszej.

## Architektura

Aby popatrzyć na różne możliwości wdrożeń przyjmijmy takie oto założenia:

* Nancy host jest wdrażany na standardowy Web Hosting gdzie mamy ograniczone możliwości konfiguracji
* NServiceBus host jest wdrażany na maszynę wirtualną nad którą mamy pełną kontrolę i pełne możliwości konfiguracji
* Warstwa transportowa NServiceBus'a to tabele w bazie danych SQL Server

![Picutre1]({{ site.url }}/assets/blog_comments_deploy/deploy_architecture.png)

Patrząc na architekturę z punktu widzenia logicznego Nancy host wysyła message'e do NServiceBus Host'a. Patrząc na architekturę z punktu widzenia fizycznego obydwa hosty mają zdefiniowane tzw. **Policy** które mówi jaki transport kolejkowy ma być użyty do przesyłania wiadomości. NServiceBus wspiera wiele różnych rodzajów [transportów][3]. Wybór transportu może mieć wpływ na to jak i gdzie NServiceBus host będzie wdrażany a co za tym idzie wpływ na rozwiązanie automatyzacji.

## Build

Zanim wdrożymy stworzone funkcjonalności chcemy wyeliminować tzw. efekt "u mnie działa" jeśli chodzi o kompilację kodu oraz rezultat unit testów. Pierwszym krokiem w procesie wdrożenia jest tzw. **Build**, którego zadaniem jest pobranie kodu z systemu kontroli wersji, skompilowanie w celu sprawdzenia czy kod jest kompletny oraz uruchomienie unit testów. Build może zawierać również inne kroki np. sprawdzanie pokrycia kodu testami jednostkowymi, zgodność kodu z regułami jego formatowania, itd. Przydatnym elementem jest oznaczanie wdrażanych artefaktów numerem wskazującym z którego commit'a z systemu kontroli wersji nastąpiło wdrożenie. Jeśli kod trzymamy na [GitHub'e][4] to możemy tworzyć tzw. [Release'y][5] oznaczając je np. wg konwencji [Semantic Versioning][6]. Następnie skorzystać z narzędzia [GitVersion][7], które oznacza artefakty numerem zdefiniowanego GitHub Release'a. Do zautomatyzowania procesu Build'owania można wykorzystać różnego rodzaju [narzędzia lub systemy][8]. W dalszej części artykułu zobaczymy jak wykorzystać narzędzie oraz język skryptowy [PowerShell][9] do zautomatyzowania zarówno Build'a jak Deploy'a stworzonych funkcjonalności.

W pierwszej kolejności zobaczmy przykładowy skrypt, który pobiera kod z GitHub'a oraz go kompiluje.

{% highlight powershell %}
[CmdletBinding()]
Param(

    [Parameter(Mandatory=$True)]
    [string]$gitExePath,
    
    [Parameter(Mandatory=$True)]
    [string]$nugetExePath,

    [Parameter(Mandatory=$True)]
    [string]$msbuildExePath,

    [Parameter(Mandatory=$True)]
    [string]$buildArtifactsPath,

    [Parameter(Mandatory=$True)]
    [string]$gitRepositoryUrl,	
	
    [Parameter(Mandatory=$True)]
    [string]$solutionRelativePath,

    [Parameter(Mandatory=$True)]
    [string]$binRelativePath
)

$ErrorActionPreference = "Stop"

try
{
    Write-Host "clean artifacts directory"
    Remove-Item "$buildArtifactsPath\*" -Recurse -Force
    
    Write-Host "clone repository"
    & $gitExePath "clone" "-q" $gitRepositoryUrl $buildArtifactsPath
    
    Write-Host "restore nuget packages"
    & $nugetExePath "restore" "$buildArtifactsPath\$solutionRelativePath"

    Write-Host "build solution"
    $buildLogFile = "$buildArtifactsPath\$binRelativePath\build.log"
    $run = $msbuildExePath
    $p1 = "$buildArtifactsPath\$solutionRelativePath"
    $p2 = "/p:Configuration=Release"
    $p3 = "/flp:logfile=$buildLogFile"

    & $run $p1 $p2 $p3

    if(!$?)
    {
        throw "Buit failed see $buildLogFile for details"
    }
}
catch
{
    throw;
}
{% endhighlight %}

Skrypt przyjmuje następujące parametry wejściowe:

* **$gitExePath** - ścieżka do pliku git.exe
* **$nugetExePath** - ścieżka do pliku nuget.exe
* **$msbuildExePath** - ścieżka do pliku msbuild.exe
* **$buildArtifactsPath** - ścieżka do katalogu gdzie zostanie pobrany kod do skompilowania
* **$gitRepositoryUrl** - ścieżka do repozytorium kodu GitHub'a
* **$solutionRelativePath** - ścieżka do pliku *.sln do skompilowania (względem $buildArtifactsPath)
* **$binRelativePath** - ścieżka do skompilowanego kodu (względem $buildArtifactsPath)

W pierwszej kolejności czyszczony jest katalog na artefakty. Następnie z GitHub'a pobierany jest kod. Kolejne dwa kroki to pobranie paczek nuget'owych oraz kompilacja kodu. Ostatecznie skompilowany kod trafia do katalogu bin skąd można go dalej procesować. Wynik kompilacji zapisywany do pliku log'a zdefiniowanego przez zmienną $buildLogFile. Zapisując powyższy skrypt do pliku build.ps1 można go uruchomić np. tak

{% highlight powershell %}
$runScript = "C:\build.ps1"
$gitExePath = "C:\Program Files\Git\bin\git.exe"
$nugetExePath = "C:\tools\Nuget-4.6.2\nuget.exe"
$msbuildExePath = "C:\Program Files (x86)\Microsoft Visual Studio\2017\Community\MSBuild\15.0\Bin\MsBuild.exe"
$buildArtifactsPath = "C:\deploy\blog-comments\build_artifacts"
$gitRepositoryUrl = "https://github.com/mikedevbo/blog-comments.git"
$solutionRelativePath = "src\blogcomments.sln"
$binRelativePath = "src\bin"

& $runScript $gitExePath $nugetExePath $msbuildExePath $buildArtifactsPath $gitRepositoryUrl $solutionRelativePath $binRelativePath
{% endhighlight %}

Zapisując powyższy skrypt do pliku run_build.ps1 można go uruchomić z konsoli PowerShell jednym poleceniem. W ten sposób mamy zautomatyzowany proces kompilacji kodu.

Zobaczmy teraz w jaki sposób można uruchomić Unit Testy.

{% highlight powershell %}
[CmdletBinding()]
Param(
    [Parameter(Mandatory=$True)]
    [string]$NunitExePath,
	
    [Parameter(Mandatory=$True)]
    [string]$binPath
)

$ErrorActionPreference = "Stop"

try
{
    $tests = (Get-ChildItem $binPath -Recurse -Include *unit.tests.dll)

    & $NunitExePath $tests --noheader --work=$binPath
}
catch
{
    throw;
}
{% endhighlight %}

Skrypt przyjmuje następujące parametry wejściowe:

* **$nunitExePath** - ścieżka do nunit3-console.exe
* **$binPath** - ścieżka do skompilowanego kodu

Skrypt zawiera dwa kroki. Pierwszy krok wyszukuje wszystkie pliki mające końcówkę *unit.tests.dll. Są to dll'ki zawierające Unit Testy. Drugi krok to uruchomienie testów i wypisanie wyniku na standardowe wyjście. Zapisując powyższy skrypt do pliku run_unit_tests.ps1 można go uruchomić np. tak:

{% highlight powershell %}
$runScript = "C:\run_unit_tests.ps1"
$nunitExePath = "C:\tools\NUnit.Console-3.8.0\nunit3-console.exe"
$binPath = "C:\deploy\blog-comments\build_artifacts"

& $runScript $nunitExePath $binPath
{% endhighlight %}

Podobnie jak dla kompilacji kodu, zapisując powyższy skrypt do pliku run_unittests.ps1 można go uruchomić z konsoli PowerShell jednym poleceniem. W ten sposób mamy zautomatyzowany proces uruchamiania testów jednostkowych.

Jeśli kompilacja kodu oraz wynik testów jednostkowych zakończą się sukcesem można przejść do procesu wdrożenia artefaktów.

## Nancy Host Deploy

### Idea

Wykorzystując standardowy Web Hosting, wdrożenie aplikacji webowej przeważnie sprowadza się do przekopiowania artefaktów w odpowiednie miejsce np. za pomocą protokołu FTP. Kopiując pliki zawsze w to samo miejsce trzeba liczyć się z tym, że aplikacja może przestać działać od momentu rozpoczęcia kopiowania do momentu jego zakończenia. Istnieją [różne wzorce wdrożeniowe][10] pozwalające poradzić sobie z tym wyzwaniem. Zobaczmy w jaki sposób można zrealizować wzorzec [Blue-Green-Deployment][11] wdrażając Nancy Host'a.

### Konfiguracja

Jeśli zarówno środowisko testowe jak i produkcyjne są takie same (lub prawie takie same) proces automatycznego wdrożenia wygląda tak samo różniąc się tylko elementami konfiguracji. W kontekście dodawania komentarzy do bloga mogą być to:

* url
    * test - testcomments.[domainname].pl/comment
    * produkcja - comments.[domainName].pl/comment
* ścieżka do repozytorium GitHub'a
    * test - branch testowy
    * produkcja - branch master
* connectionString do bazy danych
    * test - baza testowa
    * produkcja - baza produkcyjna
* ...

### Blue-Green-Deployment - Web Hosting - struktura katalogów 

Struktura katalogów powinna być taka sama zarówno na środowisku testowym jak i na produkcyjnym. Załóżmy, że głównym katalogiem do którego mamy skopiować artefakty jest katalog o nazwie **wwwroot**. W tym katalogu tworzymy dwa podkatalogi: **blue** oraz **green**. Przy kolejnych wdrożeniach artefakty będą kopiowane na zmianę raz do katalogu blue a raz do katalogu green:

* wdrożenie nr 1 -> kopiuj do katalogu blue
* wdrożenie nr 2 -> kopiuj do katalogu green
* wdrożenie nr 3 -> kopiuj do katalogu blue
* wdrożenie nr 4 -> kopiuj do katalogu green
* ...

Jeśli Web Hosting używa serwera webowego [IIS][12], zarówno katalog blue jak i green muszą być ustawione jako [Application][13], tak aby można było na nich uruchamiać osobne wersje Nancy Web Host'a. Przy takiej strukturze katalogów odwołania URL do poszczególnych wersji wyglądają tak:

* test
    * testcomments.[domainname].pl -> wwwroot
    * testcomments.[domainname].pl/blue/comment -> wwwroot/blue
    * testcomments.[domainname].pl/green.comment -> wwwroot/green
* produkcja
    * comments.[domainname].pl -> wwwroot
    * comments.[domainname].pl/blue/comment -> wwwroot/blue
    * comments.[domainname].pl/green.comment -> wwwroot/green

Zobaczmy teraz jak wygląda struktura katalogów i plików wewnątrz folderów blue oraz green:
* blue\green
    * **app_data** - katalog do którego NServiceBus zapisuje logi
    * **bin** - katalog w którym znajduje się skompilowany kod
    * **nservicebus** - katalog z którego NServiceBus pobiera informacje o licencji
    * **appsettings.config** - plik z konfiguracją
    * **connectionstrings.config** - plik z konfiguracją bazy danych
    * **log4net.config** - plik z konfiguracją logów zapisywanych za pomocą biblioteki [Log4Net][15]
    * **web.config** - główny plik konfiguracyjny

### Powrót do poprzedniej wersji

Aby przy każdym kolejnym wdrożeniu nie musieć zmieniać nic po stronie klienta, można wykorzystać właściwość [IIS URL Rewrite][14]. Klient zawsze będzie odwoływał się do głównego URL'a z którego to nastąpi przekierowanie albo do wersji blue albo do wersji green. Jednym z kroków mechanizmu automatyzującego proces wdrożenia będzie ustawienie odpowiedniej wartości w głównym pliku konfiguracyjnym web.config. Przykład definicji przekierowującej do wersji blue:

{% highlight xml %}
<?xml version="1.0" encoding="UTF-8"?>
<configuration>
  <system.webServer>
    <!-- other configurations -->
    <rewrite>
      <rules>
        <rule name="RedirectToNewVersion" stopProcessing="true">
          <match url="^$" />
          <action redirectType="Temporary" type="Redirect" url="/blue/comment" />
        </rule>
      </rules>
    </rewrite>
  </system.webServer>
</configuration>
{% endhighlight %}

W ten sposób powrót do poprzedniej wersji odbywa się przez podmianę atrybutu url. Jeśli aktualna wersja to blue powrót na green. Jeśli aktualna wersja to green powrót na blue.

### Deploy - proces

Mając zdefiniowaną strukturę katalogów na Web Host'e najlepiej jest utworzyć jej kopię na maszynie z której będzie przeprowadzane wdrożenie. Dodatkowo w tej strukturze można umieścić wszystkie pliki konfiguracyjne:

* test
    * web
        * wwwroot - katalog z taką samą strukturą jak na Web Host'e
            * blue
            * green
            * web.config
        * connectionstrings - katalog z plikiem konfiguracyjnym ścieżki do bazy danych
        * nservicebus - katalog z plikiem licencji NServiceBus'a
        * settings - katalog z plikami appsettings.config, log4net.config, web.config
* production
    * web
        * wwwroot
            * blue
            * green 
            * web.config
        * connectionstrings
        * nservicebus
        * settings       

Proces wdrożenia nowej wersji w opcji blue na środowisko testowe może wyglądać np. tak:

1. Wyczyść lokalny katalog blue
2. Przekopiuj skompilowane artefakty do katalogu blue
3. Utwórz katalog app_data
4. Przekopiuj konfiguracje appsettings.config, connectionstrings.config, log4net.config, web.config
5. Wyczyść katalog blue na FTP
6. Przekopiuj artefakty z lokalnego katalogu blue to katalogu blue na FTP
7. Wywołaj żądanie na podstronie blue - testcomments.[domainname].pl/blue/comment
9. Ustaw przekierowanie w lokalnym głównym pliku konfiguracyjnym web.config na wersję blue
10. Przekopiuj główny plik konfiguracyjny web.config do głównego katalogu wwwroot na FTP. 
11. Wywołaj żądanie na głównej stronie testcomments.[domainname].pl

Wdrożenie wersji green wygląda analogicznie z tą różnicą że wszystko odbywa się na katalogach green. Podobnie wdrożenie wersji produkcyjnej wygląda analogicznie, zmienia się tylko miejsce docelowe (środowisko produkcyjne) oraz pliki konfiguracyjne (ustawienia produkcyjne).

### Deploy - skrypt PowerShell

Zobaczmy jako może wyglądać skrypt PowerShell realizujący powyższy proces:

{% highlight powershell %}
[CmdletBinding()]
Param(

    [Parameter(Mandatory=$True)]
    [string]$destination,

    [Parameter(Mandatory=$True)]
    [string]$source,

    [Parameter(Mandatory=$True)]
    [string]$nserviceBusLicenseSourcePath,

    [Parameter(Mandatory=$True)]
    [string]$settingsSourcePath,

    [Parameter(Mandatory=$True)]
    [string]$connectionstringsSourcePath,

    [Parameter(Mandatory=$True)]
    [string]$ftpHostName,

    [Parameter(Mandatory=$True)]
    [string]$ftpUserName,

    [Parameter(Mandatory=$True)]
    [string]$ftpPassword,

    [Parameter(Mandatory=$True)]
    [string]$ftpDestinationPath,

    [Parameter(Mandatory=$True)]
    [string]$winscpDllPath,

    [Parameter(Mandatory=$True)]
    [string]$urlToWarmUp,

    [Parameter(Mandatory=$True)]
    [string]$mainWebConfigFilePath,

    [Parameter(Mandatory=$True)]
    [string]$urlRedirect,

    [Parameter(Mandatory=$True)]
    [string]$ftpMainWebConfigDestinationPath,

    [Parameter(Mandatory=$True)]
    [string]$mainUrlToWarmUp
)

$ErrorActionPreference = "Stop"

function prepareArtifactsToDeploy(
    $destination,
    $source,
    $nserviceBusLicenseSourcePath,
    $settingsSourcePath,
    $connectionstringsSourcePath)
{
    Write-Host "->clean $destination directory"
    Remove-Item "$destination\*" -Recurse -Force
    
    Write-Host "->copy artifacts for $source to $destination"
    Copy-Item "$source" -Destination "$destination\bin" -Recurse
    
    Write-Host "->create app_data directory"
    New-Item -ItemType directory -Path "$destination\app_data"

    Write-Host "->copy NServiceBusLicense"
    Copy-Item "$nserviceBusLicenseSourcePath" -Destination "$destination\nservicebus" -Recurse

    Write-Host "->copy settings"
    Copy-Item "$settingsSourcePath\*" -Destination "$destination" -Recurse

    Write-Host "->copy connectionstrings"
    Copy-Item "$connectionstringsSourcePath\*" -Destination "$destination" -Recurse
}

function ftpCleanDestination($session, $ftpDestinationPath)
{
    Write-Host "->remove files from ftp $ftpDestinationPath"
    
    $removalResult = $session.RemoveFiles("$ftpDestinationPath/*")
    if (!$removalResult.IsSuccess)
    {
        throw "Removing files failed"
    }
}

function ftpCopyFiles($session, $from, $to)
{
    Write-Host "->copy files to ftp $to"
    $session.PutFiles("$from", "$to").Check()
}

function warmUpUri($Uri, $expectedStatusCode)
{
    Write-Host "->invoke $Uri"
        
    try
    {
        Invoke-WebRequest -Uri "$Uri" -Method HEAD
    }
    catch
    {
        $responseStatusCode = $_.exception.response.statuscode.value__
            
        if ($responseStatusCode -ne $expectedStatusCode)
        {
            throw "Deployed web host is broken -> response status code $responseStatusCode"
        }
    }
}

function setMainWebConfig($mainWebConfigFilePath, $urlRedirect)
{
    Write-Host "->set main web.config"
    $doc = New-Object System.Xml.XmlDocument
    $doc.Load($mainWebConfigFilePath)

    $book = $doc.SelectSingleNode("//action[@type = 'Redirect']")
    $book.url = "$urlRedirect"

    $doc.Save($mainWebConfigFilePath)
}

#main

try
{
    prepareArtifactsToDeploy $destination $source $nserviceBusLicenseSourcePath $settingsSourcePath $connectionstringsSourcePath

    Add-Type -Path "$winscpDllPath"
    $sessionOptions = New-Object WinSCP.SessionOptions -Property @{
        Protocol = [WinSCP.Protocol]::Ftp
        HostName = "$ftpHostName"
        UserName = "$ftpUserName"
        Password = "$ftpPassword"
    }

    $session = New-Object WinSCP.Session
    
    try
    {
        Write-Host "->open ftp session"
        $session.Open($sessionOptions)

        ftpCleanDestination $session $ftpDestinationPath
        ftpCopyFiles $session "$destination\*" "$ftpDestinationPath/"

        warmUpUri $urlToWarmUp 404

        setMainWebConfig $mainWebConfigFilePath $urlRedirect
        ftpCopyFiles $session $mainWebConfigFilePath $ftpMainWebConfigDestinationPath
        
        warmUpUri $mainUrlToWarmUp 200
    }
    catch
    {
        throw;
    }
    finally
    {
        Write-Host "->dispose ftp session"
        $session.Dispose()
    }
}
catch
{
    throw;
}
{% endhighlight %}

Ponieważ skrypt jest dość długi popatrzmy na jego bazową strukturę zostawiając szczegóły do samodzielnej analizy. Parametry wejściowe skryptu są tak opisane aby w miarę łatwy sposób można było się domyśleć co znaczą. I tak np. parametr $nserviceBusLicenseSourcePath wskazuję ścieżkę z której będzie przekopiowany plik z licencją NServiceBus'a. Parametr $ftpHostName oznacza nazwę hosta FTP gdzie będą kopiowane artefakty itd. Kolejnym widocznym element jest podział kodu na mniejsze części a następnie złożenie go w jedną całość. Mniejsze kawałki to funkcje zawierające parametry wejściowe, realizujące określoną funkcjonalność:

* prepareArtifactsToDeploy - funkcja realizująca kroki od 1. do 4. proces wdrożenia
* ftpCleanDestination - funkcja realizująca krok nr 5.
* ftpCopyFiles - funkcja realizująca kroki 6. i 9.
* warmUpUri - funkcja realizująca kroki 7. i 10.
* setMainWebConfig - funkcja realizująca krok nr 8

Wykonanie skryptu zaczyna się po komentarzu #main gdzie wywoływane są poszczególne funkcje. Do operacji na protokole FTP została wykorzystana biblioteka [WinSCP][16]. Skrypt można zapisać np. w pliku web_deploy.ps1.

### Deploy - skrypt PowerShell - wywołanie

Zobaczymy teraz jak można wywołać powyższy skrypt:

{% highlight powershell %}
$runScript = "C:\web_deploy.ps1"
$destinationPath = "C:\deploy\blog-comments\test\web\wwwroot\blue"
$sourcePath = "C:\deploy\blog-comments\build_artifacts\src\bin\web\net47"
$nserviceBusLicenseSourcePath = "C:\deploy\blog-comments\test\web\nservicebus"
$settingsSourcePath = "C:\deploy\blog-comments\test\web\settings"
$connectionstringsSourcePath = "C:\deploy\blog-comments\test\web\connectionstrings"
$ftpHostName = "[host_name_or_host_ip_address]"
$ftpUserName = "[ftp_user_name]"
$ftpPassword = "[ftp_user_password]"
$ftpDestinationPath = "/testcomments.[domainname].pl/wwwroot/blue"
$winscpDllPath = "C:\deploy\blog-comments\winscp\WinSCPnet.dll"
$UrlToWarmUp = "http://testcomments.[domainname].pl/blue"
$mainWebConfigFilePath = "C:\deploy\blog-comments\test\web\wwwroot\web.config"
$urlRedirect = "/blue/comment"
$ftpMainWebConfigDestinationPath = "/testcomments.[domainname].pl/wwwroot/web.config"
$mainUrlToWarmUp = "http://testcomments.[domainname].pl"

& $runScript $destinationPath $sourcePath $nserviceBusLicenseSourcePath $settingsSourcePath $connectionstringsSourcePath $ftpHostName $ftpUserName $ftpPassword $ftpDestinationPath $winscpDllPath $UrlToWarmUp $mainWebConfigFilePath $urlRedirect $ftpMainWebConfigDestinationPath $mainUrlToWarmUp
{% endhighlight %}

Powyższe parametry można dostosowywać w zależności od tego którą opcję  i na jakie środowisko wdrażamy nową wersję. Zapisując poszczególne konfiguracje do osobnych plików *.ps1 możemy jednym wywołaniem w konsoli PowerShell wdrożyć nową wersję:

* run_blue_web_deploy_test.ps1 - wersja blue na środowisko testowe
* run_green_web_deploy_test.ps1 - wersja green na środowisko testowe
* run_blue_web_deploy_production.ps1 - wersja blue na środowisko produkcyjne
* run_blue_web_deploy_production.ps1 - wersja green na środowisko produkcyjne

## NServiceBus Host Deploy

### Idea

Mając pełny dostęp do maszyny wirtualnej, wdrożenie NServiceBus'a przeważnie sprowadza się do wykorzystania [hostingu typu Windows Service][17]. Ponieważ NServiceBus bazuje na kolejkach i messaging'u usługę taką można zatrzymać na czas kopiowania plików z nową wersją. W takiej sytuacji wiadomości lądują w kolejce i czekają na przetworzenie, które nastąpi po ponownym uruchomieniu usługi. Więcej na temat zalet NServiceBus'a można przeczytać [np. w tym artykule]({% post_url 2017-04-29-wytwarzanie_oprogramowania_IV %}).

### Konfiguracja

Sytuacja jest analogiczna jak w przypadku Nancy Host'a. Cały proces wdrożenia NServiceBus Host'a powinien wyglądać tak samo zarówno dla środowiska testowego jak i produkcyjnego, różniąc się tylko elementami konfiguracji.

### Blue-Green-Deployment - Virtual Machine - struktura katalogów

Struktura katalogów na artefakty, które będą instalowane jako Usługa Windows, powinna być taka sama zarówno na środowisku testowym jak i na produkcyjnym. Może wyglądać np. tak:

* blog-comments - odpowiednik katalogu wwwroot dla Nancy Host'a
    * blue
    * green

Nowe wersje będą wdrażane, na zmianę, raz do katalogu blue a raz do katalogu green:

* wdrożenie nr 1 -> kopiuj do katalogu blue
* wdrożenie nr 2 -> kopiuj do katalogu green
* wdrożenie nr 3 -> kopiuj do katalogu blue
* wdrożenie nr 4 -> kopiuj do katalogu green
* ...

W odróżnieniu do Nancy Host'a tutaj wszystkie pliki konfiguracyjne będą trzymane w tym samym katalogu co skompilowany kod.

### Powrót do poprzedniej wersji

Powrót do poprzedniej wersji również się upraszcza i polega na zatrzymaniu nowo wdrożonej Usługi Windows i uruchomieniu poprzednio zatrzymanej.

### Deploy - proces

Mając pełny dostęp do maszyny wirtualnej możemy uruchomić skrypt wdrożeniowy bezpośrednio na niej kopiując artefakty bezpośrednio do katalogu z którego będzie instalowana Usługa Windows. Wcześniej można przygotować strukturę katalogów dla plików konfiguracyjnych

* host
    * connectionstrings - katalog z plikiem konfiguracyjnym ścieżki do bazy danych
    * settings - katalog z plikami appsettings.config, log4net.config, web.config

Proces wdrożenia nowej wersji w opcji blue na środowisko testowe może wyglądać np. tak:

1. Wyczyść katalog blue skąd będzie uruchamiana Usługa Windows
2. Przekopiuj skompilowane artefakty do katalogu blue
3. Przekopiuj konfiguracje appsettings.config, connectionstrings.config, Host.exe.config
4. Zatrzymaj aktualnie działającą Usługę Windows w opcji green
5. Utwórz nową Usługę Windows jeśli nie istnieje
6. Wystartuj nowo stworzoną Usługę Windows

Wdrożenie wersji green wygląda analogicznie z tą różnicą że wszystko odbywa się na katalogu green. Podobnie wdrożenie wersji produkcyjnej wygląda analogicznie, zmienia się tylko miejsce docelowe (środowisko produkcyjne) oraz pliki konfiguracyjne (ustawienia produkcyjne).

### Deploy - skrypt PowerShell

Zobaczmy jako może wyglądać skrypt PowerShell realizujący powyższy proces:

{% highlight powershell %}
[CmdletBinding()]
Param(

    [Parameter(Mandatory=$True)]
    [string]$destination,

    [Parameter(Mandatory=$True)]
    [string]$source,

    [Parameter(Mandatory=$True)]
    [string]$settingsSourcePath,

    [Parameter(Mandatory=$True)]
    [string]$connectionstringsSourcePath,

    [Parameter(Mandatory=$True)]
    [string]$previousWindowsServiceName,
    
    [Parameter(Mandatory=$True)]
    [string]$newWindowsServiceName,

    [Parameter(Mandatory=$True)]
    [string]$newWindowsServiceDescription,

    [Parameter(Mandatory=$True)]
    [string]$windowsServiceBinPath
)

$ErrorActionPreference = "Stop"

function prepareArtifactsToDeploy(
    $destination,
    $source,
    $settingsSourcePath,
    $connectionstringsSourcePath)
{
    Write-Host "->clean $destination directory"
    Remove-Item "$destination\*" -Recurse -Force
    
    Write-Host "->copy artifacts for $source to $destination"
    Copy-Item "$source\*" -Destination "$destination" -Recurse

    Write-Host "->copy settings"
    Copy-Item "$settingsSourcePath\*" -Destination "$destination" -Recurse

    Write-Host "->copy connectionstrings"
    Copy-Item "$connectionstringsSourcePath\*" -Destination "$destination" -Recurse
}

function windowsServiceExists($serviceName)
{
    $service = Get-Service $serviceName -ErrorAction SilentlyContinue

    if ($service)
    {
        return $True;
    }

    return $False
}

function stopWindowService($serviceName)
{
    Write-Host "stop windows service $serviceName"
    $service = Get-Service $serviceName -ErrorAction SilentlyContinue
    
    if (!$service)
    {
        return;
    }

    if (!($service.Status -eq 'Running'))
    {
        return;
    }

    Stop-Service $serviceName
}

#main

try
{
    prepareArtifactsToDeploy $destination $source $settingsSourcePath $connectionstringsSourcePath
    stopWindowService $previousWindowsServiceName

    if (!(windowsServiceExists $newWindowsServiceName))
    {
        Write-Host "create windows service $newWindowsServiceName"
        sc.exe create $newWindowsServiceName start= demand binpath= "$windowsServiceBinPath"
        sc.exe description $newWindowsServiceName $newWindowsServiceDescription
        sc.exe failure $newWindowsServiceName reset= 3600 actions= restart/5000/restart/10000/restart/60000
    }

    Write-Host "start windows service $newWindowsServiceName"
    Start-Service $newWindowsServiceName
}
catch
{
    throw;
}
{% endhighlight %}

Skrypt również został podzielony na mniejsze kawałki za pomocą funkcji:

* prepareArtifactsToDeploy - funkcja realizująca kroki od 1 do 3 procesu
* windowsServiceExists - funkcja realizująca krok nr 5
* stopWindowService - funkcja realizująca krok nr 4

Wykonanie skryptu zaczyna się po komentarzu #main gdzie wywoływane są poszczególne funkcje. Ostatnim elementem skryptu jest wykonanie kroku nr 6 procesu. Skrypt można zapisać np. w pliku host_deploy.ps1.

### Deploy - skrypt PowerShell - wywołanie

Zobaczymy teraz jak można wywołać powyższy skrypt:

{% highlight powershell %}
$runScript = "C:\host_deploy.ps1"
$destinationPath = "C:\applications\blog-comments\test\blue"
$sourcePath = "C:\deploy\blog-comments\build_artifacts\src\bin\host\net47"
$settingsSourcePath = "C:\deploy\blog-comments\test\host\settings"
$connectionstringsSourcePath = "C:\deploy\blog-comments\test\host\connectionstrings"
$previousWindowsServiceName = "blogcomments-green-test"
$newWindowsServiceName = "blogcomments-blue-test"
$newWindowsServiceDescription = "Service for hosting blogcomments endpoint."
$windowsServiceBinPath = "C:\applications\blog-comments\test\blue\Host.exe"

& $runScript $destinationPath $sourcePath $settingsSourcePath $connectionstringsSourcePath $previousWindowsServiceName $newWindowsServiceName $newWindowsServiceDescription $windowsServiceBinPath
{% endhighlight %}

Powyższe parametry można dostosowywać w zależności od tego którą opcję i na jakie środowisko wdrażamy nową wersję. Zapisując poszczególne konfiguracje do osobnych plików *.ps1 możemy jednym wywołaniem w konsoli PowerShell wdrożyć nową wersję:

* run_blue_host_deploy_test.ps1 - wersja blue na środowisko testowe
* run_green_host_deploy_test.ps1 - wersja green na środowisko testowe
* run_blue_host_deploy_production.ps1 - wersja blue na środowisko produkcyjne
* run_blue_host_deploy_production.ps1 - wersja green na środowisko produkcyjne

## Podsumowanie

Ten post kończy serię pokazującą jak na przykładzie funkcjonalności dodawania komentarzy do bloga można przejść przez proces wytwarzania oprogramowania od momentu zaprojektowania rozwiązania, poprzez jego realizację, testowanie, aż po wdrożenie. Widzieliśmy też jakie możliwości daje messaging and queueing oraz w jaki sposób framework NServiceBus ułatwia realizację funkcjonalności w takiej architekturze. W ostatniej części widzieliśmy przykład automatyzacji wdrażania stworzonych funkcjonalności.

[1]: https://martinfowler.com/articles/continuousIntegration.html "Continuous Integration"
[2]: https://www.atlassian.com/continuous-delivery/ci-vs-ci-vs-cd "ci-vs-ci-vs-cd"
[3]: https://docs.particular.net/transports/ "NServiceBus transports"
[4]: https://github.com/ "GitHub"
[5]: https://help.github.com/articles/about-releases/ "GitHub releases"
[6]: https://semver.org/ "Semantic versioning"
[7]: https://gitversion.readthedocs.io/en/latest/ "GitVersion"
[8]: https://en.wikipedia.org/wiki/List_of_build_automation_software "List of build automation software"
[9]: https://docs.microsoft.com/en-us/powershell/scripting/powershell-scripting?view=powershell-6 "powershell"
[10]: https://octopus.com/docs/deployment-patterns "octopus deploy deployment patterns"
[11]: https://martinfowler.com/bliki/BlueGreenDeployment.html "blue green deployment"
[12]: https://www.iis.net/ "IIS Web Server"
[13]: https://docs.microsoft.com/en-us/iis/get-started/planning-your-iis-architecture/understanding-sites-applications-and-virtual-directories-on-iis "IIS Sites, Applications, Virtual Directories"
[14]: https://docs.microsoft.com/en-us/iis/extensions/url-rewrite-module/creating-rewrite-rules-for-the-url-rewrite-module "IIS URL Rewrite"
[15]: https://logging.apache.org/log4net/ "Log4Net"
[16]: https://winscp.net/eng/docs/library "WinScp .NET Assembly and COM library"
[17]: https://docs.particular.net/nservicebus/hosting/windows-service "NServiceBus Windows Service"