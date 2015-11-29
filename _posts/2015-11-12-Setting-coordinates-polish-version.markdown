---
layout: post
title:  "Setting coordinates for atoms - Polish version"
date:   2015-11-12 00:30:00
categories:
hideFromIndex: true
---

## Jak działa ustawianie pozycji atomów

Wejściem do algorytmu ustawiania pozycji jest graf cząsteczki.

Przed przystąpieniem do ustalania pozycji graf musi mieć ustalone struktury: spójne części, dwuspójne składowe, pierścienie (wykrywanie za pomocą algorytmu SSSR - smallest set of smallest rings). Ich wykrywanie odbywa się w module `SMILES.detectStructure`.

# Odwracanie pierścieni

Proces ułatwiający późniejsze kroki.

Pierścień ułożony na płaszczyźnie może być opisany na dwa sposoby: jego elementy mogą być podane w kolejności zgodnej z ruchem wskazówek zegara lub przeciwnej (można też o tym myśleć jako o patrzeniu na niego od góry lub od dołu). Na tym etapie obiekty opisujące pierścienie są przekształcane w ten sposób, aby były ze sobą spójne - opisywały pierścienie w tych samych kierunkach. Nie ma tu znaczenia absolutna konfiguracja.

Działanie algorytmu można opisać w ten sposób: znajdź dwa pierścienie (`A`, `B`), które mają wspólną krawędź (`x-y`) a następnie odwróć je tak aby pierścień `A` miał postać `...xy...`, a pierścień `B` postać `...yx...` (lub też na odwrót). Oczywiście należy zadbać o to, aby nie odwracać cały czas tych samych pierścieni - w tym projekcie użyty jest DFS po pierścieniach, gdzie sąsiedztwo rozumiane jest jako posiadanie wspólnej krawędzi.

Warto zauważyć, że możliwa jest sytuacja, gdy nie da się tak odwrócić pierścieni, aby spełnić wymagania. Tak sytuacja oznacza, że cząsteczka ma pierścienie, wystające pod lub nad płaszczyznę. Przykładami takich cząsteczek są: morfina, fuleren C60, kuban (C8H8). W przypadku fulerenu C60, kubanu, oraz innych związków mających postać klatki może się wydawać, ze problem nie występuje - dzieje się tak dlatego, że algorytm znajdowania pierścieni (SSSR) nie wykrywa wszystkich ścian takich struktur jako cykli.

Aktualnie w naszej implementacji błąd odwracania pierścieni jest sygnalizowany przez wyjątek.

# Zachłanne ustawianie pierścieni

Na tym etapie każda dwuspójna składowa przetwarzana jest osobno, a współrzędne wierzchołków, które powstają, są obliczane względem tej składowej. Na przykład wierzchołek, który znajduje się na styku wielu dwuspójnych składowych, będzie miał wygenerowanych wiele różnych współrzędnych.

Opisując działanie tego kroku krótko: mając ustawiony pewien zbiór wierzchołków próbujemy ustawić kolejne tak, aby pierścienie występujące w grafie były wielokątami foremnymi. Zawsze rozważamy jeden pierścień na raz, nie zastanawiamy się nad tym co będzie dalej, a ustawionych raz wierzchołków już nie ruszamy - jest to algorytm zachłanny.

Pierścienie przechodzone są DFSem, gdzie sąsiedztwo rozumiemy jako posiadanie wspólnej krawędzi. Dla każdego pierścienia:

 * Generowany jest zestaw punktów będących wierzchołkami wielokąta foremnego o liczbie wierzchołków równej liczbie wierzchołków rozpatrywanego pierścienia.
 * Obliczana jest optymalna rotacja i translacja, tak aby odpowiednie wierzchołki wygenerowanego wielokąta foremnego, po zastosowaniu przekształceń, nałożyły się jak najlepiej (minimalna suma kwadratów) z ustawionymi już wierzchołkami pierścienia.
 * Obliczone przekształcenia są zastosowywane do punktów wielokąta foremnego.
 * Wierzchołki pierścienia, które nie mają jeszcze ustawionej pozycji są ustawiane w miejscach, w których znajdują się przekształcone wierzchołki wielokąta foremnego.

W obrębie pojedynczej dwuspójnej składowej graf pierścieni jest spójny, więc przetworzymy wszystkie pierścienie tej składowej. Oprócz tego każdy wierzchołek dwuspójnej składowej musi być w choć jednym cyklu (jedyny wyjątek to składowe jedno i dwuelementowe), a zatem każdy wierzchołek będzie miał ustawioną pozycję.

Ten algorytm działa dobrze dla cząsteczek, których pierścienie w obrębie każdej dwuspójnej składowej tworzą strukturę drzewiastą (bez cykli), oraz dla cząsteczek, których cykle nie są naprężone.

Dla cząsteczek nie spełniających tych warunków, algorytm ustawi pozycje jednak wizualizacja stworzona na ich podstawie nie będzie wyglądała dobrze.

# Trywialne dwuspójne składowe

W tym momencie każda dwuspójna składowa, która zawierała choć jeden cykl ma ustawione współrzędne wszystkich swoich wierzchołków. Należy zająć się składowymi bez cykli. Mogą one mieć jeden lub dwa wierzchołki, nie więcej. Postępujemy tak:

 * dla składowej o jednym wierzchołku: ustawiamy jego pozycję wewnątrz tej składowej na `(0, 0)`,
 * dla składowej o dwóch wierzchołkach: ustawiamy ich pozycje wewnątrz tej składowej na `(0, 0)` oraz `(d, 0)`

gdzie `d` jest długością krawędzi.

# Termiczne ustawianie pierścieni

Ten etap ma na celu poprawić niedoskonałości zachłannego algorytmu ustawiania pierścieni, działającego na naprężonych cząsteczkach. W skrócie algorytm można opisać tak: pozwalamy na przemieszczanie się wierzchołków oraz dodajemy siły, które chcą uczynić z każdego pierścienia wielokąt foremny.

Algorytm przeprowadza wiele iteracji poprawiania pozycji i przerywa powtarzanie, gdy rozwiązanie przestaje się bardzo zmieniać. ("Bardzo" jest definiowane przez liczbę.) Miarą zmiany rozwiązania jest średnia długość przesunięcia wierzchołka, które wystąpiło podczas tego kroku.

Pozycje wierzchołków są cały czas rozumiane jako pozycje wewnątrz dwuspójnych składowych. Można sobie wyobrażać, że każda dwuspójna składowa to inna cząsteczka. Ich złączenie nastąpi później.

Pojedyncza iteracja dla każdego pierścienia:

 * oblicza pozycje wierzchołków wielokąta foremnego o liczbie wierzchołków równej liczbie wierzchołków przetwarzanego pierścienia,
 * przesuwa i obraca wielokąt tak, aby jak najlepiej nałożył się z wierzchołkami przetwarzanego pierścienia (minimalna suma kwadratów),
 * zapisuje pozycje wierzchołków przekształconego wielokąta do tablic powiązanych z odpowiednimi wierzchołkami grafu

Efektem tych operacji jest uzyskanie, dla każdego wierzchołka, tablic pożądanych pozycji. Z tych tablic liczona jest średnia, a następnie pozycja wierzchołka jest zastępowana tą średnią. Kolejna iteracja będzie operowała na nowych współrzędnych.

Średnia może być ważona - na przykład jeśli zależy nam aby mniejsze pierścienie miały większy "priorytet foremności" możemy pozycje wynikające z mniejszych pierścieni brać z większą wagą niż te wynikające z pierścieni dużych.

# Uzgadnianie dwuspójnych składowych

Na tym etapie mamy ustawione pozycje każdego wierzchołka wewnątrz każdej dwuspójnej składowej do której on należy. Aby uzyskać pozycje przydatne do wizualizacji należy uzgodnić współrzędne pomiędzy składowymi. Przez uzgadnianie należy rozumieć znalezienie dla każdej składowej wektora translacji i kąta rotacji, które trzeba zastosować do współrzędnych wewnątrz tej składowej aby uzyskać współrzędne absolutne.

Punktem wyjścia do znalezienia odpowiednich przekształceń będą wierzchołki, które należą do więcej niż jednej składowej - nazywam je złączeniami (junctions). Są tutaj dwa warunki:

 * Złączenie ma jedną pozycję absolutną. To oznacza, że współrzędne złączenia w każdej ze składowych przekształcone przez translację i rotację odpowiednich składowych muszą dać jeden punkt.
 * Złączenie reprezentuje konkretny atom, który na rysunku ma mieć określone kąty pomiędzy swoimi wiązaniami. To nakłada warunek na rotację poszczególnych składowych.

Dwuspójne składowe wewnątrz pojedynczego spójnego fragmentu grafu tworzą drzewo. Naturalne wydaje się przetwarzanie złączeń w ustalonym porządku obchodzenia drzewa.

W aktualnej implementacji najpierw, w porządku sufiksowym (nie ma on znaczenia), ustalane są lokalne transformacje - to znaczy dla składowej `A` oraz pewnego jej dziecka (w sensie drzewa) `B` obliczana jest transformacja, której zastosowanie do składowej `B` spowoduje, że w miejscu złączenia będzie "pasować" do składowej `A`. Następnie w porządku prefiksowym składowe są przekształcane zgodnie z wyliczonymi wcześniej wartościami, z tą różnicą, że przekształcenia są przekazywane w dół drzewa i akumulowane, aby składowe pasowały do siebie globalnie.

# Ustawianie spójnych części

Ostatnim krokiem jest rozsunięcie spójnych podgrafów tak, żeby były rysowane w innych miejscach i graficznie na siebie nie zachodziły. W tym celu dla każdej spójnej części obliczane są maksymalne i minimalne współrzędne x i y osiągane przez wierzchołki tej części. Potem na podstawie wyliczonych wartości podgrafy są przesuwane tak, aby w pionie były wyśrodkowane, a w poziomie były ustawione jeden za drugim, z określonymi odstępami pomiędzy.

