---
layout: post
title:  "System komentarzy do bloga I - Design"
date:   2018-03-11
---

Posty z tej serii:

* [System komentarzy do bloga I - Design]({% post_url 2018-03-11-blog_comments_I_Design %})
* [System komentarzy do bloga II - Develop]({% post_url 2018-03-18-blog_comments_II_Develop %})
* [System komentarzy do bloga III - Test]({% post_url 2018-03-30-blog_comments_III_Test %})

W poście [Wytwarzanie oprogramowania IV]({% post_url 2017-04-29-wytwarzanie_oprogramowania_IV %}), na przykładzie [NServiceBus'a][1], widzieliśmy jak Messaging and Queueing stawia krok do przodu w kierunku polepszenia jakości realizowanych systemów. Ten post otwiera serię pokazującą jak za pomocą tej samej techniki zrealizować system dodawania komentarzy do bloga. Trzy główne wątki tego posta to: **Idea**, **User Stories** oraz **Design** rozwiązania.

## Idea

Używając [Jekyll'a][2] jako silnika do blogowania mamy możliwość hostowania zawartości na serwerach [GitHub'a][3] przy pomocy [GitHub Pages][4]. Każdy nowy post to [git-push][5] do repozytorium kodu. Każdy komentarz można więc potraktować jako [GitHub pull request][6] do opublikowanego posta.

## User Stories

Trzy podstawowe wymagania opisane za pomocą techniki [User Story][7] mogą wyglądać tak:

1. Jako czytelnik bloga chcę dodać komentarz do przeczytanego posta
2. Jako autor bloga chcę zweryfikować, a następnie odpowiedzieć na dodany komentarz lub go odrzucić
3. Jako czytelnik bloga chcę być poinformowany o rezultacie dodanego przeze mnie komentarza do posta

## Design

Poniższy diagram przedstawia projekt rozwiązania na platformę [.NET][8] z użyciem:
* [Nancy][9] - Web Framework
* [NServiceBus][1] - Messaging Framework
* [GitHub API][10] - Komunikacja z GitHub'em

![Picutre1]({{ site.url }}/assets/blog_comments_design/Diagram1.png)

Na diagramie numerki od 1 do 22 przedstawiają kolejność wykonywania poszczególnych kroków, a poszczególne strzałki oznaczają rodzaje operacji:
* <---> - wywołanie `Request\Response`
* ----> - wysłanie wiadomości do kolejki\zlecenie działania (`command`)
* ....> - wysłanie wiadomości do kolejki\poinformowanie, że działanie zostało zakończone (`event`)

Przeanalizujmy diagram krok po kroku:
* 1 - komentarz przychodzi w postaci żądania `HTTP`
* 2 - moduł webowy tworzy message z danymi typu `command` i wysyła go do kolejki
* 3 - moduł webowy zwraca odpowiedź w stylu **"dziękuję twoje zgłoszenie zostało przyjęte"**
* 4 - message handler podnosi message z kolejki i wysyła kolejny message z żądaniem utworzenia na `GitHub'e` nowego [branch'a][11] z `branch'a master` gdzie znajduje się post
* 5 - message handler podnosi message z kolejki i używając `GitHub API` tworzy `branch'a`
* 6 - `GitHub API` zwraca odpowiedź
* 7 - message handler publikuje message typu `event` z informacją, że `branch` został utworzony
* 8..9..10..11 - na tej samej zasadzie jak w krokach od 4 do 7 wykonywana jest operacja dodania komentarza do posta na nowo utworzonym `branch'u`
* 12..13..14..15 - na tej samej zasadzie jak w krokach od 4 do 7 wykonywana jest operacja utworzenia `pull request'a`
* 16 - message handler wysyła message do siebie samego w stylu **"mam się zbudzić z X minut"**
* 17..18..19..20 - message handler wybudza się oraz sprawdza stan `pull request'a`
    * jeśli `pull request` jest otwarty, sterowanie wraca do kroku 16
    * jeśli `pull request` jest zamknięty sterowanie przechodzi do kroku 21
* 21..22 - message handler podnosi message z kolejki i wysyła email z informacją w stylu:
    * **"do komentarza została dodana odpowiedź"**, jeśli `pull request` został `z'merge'owany` z `master'em` i zamknięty
    * **"komentarz został odrzucony"**, jeśli `pull request` został zamknięty bez `merge'a` z `master'em`

Punkty od 1 do 16 realizują User Story nr 1. natomiast punkty od 17 do 22 realizują User Story nr 3. Gdzie jest zatem realizacja User Story nr 2.? Tutaj niestety **"samo się nie zrobi"**. Aby zweryfikować posta można skorzystać z portalu `GitHub'a` dodając odpowiedź i `merge'ując pull request'a` do `master'a` lub odrzucając komentarz poprzez samo zamknięcie `pull request'a`.

Punkt nr 16 pokazuje interesującą rzecz. Pomiędzy dodaniem komentarza, a jego obsługą może upłynąć dość długi czas. Powoduje to, że całość staje się tak zwanym `Long-running Business Process`. Taki proces w języku `NServiceBus'a` reprezentowany jest jako [Saga][12]. Na diagramie rolę `Sagi` pełni Message Handler Saga. `Saga` to również odpowiednik jednego ze wzorców [Enterprise Integration Patterns (EIP)][13] zwanego [Process Manager][14].

To tyle w temacie projektowania rozwiązania. W następnym poście zobaczymy w jaki sposób można zaimplementować powyższe rozwiązanie.

[1]: https://particular.net/nservicebus "NServiceBus"
[2]: https://jekyllrb.com "Jeykyll"
[3]: https://github.com "GitHub"
[4]: https://pages.github.com/ "GitHub Pages"
[5]: https://git-scm.com/docs/git-push/ "git-push"
[6]: https://help.github.com/articles/about-pull-requests/ "GitHub pull request"
[7]: https://en.wikipedia.org/wiki/User_story/ "User Story"
[8]: https://www.microsoft.com/net/ ".NET"
[9]: http://nancyfx.org/ "Nancy"
[10]: https://developer.github.com/v3/ "GitHub API"
[11]: https://git-scm.com/book/en/v2/Git-Branching-Basic-Branching-and-Merging/ "git-branching"
[12]: https://docs.particular.net/nservicebus/sagas/ "Sagas"
[13]: http://www.enterpriseintegrationpatterns.com/ "EIP"
[14]: http://www.enterpriseintegrationpatterns.com/patterns/messaging/ProcessManager.html "Process Manager"