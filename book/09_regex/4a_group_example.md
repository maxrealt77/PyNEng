### Разбор вывода команды show ip dhcp snooping с помощью именованных групп

Рассмотрим еще один пример использования именованных групп.
В этом примере, задача в том, чтобы получить из вывода команды show ip dhcp snooping binding
поля: MAC-адрес, IP-адрес, VLAN и интерфейс.


В файле dhcp_snooping.txt находится вывод команды show ip dhcp snooping binding: 
```
MacAddress          IpAddress        Lease(sec)  Type           VLAN  Interface
------------------  ---------------  ----------  -------------  ----  --------------------
00:09:BB:3D:D6:58   10.1.10.2        86250       dhcp-snooping   10    FastEthernet0/1
00:04:A3:3E:5B:69   10.1.5.2         63951       dhcp-snooping   5     FastEthernet0/10
00:05:B3:7E:9B:60   10.1.5.4         63253       dhcp-snooping   5     FastEthernet0/9
00:09:BC:3F:A6:50   10.1.10.6        76260       dhcp-snooping   10    FastEthernet0/3
Total number of bindings: 4
```


Для начала, попробуем разобрать одну строку:
```python
In [1]: line = '00:09:BB:3D:D6:58  10.1.10.2 86250   dhcp-snooping   10  FastEthernet0/1'
```

В регулярном выражении, именованные группы используются для тех частей вывода, которые нужно запомнить:
```python
In [2]: match = re.search('(?P<mac>.+?) +(?P<ip>.*?) +(\d+) +([\w-]+) +(?P<vlan>\d+) +(?P<int>.*$)', line)
```

Комментарии к регулярному выражению:
* ```(?P<mac>.+?) +``` - в группу с именем 'mac' попадают любые символы. Но, так как выражение не жадное, а следом указано, что должны идти пробелы (один или более), получается, что выражение описывает последовательность любых символов, до пробела
* ```(?P<ip>.*?) +``` - тут аналогично, последовательность любых символов до пробела. Имя группы 'ip'
* ```(\d+) +``` - числовая последовательность (одна или более цифр), а затем один или более пробелов
 * сюда попадет значение Lease
* ```([\w-]+) +``` - буквы или ```-```, в количестве одного или более
 * сюда попадает тип соответствия (в данном случае, все они dhcp-snooping)
* ```(?P<vlan>\d+) +``` - именованная группа 'vlan'. Сюда попадают только числовые последовательности, с одним или более символами
* ```(?P<int>.*$)``` - именованная группа 'int'. Сюда попадают любые символы, которые находятся в конце строки (в предыдущем выражении эта группа ограничена пробелом)

Обратите внимание, что для первых двух групп элементов отключена жадность.

Для остальных жадность можно не отключать, так как в них более четко указаны какие именно символы должны быть.

В результате, метод groupdict вернет такой словарь:
```python
In [3]: match.groupdict()
Out[3]: 
{'int': 'FastEthernet0/1',
 'ip': '10.1.10.2',
 'mac': '00:09:BB:3D:D6:58',
 'vlan': '10'}
```

Так как регулярное выражение отработало как нужно, можно создавать скрипт.
В скрипте, перебираются все строки файла dhcp_snooping.txt и на стандартный поток вывода, выводится информация об устройствах.

Файл parse_dhcp_snooping.py:
```python
# -*- coding: utf-8 -*-
import re

regex = re.compile('(?P<mac>.+?) +(?P<ip>.*?) +(\d+) +([\w-]+) +(?P<vlan>\d+) +(?P<int>.*$)')
result = []

with open('dhcp_snooping.txt') as data:
    for line in data:
        if line[0].isdigit():
            result.append(regex.search(line).groupdict())

print "К коммутатору подключено %d устройства" % len(result)

for num, comp in enumerate(result, 1):
    print "Параметры устройства %s:" % num
    for key in comp:
        print "\t%s:\t%s" % (key,comp[key])
```

Результат выполнения:
```
$ python parse_dhcp_snooping.py
К коммутатору подключено 4 устройства
Параметры устройства 1:
	int:	FastEthernet0/1
	ip:	10.1.10.2
	mac:	00:09:BB:3D:D6:58
	vlan:	10
Параметры устройства 2:
	int:	FastEthernet0/10
	ip:	10.1.5.2
	mac:	00:04:A3:3E:5B:69
	vlan:	5
Параметры устройства 3:
	int:	FastEthernet0/9
	ip:	10.1.5.4
	mac:	00:05:B3:7E:9B:60
	vlan:	5
Параметры устройства 4:
	int:	FastEthernet0/3
	ip:	10.1.10.6
	mac:	00:09:BC:3F:A6:50
	vlan:	10
```

