# 5symbol

Запрос помогает найти правильное слово в игре [5 букв](https://5bukv.tinkoff.ru/). 

## Правила игры «5 букв»

Каждый день мы загадываем существительное из пяти букв в единственном числе.
Отгадайте слово и выиграйте призы.

На разгадку у вас шесть попыток. После каждой попытки цвет букв меняется. Вот что означают цвета.

```plain
(' м-ы-с л ь')
```

Буквы «М», «Л», «Ь» белые — они есть в загаданном слове, но стоят в других местах. Буквы «Ы», «С» серые — их в слове нет.

```plain
('+ф+и-к-у-с')
```

Буквы «Ф», «И» желтые — они есть в загаданном слове и стоят на нужных местах.

```plain
('+ф+и+л+ь+м')
```

Когда вы угадаете слово, все буквы окрасятся в желтый.

В загаданном слове могут встретиться повторяющиеся буквы, например буква «Т» в слове «Театр». Буквы «Ё» в игре нет, вместо нее используйте «Е» — как в кроссворде.

Начиная с 2 ноября, за хорошую игру вы получаете призы. Первый приз сможете выиграть за первое отгаданное слово. Далее — за каждые пять побед

## Старт игры

В самом начале выбираем одно из слов, которое дает шанс не менее 30% что хотя бы одна буква будет угадана.

```plain
крона
сотка
комар
кобра
катер
тропа
вокал
лодка
```

## Следующее слово

Идем на [онлайн бд](https://sqliteonline.com/) и загружаем БД [Rus.db](https://github.com/VASilaev/5symbol/blob/main/rus.db).

Далее выполняем запрос дополняя значения. Перед каждой буквой ставим "-" если такой буквы нет в слове, "+" Если угадали букву и она на своем месте, " "(пробел) если буква не на своем месте. 

Обратите внимание, что если в слове две одинаковые буквы, то только первая помечается как нет на месте. все последующие будут помечены как нет в слове.

```sql
with qPass as (select * from (values 		 
  ('-к-р-о-н а'),('-б а л е-т'),('-и-д е а+л') --Попытки
)),
qMatch as (
select 
 value,
 case when p1 like '^%' and not ml like '%' || substr(value, 1,1) || '%' then substr(value, 1,1) else '' end ||
 case when p2 like '^%' and not ml like '%' || substr(value, 2,1) || '%' then substr(value, 2,1) else '' end ||
 case when p3 like '^%' and not ml like '%' || substr(value, 3,1) || '%' then substr(value, 3,1) else '' end ||
 case when p4 like '^%' and not ml like '%' || substr(value, 4,1) || '%' then substr(value, 4,1) else '' end || 
 case when p5 like '^%' and not ml like '%' || substr(value, 5,1) || '%' then substr(value, 5,1) else '' end NewLetter,
 m
from 
 (
	select 
	  replace(coalesce(max(s) filter (where o = '+' and ns=0),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=0)))),',','') p1,
	  replace(coalesce(max(s) filter (where o = '+' and ns=1),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=1)))),',','') p2,
	  replace(coalesce(max(s) filter (where o = '+' and ns=2),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=2)))),',','') p3,
	  replace(coalesce(max(s) filter (where o = '+' and ns=3),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=3)))),',','') p4,
	  replace(coalesce(max(s) filter (where o = '+' and ns=4),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=4)))),',','') p5,
	  replace(group_concat(distinct s) filter (where o=' '),',','') m, 
	  replace(group_concat(distinct s) filter (where o in (' ', '+')),',','') ml, 
	  count(distinct s) filter (where o=' ') c
	from
	 (
		select  
		 k.column1 ns, substr(t.column1, k.column1 * 2 + 2, 1) s, substr(t.column1, k.column1 * 2 + 1, 1) o
		from (values (0),(1),(2),(3),(4)) k, qPass as t
		order by s
	 ) t
	where s <> ''
 ) t,
 word 
where 
 ((p1 like '%'||substr(value,1,1)||'%') + case when p1 like'^%'then 1 else 0 end)&1=1 and 
 ((p2 like '%'||substr(value,2,1)||'%') + case when p2 like'^%'then 1 else 0 end)&1=1 and
 ((p3 like '%'||substr(value,3,1)||'%') + case when p3 like'^%'then 1 else 0 end)&1=1 and
 ((p4 like '%'||substr(value,4,1)||'%') + case when p4 like'^%'then 1 else 0 end)&1=1 and
 ((p5 like '%'||substr(value,5,1)||'%') + case when p5 like'^%'then 1 else 0 end)&1=1 and
 coalesce((select count(distinct nullif(instr(m,substr(value,column1,1)),0)) from (values (1),(2),(3),(4),(5))),0) = coalesce(c,0)
),

qWeight as (
select 
 s, cast(c as double) / sum(c) over () p 
from 
 (
	select s, count(*) c from (
	select substr(NewLetter,1,1) s from qMatch
	union all
	select substr(NewLetter,2,1) from qMatch
	union all
	select substr(NewLetter,3,1) from qMatch
	union all
	select substr(NewLetter,4,1) from qMatch
	union all
	select substr(NewLetter,5,1) from qMatch
	) t 
	where s <> ''
	group by s
 ) t
)

select 
  value,
  (select sum(p)from (select distinct substr(value,column1,1)s,p from(values(1),(2),(3),(4),(5)),qWeight where qWeight.s=substr(value,column1,1)) where (m is null or not m like '%'||s||'%')) as nRate
from
  qMatch  
order by 2 desc
```
