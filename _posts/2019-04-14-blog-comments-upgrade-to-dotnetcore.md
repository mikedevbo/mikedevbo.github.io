---
layout: post
title:  ".NET Core - pierwsze starcie"
date:   2019-04-14
---

Gdzieś kiedyś przeczytałem/usłyszałem _"Idziesz na informatykę? Całe życie będziesz się uczył!"_. Po X latach działania w branży IT śmiało mogę powiedzieć, że to stwierdzenie jest jak najbardziej prawdziwe. Świat technologii zmienia się bardzo szybko. Nowe języki programowania, nowe biblioteki, nowe frameworki, nowe koncepcje wytwarzania oprogramowania. Choć bazują one na pewnych uniwersalnych zasadach to jednak, aby biegle się nimi posługiwać, trzeba przysiąść, pouczyć się ich oraz potrenować ich stosowanie w praktyce. Raz na jakiś czas następuje też duża zmiana w znanej nam, ustabilizowanej technologii. Wtedy również musimy odrobić pracę domową i nauczyć się, jak na nowo zrealizować coś, co do tej pory było nam znane. Przykładem takiej zmiany jest framework .NET. Poprzednia wersja zwana **.NET Framework** umożliwia wytwarzanie oprogramowania uruchamianego tylko na platformie **Windows**. Najnowsza wersja zwana **.NET Core** umożliwia wytwarzanie oprogramowania uruchamianego na wielu platformach. Jest to tylko jedna z wielu cech nowego frameworka. Więcej szczegółów można znaleźć na [stronie dokumentacji][1]. Na dzień powstawania tego artykułu najnowsza stabilna wersja .NET Core to wersja 2.2. Na horyzoncie widać już wersję 3.0. Można więc spokojnie uznać, że framework jest stabilny. Dodatkowo coraz więcej bibliotek, frameworków, platform stworzonych na podstawie .NET jest już dostępnych właśnie dla .NET Core. W związku z tym ja również postanowiłem się z nim zmierzyć, upgradując system komentarzy na blogu z wersji .NET Framework 4.7 do wersji .NET Core 2.2. Dalsza część artykułu opisuje najbardziej znaczące wyzwania, z jakimi miałem do czynienia w trakcie prac.

## Design

W tym przypadku faza Designu polegała na przygotowaniu planu upgradu. Okazało się, że strona dokumentacji zawiera [rekomendowany opis procesu][2] przejścia na .NET Core. Pozostało więc podążać wg wskazówek.

#### Analiza third-party dependencies.

System komentarzy na blogu opiera się na frameworkach **NServiceBus** oraz **Nancy**, więc pierwszym krokiem było sprawdzenie, czy można ich używać z .NET Core. Odpowiedź okazała się twierdząca:

* [NServiceBus][3]
* [Nancy][4]

Dodatkowym krokiem było sprawdzenie, czy środowiska, na które wdrażany jest system, również wspierają .NET Core. Odpowiedź także okazała się twierdząca, można więc było pójść dalej.

#### Aktualizacja do .NET Framework 4.7.2

Kolejnym krokiem była aktualizacja projektów do .NET Frameworka 4.7.2. Był to też dobry moment na aktualizację narzędzi developerskich:

* aktualizacja Visual Studio
* [instalacja .NET Core SDK][5]

#### Uruchomienie narzędzia Portability Analyzer

Ostatnim krokiem było sprawdzenie za pomocą narzędzia [Portability Analyzer][6], z jakimi elementami może być problem w trakcie migracji. Przykładowo z poniższego screena widać, że .NET Core w wersji 2.2 nie wspiera operacji pobierania wartości konfiguracyjnych oraz connectionstringów do bazy z pliku App.config za pomocą klasy **System.Configuration.ConfigurationManager.**

![Picutre1]({{ site.url }}/assets/blog_comments_upgrade_to_dotnetcore/portability_analyzer.png)

## Develop & Test

Po przejściu przez fazę Design nadszedł czas na fazę Develop, która automatycznie połączona była z fazą Test. Już przed pierwszą zmianą pojawiło się kluczowe pytanie: Czy zmieniać wszystko od razu, czy może podzielić zmianę na kawałki i przenosić jeden po drugim? System komentarzy na blogu oparty jest o styl architektoniczny zwany Messagingiem, którego jednym z głównych założeń jest budowanie luźno powiązanych ze sobą komponentów. W związku z tym lepiej było wyodrębnić te niezależne komponenty i aktualizować je jeden po drugim. Sposób podziału wynikał wprost z projektu rozwiązania:

1. Część przetwarzająca Message dodania nowego komentarza - NServiceBus Host
2. Część zgłaszająca Message dodania nowego komentarza - Nancy Web Host

Obydwa hosty mają tylko dwie części wspólne: Message Contract oraz Message Endpoint Address. Pierwszym krokiem było więc ustawienie wersji .NET Core projektu Messages.csproj.

{% highlight xml %}
<PropertyGroup>
    <TargetFramework>netcoreapp2.2</TargetFramework>
    <GenerateAssemblyInfo>false</GenerateAssemblyInfo>
</PropertyGroup>
{% endhighlight %}

### NServiceBus Host

#### Konfiguracja oraz plik App.config

Tak jak pokazał Portability Analyzer, nie można było już korzystać z klasy System.Configuration.ConfigurationManager do odczytywania wartości z pliku **App.config**. Trzeba więc było zastosować inny mechanizm. Rozwiązaniem okazało się skorzystanie z paczki Nugetowej **Microsoft.Extensions.Configuration** udostępniającej funkcjonalności czytania danych konfiguracyjnych z różnych źródeł. Ponieważ cała funkcjonalność pobierania danych konfiguracyjnych została "zamknięta" za interfejsem **IConfigurationManager**, wystarczyło w klasie **ConfigurationManager**, dziedziczącej po tym interfejsie, zmienić wewnętrzną implementację. Klasy korzystające z IConfigurationManager nie musiały nic wiedzieć o tej zmianie. Jest to klasyczny przypadek pokazujący, że używanie interfejsów jako zależności między klasami jest bardzo dobrym wzrocem.

implementacja przed zmianą

{% highlight csharp %}
public class ConfigurationManager : IConfigurationManager
{
    public string UserAgent
    {
        get
        {
            return System.Configuration
                         .ConfigurationManager
                         .AppSettings["UserAgent"];
        }

        //...
    }
}
{% endhighlight %}

implementacja po zmianie

{% highlight csharp %}
public class ConfigurationManager : IConfigurationManager
{
    private readonly IConfiguration configuration;

    public ConfigurationManager(IConfiguration configuration)
    {
        this.configuration = configuration;
    }

    public string UserAgent
    {
        get
        {
            return this.configuration["UserAgent"];
        }
    }

    //...
}
{% endhighlight %}

Do klasy ConfigurationManager została dodana zależność do interfejsu **IConfiguration**, który wchodzi w skład paczki Microsoft.Extensions.Configuration. Odwołanie do wartości konfiguracji odbywa się tak jak poprzednio poprzez property o odpowiedniej nazwie.

Skoro do konstruktora klasy ConfigurationManager została dodana zależność, to wszędzie tam, gdzie klasa ta jest tworzona, trzeba było dostarczyć konkretną implementację tej zależności. W kontekście NServiceBus Hosta były to dwa miejsca: sam Host oraz testy integracyjne. W takim przypadku widać również moc konstruowania testów jednostkowych. Ponieważ sam interfejs IConfigurationManager nie zmienił się, a w testach tych wszystkie zależności dostarczane jako interfejs są fakeowane, więc nic w tych testach nie trzeba było zmieniać. Jak w takim razie dostarczyć funkcjonalność w Hoście i testach integracyjnych oraz gdzie trzymać wartości konfiguracji? Jedną z możliwości jest czytanie danych z pliku **json**. Microsoft dostarcza implementację takiej funkcjonalności w postaci dwóch paczek Nugetowych:

* **Microsoft.Extensions.Configuration.FileExtensions**
* **Microsoft.Extensions.Configuration.Json**

testy integracyjne - implementacja przed zmianą

{% highlight csharp %}
[TestFixture]
[Ignore("only for manual tests")]
public class GitHubApiTests
{
    private readonly IConfigurationManager configurationManager =
        new Common.ConfigurationManager();

    //...
}
{% endhighlight %}

testy integracyjne - implementacja po zmianie

{% highlight csharp %}
[TestFixture]
[Ignore("only for manual tests")]
public class GitHubApiTests
{

    private readonly IConfiguration config;
    private readonly IConfigurationManager configurationManager;

    public GitHubApiTests()
    {
        this.config = new ConfigurationBuilder()
                        .SetBasePath(Directory.GetCurrentDirectory())
                        .AddJsonFile("appsettings.components.integration.tests.json", false, true)
                        .Build();

        this.configurationManager = new ConfigurationManager(this.config);
    }

    //...
}
{% endhighlight %}

Klasa **ConfigurationBuilder** służy do dostarczenia implementacji interfejsu IConfiguration, pobierającej dane z pliku **appsettings.components.integration.tests.json**, który znajduje się w tej samej lokalizacji co uruchamiany test.

host - implementacja przed zmianą

{% highlight csharp %}
//...

protected async Task AsyncOnStart()
{
    var configurationManager = new ConfigurationManager();
}

//...
{% endhighlight %}

host - implementacja po zmianie

{% highlight csharp %}
//...

protected async Task AsyncOnStart()
{
    IConfiguration config = new ConfigurationBuilder()
                    .SetBasePath(Path.GetDirectoryName(System.Reflection.Assembly.GetExecutingAssembly().Location))
                    .AddJsonFile("appsettings.host.dev.json", true, true)
                    .AddJsonFile("appsettings.host.test.json", true, true)
                    .AddJsonFile("appsettings.host.production.json", true, true)
                    .Build();

    var configurationManager = new ConfigurationManager(config);
}

//...
{% endhighlight %}

Podobnie jak przy testach integracyjnych również wykorzystana jest klasa ConfigurationBuilder do dostarczenia implementacji interfejsu IConfiguration. Dodatkowym elementem jest to, że w przypadku hosta możemy dostarczyć różne konfiguracje w zależności od tego, na jakim środowisku uruchamiamy funkcjonalność. W powyższym przykładzie do konfiguracji dodawane są trzy pliki: **appsettings.host.dev.json**, **appsettings.host.test.json** oraz **appsettings.host.production.json**. Kolejność ma znaczenie w takim sensie, że ostatni znaleziony plik wygrywa, więc jakby się zdarzyło tak, że w lokalizacji skąd uruchamiany jest Host, znalazłyby się wszystkie trzy pliki, to wartości będą czytane z pliku appsettings.host.production.json. Oczywiście przy realizacji mechanizmu wdrażania nowych wersji, trzeba zapewnić dostarczenie odpowiedniego pliku na odpowiednie środowisko, ale o tym w dalszej części artykułu.

#### Hostowanie jako Windows Service

Po uporaniu się z konfiguracją przyszedł czas na sposób wdrażania Hosta. Portability Analyzer pokazał, że nie będzie można skorzystać z klasy **System.ServiceProcess.ServiceBase** odpowiedzialnej za obsługę **Windows Service**. Ponowny research i okazało się, że klasa ta została przeniesiona do paczki Nugetowej **System.ServiceProcess.ServiceController**. Po ściągnięciu paczki niczego nie trzeba było dostosowywać, ponieważ nie było żadnych łamiących zmian, jeśli chodzi o sygnatury używanych metod. Kod od razu zaczął się kompilować.

W tym momencie wszystkie testy integracyjne oraz jednostkowe działały tak jak poprzednio. Sam NServiceBus Host również uruchamiał się bez żadnych błędów, ponieważ korzystał już z siódmej wersji NServiceBusa, która to wersja wspiera zarówno pełny .NET, jak i .NET Core. Można więc było przejść do aktualizacji Nancy Web Hosta.

### Nancy Web Host

#### ASP.NET Core jako host dla Nancy

Podobnie jak w przypadku samej platformy .NET również framework **ASP.NET** przeszedł gruntowaną przebudowę, jeśli chodzi o wsparcie dla .NET Core. W związku z tym całkowicie zmienił się sposób hostowania frameworka Nancy w nowym **ASP.NET Core**. Trzeba było utworzyć nowy projekt, wybierając w Visual Studio szablon **ASP.NET Core Web Application** i na ten projekt nanieść odpowiednie zmiany. Pierwszą było dodanie odpowiednich paczek Nugetowych.

paczki przed zmianą

{% highlight xml %}
<PackageReference Include="FluentValidation" Version="6.4.1" />
<PackageReference Include="log4net" Version="2.0.8" />
<PackageReference Include="Nancy" Version="1.4.5" />
<PackageReference Include="Nancy.Validation.FluentValidation" Version="1.4.1" />
<PackageReference Include="Nancy.Hosting.Aspnet" Version="1.4.1" />
{% endhighlight %}

paczki po zmianie

{% highlight xml %}
<PackageReference Include="Microsoft.AspNetCore.Hosting" Version="2.2.0" />
<PackageReference Include="Microsoft.AspNetCore.Server.Kestrel" Version="2.2.0" />
<PackageReference Include="Microsoft.AspNetCore.Owin" Version="2.2.0" />
<PackageReference Include="Microsoft.AspNetCore.Server.IISIntegration" Version="2.2.1" />
<PackageReference Include="Microsoft.Extensions.Configuration" Version="2.2.0" />
<PackageReference Include="Microsoft.Extensions.Configuration.FileExtensions" Version="2.2.0" />
<PackageReference Include="Microsoft.Extensions.Configuration.Json" Version="2.2.0" />
<PackageReference Include="Microsoft.Extensions.Logging.Log4Net.AspNetCore" Version="2.2.10" />
<PackageReference Include="log4net" Version="2.0.8" />
<PackageReference Include="Nancy" Version="2.0.0-clinteastwood" />
<PackageReference Include="Nancy.Validation.FluentValidation" Version="2.0.0-clinteastwood" />
{% endhighlight %}

Kolejna zmiana to sposób inicjalizowania poszczególnych elementów składających się na całość funkcjonalności. W pierwotnej wersji sercem konfiguracji była klasa **Bootstrapper**. To w niej tworzony i rejestrowany był endpoint do wysyłania wiadomości oraz logger. Definicja przekierowywania żądań HTTP do silnika Nancy znajdowała się w pliku **Web.config**. W ASP.NET Core cała konfiguracja odbywa się w klasach **Program** oraz **Startup**. Endpoint oraz logger przekazywane są jako zależności do klasy Bootstrapper, która tak jak poprzednio rejestruje te zależności w kontenerze Nancy oraz definiuje logowanie nieprzechwyconych wyjątków. Przekierowywanie żądań HTTP do silnika Nancy odbywa się poprzez interfejs [**OWIN**][7].

{% highlight csharp %}
public static class Startup
{
    //...
    
    app.UseOwin(x => x.UseNancy(opt => opt.Bootstrapper = new Bootstrapper(
        this.endpointInstance,
        commentValidator,
        logger)));
    
    //...
}
{% endhighlight %}

Niemałym wyzwaniem było również uruchomienie **Log4Neta**. Poprzednia konfiguracja poprzez wywołanie **XmlConfigurator.Configure()** nie chciała działać. Kolejny research i okazało się, że trzeba pobrać paczkę Nugetową **Microsoft.Extensions.Logging.Log4Net.AspNetCore**, rozszerzyć metodę **Configure** w klasie Startup o parametr typu **ILoggerFactory**, a następnie utworzyć loggera i przekazać go dalej jako zależność.

{% highlight csharp %}
//...

public void Configure(
    IApplicationBuilder app,
    IApplicationLifetime applicationLifetime,
    ILoggerFactory loggerFactory)
{
    //...
    
    var logger = loggerFactory.AddLog4Net().CreateLogger("Web");
    
    //...
}

//...
{% endhighlight %}

Do zmiany była również funkcjonalność pobierania wartości konfiguracyjnych. Tak jak poprzednio, można było wykorzystać interfejs IConfiguration.

{% highlight csharp %}
public class Startup
{
    private readonly IConfiguration config;
    private IEndpointInstance endpointInstance;

    public Startup(IHostingEnvironment env)
    {
        this.config = new ConfigurationBuilder()
                        .SetBasePath(Directory.GetCurrentDirectory())
                        .AddJsonFile("appsettings.web.dev.json", true, true)
                        .AddJsonFile("appsettings.web.test.json", true, true)
                        .AddJsonFile("appsettings.web.production.json", true, true)
                        .Build();
    }

    //...
}
{% endhighlight %}

Całość implementacji można zobaczyć na [GitHubie][8].

W tym momencie zarówno NServiceBus Host, jak i Nancy Web Host kompilowały się i działały prawidłowo. Można było przejść do ostatniej fazy migracji.

## Deploy

W przypadku skryptów wdrożeniowych również trzeba było dokonać kilku zmian. Najwięcej zmian przypadło na krok przygotowania artefaktów. Doszedł jeden nowy krok o nazwie **Publish**. Najmniej zmian trzeba było dokonać w krokach kopiujących i instalujących artefakty na środowisku testowym oraz produkcyjnym.

#### Build

Instalując .NET Core SDK, dostajemy nowe narzędzie o nazwie [**dotnet**][9]. W kontekście Builda systemu komentarzy na blogu narzędzie to zostało wykorzystane do skompilowania kodu źródłowego oraz uruchomienia **Unit Testów**.

kompilacja kodu przed zmianą

{% highlight powershell %}
#...
Write-Host "build solution"
$buildLogFile = "$buildArtifactsPath\$binRelativePath\build.log"
$run = $msbuildExePath
$p1 = "$buildArtifactsPath\$solutionRelativePath"
$p2 = "/p:Configuration=Release"
$p3 = "/flp:logfile=$buildLogFile"

& $run $p1 $p2 $p3
#...
{% endhighlight %}

kompilacja kodu po zmianie

{% highlight powershell %}
#...
Write-Host "build solution"
$buildLogFile = "$buildArtifactsPath\$solutionRelativePath\bin\build.log"

$path = "$buildArtifactsPath\$solutionRelativePath"

& $dotnetExePath build $path --configuration Release "/flp:logfile=$buildLogFile"
#...
{% endhighlight %}

Najważniejszym zmienionym elementem jest sposób wywołania builda. Poprzednio było to użycie narzędzia **msbuild**. Po nowemu zmienna o nazwie **$dotnetExePath** wskazuje ścieżkę, gdzie zainstalowane jest narzędzie dotnet. Następnie za pomocą komendy **build** kod ze wskazanej ścieżki **$path** kompilowany jest w konfiguracji **Release**. Tak jak poprzednio logi builda zapisywane są w miejsce wskazujące przez zmienną **$buildLogFile**.

uruchamianie Unit Testów przed zmianą

{% highlight powershell %}
#...
$tests = (Get-ChildItem $binPath -Recurse -Include *unit.tests.dll)

& $NunitExePath $tests --noheader --work=$binPath
#...
{% endhighlight %}

uruchamianie Unit Testów po zmianie

{% highlight powershell %}
#...
& $dotnetExePath test $solutionPath --no-build --configuration Release
#...
{% endhighlight %}

We wcześniejszej wersji trzeba było manualnie wskazać, w których dllkach należy szukać testów, a następnie uruchomić je za pomocą narzędzia **nunit3-console**, którego ścieżkę instalacji wskazuje zmienna **$NunitExePath**. Po zmianie samo narzędzie dotnet wie, gdzie szukać testów do uruchomienia. Znaczenie poszczególnych zmiennych:

* **$dotnetExePath** - ścieżka, gdzie zainstalowane jest narzędzie dotnet
* **test** - komenda uruchamiająca testy
* **$solutionPath** - ścieżka do skompilowanych artefaktów - tam gdzie znajduje się plik solucji **sln**
* **- -no-build** - uruchomienie testów bez ponownej kompilacji kodu
* **- -configuration Release** - uruchomienie testów w konfiguracji **Release**

#### Publish

Po wgraniu artefaktów na środowisko testowe usługa windows reprezentująca NServiceBus Hosta nie chciała się uruchomić, a strona webowa reprezentująca Nancy Web Hosta zwracała błąd. Dziwne to było zachowanie, ponieważ całość działała przy uruchamianiu z **Visual Studio**. Ponowny research i okazało się, że wynikiem kompilacji kodu pod .NET Core są tylko te elementy, które wchodzą w skład kompilowanego projektu (dllki, pliki konfiguracyjne...). Przy uruchamianiu z Visual Studio wszystkie dllki systemowe dołączane są automatycznie. Przy uruchamianiu spoza Visual Studio już tak się nie dzieje. Z tego właśnie powodu do procesu wdrażania doszedł nowy  krok o nazwie Publish.

{% highlight powershell %}
[CmdletBinding()]
Param(
    [Parameter(Mandatory=$True)]
    [string]$dotnetExePath,
	
    [Parameter(Mandatory=$True)]
    [string]$projectPath,

    [string]$rid
)

$ErrorActionPreference = "Stop"

try
{
	if ($rid -eq "")
	{
		Write-Host "publish without RID"
		& $dotnetExePath publish $projectPath --configuration Release
	}
	else
	{
		Write-Host "publish with RID: $rid"
		& $dotnetExePath publish $projectPath --configuration Release --runtime $rid
	}
	
    if(!$?)
    {
        throw "Publish failed."
    }	
}
catch
{
    throw;
}
{% endhighlight %}

Komenda **publish** narzędzia dotnet publikuje artefakty wraz ze wszystkimi wymaganymi zależnościami. Tak jak poprzednio zmienna **$dotnetExePath** wskazuje ścieżkę, gdzie zainstalowane jest narzędzie dotnet. Zmienna **$projectPath** wskazuje ścieżkę do projektu, który ma zostać opublikowany, a parametr **- -configuration** określa publikowanie w konfiguracji **Release**.

O ile w przypadku Nancy Web Hosta zwykły publish był wystarczający, tak w przypadku NServiceBus Hosta było to za mało. Usługa Windows dalej nie chciała się uruchomić. Jak sama nazwa wskazuje, usługa windows jest sposobem hostowania funkcjonalności tylko na platformie Windows. W związku z tym przy publikacji artefaktów trzeba jawnie wskazać, że host będzie uruchamiany na tej platformie, podając dodatkowy parametr **--runtime** z wartością tzw. [**Runtime IDentifier (RID)**][10] równą **win10-x64**. W powyższym skrypcie RID jest reprezentowany przez opcjonalną zmienną **$rid**.

#### Deploy NServiceBus Host & Nancy host

Ostatnim krokiem w procesie wdrażania było dostosowanie skryptów kopiujących opublikowane artefakty na docelowe środowisko. Pierwszą zmianą było zastąpienie poprzednich plików konfiguracyjnych nowymi plikami. W nazwach nowych plików pojawiły się nazwy środowisk (**test**, **production**) tak, aby przy testach skryptów mieć pewność, że odpowiednie pliki trafiły na odpowiednie środowisko. Jak widzieliśmy we wcześniejszej części artykułu, nowy mechanizm czytania plików konfiguracyjnych szuka plików właśnie o takich nazwach:

* NServiceBus Host
    * środowiska test i production przed zmianą
        * appsettings.config
        * connectionstrings.config        
        * Host.exe.config
    * środowisko test po zmianie
        * **appsettings.host.test.json**
    * środowisko production po zmianie
        * **appsettings.host.production.json**
* Nancy Web Host
    * środowiska test i production przed zmianą
        * appsettings.config
        * connectionstrings.config        
        * web.config
    * środowisko test po zmianie
        * **appsettings.web.test.json**
    * środowisko production po zmianie
        * **appsettings.web.production.json** 

Drugą zmianą było usunięcie kodu kopiującego plik connectionstrings.config. W nowej wersji connectionStringi do bazy zawarte są w pliku json razem z pozostałymi konfiguracjami.

Ostatnią krokiem była zmiana docelowej ścieżki dla artefaktów Nancy Web Hosta, ponieważ w ASP.NET Core nie ma już pliku **bin**, w którym musiały znajdować się wszystkie pliki dla wersji ASP.NET. Teraz wszystko znajduje się bezpośrednio w katalogu, który reprezentuje **IIS Web Application**.

## Podsumowanie

Patrząc na rodzaj zmian, jakich trzeba było dokonać, aby przejść na .NET Core, można zauważyć, że zmiany te dotyczyły głównie infrastruktury: zmiany w plikach csproj, nowe paczki Nugetowe, konfiguracja, hostowanie jako usługa windows, nowy bazowy framework webowy, skrypty wdrożeniowe. W kodzie implementującym logikę dodawania komentarza na blogu jako **Pull Requesta** na **GitHubie** nie trzeba było zmieniać nic. Można więc pokusić się o stwierdzenie, że logika ta jest dobrze wyizolowana i gotowa na ewentualne przyszłe tego rodzaju zmiany. Ponieważ .NET Core umożliwia wytwarzanie oprogramowania uruchamianego na wielu platformach, otwiera się furtka do nauki, aby system komentarzy na blogu uruchomić np. na platformie **Linux** lub **macOS**. To już jest jednak całkiem inna historia...

[1]: https://dotnet.microsoft.com/ ".NET"
[2]: https://docs.microsoft.com/en-us/dotnet/core/porting/ "NET CORE Porting"
[3]: https://particular.net/blog/nservicebus-7-for-net-core-is-here "nservicebus-7-for-net-core-is-here"
[4]: https://github.com/NancyFx/Nancy/tree/master/samples/Nancy.Demo.Hosting.Kestrel "Nancy.Demo.Hosting.Kestrel"
[5]: https://dotnet.microsoft.com/download/visual-studio-sdks "visual-studio-sdks"
[6]: https://docs.microsoft.com/en-us/dotnet/standard/analyzers/portability-analyzer "portability-analyzer"
[7]: http://owin.org/ "OWIN"
[8]: https://github.com/mikedevbo/blog-comments "GitHub blog-comments"
[9]: https://docs.microsoft.com/en-us/dotnet/core/tools/dotnet?tabs=netcore21 "dotnet tool"
[10]: https://docs.microsoft.com/en-us/dotnet/core/rid-catalog "RID"

{{ site.mark_post_as_end }}

### {{ site.comments }}