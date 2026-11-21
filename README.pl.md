# Problem komiwojażera: trasa objazdowa 10 politechnik w Polsce

*English version: [README.md](README.md)*

## Overview

Projekt na przedmiot Laboratorium zastosowań sztucznej inteligencji (LZSI). Algorytm ewolucyjny szuka najkrótszej trasy zamkniętej, która odwiedza każdą z 10 politechnik dokładnie raz i wraca do punktu startowego.

To klasyczny problem komiwojażera (TSP). Liczba tras rośnie silniowo z liczbą miast, więc przy większych instancjach nie da się sprawdzić wszystkich. Przy 10 miastach przegląd zupełny jeszcze się mieści w czasie poniżej sekundy, dzięki czemu można sprawdzić, czy algorytm trafił w optimum.

Na liście są politechniki: Warszawska, Krakowska, Gdańska, Wrocławska, Śląska (Gliwice), Łódzka, Poznańska, Lubelska, Białostocka i Rzeszowska. Współrzędne ich głównych budynków notatnik pobiera przy każdym uruchomieniu z modelu Gemini. Gdy zapytanie się nie uda albo odpowiedź nie przejdzie kontroli, używa pliku `politechniki_alt.csv`.

W ostatnim uruchomieniu (dane z Gemini) algorytm znalazł trasę długości 1720,3 km w 594. pokoleniu. Przegląd zupełny dał tę samą wartość, czyli trasa jest optymalna. Na danych z pliku zapasowego ta sama trasa ma 1721,3 km.

## Technologies

| Narzędzie | Zastosowanie |
|---|---|
| Python 3.13 | język implementacji |
| Jupyter Notebook | środowisko pracy |
| Gemini API (`google-genai`, model `gemini-3.8-flash`) | pobranie współrzędnych politechnik |
| python-dotenv | wczytanie klucza API z pliku `.env` |
| pandas | odczyt CSV z odpowiedzi AI lub z pliku zapasowego, kontrola danych |
| NumPy | macierz odległości, permutacje, losowanie z zadanym rozkładem |
| Matplotlib | wykres zbieżności i statyczna mapa trasy |
| folium | interaktywna mapa trasy na podkładzie OpenStreetMap |
| `random`, `itertools`, `sys`, `io` | biblioteka standardowa |

## Installation

1. Pobierz folder projektu.
2. Opcjonalnie utwórz i aktywuj środowisko wirtualne:

   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\Activate.ps1
   # Linux / macOS:
   source .venv/bin/activate
   ```

3. Zainstaluj zależności:

   ```bash
   pip install -r requirements.txt
   ```

4. Utwórz w folderze projektu plik `.env` z kluczem do Gemini API (klucz generuje się w Google AI Studio):

   ```
   GOOGLE_API_KEY=twój_klucz_api
   ```

   Zamiast pliku `.env` można ustawić zmienną środowiskową `GOOGLE_API_KEY`, ale trzeba to zrobić przed uruchomieniem Jupytera. Bez klucza notatnik też działa, tylko bierze współrzędne z `politechniki_alt.csv`.

5. Uruchom notatnik i wykonaj wszystkie komórki po kolei (Run All):

   ```bash
   jupyter notebook tsp-polish-polytechnics_pl.ipynb
   ```

Ziarno losowości jest ustawione na 42 dla `random` i `numpy.random`, więc przy tych samych współrzędnych wynik się powtarza. Gemini potrafi jednak przy kolejnych uruchomieniach zwrócić trochę inne współrzędne, mimo `temperature=0`. Kolejność miast na trasie zostaje wtedy ta sama, ale długość trasy może się zmienić w granicach około ±10 km, a algorytm może dojść do niej w innym pokoleniu. Z tego samego powodu wykresy w notatniku i pliki PNG mogą się nieznacznie różnić między uruchomieniami. Plik zapasowy daje zawsze ten sam wynik: 1721,3 km.

## Project Structure

```
original_LZSI/
├── tsp-polish-polytechnics_pl.ipynb   # notatnik po polsku: kod, wyniki, wykresy i wnioski
├── tsp-polish-polytechnics_en.ipynb   # ten sam notatnik po angielsku
├── politechniki_alt.csv               # zapasowe współrzędne politechnik
├── raport.tex                         # raport z projektu (LaTeX)
├── requirements.txt                   # zależności Pythona
├── README.pl.md
└── README.md
```

Uruchomienie polskiego notatnika tworzy pliki `zbieznosc.png` (wykres zbieżności) i `trasa.png` (mapa trasy), a angielskiego – `convergence.png` i `route.png`.

Pliku `.env` z prywatnym kluczem API i folderu `.venv` nie należy nikomu udostępniać.

Notatnik ma sześć sekcji:

1. Dane wejściowe: zapytanie do Gemini, kontrola odpowiedzi i w razie potrzeby plik zapasowy.
2. Macierz odległości liczona wzorem haversine, w kilometrach.
3. Algorytm ewolucyjny: funkcje, parametry i uruchomienie.
4. Wynik: najlepsza trasa odcinek po odcinku i wykres zbieżności.
5. Weryfikacja przeglądem zupełnym.
6. Wizualizacja trasy w Matplotlib i na mapie folium.

## How It Works

### Dane wejściowe

Notatnik prosi Gemini o CSV z kolumnami `nazwa,miasto,szerokosc,dlugosc` (6 miejsc po przecinku, `temperature=0`), usuwa ewentualne znaczniki ``` i wczytuje tekst przez pandas. Funkcja `validate` sprawdza nazwy kolumn, czy jest dokładnie 10 oczekiwanych miast i czy współrzędne leżą w prostokącie 49–55° N, 14–24,2° E. Jeśli zapytanie rzuci wyjątek albo kontrola coś znajdzie, dane idą z `politechniki_alt.csv`.

### Odległości

Dane to stopnie geograficzne, więc odległość liczy wzór haversine po powierzchni kuli o promieniu R = 6371 km (w linii prostej). Wynikiem jest symetryczna macierz 10 × 10.

### Algorytm ewolucyjny

Algorytm jest oparty na przykładzie z zajęć.

| Element | Realizacja |
|---|---|
| Osobnik | permutacja numerów politechnik, np. `[3, 0, 7, …]` to trasa 3 → 0 → 7 → … → 3 |
| Populacja początkowa | losowe permutacje z `np.random.permutation` |
| Ocena (`fitness`) | długość trasy zamkniętej w km, minimalizowana |
| Selekcja (`select`) | ruletka z prawdopodobieństwem proporcjonalnym do 1/L; połowa populacji, bez zwracania |
| Mutacja (`mutate`) | odwrócenie losowego fragmentu trasy |
| Krzyżowanie (`cross`) | fragment od jednego rodzica, pozostałe miasta w kolejności z drugiego; powstaje dwóch potomków |
| Nowe pokolenie | 50 mutantów i 100 potomków, z których losowanych jest 100 osobników |

Najlepsza znaleziona dotąd trasa jest trzymana w zmiennej `best_solution`, osobno od populacji.

Parametry:

```python
population_size   = 100    # liczba osobników w populacji
no_of_generations = 2000   # liczba pokoleń
mutation_prob     = 0.5    # prawdopodobieństwo mutacji
crossover_prob    = 0.3    # prawdopodobieństwo krzyżowania
```

### Weryfikacja

Dla 10 miast jest (10 − 1)! / 2 = 181 440 różnych tras zamkniętych. Notatnik ustala Warszawę jako start, sprawdza wszystkie 9! permutacji pozostałych miast i porównuje najkrótszą trasę z wynikiem algorytmu.
