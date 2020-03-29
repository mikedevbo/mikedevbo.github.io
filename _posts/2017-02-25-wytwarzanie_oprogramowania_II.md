---
layout: post
title:  "Wytwarzanie oprogramowania II - techniki programowania"
description: Drugi wpis z serii „Wytwarzanie oprogramowania” zawiera przegląd wybranych technik programistycznych pozwalających dostarczać ciut lepszej jakości programy. Wszystkie przykłady napisane są w języku `C#` i dotyczą paradygmatu programowania obiektowego.
date:   2017-02-25
---
Posty z tej serii:

* [Wytwarzanie oprogramowania I - paradygmaty programowania]({% post_url 2016-11-26-wytwarzanie_oprogramowania_I %})
* [Wytwarzanie oprogramowania II - techniki programowania]({% post_url 2017-02-25-wytwarzanie_oprogramowania_II %})
* [Wytwarzanie oprogramowania III - architektury systemów informatycznych]({% post_url 2017-03-14-wytwarzanie_oprogramowania_III %})
* [Wytwarzanie oprogramowania IV - NServiceBus - framework, który zmienia zasady gry]({% post_url 2017-04-29-wytwarzanie_oprogramowania_IV %})

Drugi wpis z serii „Wytwarzanie oprogramowania” zawiera przegląd wybranych technik programistycznych pozwalających dostarczać ciut lepszej jakości programy. Wszystkie przykłady napisane są w języku `C#` i dotyczą paradygmatu programowania obiektowego.

## Testowanie

### Testy jednostkowe
Pierwszą rzecz, którą chcemy zrobić po napisaniu nowego kawałka kodu, jest sprawdzenie, czy ten kod działa zgodnie z dostarczonymi wymaganiami. Najszybszym sposobem, aby się o tym przekonać, jest napisanie tzw. testów jednostkowych.  W wielkim skrócie testy jednostkowe to również kod, który uruchamia inny kod z podanymi parametrami wejściowymi i sprawdza, czy wartości zwrócone przez ten kod zgadzają się z wartościami oczekiwanymi. Główną charakterystyką testów jednostkowych jest to, że można je uruchamiać w odseparowaniu od zewnętrznych źródeł, np. dostępu do plików, bazy danych, sieci lub innych. Takie podejście wymusza odpowiednią konstrukcję kodu, który ma być testowany. Pozwala też na sprawniejsze i pewniejsze wprowadzanie zmian do kodu oraz szybkie przekonanie się, czy zmiany te działają zgodnie z oczekiwaniami i czy nie popsuły tego, co już działało.
Przykładowo: załóżmy, że mamy do zakodowania funkcjonalność wyświetlającą listę graczy, z którymi można umówić się na mecz tenisa. Gracze ci mają spełniać obowiązkowe warunki:

* zgłaszają chęć rozegrania meczu
* posiadają aktualne badania medyczne uprawniające do gry w tenisa
* mieszczą się w tym samym zakresie w rankingu graczy np. miejsca między 100 a 200

Opis algorytmu realizującego taką funkcjonalność może wyglądać tak:

1. pobierz wszystkich graczy z bazy danych
2. wybierz tych, którzy spełniają obowiązkowe warunki
3. wyświetl na ekranie dostępnych graczy

W powyższym algorytmie można zauważyć dwa odwołania do zewnętrznych źródeł: bazy danych i interfejsu użytkownika, przez co punkty 1 i 3 nie są dobrymi kandydatami do napisania dla nich testów jednostkowych. Natomiast punkt 2 jak najbardziej. Przykładowa implementacja metody sprawdzającej warunki, może wyglądać tak:

{% highlight csharp %}
public interface IPlayerChecker
{
  bool CanBeInvitedToPlay(
    bool isActive,
    bool isValidMedicalTests,
    bool isSameRankingRange);
}

public class PlayerChecker : IPlayerChecker
{
  public bool CanBeInvitedToPlay(
    bool isActive,
    bool isValidMedicalTests,
    bool isSameRankingRange)
    {
        return isActive && isValidMedicalTests && isSameRankingRange;
    }
}
{% endhighlight %}

Metoda `CanBeInvitedToPlay` interfejsu `IPlayerChecker` na wejściu bierze trzy parametry, każdy z nich określa, czy gracz posiada daną cechę czy nie (`true,  false`) i zwraca wartość `true`, kiedy wszystkie obowiązkowe warunki są spełnione. Można zauważyć, że w sumie do sprawdzenia jest 8 przypadków. Z graczem można zagrać wtedy i tylko wtedy, gdy posiada on wszystkie obowiązkowe cechy. Jeśli chociaż jeden z warunków nie jest spełniony, to z takim graczem grać nie można. Przykładowe testy jednostkowe sprawdzające poprawność implementacji z wykorzystaniem biblioteki [NUnit][1], wspomagającej pisanie takich testów, mogą wyglądać na przykład tak:

{% highlight csharp %}
[TestFixture]
public class PlayerCheckerTests
{
  [TestCase(false, false, false, false)]
  [TestCase(false, false, true, false)]
  [TestCase(false, true, false, false)]
  [TestCase(false, true, true, false)]
  [TestCase(true, false, false, false)]
  [TestCase(true, false, true, false)]
  [TestCase(true, true, false, false)]
  [TestCase(true, true, true, true)]
  public void CanBeInvitedToPlay_Input_ProperResult(
    bool isActive,
    bool isValidMedicalTests,
    bool isSameRankingRange,
    bool expectedResult)
  {
    // Arrange
    var playerChecker = this.GetPlayerChecker();

    // Act
    var result = playerChecker.CanBeInvitedToPlay(
      isActive,
      isValidMedicalTests,
      isSameRankingRange);

    // Assert
    Assert.That(result, Is.EqualTo(expectedResult));
  }

  private IPlayerChecker GetPlayerChecker()
  {
    return new PlayerChecker();
  }
}
{% endhighlight %}

Takie testy uruchamia się jednym poleceniem i w ciągu chwili dostajemy wynik, czy nasza zakodowana funkcjonalność działa czy nie. Oczywiście testy to również kod i jak w każdym kodzie, również w tym mogą zdarzyć się błędy, dlatego tak ważne jest aby, dzielić kod na małe jednostki, testować te jednostki osobno, a następnie składać większe kawałki z tych mniejszych (jak klocki lego ;)), wtedy prawdopodobieństwo popełnienia błędu spada. Testy jednostkowe mogą być uruchamianie na serwerze build'ów przez co kod jest automatycznie testowany po wprowadzeniu każdej zmiany do systemu kontroli wersji.

### Testy integracyjne
Wiedząc, że poszczególne kawałki kodu działają, od razu nasuwa się pytanie *„no dobrze, ale czy te poskładane kawałki działają jako całość ?”* Tutaj z pomocą przychodzą testy integracyjne, których celem jest przetestowanie całości funkcjonalności wraz z dostępem do zewnętrznych źródeł danych. W odróżnieniu od testów jednostkowych, testy te wymagają większej pracy, ponieważ trzeba dla nich przygotować zestaw rzeczywistych danych do przetestowania. Dodatkowo środowisko testowe musi być przygotowane do wykonywania takich testów, np. ustawiony dostęp do bazy danych, uprawnienia do plików, skonfigurowane URL'e do zasobów w sieci itp. Czasami bardzo trudno jest zestawić wszystkie potrzebne parametry na raz, więc pewnym półśrodkiem, który dość dobrze się sprawdza, jest pisanie manualnych testów integracyjnych. Polega to na wykorzystaniu technik testów jednostkowych, tzn. wydzielenia kawałku kodu, który chcemy przetestować w całości, napisania testu, oznaczenia testu specjalnym atrybutem, aby test ten nie był uruchamiany automatycznie, uruchomieniu testu ręcznie. Przykładowo dla wymagania - pobierz wszystkich graczy z bazy danych - test taki może wyglądać na przykład tak:

{% highlight csharp %}
[TestFixture]
[Ignore("manual")]
public class QueryPlayersTests
{
  [Test]
  public void Execute_Run_ProperResult()
  {
    // Arrange
    var query = this.GetQuery();

    // Act
    var result = query.Execute().ToList();

    // Assert
    Assert.True(result.Any());
  }

  private IQueryPlayers GetQuery()
  {
    return new QueryPlayers("some connection string to database");
  }
}
{% endhighlight %}

Klasa `QueryPlayers` zawiera logikę pobierania graczy z rzeczywistej bazy danych. Podając w konstruktorze „connection string” do tej bazy, możemy sprawdzić, czy zapytanie faktycznie się wykonuje i czy zwraca jakieś wyniki.

### Testy końcowe

Jeszcze innym powodem pisania testów jest czas, w jakim jesteśmy w stanie, znaleźć błąd w kodzie. Pisząc testy jednostkowe oraz integracyjne, możemy wykryć potencjalne błędy bez potrzeby angażowania działu testów oraz końcowych użytkowników, co w znacznym stopniu skraca czas realizacji funkcjonalności. Kiedy już stwierdzimy, że funkcjonalność jest skończona, warto uruchomić system i chociaż raz przejść taką funkcjonalność, tak jak będzie to robił końcowy użytkownik. Może się wtedy okazać, że jednak o czymś zapomnieliśmy, a czego testy nie wykazały np. nadać jeszcze jakieś uprawnienie. Daje nam to całościowy obraz tego, jak funkcjonalność działa i kontekst do rozmowy z użytkownikiem w razie, gdy coś jednak będzie nie tak.

W zależności od rodzaju funkcjonalności, którą realizujemy, mogą być wymagane jeszcze inne rodzaje testów, np:

* wydajnościowe
* bezpieczeństwa
* ergonomii użytkowania
* ...

Warto przeprowadzać testy, wtedy zwiększa się prawdopodobieństwo tego, że funkcjonalność będzie działać stabilnie na środowisku produkcyjnym.

## Wzorce projektowe

W programowaniu nie ma jednego, jedynego słusznego podejścia do konstruowania kodu. Najprostszym sposobem, jaki można sobie wyobrazić, jest utworzenie jednej klasy oraz jednej metody w tej klasie i zakodowanie całej funkcjonalności w tej metodzie. Odpadają wtedy wszystkie dylematy związane z dzieleniem kodu:

* ile klas utworzyć ?
* jak podzielić to na metody ?
* jak definiować zależności między klasami ?
* ...

Jednak kodując w ten sposób, bardzo szybko dochodzi się do większych problemów:

* ten kod jest nieczytelny!
* nie da się wprowadzić zmiany do tego kodu!
* nie da się napisać testów dla tej metody!

Warto więc poświęcić czas na przemyślenie konstrukcji kodu, zwłaszcza pod kątem jego późniejszych zmian. W niektórych przypadkach okazuje się, że rozwiązanie zostało już wymyślone, dobrze opisane i nazwane. Warto więc sięgnąć do tego rozwiązania, dostosowując je do własnych potrzeb. Tak właśnie jest z tzw. [wzorcami projektowymi][2]. Dobrze przemyślana konstrukcja kodu przydaje się również w sytuacji, gdy mamy do zrealizowania kolejną funkcjonalność i z opisu wymagań wynika, że *„gdzieś już realizowałem bardzo podobną funkcjonalność i wymyślone rozwiązanie sprawdziło się”*. Czasami przydaje się implementacja jeden to jeden zgodna z definicją wzorca, a czasami wystarcza własna implementacja dostosowana do potrzeb.

### Factory method - Creational pattern

We wcześniejszym przykładzie dotyczącym testów jednostkowych do utworzenia obiektu do przetestowania wykorzystany jest wzorzec `Factory method`:

{% highlight csharp %}
private IPlayerChecker GetPlayerChecker()
{
  return new PlayerChecker();
}
{% endhighlight %}

Prosta metoda `GetPlayerChecker()` zwraca konkretną implementację interfejsu `IPlayerChecker`. Jeśli np. chcielibyśmy zastąpić implementację klasy `PlayerChecker` inną implementacją (`PlayerChecker2` ;)), to wystarczy zwrócić w tej metodzie obiekt klasy `PlayerChecker2` i uruchomić testy. Jeśli mamy więcej testów, a obiekt klasy `PlayerChecker` byłby tworzony w każdym z tych testów, to trzeba by wprowadzić tyle zmian, ile jest metod z testami (czasochłonne/pracochłonne).

### Proxy - Structural pattern

Wzorzec o nazwie `Proxy` jest użyteczny w sytuacji gdy potrzebujemy odwołać się do zewnętrznego zasobu. Przykładowo, mogą to być dane, które udostępniane są przez „zewnętrznego” dostawcę przy użyciu technologii `Web Service`, co wymaga sięgnięcia po te dane na zdalny serwer poprzez np. protokół `SOAP`. W takim przypadku `Proxy` może być reprezentowane przez klasę, która będzie zawierała szczegóły implementacji, zarówno tłumaczenia obiektów na reprezentację `XML` oraz zdalnego wywołania,  jak i tłumaczenia powrotnego - odpowiedzi i danych na obiekty.
Przykładowa implementacja `Proxy` opakowująca funkcjonalność pobierania informacji o graczach może wyglądać tak:

{% highlight csharp %}
public interface IProxyQueryPlayers
{
  List<ProxyQueryPlayers.Player> Execute();
}

[WebServiceBinding(Name="ProxyQueryPlayers", Namespace= "http://playersWebService.org/")]
public class ProxyQueryPlayers : SoapHttpClientProtocol, IProxyQueryPlayers
{
  public ProxyQueryPlayers(string url)
  {
    base.Url = url;
  }

  [SoapDocumentMethod("http://playersWebService.org/Execute",
   RequestNamespace = "http://playersWebService.org/",
   ResponseNamespace = "http://playersWebService.org/",
   Use = SoapBindingUse.Literal,
   ParameterStyle = SoapParameterStyle.Wrapped)]
  public List<Player> Execute()
  {
    object[] results = this.Invoke("Execute", new object[0]);
    return ((List<Player>)(results[0]));
  }

  public class Player
  {
    public Guid Id { get; set; }

    public string Name { get; set; }

    public bool IsActive { get; set; }

    public bool IsValidMedicalTests { get; set; }

    public int RankingPosition { get; set; }
  }
}
{% endhighlight %}

Interfejs `IProxyQueryPlayers` reprezentuje deklarację funkcjonalności pobierania danych o graczach.
Klasa `SoapHttpClientProtocol` należy do standardowej biblioteki .NET `Syste.Web.Services`. Jest ona implementacją protokołu SOAP. Klasa `ProxyQueryPlayers` dziedziczy po klasie `SoapHttpClientProtocol` oraz interfejsie `IProxyQueryPlayers` i jest implementacją `Proxy`. Konstruktor tej klasy w parametrze przyjmuje URL do zdalnego zasobu, z którego można pobrać informacje o graczach. Metoda `Execute` wywołuje metodę `Invoke`, która to metoda wykonuje zdalne wywołanie oraz tłumaczy wynik z formatu `XML` na listę obiektów typu `Player`.
Przykładowe użycie takiego `Proxy` może wyglądać tak:

{% highlight csharp %}
public class Program
{
  public static void Main(string[] args)
  {
    IProxyQueryPlayers proxyQuery = new ProxyQueryPlayers("http://host:port/WebServiceName.asmx");
    var players = proxyQuery.Execute();
    players.ForEach(player => Console.WriteLine(player.IsActive));

    Console.ReadLine();
  }
}
{% endhighlight %}

### Template - Behavioral pattern

Wzorzec ten przydaje się w sytuacjach, gdy kodujemy funkcjonalność, która składa się z powtarzalnych kroków. Część z tych kroków jest wspólna, a część jest specyficzna dla konkretnej sytuacji. Najlepiej jak kroki te są „stabilne” tzn. istnieje małe prawdopodobieństwo, że będą się zmieniać w czasie. Przykładowo, pozostając w domenie gry w tenisa, uderzenie piłki z końca linii w trwającej wymianie można opisać w krokach:

* przygotowanie pozycji wyjściowej
* przygotowanie do uderzenia (skręt ciała)
* złapanie rakiety odpowiednim uchwytem
* zamach
* uderzenie
* wykończenie

Część z tych kroków jest niezależna od typu uderzenia (forehand, backhand) oraz gracza. Zawsze trzeba zacząć od pozycji wyjściowej i zawsze trzeba uderzyć piłkę. Pozostałe kroki są ściśle powiązane z typem gracza oraz z typem uderzenia. Inny jest skręt ciała do forehand'u a inny do backhand'u. Są różne rodzaje uchwytów pozwalających celnie uderzyć piłkę itd. Tenis jako gra cały czas ewoluuje, natomiast jest mało prawdopodobne by ten bazowy wzorzec kroków zmienił się radykalnie. Implementacja takiego wzorca, z prostą funkcjonalnością wypisującą na standardowe wyjście wybrane kroki uderzenia może wyglądać tak:

{% highlight csharp %}
public interface IGroundstroke
{
  void HitBall();
}

public abstract class Groundstroke : IGroundstroke
{
  private void ReadyPosition() { Console.WriteLine("Ready position."); }

  protected abstract void Preparation();

  private void ContactPoint() { Console.WriteLine("Contact point."); }

  protected abstract void FollowThrough();

  public void HitBall()
  {
    this.ReadyPosition();
    this.Preparation();
    this.ContactPoint();
    this.FollowThrough();
  }
}

public class Forehand : Groundstroke
{
  protected override void Preparation()
  {
    Console.WriteLine("Forehand preparation.");
  }

  protected override void FollowThrough()
  {
    Console.WriteLine("Forehand follow through.");
  }
}

public class TwoHandedBackHand : Groundstroke
{
  protected override void Preparation()
  {
    Console.WriteLine("Backhand preparation.");
  }

  protected override void FollowThrough()
  {
    Console.WriteLine("Backhand follow through.");
  }
}

public class Program
{
  IGroundstroke forehand = new Forehand();
  forehand.HitBall();

  IGroundstroke backhand = new TwoHandedBackHand();
  backhand.HitBall();

  Console.ReadLine();
}
{% endhighlight %}

Wspólne kroki uderzenia, niezależnie od tego, czy jest to forehand czy backhand, reprezentowane są przez metody `ReadyPosition()` oraz `ContactPoint` abstrakcyjnej klasy o nazwie `Groundstroke`, którą to klasę można traktować jako `Template`. Kroki różniące się w zależności od rodzaju uderzenia to kroki reprezentowane przez metody `Preparation()` oraz `FollowThrough()`, których implementacja znajduje się w klasach `Forehand` oraz `Backhand` dziedziczących po klasie `Groundstroke`.
Metoda `HitBall()` należąca do interfejsu `IGroundstroke` reprezentuje uderzenie piłki tenisowej.

## Wzorce aplikacyjne

Przy konstruowaniu kodu równie pomocne mogą okazać się [Wzorce aplikacyjne][3], które oprócz wskazania jak można układać kod w klasy, opisują, w jaki sposób można wyodrębniać i łączyć różne elementy składające się na działanie systemu, np. funkcjonalności interfejsu użytkownika, logiki działania oraz dostępu do bazy danych. Podobnie jak w przypadku wzorców projektowych, w pewnych sytuacjach przydaje się pełna implementacja zgodna z definicją, a w innych wystarcza częściowa.

### Query Object

We wcześniejszych przykładach dotyczących pobierania danych o graczach wykorzystywany jest obiekt `Query`.  W implementacji klasy takiego obiektu można zaszyć zarówno logikę pobierania danych, np. z bazy danych, jak i klasę reprezentującą pobrane dane. W ten sposób całość możemy „zamknąć” w jednej klasie i traktować `Query` jak komponent odpowiedzialny za pobranie konkretnych danych.

{% highlight csharp %}
public interface IDbQueryPlayers
{
  IList<DbQueryPlayers.Player> Execute();
}

public class DbQueryPlayers
{
  private readonly string connectionString;

  public DbQueryPlayers(string connectionStrig)
  {
    this.connectionString = connectionStrig;
  }

  public IList<Player> Execute()
  {
    var result = new List<Player>();

    ////result = "use some database access technology"

    return result;
  }

  public class Player
  {
    public Guid Id { get; set; }

    public string Name { get; set; }

    public bool IsActive { get; set; }

    public bool IsValidMedicalTests { get; set; }

    public int RankingPosition { get; set; }
  }
}
{% endhighlight %}

Patrząc na wcześniejszy przykład wzorca `Proxy`, można zauważyć, że do jego implementacji został wykorzystany wzorzec `Query` dostosowany do funkcjonalności pobierania danych ze zdalnego zasobu, także nic nie stoi na przeszkodzie, aby łączyć ze sobą różne wzorce w celu uzyskania pożądanego efektu.

### Model View Controller (MVC)

Ostatni opisany wzorzec, w tym już dość długim poście, to wzorzec pomagający w zaprojektowaniu interakcji między interfejsem użytkownika (UI), jego logiką a pozostałą częścią systemu. Większość technologii UI opiera się na wyświetlaniu zawartości (grafiki, treści, ..) oraz interakcji z użytkownikiem poprzez zdarzenia. Użytkownik klika guzik, generując zdarzenie, po którym wyświetla mu się odpowiednia zawartość strony (treść). Innym przykładem jest wypełnienie formularza, a następnie kliknięcie guzika „wyślij”, po którym następuje przesłanie zawartości formularza do przetworzenia. Mieszając kod logiki z kodem interfejsu użytkownika, łatwo można wpaść w pułapkę niemożliwości przetestowania funkcjonalności bez uruchomienia systemu i jego „przeklikaniu” (co samo w sobie jest bardzo wskazane, zwłaszcza w końcowej fazie realizacji). Wzorzec MVC (lub jego odmiana Model View Presenter) umożliwia uniknięcie tej pułapki. Dzięki odseparowaniu zachowania danego interfejsu od sposobu jego wyświetlania, można nawet zacząć kodowanie od pierwszej części, skupiając się na poprawności działania, bez konieczności natychmiastowego kodowania kontrolek UI. Trzymając się przykładu pobierania informacji o graczach tenisa, wzorzec MVC można zakodować na przykład tak:

{% highlight csharp %}
public interface IViewShowPlayers
{
  void Show(IList<PresenterShowPlayers.PlayerViewModel> players);
}

public interface IPresenterShowPlayers
{
  void Show();
}

public class PresenterShowPlayers : IPresenterShowPlayers
{
  private readonly IViewShowPlayers view;

  public PresenterShowPlayers(IViewShowPlayers view)
  {
    this.view = view;
  }

  public void Show()
  {
    var result = new List<PlayerViewModel>();

    ////result = "use some query to get players and map result to view model"

    this.view.Show(result);
  }

  public class PlayerViewModel
  {
    public Guid Id { get; set; }

    public string Name { get; set; }

    public bool IsActive { get; set; }

    public bool IsValidMedicalTests { get; set; }

    public int RankingPosition { get; set; }
  }
}
{% endhighlight %}

Interfejs `IViewShowPlayers` reprezentuje deklarację widoku, który będzie wyświetlał informacje o graczach po wywołaniu metody `Show(IList<PresenterShowPlayers.PlayerViewModel> players)`. Metodę tę wywołuje prezenter, którego deklarację reprezentuje interfejs `IPresenterShowPlayers` a definicję klasa `PresenterShowPlayers`. Jeśli wymaganie opisuje ładowanie informacji o graczach po kliknięciu guzika na interfejsie, to w momencie, gdy użytkownik klika guzik widok wywołuje metodę `Show` prezentera. Prezenter pobiera dane o graczach przez `Proxy` ze zdalnego serwera lub przez `Query` bezpośrednio z bazy oraz mapuje wyniki na model `PlayerViewModel`, który to model przekazywany jest do widoku poprzez wywołanie metody `Show` na tym widoku. W ten sposób można odseparować renederowanie danych od sposobu pobierania danych. Klasę `PresenterShowPlayers` można przetestować testami jednostkowymi bez konieczności uruchamiania systemu.

## Podsumowanie

Powyższe przykłady są tylko niewielką częścią zagadnień, na temat których powstało wiele książek oraz artykułów. Oczywiście nie ma żadnego przymusu stosowania wzorców w każdej implementacji, a nawet stosowania testów czy to jednostkowych, czy też integracyjnych. Czasami samo „przeklikanie” funkcjonalności w zupełności wystarcza, natomiast kod powinien być zawsze tak napisany, aby w każdym momencie można było do niego łatwo dopisać testy. Gdy jednak okazuje się, że w kodowanej funkcjonalności jest wiele ścieżek przebiegu, testy jednostkowe pozwalają uzyskać efekt ***„o k...! jak bym to puścił/puściła na produkcję to byłoby niewesoło, dobrze, że napisałem/napisałam do tego testy!”***. Podobnie jest ze wzorcami. Jeśli funkcjonalność jest prosta, to może do jej realizacji wystarczą dwie metody, zamiast implementacja złożonego wzorca, natomiast jak funkcjonalność jest bardziej złożona, lub z prostej staje się złożona, to warto przeanalizować rozwiązanie pod kątem użycia konkretnego wzorca. Uniwersalną zasadą jest zasada, aby kod był prosty i czytelny, tak aby w późniejszym czasie można było go w miarę łatwo modyfikować, dlatego w trakcie kodowania warto zadawać sobie pytanie: ***„Czy jakbym miał/miała zmieniać coś w tym kodzie za pół roku, to będzie to dla mnie przyjemnością czy męką :)”***. Kolejna część cyklu będzie dotyczyć rodzajów aplikacji oraz sposobów ich realizacji.

[1]: https://www.nunit.org "NUnit"
[2]: http://wazniak.mimuw.edu.pl/index.php?title=Io-8-wyk-toc "Wzorce projektowe"
[3]: https://www.martinfowler.com/eaaCatalog/ "Wzorce aplikacyjne"

{{ site.mark_post_as_end }}

### {{ site.comments }}