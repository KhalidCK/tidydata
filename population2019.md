---
jupyter:
  jupytext:
    formats: ipynb,md
    text_representation:
      extension: .md
      format_name: markdown
      format_version: '1.2'
      jupytext_version: 1.5.2
  kernelspec:
    display_name: Python 3
    language: python
    name: python3
---

## Support

```python
import sys
import logging

import pandas as pd
pd.set_option("display.max_rows", 120)
pd.set_option("display.max_columns", 120)

logging.basicConfig(format='[%(asctime)s] %(levelname)s - %(message)s',
                        level=logging.INFO)
```

```python
def pandas_df_to_markdown_table(df,name):
    '''
    Write the df in name to markdown format
    '''
    fmt = ['---' for i in range(len(df.columns))]
    df_fmt = pd.DataFrame([fmt], columns=df.columns)
    df_formatted = pd.concat([df_fmt, df])
    df_formatted.to_csv(name,sep="|", index=False)
```

```python
logging.info('start')
```

## Dataset

```python
!mkdir -p data
!wget -q -O data/estim-pop-dep-sexe-gca-1975-2019.xls data https://raw.githubusercontent.com/KhalidCK/tidydata/master/data/estim-pop-dep-sexe-gca-1975-2019.xls
```

```python
!ls data
```

<!-- #region -->
>Chaque année, l'Insee estime la population des régions et des départements (France métropolitaine et DOM) à la date du 1ᵉʳ janvier. Ces estimations annuelles de population sont déclinées par sexe et par âge (quinquennal, classes d'âge).


[ref](https://www.insee.fr/fr/statistiques/1893198)
<!-- #endregion -->

```python
import pandas as pd
```

```python
pop2019 = pd.read_excel('data/estim-pop-dep-sexe-gca-1975-2019.xls'
                        ,sheet_name='2019'
                        ,skiprows=4)
```

```python
pop2019.head()
```

Des colonnes ne sont pas detecté par pandas.La feuille de donnée essaye de créer une agrégation sur des colonnes visuellement.

On donne des noms pertients à ces colonnes en faisant une inspection de la feuille Excel.

```python
pop2019.columns = ["departements","nom_departement"]+list(pop2019.columns[2:])
```

C'est souvent une bonne idée de trouver une colonne qui pourra faire office de clé (index)

```python
assert len(pop2019.departements) == len(pop2019.departements.unique())
```

les departements sont usuellement sur 2 ou 3 (DOM) caractères

```python
all(pop2019.departements.str.len() <=3)
```

Quelles sont les departements avec plus de deux 3 digits

```python
pop2019[pop2019.departements.str.len()>3][['departements']]
```

```python
dep_nom = pop2019[["departements","nom_departement"]].copy()
```

```python
pop2019=(pop2019
                .loc[pop2019.departements.str.len()<3]
                .drop(columns="nom_departement")
                .set_index('departements'))
```

```python
pop2019.tail()
```

```python
assert all(pop2019.index.str.len() <=3)
```

Les colonnes sont assignés à 3 groupes en fonction de la portion de la colonne (tous les 6 elements => nouvelle section).

Dans l'ordre d'apres le fichier initial : 'ensemble','hommes','femmes'

```python
from itertools import zip_longest

#https://docs.python.org/3.7/library/itertools.html
def grouper(iterable, n, fillvalue=None):
    "Collect data into fixed-length chunks or blocks"
    # grouper('ABCDEFG', 3, 'x') --> ABC DEF Gxx"
    args = [iter(iterable)] * n
    return zip_longest(*args, fillvalue=fillvalue)
```

```python
elements = ['ensemble','hommes','femmes']
data = {}
for element,cols in zip(elements,list(grouper(pop2019.columns,6))):
    df = pop2019.loc[:,cols]
    df.columns = [col.split('.')[0] for col in df.columns]
    data[element] = df.drop(columns='Total')
```

```python
data.keys()
```

```python
data["hommes"].head()
```

```python
data["femmes"].head()
```

```python
pandas_df_to_markdown_table(data["ensemble"].head().reset_index(),'peek-csv-table.md')
```

Le nom des colonnes sont des variables

```python
def to_tidy(df)->pd.DataFrame:
    return df.reset_index().melt(id_vars='departements',var_name="age",value_name="nb")
```

```python
tidys = {segment:to_tidy(df) for segment,df in data.items()}
```

```python
tidys['femmes'].head()
```

```python
france = tidys['ensemble']
france.nb=france.nb.astype(int)
```

```python
tidy_sample=france.sample(10)
```

```python
tidy_sample
```

```python
pandas_df_to_markdown_table(tidy_sample,'sample-tidy-france.md')
```

```python
france = france[france.age!="Total"]
```

```python
france_total_dep = (france[['departements','nb']]
                    .groupby('departements')
                    .sum()
                    .rename(columns={'nb':'total'}))
```

```python
france_total_dep.head()
```

```python
#vérification, ordre de grandeur ok
france_total_dep.sum()/10**6
```

```python
pop = "ensemble"
```

```python
deps = ['59','75','67']
```

```python
sub = tidys[pop]
sub = sub[sub.departements.isin(deps)]
```

```python
france[france.departements.isin(['59','75','67'])]
```

```python
deps = ['59','75','67']
```

```python
sub=sub.join(france_total_dep,on="departements")
```

```python
sub['pourcentage'] = ((sub['nb'] / sub['total'])*100).round(2)
```

```python
sub.sort_values('departements')
```

```python
import altair as alt

alt.Chart(sub).mark_bar().encode(
    x="age:O",
    y="pourcentage:Q",
    color="age:N",
    column='departements:N')
```
