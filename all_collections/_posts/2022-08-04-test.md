---
layout: post
title: Оптимизация работы с Пандас
date: 2020-06-21
categories: ["ru", "dev"]
---

#### Считывание больших данных

Очень полезная возможность Pandas — итерирование по датафрейму, если вдруг он не влезает в память (такое часто бывает в реальной работе).

```python
data_iter = pd.read_csv('babynames.csv', chunksize=1000)

def data_prep(df, func):
    """ Отфильтруем данные и сложим в другой файлик
    """
    for chunk in df:
        tmp = func(chunk)
        tmp.to_csv('dataset.csv')
        
def func_filter(data):
    return data.head(5)

data_prep(data_iter, func_filter)
```

#### Понижение размера дата фрейма

При считывании таблицы в pandas есть возможность повлиять на размер занимаемых данных путем снижения количества выделяемой памяти, но с этим нужно быть осторожным, если у вас очень большие числа, то снижение 64 бит до 32 может попросту вызвать переполнение и неверные данные. Тут лучше небольшой бенчмарк.

```python
# Загрузка данных из файла
data = pd.read_csv('babynames.csv')
data.info()

<class 'pandas.core.frame.DataFrame'>
RangeIndex: 1690784 entries, 0 to 1690783
Data columns (total 4 columns):
 #   Column  Non-Null Count    Dtype 
---  ------  --------------    ----- 
 0   Name    1690784 non-null  object
 1   Gender  1690784 non-null  object
 2   Count   1690784 non-null  int64 
 3   Year    1690784 non-null  int64 
dtypes: int64(2), object(2)
memory usage: 51.6+ MB <<<<<< ТУТ
```

```python
# Загрузка данных из файла с оптимизацией поля Gender
data = pd.read_csv('babynames.csv', dtype={'Name': object, 'Gender': 'category','Count': 'int32','Year': 'int16'})
data.info()

<class 'pandas.core.frame.DataFrame'>
RangeIndex: 1690784 entries, 0 to 1690783
Data columns (total 4 columns):
 #   Column  Non-Null Count    Dtype   
---  ------  --------------    -----   
 0   Name    1690784 non-null  object  
 1   Gender  1690784 non-null  category
 2   Count   1690784 non-null  int32   
 3   Year    1690784 non-null  int16   
dtypes: category(1), int16(1), int32(1), object(1)
memory usage: 24.2+ MB <<<<<< ТУТ
```
Итак что мы сделали? По умолчанию Pandas для каждого значения выделяет 64 бита для Int, мы понизили его до 32 и 16 соответственно. И также поменяли тип Gender на category, данный тип очень полезен если ваш столбец имеет низкую гранулярность, тогда можно существенно снизить объем, дело в том что каждое значение будет закодировано числом, а текст будет лежать как словарь отдельно.

Итак нам удалось снизить размер в 2 раза, хороший результат!

#### Парсинг даты при чтении

Корректный разбор даты и тип столбца datetime который снизит размер вашего DF (в моей практике был реальный случай когда в финансовых данных из 1C, господи прости, дата была в очень странном виде, и распарсилась таким образом что месяц стал днем и все перепуталось). В общем указывайте явно, это может быть дольше с точки зрения чтения дата фрейма, но зато очевидно и поможет сократить место на диске.


```python
%time
# Пусть dateutils сам разберется как парсить дату
data = pd.read_csv('babynames_time_test.csv', parse_dates=['Year'])

CPU times: user 3 µs, sys: 0 ns, total: 3 µs
Wall time: 7.15 µs
```

```python
%time
# Говорим ему как разобрать дату
date_parse_func = lambda x: pd.datetime.strptime(x, "%Y-%m-%d %H:%M:%S")
data = pd.read_csv('babynames_time_test.csv', parse_dates=['Year'], date_parser=date_parse_func)

CPU times: user 3 µs, sys: 0 ns, total: 3 µs
Wall time: 6.68 µs
```


#### Индексирование и join

Индексирование полей помогает при мерже, причем весьма сильно. Немного бенчмарков.

```python
%%timeit
data.merge(data, on='Name')

18 s ± 148 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

```python
%%timeit
data.set_index('Name')
data.merge(data, left_index=True, right_index=True)

184 ms ± 3.29 ms per loop (mean ± std. dev. of 7 runs, 10 loops each)
```

Также замечу что индекс поможет вам при использовании функций at, поиск по индексу, это аналог loc.

#### Векторные операции

Старайтесь избегать действий с map/apply/applymap, эти функции так или иначе используют цикл в своих кишках, на уровне ниже, вместо этого стоит воспользоваться встроенным механизмом векторных операций, ну и очевидно что нужно по максимуму отказываться от работы с циклами или itertools, если не хотите выполнять свою работу очень медленно. Немного бенчмарков.

```python
%%timeit
data['Count'].map(lambda x: x - x/2)

497 ms ± 19.4 ms per loop (mean ± std. dev. of 7 runs, 1 loop each)
```

```python
%%timeit
data['Count'] - data['Count']/2

12.4 ms ± 244 µs per loop (mean ± std. dev. of 7 runs, 100 loops each)
```
