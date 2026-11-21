# Travelling Salesman Problem: a tour of 10 Universities of Technology in Poland

*Polish version: [README.pl.md](README.pl.md)*

## Overview

A project for the course Laboratory of Artificial Intelligence Applications (LZSI). An evolutionary algorithm searches for the shortest closed route that visits each of 10 Polish universities of technology exactly once and returns to the starting point.

This is the classic Travelling Salesman Problem (TSP). The number of routes grows factorially with the number of cities, so for larger instances it is impossible to check them all. With 10 cities an exhaustive search still runs in under a second, which makes it possible to check whether the algorithm found the optimum.

The list includes the universities of technology in Warsaw, Cracow, Gdańsk, Wrocław, Gliwice (Silesian), Łódź, Poznań, Lublin, Białystok and Rzeszów. The notebook fetches the coordinates of their main buildings from the Gemini model on every run. If the request fails or the response does not pass validation, it uses the file `politechniki_alt.csv`.

In the latest run (data from Gemini) the algorithm found a route of 1,720.3 km in generation 594. The exhaustive search returned the same value, so the route is optimal. With the fallback data the same route is 1,721.3 km long.

The notebook is available in English (`tsp-polish-polytechnics_en.ipynb`) and Polish (`tsp-polish-polytechnics_pl.ipynb`). The report `raport.tex` is in Polish.

## Technologies

| Tool | Purpose |
|---|---|
| Python 3.13 | implementation language |
| Jupyter Notebook | working environment |
| Gemini API (`google-genai`, model `gemini-3.8-flash`) | fetching the coordinates of the universities |
| python-dotenv | loading the API key from the `.env` file |
| pandas | reading the CSV from the AI response or the fallback file, data validation |
| NumPy | distance matrix, permutations, weighted random sampling |
| Matplotlib | convergence plot and static route map |
| folium | interactive route map on an OpenStreetMap base layer |
| `random`, `itertools`, `sys`, `io` | standard library |

## Installation

1. Download or clone the project folder.
2. Optionally create and activate a virtual environment:

   ```bash
   python -m venv .venv
   # Windows:
   .venv\Scripts\Activate.ps1
   # Linux / macOS:
   source .venv/bin/activate
   ```

3. Install the dependencies:

   ```bash
   pip install -r requirements.txt
   ```

4. Create a `.env` file in the project folder with your Gemini API key (you can generate one in Google AI Studio):

   ```
   GOOGLE_API_KEY=your_api_key
   ```

   Instead of the `.env` file you can set the `GOOGLE_API_KEY` environment variable, but it has to be set before Jupyter is started. The notebook also works without a key; it then takes the coordinates from `politechniki_alt.csv`.

5. Open the notebook and run all cells in order (Run All):

   ```bash
   jupyter notebook tsp-polish-polytechnics_en.ipynb
   ```

The random seed is set to 42 for both `random` and `numpy.random`, so with the same coordinates the result is reproducible. However, Gemini may return slightly different coordinates on subsequent runs, even with `temperature=0`. The order of the cities on the route stays the same, but its length may change by roughly ±10 km and the algorithm may reach it in a different generation. For the same reason, the plots in the notebook and the PNG files may differ slightly between runs. The fallback file always gives the same result: 1,721.3 km.

## Project Structure

```
original_LZSI/
├── tsp-polish-polytechnics_en.ipynb   # notebook in English: code, results, plots and conclusions
├── tsp-polish-polytechnics_pl.ipynb   # the same notebook in Polish
├── politechniki_alt.csv               # fallback coordinates of the universities
├── raport.tex                         # project report (Polish, LaTeX)
├── requirements.txt                   # Python dependencies
├── README.md
└── README.pl.md
```

Running the English notebook creates `convergence.png` (convergence plot) and `route.png` (route map).

The `.env` file with your private API key and the `.venv` folder should never be shared; both are listed in `.gitignore`.

The notebook has six sections:

1. Input data: the Gemini request, response validation and, if needed, the fallback file.
2. Distance matrix computed with the haversine formula, in kilometres.
3. Evolutionary algorithm: functions, parameters and the run.
4. Result: the best route leg by leg and the convergence plot.
5. Verification by exhaustive search.
6. Route visualisation in Matplotlib and on a folium map.

## How It Works

### Input data

The notebook asks Gemini for a CSV with the columns `name,city,latitude,longitude` (6 decimal places, `temperature=0`), strips any ``` markers and reads the text with pandas. The `validate` function checks the column names, that there are exactly the 10 expected cities, and that the coordinates lie within the rectangle 49–55° N, 14–24.2° E. If the request raises an exception or validation finds a problem, the data comes from `politechniki_alt.csv` (its Polish column names are mapped to the English ones after loading).

### Distances

The data is in geographic degrees, so distances are computed with the haversine formula on a sphere of radius R = 6371 km (straight-line distance). The result is a symmetric 10 × 10 matrix.

### Evolutionary algorithm

The algorithm is based on the example from class.

| Element | Implementation |
|---|---|
| Individual | a permutation of university indices, e.g. `[3, 0, 7, …]` is the route 3 → 0 → 7 → … → 3 |
| Initial population | random permutations from `np.random.permutation` |
| Fitness (`fitness`) | length of the closed route in km, minimised |
| Selection (`select`) | roulette wheel with probability proportional to 1/L; half of the population, without replacement |
| Mutation (`mutate`) | reversal of a random segment of the route |
| Crossover (`cross`) | a segment from one parent, the remaining cities in the order of the other parent; produces two offspring |
| Next generation | 50 mutants and 100 offspring, from which 100 individuals are drawn |

The best route found so far is kept in the `best_solution` variable, separately from the population.

Parameters:

```python
population_size   = 100    # number of individuals in the population
no_of_generations = 2000   # number of generations
mutation_prob     = 0.5    # mutation probability
crossover_prob    = 0.3    # crossover probability
```

### Verification

For 10 cities there are (10 − 1)! / 2 = 181,440 distinct closed routes. The notebook fixes Warsaw as the start, checks all 9! permutations of the remaining cities and compares the shortest route with the algorithm's result.
