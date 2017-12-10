# Алгоритм кластеризации Маркова
* [Теоретическая часть](../master/Theoretical_Part.md)
* [Использование программы](#1)
* [Логика программы](#2)
* [Тесты](#3)
* [Оценка сложности](#4)

<a name="1"></a>
## Использование программы

Для запуска необходим python версии не менее 2.7. Команда:
```
python markov_clustering.py <input_file.csv> <output>
```
Обязательным параметром для запуска является ```<input_file.csv>```, прочий необязателен. По умолчанию файл вывода будет создан в корневой папке проекта с именем ```output```.
Также в консоль выводится краткая информация о выполненной программе.

Пример использования:
![example](../master/public/static/usage_example.png)

```input_file.csv``` - это граф, заданный матрицей смежности взвешенного графа в виде таблицы формата \*.csv.

Файл выхода программы, по умолчанию ```output```, содержит полученные кластеры графа, выведенные в виде ассоциативного массива: индекс кластера -> список содержащихся в нем вершин по индексам, взятым из матрицы смежности. (см. [пример](../master/tests/outputs/2) из двух кластеров)

Точность создания кластеров алгоритмом регулируется задаваемыми константами, которые можно найти в файле [config.py](../master/public/config.py). Среди них:

* ```inflate_factor``` - фактор накачивания - константа, использующаяся в операторе накачивания алгоритма как степень r (см. [формула операции накачивания](../master/Theoretical_Part.md#3)). При значениях ```inflate_factor > 1``` накачивание преобразует элементы матрицы, являющиеся вероятностями перехода из одного состояния в другое для конкретной вершины (соответующей столбцу матрицы), в элементы более вероятных прогулок. Чем выше параметр, тем больше находится кластеров.
* ```max_loop``` - максимальное количество итераций, по умолчанию стоит 100, для более высокой точности можно выставить больше итераций, но оптимально иметь 100. В действительности в большинстве случаев количество итераций будет меньше, так как будет достигнуто устойчивое состояние матрицы (разница двух подряд идущих матриц при заданной точности ```accuracy``` будет равна 0).
* ```loop_value``` - константа, задающая значения диагональных элементов матрицы, говоря понятием графа: вес петли в графе. Если верить автору алгоритма, оптимально иметь значение, равное 1. Увеличение приведет к более высокой гранулярности графа (степени разбиения).
* ```accuracy``` - точность - константа задает порядок, которым можно пренебречь при расчете вероятности. Используется в подсчете разницы двух подряд идущих матриц. Чем выше точность, тем больше итераций алгоритма, и наоборот. 

<a name="2"></a>
## Логика программы
Точка входа программы - функция ```main``` файла ```markov_clustering.py```.
Она считывает с входного потока имя входного и выходного файлов и передает их в функцию ```mcl_manager```, попутно обрабатывая исключения ввода имен файлов и исключения работы алгоритма. 

Модуль ```mcl_manager.py``` состоит из функций: 
* ```reader(filename)``` - принимает на вход файл формата \*.csv и преобразует его в питоновский лист листов, по сути в матрицу. 
* ```mcl_manager(input_file, output_file)``` - функция-менеджер алгоритма: считывает матрицу из файла (функция ```reader```), запускает алгоритм кластеризации на полученной матрице (функция ```mcl```) и записывает результат в файл (функция ```clusters_to_output```). Также выводит информацию о ходе программы.

Далее программа переходит в модуль ```mcl.py```. В ней реализованы алгоритмы, специфичные для кластеризации Маркова. Это подразумевает, что в этом же файле реализован и сам алгоритм кластеризации. Этот модуль также активно использует операции над матрицами, реализованные в модуле ```matrix.py``` Соотвественно состоит из функций:
* ```expansion(matrix)``` - реализует оператор распространения, необходимый для вычисления алгоритма (см. [распространение](../master/Theoretical_Part.md#3))
* ```inflation(matrix, inflate_factor)``` - реализует оператор накачивания (см. [накачивание](../master/Theoretical_Part.md#3))
* ```mcl(matrix, inflate_factor, max_loop, loop_value, accuracy) ``` - главная функция, реализующая алгоритм. По сути алгоритм представляет собой следующие шаги:
1. Добавить в matrix петли с весом ```LOOP_VALUE```.
2. Нормализовать matrix: привести к стохастическому виду.
3. Накачивание с ```INFLATE_FACTOR```
4. Распространение.
5. Посмотреть изменения текующей матрицы с предыдущей с заданной точностью ```ACCURACY```, если 0 или число итераций превысило ```MAX_LOOP``` - завершить алгоритм, иначе - перейти к шагу 3.
6. Получить кластеры из получившейся устойчивой матрицы.

Для работы с получением и выводом кластеров реализован модуль ```clusters.py```.
* ```get_clusters(matrix)``` - на вход матрица в устойчивом состоянии (см. [пример](../master/tests/examples/steady_state_matrix.csv) такой матрицы для графа из файла [2.csv](../master/tests/examples/2.csv)), из нее получает ассоциативный массив кластеров.
* ```clusters_to_output(clusters, output_file)``` - записывает ассоциативный массив ```clusters``` кластеров в файл ```output_file```.

В функции основного файла ```mcl.py``` алгоритма активно используются операции над матрицами, представленными в виде листа листов, реализованные в модуле ```matrix.py```.

Среди функций этого модуля наибольший интерес представляет функция перемножения матриц. Так как стандартное перемножение на практике с графом с 300 вершинами за одну итерацию дает время выполнения в 7 секунд, было решено оптимизировать алгоритмом Штрассена ```strassen_multiplication(a, b, c)```. (теперь занимает 5 секунд) На графах с менее чем 64 вершинами используется стандартное умножение ```multiply(a, b)```.

<a name="3"></a>
## Тесты
Реализовано два блока тестов: тесты модуля matrix и тесты корректной работы программы. 

Запуск тестов модуля:
```
python -m unittest -v tests.matrix_test
```

Запуск тестов программы:
```
python -m unittest -v tests.mcl_test
```

На вход тестов программы подаются файлы из директории ```tests/examples/*.csv```.
Тестовые файлы оттуда проверяют базовые случаи для графов с количеством вершин не более 7, так как для более крупных графов, начиная уже с 10 вершин, нельзя точно выявить, на какие кластеры разобьется граф и проверку тестов не получится автоматизировать. 

Подробное описание каждого теста:
1. ```1.csv``` - единичная матрица, что подразумевает под собой полный граф. Очевидно, что в таком случае кластер будет один и состоять из всех вершин.
2. ```2.csv``` - граф, в котором интуитивно понятно, что образются два кластера: соединены между собой 1, 2, 3 вершины и отдельно 4, 5, 6, 7, а между ними соединены 3 и 4.
3. ```3.csv``` - диагональная матрица, что означает, что это граф, в котором вершины между собой не связаны, но у всех есть петли. Соотвественно кластеров должно быть столько же, сколько и вершин.
4. ```4.csv``` - граф, в котором связаны только первая и вторая вершины и отсутствуют петли. Очевидно, что вершины 1, 2 в одном кластере, остальные - разделены по своим собственным.
5. ```5.csv``` - граф, практически индентичный ```2.csv```. Ожидается, что в итоге получится 2 кластера (как и в тесте 2, но выставлены случайные петли у вершин). 

Тесты модуля ```matrix.py``` проверяют правильность выполнения содержащихся в нем функций на простых примерах. 

<a name="4"></a>
## Оценка сложности
### Вычислительная сложность алгоритма
Условимся за ![n](https://latex.codecogs.com/gif.latex?n) считать количество вершин графа. 

Сперва необходимо граф прочитать из файла и записать в лист листов. Так как граф хранится в виде матрицы смежности, то сложность выйдет ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29). 

Затем работает сам алгоритм. В моем случае, в конфигурации выставлено, что максимальное количество итераций равно 100. Автор алгоритма утверждает, что количество подобных итераций будет равно ![cmplx3](https://latex.codecogs.com/gif.latex?O%28k%5E%7B2%7D%29), где ![k](https://latex.codecogs.com/gif.latex?k) - максимальная степень вершины графа. 

В алгоритме используются операции накачивания и расширения. 

Накачивание - это нормализация матрицы (то есть по сути просто пройтись по матрице) и возвести в степень по правилу Адамара (тоже один проход по матрице). Итого ![inflat](https://latex.codecogs.com/gif.latex?O%282n%5E%7B2%7D%29).

Расширение - это перемножение матрицы самой на себя. Обычное перемножение матрицы самой на себя по сложности ![kub](https://latex.codecogs.com/gif.latex?O%28n%5E%7B3%7D%29). 

Однако в программе используется алгоритм Штрассена, согласно которому умножение матриц проводится рекурсивно для матриц размера ![n/2](https://latex.codecogs.com/gif.latex?%5Cfrac%7Bn%7D%7B2%7D), над которыми проводится 7 операций перемножения из введенных дополнительных матриц: 

![strassen_m](../master/public/static/strassen_matrixes.png).

Из которых любую матрицу можно разбить на четыре соотношением: 

![strassen_values](../master/public/static/strassen_values.png)

Получается реккурентное соотношение вида: ![reccurent](https://latex.codecogs.com/svg.latex?%5Cdpi%7B120%7D%20O%28n%29%20%3D%207O%28%5Cfrac%7Bn%7D%7B2%7D%29%20&plus;%20cn%5E%7B2%7D), из которого сложность будет равна ![42](https://latex.codecogs.com/gif.latex?O%28n%5E%7Blog_%7B2%7D7%7D%29). 

Остаются функции ```converged```, ```rounding``` и ```get_clusters```, в которых опять же проходишься по матрице за ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29).

Вывод кластеров в файл занимает ![O(n)](https://latex.codecogs.com/svg.latex?%5Cdpi%7B120%7D%20O%28n%29) и плюс количество кластеров.

Итого вычислительная сложность: 

[42](https://latex.codecogs.com/gif.latex?O%28n%5E%7Blog_%7B2%7D7%7D%29)

### Сложность по памяти

Хранение матрицы смежности, как в файле csv, так и в листе листов ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29). 

Функции работы с матриксами, использующие дополнительную память, выделяют не больше чем ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29) дополнительной памяти (так как просто создают новую матрицу nxn).

Функция ```get_clusters``` смотрит диагональные элементы устойчивой матрицы и если он не равен 0, записывает эту колонку в свой лист листов. Таким образом, по памяти она тратит также ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29).

Итого ![square](https://latex.codecogs.com/gif.latex?O%28n%5E%7B2%7D%29).
