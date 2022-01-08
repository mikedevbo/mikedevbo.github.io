---
layout: post
title:  "2022 - Co było...Co jest...Co będzie..."
description: "Podsumowanie 2021 roku. Plany na rok 2022."
date:   2022-01-08
---

No i mamy styczeń! Czas podsumowań minionego roku oraz planowania działań na rok kolejny. W moim przypadku trzeci raz w blogowej formie:

* [2020][3]
* [2021][4]
* [2022][5]

Zobaczmy, co ciekawego wydarzyło się w ostatnich 12 miesiącach.

## Co było...

*"Jeden obraz wart więcej niż tysiąc słów"* (chińskie przysłowie):

[![Picture1]({{ site.url }}/assets/2022_was_is_will_be/what_was.svg)][1]

Na początku 2021 roku postanowiłem, że nie zmieniam długofalowego celu, który polegał na zdobyciu umiejętności budowania systemów zawierających **Frontend**, **Backend** oraz **Integrację z systemami zewnętrznymi** z wykorzystaniem elementów wymienionych na zdjęciu. Wprowadziłem trzy-kolorowy stopień osiągnięć:

* zielony
    * najwyższy, oznaczający umiejętności pozwalające na realizacje zagadnień
* niebieski
    * praktyczne znajomość
    * pozwala na realizację zagadnień z większym wsparciem wyszukiwania i analizowania pomocnych treści
    * do osiągnięcia najwyższego stopnia, potrzeba więcej treningów 
* czarny
    * wiedza teoretyczna, pozwalająca dyskutować w danym temacie

Planowałem większą część czasu poświęcić zagadnieniom dookoła **Frontendu**. Kierunkiem było podejście **JAMstack** z wykorzystaniem frameworków **C# Blazor** oraz **F# Bolero**.

Osiągnięciami i przemyśleniami miałem dzielić się na tym blogu.

## Co jest...

*"Jeden obraz wart więcej niż tysiąc słów"* (chińskie przysłowie):

[![Picture2]({{ site.url }}/assets/2022_was_is_will_be/what_is.svg)][2]

### Design

Przez cały poprzedni rok nie natrafiłem na materiały, które radykalnie zmieniłby moje podejście do wytwarzania oprogramowania. Niezmiennie trzymam się koncepcji z kursu [Advanced Distributed System Design (ADSD)][6].

Brakującym elementem w całej układance była wiedza z programowania **Webowego**. Sprawdziłem, o co w tym chodzi i od razu natrafiłem na styl architektoniczny [Model-View-Update (MVU)][7]. To właściwie zamknęło temat projektowania, dlatego zamieniam **JAMstack** na **MVU**, jako główny koncept projektowy.

### Develop

80% dostępnego czasu spędziłem na ćwiczeniu programowania **Webowego** z użyciem dwóch frameworków: [F# Bolero][8] oraz [C# Blazor][9]. Pierwszy z nich rozszerza drugi o implementację wzorca **Model-View-Update** oraz wiele innych przydatnych mechanizmów.

Pozostałe 20% poświęciłem [przepisywaniu][10] na język **F#** [systemu komentarzy na blogu][11], który zrealizowany jest z wykorzystaniem wybranych technik wymienionych na zdjęciu.

### Test

Dalej bez zmian. Dzielenie kodu na małe, niezależne części, które można przetestować za pomocą automatycznych testów jednostkowych oraz manualnych testów integracyjnych sprawdza się dość dobrze.

### Deploy

Również bez zmian. Tematy czekają na swoją kolej.

### Blog

 Od strony wizualnej nic się nie zmieniło. Napisałem 8 artykułów, o dwa mniej w porównaniu do poprzedniego roku.

## Co będzie...

Rok temu, o tej porze, dokładnie wiedziałem, czym będę się zajmował w kolejnych miesiącach. Dzisiaj nie mam sprecyzowanych celów. Plan jest taki, aby na bieżąco wybierać tematy z puli, zgłębiać zagadnienie i dzielić się poznaną wiedzą.

To tyle, jeśli chodzi o podsumowanie minionego roku. Dzięki za przeczytanie artykułu. Powodzenia z Twoimi planami oraz ich realizacją!

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}/assets/2022_was_is_will_be/what_was.svg
[2]: {{ site.url }}/assets/2022_was_is_will_be/what_is.svg
[3]: {{ site.url }}{% link _posts/2020-01-12-what-was-what-is-what-will-be.md %}
[4]: {{ site.url }}{% link _posts/2021-01-06-what-was-what-is-what-will-be.md %}
[5]: {{ site.url }}{% link _posts/2022-01-08-what-was-what-is-what-will-be.md %}
[6]: https://particular.net/adsd "Advanced Distributed System Design"
[7]: https://guide.elm-lang.org/architecture/ "Elm architecture"
[8]: {{ site.url }}{% link _posts/2021-01-23-fsharp-bolero-introduction.md %}
[9]: https://dotnet.microsoft.com/en-us/apps/aspnet/web-apps/blazor "C# Blazor"
[10]: {{ site.url }}{% link _posts/2021-11-14-adsd-nservicebus-csharp-fsharp-reloaded.md %}
[11]: https://github.com/mikedevbo/blog-comments "Blog Comments"
