---
layout: post
title:  "data-access"
date:   2018-11-13
---

Kodując tzw. "backend" w biznesowych systemach informatycznych w 90% mamy do czynienia z operacjami na bazie danych. Potrzebny jest zatem sposób w jaki możemy się z bazą danych porozumieć. Artykuł ten jest wycieczką po bibliotekach/technologiach umożliwiającymi kontakt z bazą danych z którymi miałem do czynienia przy różnych projektach. Poniższe przykłady są implementacją w języku C# na platformie .NET oraz zakładają połączenie z relacyjną bazą danych [MS SQL Server][1]. Całość przedstawiona jest kontekście operowania na danych dotyczących informacji o graczu tenisa:

* pobranie informacji o graczu - select z widoku bazodanowego
* dodanie nowego gracza - insert danych to tabeli
* ustawienie trenera gracza - update wybranych danych z [Optimistic Offline Lock][7]

## [ADO.NET][2]

Początki to najbardziej podstawowy sposób łączenia się z bazą danych czyli ADO.NET. W przykładach kod SQL zawarty jest w kodzie C#. W rzeczywistości kod SQL pisany był w procedurach składowanych tak aby nie mieszać go z kodem C#. ADO.NET umożliwia wywoływanie procedur składowanych poprzez odpowiednie ustawienie obiektu SQLCommand:

**command.CommandType = CommandType.StoredProcedure;**

Mapowanie danych z bazy na obiekt wygląda tak samo w obydwu przypadkach.

* select

{% highlight csharp %}
public PlayersBaseInfoView GetPlayerBaseInfo(int playerId)
{
    PlayersBaseInfoView result = null;

    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        using (var command = new SqlCommand())
        {
            command.Connection = connection;
            command.CommandType = CommandType.Text;

            command.CommandText =
                "select " +
                "  pbi.id" +
                ", pbi.FirstName" +
                ", pbi.LastName" +
                ", pbi.BirthYear" +
                ", pbi.BirthMonth" +
                ", pbi.BirthDay" +
                ", pbi.BirthplaceCountry" +
                ", pbi.BirthplaceCity" +
                ", pbi.[Weight]" +
                ", pbi.Height" +
                ", pbi.IsRightHanded" +
                ", pbi.IsTwoHandedBackhand" +
                ", pbi.CoachId" +
                ", pbi.CoachFirstName" +
                ", pbi.CoachLastName " +
                "from " +
                "  dbo.PlayersBaseInfo pbi " +
                "where " +
                "  pbi.Id = @Id";

            command.Parameters.Add("@Id", SqlDbType.Int, 4).Value = playerId;

            connection.Open();

            using (var reader = command.ExecuteReader())
            {
                if (reader.Read())
                {
                    result = new PlayersBaseInfoView
                    {
                        Id = Convert.ToInt32(reader["Id"]),
                        FirstName = reader["FirstName"].ToString(),
                        LastName = reader["LastName"].ToString(),
                        BirthYear = Convert.ToInt32(reader["BirthYear"]),
                        BirthMonth = Convert.ToInt32(reader["BirthMonth"]),
                        BirthDay = Convert.ToInt32(reader["BirthDay"]),
                        BirthplaceCountry = reader["BirthplaceCountry"].ToString(),
                        BirthplaceCity = reader["BirthplaceCity"].ToString(),
                        Weight = Convert.ToInt32(reader["Weight"]),
                        Height = Convert.ToInt32(reader["Height"]),
                        IsRightHanded = Convert.ToBoolean(reader["IsRightHanded"]),
                        IsTwoHandedBackhand = Convert.ToBoolean(reader["IsTwoHandedBackhand"]),
                        CoachId = reader["CoachId"] == DBNull.Value ? default(int?) : Convert.ToInt32(reader["CoachId"]),
                        CoachFirstName = reader["CoachFirstName"] == DBNull.Value ? null : reader["CoachFirstName"].ToString(),
                        CoachLastName = reader["CoachLastName"] == DBNull.Value ? null : reader["CoachLastName"].ToString(),
                    };
                }
            }
        }
    }

    return result;
}
{% endhighlight %}

* insert

{% highlight csharp %}
public void AddPlayer(int personId, bool isRightHanded, bool isTwoHandedBackhand)
{
    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        using (var command = new SqlCommand())
        {
            command.Connection = connection;
            command.CommandType = CommandType.Text;

            command.CommandText =
                "insert into dbo.Player(id, IsRightHanded, IsTwoHandedBackhand, CoachId) " +
                "values(@Id, @IsRightHanded, @IsTwoHandedBackhand, null)";

            command.Parameters.Add("@Id", SqlDbType.Int, 4).Value = personId;
            command.Parameters.Add("@IsRightHanded", SqlDbType.Bit).Value = isRightHanded;
            command.Parameters.Add("@IsTwoHandedBackhand", SqlDbType.Bit).Value = isTwoHandedBackhand;

            connection.Open();

            command.ExecuteNonQuery();
        }
    }
}
{% endhighlight %}

* update

{% highlight csharp %}
public void SetPlayerCoach(int playerId, int? newCoachId, int? previousCoachId)
{
    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        using (var command = new SqlCommand())
        {
            command.Connection = connection;
            command.CommandType = CommandType.Text;

            command.CommandText =
                "update " +
                "   dbo.Player " +
                "set " +
                "   CoachId = @newCoachId " +
                "where Id = @Id and " +
                    string.Format("CoachId {0}", previousCoachId.HasValue ? "= @previousCoachId" : "is null");

            command.Parameters.Add("@Id", SqlDbType.Int, 4).Value = playerId;

            if (newCoachId.HasValue)
            {
                command.Parameters.Add("@newCoachId", SqlDbType.Int, 4).Value = newCoachId;
            }
            else
            {
                command.Parameters.Add("@newCoachId", SqlDbType.Int, 4).Value = DBNull.Value;
            }

            if (previousCoachId.HasValue)
            {
                command.Parameters.Add("@previousCoachId", SqlDbType.Int, 4).Value = previousCoachId.Value;
            }

            connection.Open();

            var result = command.ExecuteNonQuery();

            if (result == 0)
            {
                throw new DbUpdateConcurrencyException();
            }
        }
    }
}
{% endhighlight %}

## [Enterprise Library - Data Access Application Block][3]

Używając ADO.NET bardzo szybko dochodzi się do wniosku, że trzeba napisać sporo bardzo podobnego oraz powtarzalnego kodu. Najbardziej mechanicznym i podatnym na błędy kodem jest mapowanie zwróconych danych z bazy na poszczególne property obiektu reprezentującego wynik. W takich sytuacjach opcje są co najmniej dwie: napisać własny generyczny kod lub poszukać biblioteki realizującej taką funkcjonalność. W moim przypadku było to skorzystanie z biblioteki Enterprise Library - Data Access Application Block umożliwiającej takie mapowanie poprzez wywołanie:

**var accessor = db.CreateSqlStringAccessor&lt;PlayersBaseInfoView&gt;(sql, parameterMapper);**

* select

{% highlight csharp %}
public PlayersBaseInfoView GetPlayerBaseInfo(int playerId)
{
    var db = this.GetDatabase();

    const string sql =
        "select " +
        "  pbi.id" +
        ", pbi.FirstName" +
        ", pbi.LastName" +
        ", pbi.BirthYear" +
        ", pbi.BirthMonth" +
        ", pbi.BirthDay" +
        ", pbi.BirthplaceCountry" +
        ", pbi.BirthplaceCity" +
        ", pbi.[Weight]" +
        ", pbi.Height" +
        ", pbi.IsRightHanded" +
        ", pbi.IsTwoHandedBackhand" +
        ", pbi.CoachId" +
        ", pbi.CoachFirstName" +
        ", pbi.CoachLastName " +
        "from " +
        "  dbo.PlayersBaseInfo pbi " +
        "where " +
        "  pbi.Id = @Id";

    var parameterMapper = new PlayersBaseInfoParameterMapper();
    var accessor = db.CreateSqlStringAccessor<PlayersBaseInfoView>(sql, parameterMapper);

    return accessor.Execute(playerId).FirstOrDefault();
}

public class PlayersBaseInfoParameterMapper : IParameterMapper
{
    public void AssignParameters(DbCommand command, object[] parameterValues)
    {
        DbParameter parameter = command.CreateParameter();
        parameter.ParameterName = "@Id";
        parameter.Value = parameterValues[0];
        command.Parameters.Add(parameter);
    }
}

private Database GetDatabase()
{
    return new DatabaseProviderFactory().Create(Helper.ConnectionName);
}
{% endhighlight %}

* insert

{% highlight csharp %}
public void AddPlayer(int personId, bool isRightHanded, bool isTwoHandedBackhand)
{
    var db = this.GetDatabase();

    using (var command = new SqlCommand())
    {
        command.CommandType = CommandType.Text;

        command.CommandText =
            "insert into dbo.Player(id, IsRightHanded, IsTwoHandedBackhand, CoachId) " +
            "values(@Id, @IsRightHanded, @IsTwoHandedBackhand, null)";

        command.Parameters.Add("@Id", SqlDbType.Int, 4).Value = personId;
        command.Parameters.Add("@IsRightHanded", SqlDbType.Bit).Value = isRightHanded;
        command.Parameters.Add("@IsTwoHandedBackhand", SqlDbType.Bit).Value = isTwoHandedBackhand;

        db.ExecuteNonQuery(command);
    }
}
{% endhighlight %}

* update

{% highlight csharp %}
public void SetPlayerCoach(int playerId, int? newCoachId, int? previousCoachId)
{
    var db = this.GetDatabase();

    using (var command = new SqlCommand())
    {
        command.CommandType = CommandType.Text;

        command.CommandText =
            "update " +
            "   dbo.Player " +
            "set " +
            "   CoachId = @newCoachId " +
            "where Id = @Id and " +
                string.Format("CoachId {0}", previousCoachId.HasValue ? "= @previousCoachId" : "is null");

        command.Parameters.Add("@Id", SqlDbType.Int, 4).Value = playerId;

        if (newCoachId.HasValue)
        {
            command.Parameters.Add("@newCoachId", SqlDbType.Int, 4).Value = newCoachId;
        }
        else
        {
            command.Parameters.Add("@newCoachId", SqlDbType.Int, 4).Value = DBNull.Value;
        }

        if (previousCoachId.HasValue)
        {
            command.Parameters.Add("@previousCoachId", SqlDbType.Int, 4).Value = previousCoachId.Value;
        }

        var result = db.ExecuteNonQuery(command);

        if (result == 0)
        {
            throw new DbUpdateConcurrencyException();
        }
    }
}
{% endhighlight %}

## ORM

Używając dwóch powyższych podejść w wielu projektach a także kodując większość logiki biznesowej w języku C# nasunął mi się wniosek, że tak naprawdę 90% kodu SQL to są podstawowe operacje select, insert oraz update pisane w powtarzalny, niemal automatyczny sposób. Pojawiła się więc myśl czy istnieją rozwiązania, które mogą taki kod SQL generować automatycznie. Okazało się, że tak i że takie podejście można zrealizować w tzw. [Object Relational Mapping][8]. Niestety w moim przypadku bariera wejścia okazała się zbyt duża, ponieważ materiały na które natrafiałem sugerowały, że aby poprawnie używać ORMa najlepiej jest projektować rozwiązania w oparciu o model dziedzinowy co samo w sobie było wyzwaniem. (w dalszej części artykułu zobaczymy, że jednak da się inaczej :))

Jako ćwiczenie zachęcam do zakodowania trzech operacji zdefiniowanych na początku artykułu używając np. biblioteki [NHibernate][9].

## [Dapper][4]

Trochę zniechęcony powróciłem do dobrze znanego mi ADO oraz EntLiba. W między czasie usłyszałem o bibliotece zwanej Dapper. Szybki research i okazało się, że podobnie jak EntLib, Dapper również potrafi mapować dane z bazy na obiekt a dodatkowo sam zarządza odpowiednim konstruowaniem parametrów wejściowych zapytań SQL przez co napisany kod upraszcza się. Ponieważ dalej to my definiujemy kod SQL to w przypadku operacji update aby zachować Optimistic Offline Lock, tak jak poprzednio, musimy skonstruować odpowiednie zapytanie w zależności od tego czy poprzednia wartość na której opieramy implementację ma wartość null czy nie. W przykładach jest to wartość zmiennej **previousCoachId**. Kluczową właściwością Dappera jest jego szybkość działania. Porównanie wyników z innymi bibliotekami można zobaczyć na stronie dokumentacji.

* select

{% highlight csharp %}
public PlayersBaseInfoView GetPlayerBaseInfo(int playerId)
{
    PlayersBaseInfoView result = null;

    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        result = connection.Query<PlayersBaseInfoView>(
            "select " +
            "  pbi.id" +
            ", pbi.FirstName" +
            ", pbi.LastName" +
            ", pbi.BirthYear" +
            ", pbi.BirthMonth" +
            ", pbi.BirthDay" +
            ", pbi.BirthplaceCountry" +
            ", pbi.BirthplaceCity" +
            ", pbi.[Weight]" +
            ", pbi.Height" +
            ", pbi.IsRightHanded" +
            ", pbi.IsTwoHandedBackhand" +
            ", pbi.CoachId" +
            ", pbi.CoachFirstName" +
            ", pbi.CoachLastName " +
            "from " +
            "  dbo.PlayersBaseInfo pbi " +
            "where " +
            "  pbi.Id = @Id",
            new { Id = playerId }).FirstOrDefault();
    }

    return result;
}
{% endhighlight %}

* insert

{% highlight csharp %}
public void AddPlayer(int personId, bool isRightHanded, bool isTwoHandedBackhand)
{
    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        connection.Execute(
            "insert into dbo.Player(id, IsRightHanded, IsTwoHandedBackhand, CoachId) " +
            "values(@Id, @IsRightHanded, @IsTwoHandedBackhand, null)",
            new { Id = personId, isRightHanded, isTwoHandedBackhand });
    }
}
{% endhighlight %}

* update

{% highlight csharp %}
public void SetPlayerCoach(int playerId, int? newCoachId, int? previousCoachId)
{
    using (var connection = new SqlConnection(ConfigurationManager.ConnectionStrings[Helper.ConnectionName].ConnectionString))
    {
        var result = connection.Execute(
            "update " +
            "   dbo.Player " +
            "set " +
            "   CoachId = @newCoachId " +
            "where Id = @Id and " +
                string.Format("CoachId {0}", previousCoachId.HasValue ? "= @previousCoachId" : "is null"),
            new { Id = playerId, newCoachId, previousCoachId });

        if (result == 0)
        {
            throw new DbUpdateConcurrencyException();
        }
    }
}
{% endhighlight %}

## [Simple.Data][5]

W nowym projekcie zapadła decyzja aby jeden z podsystemów zrealizować w oparciu o platformę [Particular.net][10], której sercem jest framework [NServiceBus][11]. W tamtym czasie jego najnowszą wersją była wersja v5. Wybór technologii dostępu do bazy danych miał być prosty - Dapper, ale natrafiłem na bibliotekę Simple.Data. Okazała się ona strzałem w dziesiątkę. Spełniała wszystkie kryteria, których potrzebowałem. Pierwszą istotną zmianą w stosunku do poprzedników (nie wliczając ORMa) jest to, że kod SQL generowany jest automatycznie przez co zauważalnie zmniejsza się ilość kodu C# do napisania. Drugą istotną zmianą jest to, że Simple.Data opiera się o typ **dynamic** wprowadzony w **C# 4.0**. Trzecią istotną zmianą jest sposób w jaki konstruowany jest update z zachowaniem Optimistic Offline Lock. Metoda **UpdateAll** ze zdefiniowanym odpowiednim **Condition** sama wygeneruje odpowiedni warunek **WHERE** w zależności od tego czy poprzednia wartość zmiennej na której opieramy implementację (**previousCoachId**) ma wartość null czy nie.

* select

{% highlight csharp %}
public PlayersBaseInfoView GetPlayerBaseInfo(int playerId)
{
    dynamic db = this.GetDatabase();
    return db.PlayersBaseInfo.FindAllById(playerId).FirstOrDefault();
}

private Database GetDatabase()
{
    return Database.OpenNamedConnection(Helper.ConnectionName);
}
{% endhighlight %}

Wygenerowany kod SQL

{% highlight sql %}
N'select  TOP 1 [dbo].[PlayersBaseInfo].[id],[dbo].[PlayersBaseInfo].[FirstName],[dbo].[PlayersBaseInfo].[LastName],[dbo].[PlayersBaseInfo].[BirthYear],[dbo].[PlayersBaseInfo].[BirthMonth],[dbo].[PlayersBaseInfo].[BirthDay],[dbo].[PlayersBaseInfo].[BirthplaceCountry],[dbo].[PlayersBaseInfo].[BirthplaceCity],[dbo].[PlayersBaseInfo].[Weight],[dbo].[PlayersBaseInfo].[Height],[dbo].[PlayersBaseInfo].[IsRightHanded],[dbo].[PlayersBaseInfo].[IsTwoHandedBackhand],[dbo].[PlayersBaseInfo].[CoachId],[dbo].[PlayersBaseInfo].[CoachFirstName],[dbo].[PlayersBaseInfo].[CoachLastName] 
from [dbo].[PlayersBaseInfo] 
WHERE [dbo].[PlayersBaseInfo].[id] = @p1',N'@p1 int',@p1=1
{% endhighlight %}

* insert

{% highlight csharp %}
public void AddPlayer(int personId, bool isRightHanded, bool isTwoHandedBackhand)
{
    var player = new PlayerTable
    {
        Id = personId,
        IsRightHanded = isRightHanded,
        IsTwoHandedBackhand = isTwoHandedBackhand
    };

    dynamic db = this.GetDatabase();
    db.Player.Insert(player);
}
{% endhighlight %}

Wygenerowany kod SQL

{% highlight sql %}
N'INSERT INTO [dbo].[Player] ([Id],[IsRightHanded],[IsTwoHandedBackhand],[CoachId])
VALUES (@p0,@p1,@p2,@p3)',
N'@p0 int,@p1 bit,@p2 bit,@p3 int',@p0=1,@p1=1,@p2=1,@p3=NULL
{% endhighlight %}

* update

{% highlight csharp %}
public void SetPlayerCoach(int playerId, int? newCoachId, int? previousCoachId)
{
    dynamic db = this.GetDatabase();

    var result = db.Player.UpdateAll(
        CoachId: newCoachId,
        Condition: (db.Player.Id == playerId) && (db.Player.CoachId == previousCoachId));

    if (result.ReturnValue == 0)
    {
        throw new DbUpdateConcurrencyException();
    }
}
{% endhighlight %}

Wygenerowany kod SQL dla previousCoachId == null

{% highlight sql %}
N'update [dbo].[Player] set [CoachId] = @p1 
where ([dbo].[Player].[Id] = @p2 AND [dbo].[Player].[CoachId] IS NULL)',
N'@p1 int,@p2 int',@p1=2,@p2=1
{% endhighlight %}

Wygenerowany kod SQL dla previousCoachId != null

{% highlight sql %}
N'update [dbo].[Player] set [CoachId] = @p1 
here ([dbo].[Player].[Id] = @p2 AND [dbo].[Player].[CoachId] = @p3)',
N'@p1 int,@p2 int,@p3 int',@p1=3,@p2=1,@p3=2
{% endhighlight %}

## [Entity Framework][6]

Kolejny projekt, kolejne decyzje projektowe. Pierwszy wybór to oczywiście realizacja funkcjonalności w oparciu o platformę Particular.net oraz framework NServiceBus. W tym czasie zespół rozwijający framework wydał jego v6 wersję wprowadzając [wiele nowości][12]. Jedną z nich jest pełne wsparcie [C# 5.0 async/await][13]. W kontekście wyboru sposobu łączenia się z bazą danych wskazanym było mieć bibliotekę, która również wspierała asynchroniczność przy wywoływaniu zapytań do bazy. Simple.Data jako produkt skończony nie oferowała takiego wsparcia, trzeba więc było wybrać coś innego. Pierwsza myśl - powrót do Dappera. Druga myśl - "A może dać ponownie szansę podejściu ORM?". Decyzja o ponownym wykorzystaniu ORMa nie była łatwa, zwłaszcza po wcześniejszych niezbyt udanych doświadczeniach. W ramach tzw. [Spikea][14] postawiłem bibliotece takie oto wymagania do spełnienia aby zdecydować się na jej wybór:

* chcę jej używać jako warstwa dostępu do bazy danych tak samo jak pozostałe biblioteki w oderwaniu od sposobu projektowania rozwiązania
* wsparcie dla trzech operacji wymienionych na początku tego artykułu:
    * select z widoku bazodanowego
    * insert do tabeli
    * update wybranych danych z Optimistic Offline Lock
* w razie potrzeby chcę mieć możliwość napisania własnego SQLa lub wywołania procedury składowanej
* kod C# ma być prosty i czytelny

Po zakończeniu Spikea okazało się, że Entity Framework spełnia wszystkie te wymagania a dodatkowo sprawdza się przy projektowaniu rozwiązań opartych o messaging. Podobnie jak w przypadku Simple.Data kod SQL generowany jest automatycznie. W przypadku konstruowania operacji update z Optimistic Offline Lock odpowiedni warunek **WHERE** również generowany jest automatycznie w zależności od tego czy poprzednia wartość zmiennej na której opieramy implementację ma wartość null czy nie. Różnicą w porównaniu do Simple.Data jest sposób poinstruowania Entity Frameworka, które property obiektu ma być uwzględnione przy generowaniu zapytania SQL. Takie property należy oznaczyć specjalnym atrybutem:

{% highlight csharp %}
[ConcurrencyCheck]
public int? CoachId { get; set; }
{% endhighlight %}

Nie trzeba również ręcznie zgłaszać wyjątku **DbUpdateConcurrencyException**. Całość dzieje się automatycznie.

Jeśli udało ci się zakodować rozwiązanie z sekcji ORM z użyciem np. NHibernatea to zapewne również tej biblioteki można używać w takim podejściu.

* select

{% highlight csharp %}
public Task<PlayersBaseInfoView> GetPlayerBaseInfo(int playerId)
{
    return this.PlayersBaseInfoView.FirstOrDefaultAsync(p => p.Id == playerId);
}
{% endhighlight %}

Wygenerowany kod SQL

{% highlight sql %}
N'SELECT TOP (1) 
    [Extent1].[Id] AS [Id], 
    [Extent1].[FirstName] AS [FirstName], 
    [Extent1].[LastName] AS [LastName], 
    [Extent1].[BirthYear] AS [BirthYear], 
    [Extent1].[BirthMonth] AS [BirthMonth], 
    [Extent1].[BirthDay] AS [BirthDay], 
    [Extent1].[BirthplaceCountry] AS [BirthplaceCountry], 
    [Extent1].[BirthplaceCity] AS [BirthplaceCity], 
    [Extent1].[Weight] AS [Weight], 
    [Extent1].[Height] AS [Height], 
    [Extent1].[IsRightHanded] AS [IsRightHanded], 
    [Extent1].[IsTwoHandedBackhand] AS [IsTwoHandedBackhand], 
    [Extent1].[CoachId] AS [CoachId], 
    [Extent1].[CoachFirstName] AS [CoachFirstName], 
    [Extent1].[CoachLastName] AS [CoachLastName]
    FROM [dbo].[PlayersBaseInfo] AS [Extent1]
    WHERE [Extent1].[Id] = @p__linq__0',N'@p__linq__0 int',@p__linq__0=1
{% endhighlight %}

* insert

{% highlight csharp %}
public Task AddPlayer(int personId, bool isRightHanded, bool isTwoHandedBackhand)
{
    var player = new PlayerTable
    {
        Id = personId,
        IsRightHanded = isRightHanded,
        IsTwoHandedBackhand = isTwoHandedBackhand
    };

    this.PlayerTable.Add(player);

    return this.SaveChangesAsync();
}
{% endhighlight %}

Wygenerowany kod SQL

{% highlight sql %}
N'INSERT [dbo].[Player]([Id], [IsRightHanded], [IsTwoHandedBackhand], [CoachId])
VALUES (@0, @1, @2, NULL)
',N'@0 int,@1 bit,@2 bit',@0=1,@1=1,@2=1
{% endhighlight %}

* update

{% highlight csharp %}
public Task SetPlayerCoach(int playerId, int? newCoachId, int? previousCoachId)
{
    var player = new PlayerTable { Id = playerId };
    this.PlayerTable.Attach(player);

    this.Entry(player).Property(p => p.CoachId).OriginalValue = previousCoachId;
    player.CoachId = newCoachId;

    return this.SaveChangesAsync();
}
{% endhighlight %}

Wygenerowany kod SQL dla previousCoachId == null

{% highlight sql %}
N'UPDATE [dbo].[Player]
SET [CoachId] = @0
WHERE (([Id] = @1) AND [CoachId] IS NULL)
',N'@0 int,@1 int',@0=2,@1=1
{% endhighlight %}

Wygenerowany kod SQL dla previousCoachId != null

{% highlight sql %}
N'UPDATE [dbo].[Player]
SET [CoachId] = @0
WHERE (([Id] = @1) AND ([CoachId] = @2))
',N'@0 int,@1 int,@2 int',@0=3,@1=1,@2=2
{% endhighlight %}

## Podsumowanie

Znając różne możliwości łączenia się z bazą danych jesteśmy bardziej elastyczni w konstruowaniu rozwiązań. 
Przykładowo jeśli całość oparta jest na jednym podejściu, a któreś z zapytań do bazy danych trwa zbyt długo, to można tę jedną część zoptymalizować używając innego podejścia. Jeśli trafi nam się rozwój produktu gdzie zastosowano jedno z podejść to nie będzie dla nas problemem kontynuowanie pracy w tym podejściu. Oczywiście powyższa lista bibliotek nie jest listą skończoną podobnie jak wybór modelu relacyjnego do przechowywania danych nie jest jedynym możliwym wyborem.

Wszystkie powyższe przykłady dostępne są na [GitHubie].

Szybki dostęp:
* [ADO.NET](#adonet)
* [Enterprise Library - Data Access Application Block](#enterprise-library---data-access-application-block)
* [ORM](#orm)
* [Dapper](#dapper)
* [Simple.Data](#simpledata)
* [Entity Framework](#entity-framework)

[1]: https://www.microsoft.com/en-us/sql-server/ "MS SQL Server"
[2]: https://docs.microsoft.com/en-us/dotnet/framework/data/adonet/ "ADO.NET"
[3]: https://docs.microsoft.com/en-us/previous-versions/msp-n-p/dn440726(v=pandp.60) "Enterprise Library - Data Access Application Block"
[4]: https://github.com/StackExchange/Dapper "Dapper"
[5]: http://simplefx.org/simpledata/docs/ "Simple.Data"
[6]: https://docs.microsoft.com/en-us/ef/#pivot=entityfmwk "Entity Framework"
[7]: https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html "Optimistic Offline Lock"
[8]: https://en.wikipedia.org/wiki/Object-relational_mapping "Object Relational Mapping"
[9]: https://nhibernate.info/ "NHibernate"
[10]: https://www.particular.net/ "Particular.net"
[11]: https://particular.net/nservicebus "NServiceBus"
[12]: https://particular.net/blog/nservicebus-6.0-public-beta "NServiceBus 6.0 public beta"
[13]: https://particular.net/blog/async-await-its-time "Async/await its time"
[14]: https://www.mountaingoatsoftware.com/blog/spikes "Spikes"