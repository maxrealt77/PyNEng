## TextFSM CLI Table

Благодаря TextFSM, можно обрабатывать вывод команд и получать структурированный результат. 
Но, всё ещё надо вручную прописывать каким шаблоном обрабатывать команды show, каждый раз, когда используется TextFSM.

Было бы намного удобней иметь какое-то соответствие между командой и шаблоном.
Чтобы можно было написать общий скрипт, который выполняет подключения к устройствам, отправляет команды, сам выбирает шаблон и парсит вывод в соответствии с шаблоном.

В TextFSM есть такая возможность.

Для того, чтобы ей можно было воспользоваться, надо создать файл в котором описаны соответствия между командами и шаблонами. В TextFSM он называется index.

Этот файл должен находиться в каталоге с шаблонами и должен иметь такой формат:
* первая строка - названия колонок
* каждая следующая строка - это соответствие шаблона команде
* обязательные колонки, местоположение которых фиксировано (должны быть обязательно первой и последней, соответственно):
 * первая колонка - имена шаблонов
 * последняя колонка - соответствующая команда
   * в этой колонке используется специальный формат, чтобы описать то, что команда может быть написана не полностью
* остальные колонки могут быть любыми
 * например, в примере ниже будут колонки Hostname, Vendor. Они позволяют уточнить информацию об устройстве, чтобы определить какой шаблон использовать.
   * например, команда show version может быть у оборудования Cisco и HP. Соответственно, только команды недостаточно, чтобы определить какой шаблон использовать. В таком случае, можно передать информацию о том, какой тип оборудования используется, вместе с командой, и тогда получится определить правильный шаблон.
* во всех столбцах, кроме первого, поддерживаются регулярные выражения
 * в командах, внутри ```[[]]``` регулярные выражения не поддерживаются

Пример файла index:
```
Template, Hostname, Vendor, Command
sh_cdp_n_det.template, .*, Cisco, sh[[ow]] cdp ne[[ighbors]] de[[tail]]
sh_clock.template, .*, Cisco, sh[[ow]] clo[[ck]]
sh_ip_int_br.template, .*, Cisco, sh[[ow]] ip int[[erface]] br[[ief]]
sh_ip_route_ospf.template, .*, Cisco, sh[[ow]] ip rou[[te]] o[[spf]]
```

Обратите внимание на то, как записаны команды:
* ```sh[[ow]] ip int[[erface]] br[[ief]]```
 * эта запись будет преобразована в выражение sh((ow)?)? ip int((erface)?)? br((ief)?)?
 * это значит, что TextFSM сможет определить какой шаблон использовать, даже если команда набрана не полностью
 * например, такие варианты команды сработают:
     * sh ip int br
     * show ip inter bri


### Как использовать CLI table

Посмотрим как пользоваться классом clitable и файлом index.

В каталоге templates такие шаблоны и файл index:
```
sh_cdp_n_det.template
sh_clock.template
sh_ip_int_br.template
sh_ip_route_ospf.template
index
```

Сначала попробуем поработать с CLI Table в ipython, чтобы посмотреть какие возможности есть у этого класса, а затем посмотрим на финальный скрипт.

> Если при работе с clitable возникнет ошибка, что нет модуля terminal, посмотрите в каталоге, в котором находится установленный TextFSM (в зависимости от ОС), файл terminal.py. Если его нет, просто создайте его вручную и скопируйте содержимое этого файла из [репозитория TextFSM](https://github.com/google/textfsm/blob/master/terminal.py).  Определить, где находится модуль, можно таким образом: ```print textfsm.__file__```

Для начала, импортируем класс clitable:
```python
In [1]: import textfsm.clitable as clitable
```

Проверять работу clitable будем на последнем примере из прошлого раздела - выводе команды show ip route ospf. Считываем вывод, который хранится в файле output/sh_ip_route_ospf.txt, в строку:
```python
In [2]: output_sh_ip_route_ospf = open('output/sh_ip_route_ospf.txt').read()
```

Сначала надо инициализировать класс, передав ему имя файла, в котором хранится соответствие между шаблонами и командами, и указать имя каталога, в котором хранятся шаблоны:
```python
In [3]: cli_table = clitable.CliTable('index', 'templates')
```

Надо указать какая команда передается и указать дополнительные атрибуты, которые помогут идентифицировать шаблон.
Для этого, нужно создать словарь, в котором ключи - имена столбцов, которые определены в файле index.
В данном случае, не обязательно указывать название вендора, так как команде sh ip route ospf соответствет только один шаблон. 
```python
In [4]: attributes = {'Command': 'show ip route ospf' , 'Vendor': 'Cisco'}
```

Методу ParseCmd надо передать вывод команды и словарь с параметрами:
```python
In [5]: cli_table.ParseCmd(output_sh_ip_route_ospf, attributes)
```

В результате, в объекте cli_table, получаем обработанный вывод команды sh ip route ospf.

Методы cli_table (чтобы посмотреть все методы, надо вызвать dir(cli_table)):
```python
In [6]: cli_table.
cli_table.AddColumn        cli_table.NewRow           cli_table.index            cli_table.size
cli_table.AddKeys          cli_table.ParseCmd         cli_table.index_file       cli_table.sort
cli_table.Append           cli_table.ReadIndex        cli_table.next             cli_table.superkey
cli_table.CsvToTable       cli_table.Remove           cli_table.raw              cli_table.synchronised
cli_table.FormattedTable   cli_table.Reset            cli_table.row              cli_table.table
cli_table.INDEX            cli_table.RowWith          cli_table.row_class        cli_table.template_dir
cli_table.KeyValue         cli_table.extend           cli_table.row_index
cli_table.LabelValueTable  cli_table.header           cli_table.separator
```

Например, если вызвать ```print cli_table```, получим такой вывод:
```python
In [7]: print cli_table
Network, Mask, Distance, Metric, NextHop
10.0.24.0, /24, 110, 20, ['10.0.12.2']
10.0.34.0, /24, 110, 20, ['10.0.13.3']
10.2.2.2, /32, 110, 11, ['10.0.12.2']
10.3.3.3, /32, 110, 11, ['10.0.13.3']
10.4.4.4, /32, 110, 21, ['10.0.13.3', '10.0.12.2', '10.0.14.4']
10.5.35.0, /24, 110, 20, ['10.0.13.3']
```

Метод FormattedTable позволяет получить вывод в виде таблицы:
```python
In [8]: print cli_table.FormattedTable()
 Network    Mask  Distance  Metric  NextHop
====================================================================
 10.0.24.0  /24   110       20      10.0.12.2
 10.0.34.0  /24   110       20      10.0.13.3
 10.2.2.2   /32   110       11      10.0.12.2
 10.3.3.3   /32   110       11      10.0.13.3
 10.4.4.4   /32   110       21      10.0.13.3, 10.0.12.2, 10.0.14.4
 10.5.35.0  /24   110       20      10.0.13.3
```

Такой вывод может пригодиться для отображения информации.

Чтобы получить из объекта cli_table структурированный вывод, например, список списков, надо обратиться к объекту таким образом:
```python
In [9]: data_rows = []

In [10]: for row in cli_table:
   ....:     current_row = []
   ....:     for value in row:
   ....:         current_row.append(value)
   ....:     data_rows.append(current_row)
   ....:

In [11]: data_rows
Out[11]:
[['10.0.24.0', '/24', '110', '20', ['10.0.12.2']],
 ['10.0.34.0', '/24', '110', '20', ['10.0.13.3']],
 ['10.2.2.2', '/32', '110', '11', ['10.0.12.2']],
 ['10.3.3.3', '/32', '110', '11', ['10.0.13.3']],
 ['10.4.4.4', '/32', '110', '21', ['10.0.13.3', '10.0.12.2', '10.0.14.4']],
 ['10.5.35.0', '/24', '110', '20', ['10.0.13.3']]]

```

Отдельно можно получить названия столбцов:
```python
In [12]: cli_table.header.viewvalues()
Out[12]: dict_values([])

In [13]: header = []

In [13]: for name in cli_table.header:
   ....:     header.append(name)
   ....:

In [14]: header
Out[14]: ['Network', 'Mask', 'Distance', 'Metric', 'NextHop']
```

Теперь вывод аналогичен тому, который был получен в прошлом разделе.


Соберем всё в один скрипт (файл textfsm_clitable.py):
```python
import textfsm.clitable as clitable

output_sh_ip_route_ospf = open('output/sh_ip_route_ospf.txt').read()
cli_table = clitable.CliTable('index', 'templates')
attributes = {'Command': 'show ip route ospf' , 'Vendor': 'Cisco'}
cli_table.ParseCmd(output_sh_ip_route_ospf, attributes)

print "CLI Table output:\n", cli_table
print "Formatted Table:\n", cli_table.FormattedTable()

data_rows = []

for row in cli_table:
    current_row = []
    for value in row:
        current_row.append(value)
    data_rows.append(current_row)

header = []
for name in cli_table.header:
    header.append(name)

print header
for row in data_rows:
    print row
```

> В упражнениях к этому разделу будет задание, в котором надо объединить описанную процедуру в функцию. А также вариант с получением списка словарей.

Вывод будет таким:
```
$ python textfsm_clitable.py
CLI Table output:
Network, Mask, Distance, Metric, NextHop
10.0.24.0, /24, 110, 20, ['10.0.12.2']
10.0.34.0, /24, 110, 20, ['10.0.13.3']
10.2.2.2, /32, 110, 11, ['10.0.12.2']
10.3.3.3, /32, 110, 11, ['10.0.13.3']
10.4.4.4, /32, 110, 21, ['10.0.13.3', '10.0.12.2', '10.0.14.4']
10.5.35.0, /24, 110, 20, ['10.0.13.3']

Formatted Table:
 Network    Mask  Distance  Metric  NextHop
====================================================================
 10.0.24.0  /24   110       20      10.0.12.2
 10.0.34.0  /24   110       20      10.0.13.3
 10.2.2.2   /32   110       11      10.0.12.2
 10.3.3.3   /32   110       11      10.0.13.3
 10.4.4.4   /32   110       21      10.0.13.3, 10.0.12.2, 10.0.14.4
 10.5.35.0  /24   110       20      10.0.13.3

['Network', 'Mask', 'Distance', 'Metric', 'NextHop']
['10.0.24.0', '/24', '110', '20', ['10.0.12.2']]
['10.0.34.0', '/24', '110', '20', ['10.0.13.3']]
['10.2.2.2', '/32', '110', '11', ['10.0.12.2']]
['10.3.3.3', '/32', '110', '11', ['10.0.13.3']]
['10.4.4.4', '/32', '110', '21', ['10.0.13.3', '10.0.12.2', '10.0.14.4']]
['10.5.35.0', '/24', '110', '20', ['10.0.13.3']]
```

Теперь, с помощью TextFSM, можно не только получать структурированный вывод, но и автоматически определять какой шаблон использовать, по команде и опциональным аргументам.


