---
layout: post
title:  "Wytwarzanie oprogramowania I - paradygmaty programowania"
description: Jest to pierwszy wpis z serii, którą nazwałem "Wytwarzanie oprogramowania". W serii tej, przedstawię elementy, które pomagają mi zrozumieć, czym jest programowanie, a także elementy, które pomogły\pomagają mi w codziennej pracy, od momentu kiedy byłem młodszym programistą do teraz, kiedy jestem trochę starszym programistą ;).
date:   2016-11-26
---

Posty z tej serii:

* [Wytwarzanie oprogramowania I - paradygmaty programowania]({% post_url 2016-11-26-wytwarzanie_oprogramowania_I %})
* [Wytwarzanie oprogramowania II - techniki programowania]({% post_url 2017-02-25-wytwarzanie_oprogramowania_II %})
* [Wytwarzanie oprogramowania III - architektury systemów informatycznych]({% post_url 2017-03-14-wytwarzanie_oprogramowania_III %})
* [Wytwarzanie oprogramowania IV - NServiceBus - framework, który zmienia zasady gry]({% post_url 2017-04-29-wytwarzanie_oprogramowania_IV %})

Jest to pierwszy wpis z serii, którą nazwałem "Wytwarzanie oprogramowania". W serii tej, przedstawię elementy, które pomagają mi zrozumieć, czym jest programowanie, a także elementy, które pomogły\pomagają mi w codziennej pracy, od momentu kiedy byłem młodszym programistą do teraz, kiedy jestem trochę starszym programistą ;).

Wpis ten zawiera krótki przegląd wybranych paradygmatów programowania.

## Programowanie niskopoziomowe

### Język maszynowy
Z punktu widzenia komputera najbardziej przyjaznym językiem do komunikacji jest język składający się z dwóch znaków: 0 i 1. Jest to tak zwany język maszynowy. "Rozmowa" z komputerem w tym języku odbywa się poprzez wprowadzanie rozkazów reprezentowanych przez ciągi zer i jedynek. Każdy komputer posiada jednostkę zwaną procesorem, a każdy rodzaj procesora posiada swoją interpretację języka maszynowego. Dla przykładu, poniższy kod reprezentuje język maszynowy zrozumiały dla procesora [Sextium II][1]. "Rozkazuje" on komputerowi wczytanie dwóch liczb ze standardowego wejścia, dodanie ich do siebie oraz wypisanie wyniku na standardowe wyjście:

{% highlight plaintext %}
adres            zawartość słowa
0000000000000000 1010 0101 0011 0010
0000000000000001 0000 0000 0000 1001
0000000000000010 1010 0101 0011 0010
0000000000000011 0000 0000 0000 1010
0000000000000100 1010 0101 0001 0110
0000000000000101 0000 0000 0000 1001
0000000000000110 1010 0101 0001 1011
0000000000000111 0000 0000 0000 1010
0000000000001000 0100 1111 0000 0000
0000000000001001 0000 0000 0000 0000
0000000000001010 0000 0000 0000 0000
{% endhighlight %}

Wersja w zapisie 16-kowym:
{% highlight plaintext %}
adres zawartość słowa
0000  A532
0001  0009
0002  A532
0003  000A
0004  A516
0005  0009
0006  A51B
0007  000A
0008  4F00
0009  0000
000A  0000
{% endhighlight %}

### Asembler
Dla komputera powyższy kod jest jasny i łatwy w zrozumieniu, natomiast dla człowieka już nie (a może tak ;)), dlatego następnym etapem w rozwoju języków programowania było powstanie tak zwanych języków adresów symbolicznych (Asemblerów). Program napisany w takim języku, reprezentowany jest w postaci symboli, które reprezentują rozkazy procesora. Symbole te  następnie tłumaczone są na język maszynowy procesora. Poniższy kod jest tym samym programem z poprzedniego akapitu, napisanym w języku Asemblera zrozumiałego dla procesora Sextium II:

{% highlight plaintext %}
CONST x
SWAPA
READ
STORE
CONST y
SWAPA
READ
STORE
CONST x
SWAPA
LOAD
SWAPD
CONST y
SWAPA
LOAD
ADD
WRITE
HALT
x: DATA
y: DATA
{% endhighlight %}

Tłumaczenie z Asemblera na kod maszynowy

{% highlight plaintext %}
adres zawartość słowa   opis zawartości słowa
0000      A532          CONST SWAPA READ STORE
0001      0009          #adres zmiennej x
0002      A532          CONST SWAPA READ STORE
0003      000A          #adres zmiennej y
0004      A516          CONST SWAPA LOAD SWAPD
0005      0009          #adres zmiennej x
0006      A51B          CONST SWAPA LOAD ADD
0007      000A          #adres zmiennej y
0008      4F00          WRITE HALT NOP NOP
0009      0000          #miejsce przechowywania zmiennej x
000A      0000          #miejsce przechowywania zmiennej y
{% endhighlight %}

## Języki wysokiego poziomu
Asembler pozwala komunikować się z komputerem w dużo bardziej przyjazny sposób niż ciągi zer i jedynek, umożliwiając pisanie bardziej zaawansowanych programów. Jednak nadal jest to język dość trudny, zwłaszcza, jeśli chodzi o analizę kodu lub jego modyfikację. Między innymi z tego powodu następnym krokiem w rozwoju języków programowania było powstanie tak zwanych języków wysokiego poziomu. Języki te umożliwiają porozumiewanie się z komputerem w jeszcze bardziej przyjazny sposób, uwalniając programistę od konieczności znajomości konkretnej architektury procesora. Kod napisany w języku wysokiego poziomu jest tłumaczony na język Asemblera (lub bezpośrednio na język maszynowy) w tak zwanym procesie kompilacji. Jednym z pierwszych stworzonych języków z tej kategorii jest język o nazwie [Fortran][2]. Niektóre jego cechy to:

* deklaracje zmiennych
* definiowanie procedur
* operatory logiczne
* operatory arytmetyczne
* operatory działań na liczbach

Poniższy kod jest tym samym programem z poprzedniego akapitu napisanym w języku Fortran:

{% highlight fortran %}
IMPLICIT NONE
INTEGER :: x,y
READ * ,x,y
PRINT * ,'wynik',x+y
END
{% endhighlight %}

### Programowanie imperatywne
Dzięki językom wysokiego poziomu komunikacja z komputerem została mocno uproszczona w stosunku do języka Asemblera, przez co analiza programów oraz ich modyfikacja jest dużo prostsza i mniej podatna na błędy. Na bazie takiego poziomu abstrakcji, powstały różnego rodzaju paradygmaty (lub metodyki) programowania. Jednym z takich paradygmatów jest tzw. programowanie imperatywne, które charakteryzuje się tym, że program reprezentowany jest w postaci stanu maszyny, który zmieniany jest przez ciąg instrukcji. Innymi słowy, programując imperatywnie mówimy komputerowi jak ma rozwiązać dany problem. Przykładowo, poniższy kod napisany w języku [C][3] oblicza sumę kwadratów liczb od 1 do n:

{% highlight c %}
#include <stdio.h>

int main()
{
  int n = 10;
  int sum = 0;
  int i = 1;

  loop:
    if (i > n)
    {
      goto end;
    }		

    sum += i*i;
    i += 1;
    goto loop;

  end:

  printf("%i\n", sum);
}
{% endhighlight %}

### Programowanie strukturalne
Patrząc na powyższy przykład, można zauważyć, że wykorzystuje on tak zwaną instrukcję skoku `goto`. Instrukcja ta nakazuje komputerowi, przy spełnieniu odpowiednich warunków,  przejście do odpowiedniej części programu i kontynuowanie wykonywania programu od tego miejsca. Takie "skakanie" bardzo utrudnia analizę kodu. Paradygmat programowania strukturalnego ogranicza używanie instrukcji skoku oraz definiuje strukturę kodu. Kod taki powinien składać się z kilku dobrze zdefiniowanych instrukcji:

* sekwencja – wykonanie instrukcji jedna po drugiej
* wybór – if-then-else
* iteracja – for, while

Przykład, w języku C obliczania sumy kwadratów liczb od 1 do n z użyciem powyższych instrukcji może wyglądać np. tak:

{% highlight c %}
#include <stdio.h>

int main()
{
  int n = 10;
  int sum = 0;
  for (int i = 1; i <= n; i++)
  {
    sum += i*i;
  }

  printf("%i\n", sum);
}
{% endhighlight %}

### Programowanie proceduralne
Głównym założeniem tego paradygmatu jest to, aby kod dzielić na mniejsze kawałki, wykonujące jasno określone operacje. Większość języków wysokiego poziomu (a może wszystkie ;)) dostarczają konstrukcję zwaną procedurą. W procedurach można zakodować poszczególne fragmenty programu tak, aby odseparować je od reszty, a także móc je wykorzystać w innych miejscach. Przykład w języku C, podziału na procedury (`main(...), squaresSum(...), square(...)`) powyższego programu może wyglądać np. tak:

{% highlight c %}
#include <stdio.h>

int squaresSum(int n);
int square(int i);

int main()
{
  int sum = squaresSum(10);
  printf("%i\n", sum);
}

int squaresSum(int n)
{
  int sum = 0;
  for (int i = 1; i <= n; i++)
  {
    sum += square(i);
  }

  return sum;
}

int square(int i)
{
  return i*i;
}
{% endhighlight %}

### Programowanie obiektowe
Pewną trudnością w programowaniu proceduralnym jest to, że dane, na których operują procedury nie są ze sobą powiązane. Dotyczy to zwłaszcza zadeklarowanych zmiennych globalnych, do których można mieć dostęp z dowolnego fragmentu kodu. Ponieważ główną koncepcją programowania imperatywnego, a co za tym idzie również programowania proceduralnego, jest zmiana stanu maszyny, a stan ten może być zmieniany w dowolnym fragmencie kodu, może prowadzić to do powstawania błędów w programie, poprzez nieprawidłowe ustawienie tego stanu. Aby zminimalizować takie operacje, powstały tzw. języki obiektowe, a co za tym idzie również paradygmat programowania obiektowego. Głównym założeniem tego paradygmatu jest to, aby dane oraz procedury, które operują na tych danych, były ze sobą powiązane w takim sensie, aby zmiany na tych danych mogły dokonywać tylko te jasno określone procedury. W większości języków obiektowych konstrukcja do budowy takich powiązań nazywa się klasą (`class`), a instancję konkretnej klasy nazywa się obiektem (`object`). Procedury operujące na danych obiektu nazywa się metodami (`method`). Przykład programu w języku [C#][4], obliczającego sumę kwadratów liczb od 1 do n może wyglądać np. tak:

{% highlight csharp %}
using System.IO;
using System;

public class Program
{
  public static void Main()
  {
    var calc = new SquaresSumCalculator();
    var sum = calc.Calculate(10);
    Console.WriteLine(sum);
  }
}

public class SquaresSumCalculator
{
  public int Calculate(int n)
  {
    return this.SquaresSum(n);
  }

  private int SquaresSum(int n)
  {
    int sum = 0;
    for (int i = 1; i <= n; i++)
    {
      sum += this.Square(i);
    }

    return sum;
  }

  private int Square(int i)
  {
    return i*i;
  }
}
{% endhighlight %}

W metodzie `Main` jest tworzony obiekt `SquaresSumCalculator`, na którym to obiekcie wywoływana jest metoda `Calculate`. Metoda ta jest zadeklarowana jako publiczna w klasie `SquaresSumCalculator`. Pozostałe metody są oznaczone modyfikatorem dostępu `private`, który to modyfikator "zabrania" dostępu do tych metod fragmentom kodu innym niż te, które znajdują się w klasie `SquaresSumCalculator`. Jak widać na przykładzie w ramach implementacji klasy, wykorzystane jest programowanie proceduralne, ale mogłoby być użyte dowolne inne. "Na zewnątrz klasy" dostępna jest tylko metoda `Calculate`, a konsumer (klasa `Program`) wywołujący tę metodę nie wie nic o szczegółach jej wewnętrznej implementacji.

### Programowanie funkcyjne
Równolegle z rozwojem paradygmatów bazujących na programowaniu imperatywnym rozwijała się koncepcja programowania funkcyjnego. W programowaniu imperatywnym główną koncepcją jest stan maszyny oraz zmiana tego stanu, natomiast w programowaniu funkcyjnym główną koncepcją jest funkcja oraz wyliczanie wyrażeń przez funkcje. Mówimy komputerowi bardziej, co ma zrobić, a nie, jak ma to zrobić. W większości przypadków języki funkcyjne są bardziej zwięzłe w konstrukcji od języków imperatywnych. Dla porównania przykład programu obliczającego sumę kwadratów liczb od 1 do n, zapisany w języku [F#][5], może wyglądać np. tak:

{% highlight ocaml %}
let square x = x * x

let squaresSum n =
  [1..n]
  |> List.map square
  |> List.sum

let sum = squaresSum 10

printfn "%i" sum
{% endhighlight %}

### Programowanie logiczne
Jeszcze innym paradygmatem jest tzw. programowanie logiczne, gdzie program reprezentowany jest przez  zapis stwierdzeń rachunku predykatów pierwszego rzędu, a wyniki programu jako rezultat automatycznego wnioskowania z tych stwierdzeń. Przykład programu stwierdzającego relację rodzeństwa w języku [Prolog][6]:

{% highlight prolog %}
parent(a,x).
parent(a,y).
parent(a,z).
parent(b,x).
parent(b,y).
parent(b,z).
sibilings(M,N) :-
  parent(K, M),
  parent(K, N).

sibilings(x, y).
{% endhighlight %}

## Podsumowanie
Z komputerem można "rozmawiać" na wiele sposobów, ale sama znajomość języków programowania oraz paradygmatów programowania nie wystarcza do tworzenia złożonych programów cechujących się dobrą jakością. W kolejnych seriach z cyklu "Wytwarzanie oprogramowania" zostaną opisane wybrane techniki pozwalające zbliżyć się do jakości akceptowalnej przez odbiorcę, bo przecież nie istnieją programy w 100% odporne na błędy, a może tak ;).

[1]: https://www.ii.uni.wroc.pl/~prz/200506/2006lato/cpp/zadania/sextium.pdf "Sextium II processor"
[2]: https://gcc.gnu.org/wiki/GFortranStandards "Fortran language"
[3]: http://www.open-std.org/jtc1/sc22/wg14/www/docs/n1124.pdf "C language"
[4]: https://msdn.microsoft.com/en-us/library/ms228593.aspx "C# language"
[5]: http://fsharp.org/specs/language-spec/ "F# language"
[6]: http://www.deransart.fr/prolog/docs.html "Prolog language"

{{ site.mark_post_as_end }}

### {{ site.comments }}
begin-**test_user**-test_comment-2020-09-12 11:22 UTC 
