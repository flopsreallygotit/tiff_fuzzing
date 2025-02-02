# Исследование библиотеки [CIMG](https://github.com/GreycLab/CImg) с помощью фаззера [AFL](https://github.com/google/AFL)

## Запуск фаззера

В поддиректории `\example` проекта [CIMG](https://github.com/GreycLab/CImg) находятся программы, использующие методы и функции этой библиотеки. Одна из них была выбрана в качестве `fuzzing target`.

Программа на вход принимает файлы формата `*.tiff`. 


При наличии исходного кода необходимо скомпилировать программу с помощью afl-gcc, так будет добавлени инструментарий фаззера, влияющий на производительность (afl-gcc/g++/clang является как бы заменой gcc/g++/clang).

~~~
[environment vars]/home/ask0later/AFL__/AFL/afl-gcc [флаги]target.c -o target 
~~~

После этого можно запускаем фаззер при помощи:
```
./afl-fuzz -i testcase_dir -o findings_dir /path/to/program [...params...]
```
или
```
./afl-fuzz -i testcase_dir -o findings_dir /path/to/program @@
```

Второй случай нужен для того, если программа принимает аргументы через файл.

В папке с флагом -i находится корпус - корректные входные данные.

Экран состояния фаззера:

![fuzzer](/img_for_README/fuzz.png)

## [Распараллеленный фаззинг](https://github.com/google/AFL/blob/master/docs/parallel_fuzzing.txt)

Также можно запустить несколько процессов для разных или одного target'ов при помощи следующих комманд.

В одном терминале прописываем главный (М) процесс.
```
afl-fuzz -i testcase_dir -o findings_dir -M fuzzer01 [...other stuff...]
```
В других терминалах вторичные:

```
afl-fuzz -i testcase_dir -o sync_dir -S fuzzer02 [...other stuff...]
afl-fuzz -i testcase_dir -o sync_dir -S fuzzer03 [...other stuff...]
```

## [Кастомные мутаторы](https://github.com/AFLplusplus/AFLplusplus/tree/stable/custom_mutators)

Во время выполнения фаззер мутирует входные данные (корпус) для бОльшего покрытия программы. Зная особенности формата файла `*.tiff` можно написать свой ``mutator`` и запустить с ним фаззер.

### Формат .TIFF

Структура файла с форматом .tiff состоит из:

1. Заголовка файла изображения - Image File Header (IFH).

2. Каталога файлов изображений - Image File Directory (IFD).

3. Записи в каталоге -  Directory Entry (DE).

В начале файла находится header, его поля - это тег, версия файла и смещение от начала файла до первого каталога файлов изображений.

Поля IFH - это число, обозначающее общее количество записей в этом каталоге, эти самые записи, смещение от начала файла до следующего IFH.

Одна запись DE содержит в себе тег и тип атрибута, длина данных этого типа, и смещение хранилища значений атрибута.

После header и до первого IFH находится Image data, которая хранит данные изображения.

Мутация состоит в следующем - определить смещение первого IFH, и изменить Image data. 


### Запуск фаззера с кастомным мутатором

Файл `mutator.с` с измененными функциями компилируем в динамический разделяемый объектный файл при помощи команды:
```
gcc -shared -Wall -O3 mutator.c -o mutator.so
```
Далее при запуске фаззера в команду необходимо добавить переменную AFL_CUSTOM_MUTATOR_LIBRARY="/full/patg/to/mutator.so". Если необходимо использовать только свой кастомный фаззер, то ещё определим AFL_CUSTOM_MUTATOR_ONLY=1.


Экран состояния с использованием кастомного мутатора и встроеного:

![cust](/img_for_README/cust.png)

# Результаты
Заранее в команду была определенна переменная AFL_MAP_SIZE=65536 для того, чтобы размера bitmap был одинаковый во всех случаях.

Если сравнивать результаты тестирования фаззера с импользованием дефолтного встроеного мутатора и его же с кастомным мутатором, то видно, что кол-во `crashes` увеличилось. Эффективность тестирования возросла, это может использоваться в работе.

## Анализ покрытия

Утилитой библиотеки AFL можем получаить наглядный график зависимости покрытия от времени.

~~~
afl-plot findings/default/ plot_info/
~~~


### Только дефолтный мутатор
![def_coverage](/img_for_README/def_coverage.png)

### Дефолтный и кастомный мутатор
![def_and_cust_coverage](/img_for_README/def_and_cust_coverage.png)

### Только кастомный мутатор
![cust_coverage](/img_for_README/cust_coverage.png)

Видно, что кастомная мутация может оказатся полезной, так как она вносит свой вклад в расширение покрытия программы.


## Литература
1. AFL: https://github.com/google/AFL.git.
2. AFLplusplus: https://github.com/AFLplusplus/AFLplusplus.git.
3. greybox-фаззинг: https://habr.com/ru/company/bizone/blog/570312/ и https://habr.com/ru/company/bizone/blog/570534/.
4. Detailed TIFF image file format: https://programmersought.com/article/25991320372/.