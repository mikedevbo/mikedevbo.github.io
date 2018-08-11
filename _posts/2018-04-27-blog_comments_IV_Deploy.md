---
layout: post
title:  "System komentarzy do bloga IV - Deploy"
date:   2018-05-27
---

W czwartej i ostatniej części serii pokazującej w jaki sposób można wykorzystać Messaging and Queueing do realizacji systemu komentarzy do bloga, zobaczymy implementację wybranych elementów pozwalających wdrożyć stworzone funkcjonalności.

## Proces

Podobnie jak przy projektowaniu lub programowaniu, sposobów na wdrożenie jest wiele. Narzędzi wspomagających wdrożenie również jest wiele. Wszystko zależy od fizycznej architektury rozwiązania, ilości komponentów do wdrożenia, docelowego miejsca gdzie komponenty będą hostowane, idt. Niezależnie od tego jakie rozwiązanie się wybierze, wszystkie kroki związane z wdrożeniem powinny być zautomatyzowane, tak aby zminimalizować możliwość popełnienia błędu. Jest to jedna z zasad tzw. [Continous Integration software development practice]. Proces wdrożenia systemu komentarzy do bloga może wyglądać np. tak:

![Picutre1]({{ site.url }}/assets/blog_comments_deploy/deploy_process.png)

Możemy wyodrębnić dwa rodzaje automatyzacji:

* Dla każdego kroku w procesie istnieje automatyczny mechanizm realizujący ten krok:
    * Compile Code
    * Run Unit Tests
    * Deploy Nancy Host (Test and Production)
    * Deploy NServiceBus Host (Test and Production)

* Każdy kolejny krok uruchamia się automatycznie zaraz po zakończeniu poprzedniego kroku:
    * Build
    * Test
    * Production

Druga część automatyzacji zależy od tego czy będziemy trzymać się zasad zdefiniowanych wg tzw. **Continous Delivery** czy też zasad zdefiniowanych wg. tzw. **Continous Deployment**. Więcej szczegółów można przeczytać w [...].

O ile pierwsza część automatyzacji jest raczej stała o tyle druga część może się zmieniać w zależności od tego w którą stronę dany produkt będzie ewaluował. W dalszej części artykułu skupimy się na części pierwszej.

## Architektura

[wstawić diagram]

Na diagramie widzimy dwa komponenty do wdrożenia:

* Nancy host
* NServiceBus host

Patrząc na architekturę z punktu widzenia logicznego Nancy host wysyła message'e do NServiceBus Host'a. Patrzyć na architekturę z punktu widzenia fizycznego obydwa hosty mają zdefiniowane tzw. Policy które między innymi mówi jaki transport ma być użyty do przesyłania wiadomości. NServiceBus wspiera wiele różnych rodzajów [transportów]. Wybór transportu może mieć wpływ na to gdzie NServiceBus host będzie wdrażany a co za tym idzie wpływ na rozwiązanie automatyzacji.

Aby popatrzyć na różne możliwości wdrożeń przyjmijmy takie oto założenia:

* Nancy host jest wdrażany na standardowy Web Hosting [dać linka] gdzie mamy ograniczone możliwości konfiguracji
* NServiceBus host jest wdrażany na maszynę wirtualną nad którą mamy pełną kontrolę i pełne możliwości konfiguracji

## Build

- gitVersion - wstęp

Zanim wdrożymy stworzone funkcjonalności chcemy wyeliminować tzw. efekt "u mnie działa" jeśli chodzi o kompilację kodu oraz unit testy. Pierwszym krokiem w procesie wdrożenia jest tzw. build, którego zadaniem jest pobranie kodu z systemu kontroli wersji, skompilowanie w celu sprawdzenia czy kod jest kompletny i uruchomienie unit testów. Można dodawać również inne kroki np. pokrycie kodu testami jednostkowymi, zgodność kodu ze stylem itd. 

- gitVersion - wspiera różne build servery

Jeśli kod trzymany jest na [GitHub'e] można wykorzystać [GitVersion] do oznaczania dll'ek zdefiniowaną wersją [Release'a] na git hub'e. Wtedy wiemy z jakiego Release poszło wdrożenie, a z release wiemy jakie commity składają się na wdrożenie. Istnieją specjalne systemy, które pozwalają definować buildy a także wiele wiele innych rzeczy. GitVersion integruje się z różnymi takimi systemami [dać linka]

- powershell - najniższa warstwa

W dalszej części zobaczymy w jaki sposób można użyć powershella do stworzenia automatycznego deploy'a

- powershell - skrypt - build

Skrypt pobierający kod z git'a oraz uruchmiający kompilację może wyglądać tak:

- dać skrypt
- opisać skrypt

Skrypt uruchamiający testy jednostkowe może wyglądać np. tak

- dać skrypt
- opisać skrypt

Jeśli dwie powyższe operacje zakończą się sukcesem mamy przygotowane artefakty, które możemy wdrożyć na.

## Nancy Host Deploy

- blue-green deployment
- co trzeba przygotować na hoście
- kroki procesu blue -> wynik
- kroki procesu green -> wynik
- skrypt powershell -> już nie opisywać drugi raz


## NServiceBus Host Deploy

- jak zrobić blue-green
- co trzeba przygotować na hoście
- kroki procesu blue
- kroki procesu green
- skrypt powershell -> już nie opisywać drugi raz

Ten post kończy serię pokazującą jakie kroki jakie trzeba przejść od momentu zaprojektowania rozwiązania poprzez jego realizację oraz testy aż po wdrożenie. Widzieliśmy też jakie możliwości daje messaging and queueing oraz w jaki sposób framework NServiceBus ułatwia realizację funkcjonalności w takiej architekturze. Przeszliśmy też przez proces wdrażania stworzonych funkcjonalności poprzez jego automatyzację.
