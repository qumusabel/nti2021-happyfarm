#  Инженерная задача по Информационной безопасности НТИ 2021| Весёлая Ферма

## Задача
 На финальном мероприятии нам дали задание, суть которого была в том, что нам нужно было защитить подвергшийся атаке телецентр, выявить наибольшее количество уязвимых мест, разработать способы устранения атак, а также найти и обезвредить вредонос. 
 
## Вход во внутреннюю сеть

Сначала было произведено сканирование портов, найдены следующие:
```
| Порт | Прот. | Описание | |——-|——-|——————————————–| | 81 | HTTP | Хостит обфусцированный файл run.py | | 5000 | HTTP | Веб-приложение, сайт провайдера (?) NetGen | | 8037 |      UDP? | Открыт, но не отвечает | | 51515 | TCP | Какой-то сервис, просит пароль |
```
На сервисе NetGen есть страницы /register, /login, /logout Регистрируем пользователя, доступна функция создания тикетов. В параметре POST title обнаружена SSTI (предположительно, Jinja2) Некоторые ключевые слова фильтруются, причем необязательно в составе пейлоада (например, config)

Далее нами было получено SSTI RCE в сервисе на :5000

![kts](https://user-images.githubusercontent.com/67109334/110931744-ab965f80-833b-11eb-975c-01f315d1d807.png)

```
server.py:224 status_route() Possible RCE, weak filter
```
#### Решение:
>Use whitelist instead of blacklist, only allow options to uptime
```
└─$ diff ../app/server.py server.py                                                           
224c224,226
&lt;     bad_words = ['whoami', 'id', 'python', 'php', 'bash'] #hacker shouldn't pass
---
&gt;     # Only allow options to `uptime`: 
&gt;     good_words = ['-p', '--pretty', '-h', '--help', '-s', '--since', '-V', '--version']
&gt; 
227,228c229,230
&lt;         for i in bad_words:
&lt;             if i in req:
---
&gt;         for token in req.split():
&gt;             if token not in good_words:
229a232
```

* Создать пользователя " || "admin : passwd
* Зайти за " || "admin : admin <— слабая политика паролей!
* Go to /dialog?u=? -> dialog leakage
* Change " || "admin‘s passwd, now admin‘s password is changed, too
* Proposed patch: use proper DB access methods w/escaping

In future: enforce password policies

### Взлом run.py на хосте :81

#### Сначала деобфусцируем скрипт:
>
- Упростим имена, переименуем все в более понятный вид
- Упростим hex-строки, получается encode, unhexlify и прочее
- Избавимся от непонятных вызовов locals(), getattr():
  - getattr(binascii, 'unhexlify')(...) 
  - binascii.unhexlify(...)
  - locals['somefunc'](..., ...) 
  - somefunc(..., ...)
- Последний шаг – избавимся от списков чисел. Можно заметить, что они используюстя только как аругмент к somefunc, при этом функция похожа на шифр Виженера. Для того, чтобы получить оригинальные строки, просто вызовем функцию с нужными аргументами. Выясняется, что большой список в начале файла – это зашифрованный приватный ключ RSA.

#### Алгоритм:

* Загрузить ключ RSA
* Получить hex-строку от пользователя
* Расшифровать строку
* Проверить соответствие с заданной строкой, если совпадают – вывести ~~флаг~~ важную информацию

#### Взлом:

>Приватный ключ дан почти в открытом виде, зашифруем с помощью него требуемую строку, затем подключимся к серверу на порту :51515, нас попросят ввести hex, отправим шифротекст, в ответ прилетит зашифрованная информация. С помощью скрипта и коюча ее расшифруем, получим сообщение с данными для входа во внутреннюю сеть

![kts](https://user-images.githubusercontent.com/67109334/110935184-23668900-8340-11eb-9bba-bb4c503712b1.png)
