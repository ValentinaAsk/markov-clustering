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

<a name="4"></a>
## Оценка сложности


