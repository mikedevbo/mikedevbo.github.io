---
layout: post
title:  "6th sposób komunikacji z relacyjną bazą danych"
date:   2020-03-22
---

Nareszcie! Po wielu latach poszukiwań, testowania [wielu innych][1], trafiła do mnie ta jedna jedyna, która rządzi wszystkimi pozostałymi. Panie i Panowie, przedstawiam *New Kid on The Block* o nazwie [FSharp.Data.SqlClient][2], bibliotekę, która umożliwia komunikację z relacyjną bazą danych [MS SQL Server][3].

Zanim przejdziemy do szczegółów, pozwól, że nakreślę Ci pewien kontekst. Tworząc systemy informatyczne mamy duży wybór, jeśli chodzi o projektowanie, programowanie, testowanie oraz wdrażanie rozwiązań. Niektóre elementy pasują do siebie idealnie, niektóre trochę mniej. Dwa przykłady:

1. Jeśli wychodzisz od danych, to interesuje Cię model systemu oraz jego reprezentacja w bazie danych. Dopiero potem skupiasz się na zachowaniu, które będzie operować na zdefiniowanym modelu.

2. Jeśli wychodzisz od zachowania, to interesują Cię akcje, jakie system powinien wykonywać. Dopiero potem skupiasz się na danych oraz ich reprezentacji w bazie danych.

W kontekście operacji na relacyjnej bazie danych **MS SQL Server**, w pierwszym przypadku zapewne rozważysz wykorzystanie któregoś z frameworków **ORM** np. [Entity Framework][4]. W drugim przypadku możesz chcieć sięgnąć po coś "lżejszego" np. [Dapper'a][5].

Nic nie stoi na przeszkodzie, aby odwrócić wybór. W pierwszym przypadku zrezygnować z **ORM'a**, dojdzie Ci wtedy dodatkowy kod do napisania. W drugim przypadku sięgnąć po **ORMa'**, ale...możesz nie mieć modelu, wtedy "na siłę" musisz go stworzyć!

W moim przypadku podejście nr 1. nigdy się nie sprawdziło, natomiast podejście nr 2. zaskoczyło momentalnie, zwłaszcza wtedy, kiedy zacząłem realizować funkcjonalności na podstawie stylu architektonicznego [Message Queuing][6]. W podejściu tym komponent zawierający **Message Handler** odpowiedzialny jest za realizację konkretnego kawałka funkcjonalności. Przykład przetwarzania **Message'a** wygląda mniej więcej tak:

{% highlight text %}
Component
    Handler(message)
        
        // 1. get required data from database if message doesn't contain them

        // 2. execute business logic

        // 3. update database state

        // 4. send/publish another message

{% endhighlight %}

Mając dobrze wydzielony komponent oraz jasno określoną jego odpowiedzialność, operacje na bazie danych sprowadzają się do prostych **select'ów**, **update'ów** oraz **insert'ów**. Skomplikowany model nie jest potrzebny. Co więcej, jeśli zdefiniowane operacje są specyficzne dla tej konkretnej funkcjonalności, to nie potrzebna jest żadna zewnętrzna abstrakcja jak **IRepository**, **IDatabaseContext**, **IDatabaseMapper** itp. Całość można zamknąć w samym komponencie. Takie podejście minimalizuje pokusę (re)użycia kodu w innym miejscu (stworzenia zależności), co mocno upraszcza wprowadzanie zmian.

Uwalniając się od konieczności tworzenia abstrakcji oraz skomplikowanego modelu, powstaje pytanie, w jaki sposób zaprogramować operacje na bazie danych? W przypadku zmiany stanu może to być funkcja, która w parametrach przyjmuje wartości do zmiany. W przypadku pobierania danych również może to być funkcja, która w parametrach przyjmuje wartości to filtrowania (użyte w warunku **where** języka **T-SQL**), a jako wynik zwraca odpytane dane. W tym momencie zaczyna robić się ciekawie. Zwrócone dane trzeba jakoś reprezentować. Jeśli do implementacji użyjemy **Entity Framework'a** to musimy stworzyć klasę reprezentującą pobrane dane - taki mini model. Jeśli użyjemy **Dapper'a** to również musimy stworzyć taką klasę lub użyć typu [dynamic][7], ale wtedy tracimy pomoc kompilatora w wykrywaniu ewentualnych błędów.

Obecnie programując w języku **C#** w pierwszej kolejności wybieram **Dapper'a**. Za każdym razem, kiedy piszę nowe zapytanie do bazy, zatrzymuję się na decyzji o tym, czy tworzyć klasę, czy użyć typu **dynamic** - *"Przecież to zapytanie jest takie proste, po co mi ta klasa? No tak, ale fajnie jest mieć wsparcie kompilatora, podpowiadanie składni itp."* Czy jest jakiś sposób na pogodzenie tych dwóch wymagań?

Okazuje się, że jest i nazywa się [FSharp.Data.SqlClient][2]. Zobaczmy więc, co potrafi.

### FSharp.Data.SqlClient w praktyce

Tak, jak w [poprzednim artykule][1], zaprogramujmy funkcjonalności operowania na danych dotyczących informacji o graczu tenisa:
 
* pobranie informacji o graczu - select z widoku bazodanowego
* dodanie nowego gracza - insert danych do tabeli
* ustawienie trenera gracza - update wybranych danych z [Optimistic Offline Lock][8]

Zacznijmy od operacji **select**:

{% highlight fsharp %}
module SqlClient

    open FSharp.Data
    
    [<Literal>]
    let private connectionStringName = "name=Players"

    module GetPlayerBaseInfo =
        
        [<Literal>]
        let private sql = "
            select
                pbi.id
                , pbi.FirstName
                , pbi.LastName
                , pbi.BirthYear
                , pbi.BirthMonth
                , pbi.BirthDay
                , pbi.BirthplaceCountry
                , pbi.BirthplaceCity
                , pbi.[Weight]
                , pbi.Height
                , pbi.IsRightHanded
                , pbi.IsTwoHandedBackhand
                , pbi.CoachId
                , pbi.CoachFirstName
                , pbi.CoachLastName
            from
                dbo.PlayersBaseInfo pbi
            where
                pbi.Id = @Id
            "
        let execute playerId =
            use cmd = new SqlCommandProvider<sql, connectionStringName, SingleRow = true>()
            cmd.Execute(Id = playerId)    
{% endhighlight %}

Trzy pierwsze właściwości, na które warto zwrócić uwagę to:

1. Prosty kod

    Definiujemy trzy elementy:

    * `ConnectionString` - nazwa w pliku konfiguracyjnym lub pełny wpis
    * `sql` - kod bazodanowy do wykonania
    * `SqlCommandProvider` - instancja umożliwiająca wykonanie kodu **sql** poprzez wywołanie funkcji `Execute`
<br />
<br />
2. Brak konieczności definiowania zwracanego typu

    * Typ generowany jest automatycznie na podstawie danych zwracanych w zapytaniu **SQL**!:

    {% highlight fsharp %}
    Option<SqlCommandProvider<...>Record>
    {% endhighlight %}

{:start="3"}
3. Kod jest w pełni kompilowany, ale uwaga, nie tylko języka **F#**, ale też języka **T-SQL**! Dodatkowo biblioteka wykrywa nazwy parametrów wejściowych oraz nazwy zwracanych pól, weryfikując je na poziomie kompilacji kodu. Zobaczmy, jak to wygląda w **Visual Studio**:

[![Picutre1][9]][9]

Nie wiem jak u Ciebie, ale u mnie wywołało to efekt WOW!

Wracając do **Message Handler'a**, to jest właśnie to, czego potrzebowałem. Przykład wykorzystania powyższego kodu z prostą logiką wypisującą informacje o graczu na standardowe wyjście może wyglądać tak:

{% highlight fsharp %}
let Handler message =
    
    // 1. get required data from database if message doesn't contain them
    let playerInfo = SqlClient.GetPlayerBaseInfo.execute message.PlayerId

    // 2. execute business logic
    match playerInfo with
    | None -> printfn "Player with id %d doesn't exist" message.PlayerId
    | Some player -> printfn "%A" player
    
    // 3. update database state

    // 4. send/publish another message
{% endhighlight %}

Value `playerInfo` posiada typ, wygenerowany przez `FSharp.Data.SqlClient`. Podpowiadanie składni działa bez zarzutu:

[![Picutre2][10]][10]

A jak może wyglądać implementacja zmiany stanu w bazie danych? Zobaczmy to na przykładzie operacji **insert**:

{% highlight fsharp %}
module SqlClient

    open FSharp.Data
    
    [<Literal>]
    let private connectionStringName = "name=Players"

    module AddPlayer =
        
        [<Literal>]
        let private sql = "
            insert into dbo.Player(id, IsRightHanded, IsTwoHandedBackhand, CoachId)
            values(@Id, @IsRightHanded, @IsTwoHandedBackhand, null)
            "
        let execute personId isRightHanded isTwoHandedBackhand =
            use cmd = new SqlCommandProvider<sql, connectionStringName>()
            cmd.Execute(personId, isRightHanded, isTwoHandedBackhand) |> ignore
            ()
{% endhighlight %}

Schemat kodu jest dokładni taki sam, jak dla operacji typu **select**. Wszystkie wcześniej wymienione właściwości również mają zastosowanie.

Na koniec popatrzmy na operację typu **update**:

{% highlight fsharp %}
module SqlClient

    open FSharp.Data
    
    [<Literal>]
    let private connectionStringName = "name=Players"

    module SetPlayerCoach =
    
        exception DbUpdateConcurrencyException

        [<Literal>]
        let private sql = "
            declare @prevCoachId int = @prevCoachIdVal
            update
                dbo.Player
            set
                CoachId = @newCoachId
            where
                Id = @Id
                and ((@prevCoachId is null and CoachId is null) or (@prevCoachId = CoachId))
            "
        let execute (playerId:int) (newCoachId:Option<int>) (prevCoachId: Option<int>) =
            use cmd = new SqlCommandProvider<sql, connectionStringName, AllParametersOptional = true>()
            let result = cmd.Execute(prevCoachId, newCoachId, (Some playerId))
            if result = 0 then raise (DbUpdateConcurrencyException)
{% endhighlight %}

Ten sam wzorzec z zachowanymi, wcześniej wymienionymi właściwościami.

Całą implementację możesz zobaczyć na moim [GitHub'e][11].

### FSharp.Data.SqlClient - dodatkowe właściwości

Biblioteka posiada o wiele więcej możliwości. Koncepcyjnie bliżej jej do wcześniej wspomnianego **Dapper'a**. Wydajnościowo również wypada bardzo dobrze. Szczegóły możesz zobaczyć [na stronie dokumentacji][12]. Kilka dodatkowych właściwości, na które zwróciłem uwagę to:

* możliwość asynchronicznego wykonania kodu - `asyncExecute`
* obsługa pliku `app.config/web.config` nawet w **.NET Core**
* możliwość "wyniesienia" kodu **SQL** do osobnego pliku
* wsparcie dla procedur składowanych
* możliwość przekazania transakcji z zewnątrz

### Podsumowanie

Jeśli jesteś osobą, która poszukuje prostych rozwiązań dla skomplikowanych problemów, to biblioteka [FSharp.Data.SqlClient][2] powinna przypaść Ci do gustu. Jeśli chodzi o mnie, to wszystko wskazuje na to, że zostanę z nią na dłużej. Poszukiwane przeze mnie właściwości: prosty kod oraz brak konieczności definiowania własnych typów, ale z pełnym wsparciem kompilatora, zostały odnalezione. Ekstra dodatkiem jest kompilacja kodu języka **T-SQL**. Pozostaje teraz zgłębiać wiedzę na temat samej biblioteki.

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}{% link _posts/2018-12-27-5-ways-of-communicating-with-relational-database.md %}
[2]: https://fsprojects.github.io/FSharp.Data.SqlClient/index.html "FSharp Data SqlClient"
[3]: https://www.microsoft.com/en-us/sql-server/sql-server-2019 "MS Sql Server"
[4]: https://docs.microsoft.com/en-us/ef/ "Entity Framework"
[5]: https://github.com/StackExchange/Dapper "Dapper"
[6]: https://en.wikipedia.org/wiki/Message_queue "Message Queue"
[7]: https://docs.microsoft.com/en-us/dotnet/csharp/programming-guide/types/using-type-dynamic "CSharp Dynamic"
[8]: https://martinfowler.com/eaaCatalog/optimisticOfflineLock.html "Optimistic Offline Lock"
[9]: {{ site.url }}/assets/sixth_way_rdb_access/fsharp_data_sqlclient_compiletime.gif
[10]: {{ site.url }}/assets/sixth_way_rdb_access/fsharp_data_sqlclient_autocomplition.png
[11]: https://github.com/mikedevbo/data-access "data-access"
[12]: https://fsprojects.github.io/FSharp.Data.SqlClient/comparison.html "FSharp Data SqlClient Comparison"