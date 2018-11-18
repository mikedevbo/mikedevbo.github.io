---
layout: post
title:  "data-access"
date:   2018-11-13
---

Kodując tzw. "back end" w biznesowych systemach informatycznych w 99% mamy do czynienia z operacjami na bazie danych. Potrzebny zatem jest sposób w jaki możemy się z bazą danych porozumieć. Artykuł ten jest wycieczką po bibliotekach/technologiach, z którymi miałem do czynienia przy różnych projektach, umożliwiającymi kontakt z bazą danych. Poniższe przykłady są implementacją w języku C# na platformie .NET oraz zakładają połączenie z relacyjną bazą danych MS SQL Server. Całość przedstawiona jest w kontekście trzech operacji:

* select z widoku bazodanowego
* insert danych to tabeli
* update danych z Optimistic Offline Lock

Przykłady dostępne są na GitHubie.

-> rozwinięcie

## ADO.NET
* wstęp -> wszystko zaczyna się od...
* link do doc
* kod

## Enterprise Library - Data Access Application Block
* wstęp
* link do doc
* kod

## ORM
* wstęp -> NHibernate - za duża bariera wejścia
* link do doc
* kod -> implementacja jako ćwiczenie

## Dapper
* wstęp -> pierwsza próba zejścia z procek
* link do doc
* kod 

## Simple.Data
* wstęp -> dapper -> pisanie sql'a w C# -> simple jako alternatywa
* link do doc
* kod

## Entity Framework
* wstęp
    * simple -> produkt skończony -> brak async/await
    * dapper vs EF
    * kolejne podejście do ORM -> wymagania
        * tylko jako warstwa dostępu do danych -> bez DDD
        * update -> elastyczność
            * bez select
            * update -> tylko te pola, która powinny być zaktualizowane

## Podsumowanie

Znając różne możliwości łączenia się z bazą danych jesteśmy bardziej elastyczni w konstruowaniu rozwiązań. 
Przykładowo jeśli całość oparta jest na jednym podejściu, a któreś z zapytań do bazy danych trwa zbyt długo, to można tę jedną część zoptymalizować używając innego podejścia. Jeśli trafi nam się rozwój produktu gdzie zastosowano jedno z podejść to nie będzie dla nas problemem kontynuowanie pracy w tym podejściu. Oczywiście powyższa lista bibliotek/frameworków nie jest listą skończoną. Na mojej liście To Do Next są takie pozycje jak:
 
* entityframework.core
* rendle -> baza do jsona
* sql bulk copy
* no sql
* ...

[1]: https://www.microsoft.com/en-us/sql-server/ "MS SQL Server"