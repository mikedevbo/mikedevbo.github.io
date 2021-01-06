---
layout: post
title:  "2021 - Co było...Co jest...Co będzie..."
description: "...a teraz przyszła pora na kolejne. Tym razem obejmie ono krótszy okres - rok 2020. Podzielę się również planami na rok 2021. Udanego czytania!"
date:   2021-01-06
---

Nowy Rok. Nowa energia do działania. Nowe... Czas szybko płynie. Nie tak dawno pisałem [poprzednie podsumowanie][4], a teraz przyszła pora na kolejne. Tym razem obejmie ono krótszy okres - rok 2020. Podzielę się również planami na rok 2021. Udanego czytania!

## Co było...

*"Jeden obraz wart więcej niż tysiąc słów"* (chińskie przysłowie):

[![Picture1]({{ site.url }}/assets/2021_was_is_will_be/what_was.svg)][1]

Na początku 2020 roku, zdefiniowałem dla siebie długofalowy cel:

* nauczyć się realizować systemy posiadające **Frontend**, **Backend** oraz **integrację z systemami zewnętrznymi**

Wybrałem podstawę do budowania wiedzy:

* kurs [Advanced Distributed Systems Design (ADSD)][18]

Wypisałem wzorce oraz narzędzia, które pomogą mi zrealizować główny cel w taki sposób, aby końcowe rezultaty cechowały się dobrą, a może nawet i bardzo dobrą jakością:

* elementy widoczne na powyższym zdjęciu

Postanowiłem opisywać na blogu rezultaty oraz dzielić się przemyśleniami na temat poznanego zagadnienia.

## Co jest...

*"Jeden obraz wart więcej niż tysiąc słów"* (chińskie przysłowie):

[![Picture2]({{ site.url }}/assets/2021_was_is_will_be/what_is.svg)][2]

W kolejnych miesiącach 2020 roku zapoznawałem się z poszczególnymi elementami oraz utrwalałem to, co już znałem. Obecny stan wiedzy podzieliłem na trzy kategorie:

* "siadam i robię" - kolor zielony na zdjęciu powyżej
    * elementy, które znam na tyle dobrze, że jestem w stanie używać ich na co dzień
        * metafora sportowa - wychodzę na mecz i gram
<br />
<br />
* "robię, ale..." - kolor niebieski na zdjęciu powyżej
    * elementy, które znam w praktyce, ale potrzebuję większego wsparcia w postaci wyszukiwania i analizowania pomocnych treści
        * metafora sportowa - mogę wyjść na mecz, ale będą widoczne braki, trzeba więcej trenować
<br />
<br />
* "tylko teoria" - kolor czarny na zdjęciu powyżej
    * elementy, o których mogę podyskutować, znam idee, ale nie używałem ich w praktyce
        * metafora sportowa - nie wyjdę na mecz, najpierw muszę chociaż trochę potrenować

A jak to wygląda w szczegółach? Zobaczmy.

### Design

W związku z sytuacją *COVID-19*, zespół rozwijający platformę [Particular][6] postanowił czasowo [udostępnić][7] kurs online [ADSD][5] za tzw. **free**. Udało mi się ukończyć kurs, czego rezultatem jest pogłębienie wiedzy z proponowanych w nim idei.

Efektem utrwalonej wiedzy jest to, że przesunąłem do kategorii "siadam i robię" zagadnienie dotyczące **Domain Model**. Koncepcja **Sagi** bardzo mi odpowiada. Jest prosta i elegancka. Framework [NServiceBus][8] zawiera jej implementację, także od razu można korzystać z niej w praktyce.

Wiedząc jak realizować **Domain Model**, automatycznie mamy rozwiązany sposób realizacji części **Command** w podejściu **CQRS**. Część **Query** to jak najprostsze pobranie informacji potrzebnych do realizacji wymagania biznesowego. Do ustalenia pozostało mi uporządkowanie wiedzy na temat efektywnego modelu przechowywanych danych dla **Query**. Jak się z tym uporam, to będę mógł przenieść zagadnienie **CQRS** do pierwszej kategorii.

Z tematyką **Frontend** poszło trochę wolniej niż zakładałem. Przez cały poprzedni rok poszukiwałem drogi, którą chciałbym podążyć. Znalazłem pewien kierunek, który będę chciał przetestować. Więcej o tym przeczytasz w dalszej części artykułu.

**Frontend** jest ostatnim elementem, który blokuje mnie przed przeniesieniem do pierwszej kategorii zagadnień **SOA** oraz **ADSD**.

### Develop

Minął kolejny rok od momentu, w którym rozpocząłem przygodę z frameworkiem [NServiceBus][8] oraz platformą [Particular][6]. W sumie to już 7 lat. W roku 2020 podobnie, jak w poprzednich latach platforma pokazała swoją siłę w najtrudniejszych elementach związanych z wytwarzaniem oprogramowania:

* utrzymaniem oraz dalszym rozwojem istniejących funkcjonalności

Kodowanie **Message Handlerów** to wciąż czysta przyjemność.

Zdolność udzielania użytkownikom odpowiedzi na pytania, dlaczego stan systemu wygląda tak, a nie inaczej sprawia wiele satysfakcji. Taką możliwość daje przeglądanie audytu przetworzonych wiadomości za pomocą narzędzia [ServiceInsight][9].

Monitoring za pomocą narzędzia [ServicePulse][10] to bajka. Na jednym ekranie widać, czy i w jakim tempie mechanizmy przetwarzają wiadomości. Istnieje możliwość podglądu danych w czasie rzeczywistym. Krótko mówiąc, widać, czy system działa, czy jest responsywny, czy może jednak się "zapycha".

Wisienką na torcie jest koncepcja [Recoverability][11]. Mimo tylu lat używania platformy, za każdym razem, kiedy widzę w [ServicePulse][10] ponowne przetwarzanie przywróconych wiadomości, generuje mi się taki "mały" uśmiech na twarzy z myślą, ile właśnie zaoszczędziłem czasu na utrzymanie systemu w spójnym stanie.

Zeszły rok to również kontynuacja nauki języka **F#** oraz powiązanych z nim technologii. Koncepcyjnie jest coraz lepiej, jednak do wyrównania poziomu kodowania w języku **C#** potrzebuję większej ilości treningów.

W przypadku **Git** i **GitHub** nic się nie zmieniło. Nie miałem konieczności używania czegoś więcej niż to, co znam obecnie.

### Test

Z tematyką testów również nic się nie zmieniło. Jak na razie w zupełności wystarcza dzielenie kodu na małe, niezależne części, które można przetestować za pomocą automatycznych testów jednostkowych oraz manualnych testów integracyjnych. **Property-Based Testing**, a także automatyzacja testów integracyjnych/wydajnościowych czekają na swoją kolej.

### Deploy

Poznawanie narzędzia [Fake][12] dało mi możliwość potrenowania pisania kodu w języku **F#**. Napisałem mechanizm wdrażania nowych wersji systemu komentarzy, którego używam na tym blogu. **Microsoft Azure**? Trochę poklikałem, ale jeszcze zostawiam temat na stanie "tyko teoria".

### DDTD

Tak wygląda mój obecny stan wiedzy z elementów, które pomogą mi osiągnąć główny cel:

* realizować systemy posiadające **Frontend**, **Backend** oraz **integrację z systemami zewnętrznymi**

Wrócę jeszcze do platformy [Particular][6]. W zeszłym roku nastąpiła ważna zmiana w sposobie jej licencjonowania. Obecnie można używać całej platformy **bez opłat** na etapach wytwarzania oraz testowania funkcjonalności. Jeśli końcowy produkt nie przekroczył pewnej skali, to cała platforma dostępna jest również **bez opłat** do użytku na środowisku produkcyjnym. 
Dzięki temu łatwiej jest zacząć z realizacją produktu. Więcej informacji znajdziesz [tu][13].

### Blog

 Od strony wizualnej nic się nie zmieniło. Jeśli chodzi o zawartość, to udało mi się napisać 10 artykułów. Taką samą liczbę miałem w roku 2019, także utrzymałem ilość wpisów.

 Publikując treści na blogu wyszedłem do świata, dzieląc się wiedzą, przemyśleniami oraz doświadczeniami. Nie spodziewałem się tego, ale dzięki temu, zostałem wyróżniony przez zespół rozwijający platformę **Particular** i zostałem tzw. [NServiceBus Champ][14]. Bardzo miłe doświadczenie.

## Co będzie...

*"Jeden obraz wart więcej niż tysiąc słów"* (chińskie przysłowie):

[![Picture3]({{ site.url }}/assets/2021_was_is_will_be/what_will_be.svg)][3]

Nie zmieniam celu długofalowego. W każdym kolejnym roku będę wzmacniał wiedzę z tego, co już znam, podnosił umiejętności w tematach, które poznałem, a także zapoznawał się z tym, czego nie udało mi się rozpocząć. Jeśli wytrwam, to za jakiś czas wszystkie elementy widoczne na powyższym zdjęciu zapalą się na zielono - osiągną stan "siadam i robię".

Odnośnie tematu **Frontend**, to w tym roku przetestuję podejście, które mnie zaciekawiło - [JAMstack][15]. Wprowadzę jedną modyfikację - zamienię **J** na **C** i/lub **F**. Od strony **Develop** będą to [Blazor][16] oraz [Bolero][17].

Plany związane z blogiem nie ulegają zmianie. Będę dzielił się tym, co uda mi się osiągnąć oraz przemyśleniami na temat poznanego zagadnienia.

To tyle, jeśli chodzi o podsumowanie minionego roku oraz plany na najbliższą, oraz dalszą przyszłość. Dzięki za przeczytanie artykułu. Powodzenia z Twoimi planami oraz ich realizacją!

{{ site.mark_post_as_end }}

### {{ site.comments }}

[1]: {{ site.url }}/assets/2021_was_is_will_be/what_was.svg
[2]: {{ site.url }}/assets/2021_was_is_will_be/what_is.svg
[3]: {{ site.url }}/assets/2021_was_is_will_be/what_will_be.svg
[4]: {{ site.url }}{% link _posts/2020-01-12-what-was-what-is-what-will-be.md %}
[5]: https://learn.particular.net/courses/adsd-online "ADSD course online"
[6]: https://particular.net/ "Particular.net"
[7]: https://discuss.particular.net/t/free-for-a-limited-time-advanced-distributed-systems-design-videos/1708 "free for a limited time advanced distributed systems design videos"
[8]: https://particular.net/nservicebus "NServiceBus"
[9]: https://particular.net/serviceinsight "ServiceInsight"
[10]: https://particular.net/servicepulse "ServicePulse"
[11]: https://docs.particular.net/tutorials/message-replay/ "Message Replay Tutorial"
[12]: https://fake.build/ "Fake build"
[13]: https://particular.net/pricing "Pricing"
[14]: https://discuss.particular.net/t/introducing-our-newest-champ-michal-bogdanski/1840 "introducing champ"
[15]: https://jamstack.org/ "jamstack"
[16]: https://dotnet.microsoft.com/apps/aspnet/web-apps/blazor "blazor"
[17]: https://fsbolero.io/ "bolero"
[18]: https://particular.net/adsd "Particular ADSD"