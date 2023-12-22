# 5symbol

В самом начале выбираем одно из слов, которое дает шанс не менее 30% что хотя бы одна буква будет угадана

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

далее выполняем запрос дополняя значения. Перед каждой буквой ставим "-" если такой буквы нет в слове, "+" Если угадали букву и она на своем месте, " "(пробел) если буква не на своем месте 


```sql
with qWeight as (select 
 s, cast(c as double) / sum(c) over () p
from 
 (
	select s, count(*) c from (
	select substr(value,1,1) s from word
	union all
	select substr(value,2,1) from word
	union all
	select substr(value,3,1) from word
	union all
	select substr(value,4,1) from word
	union all
	select substr(value,5,1) from word
	) t 
	group by s
 ) t)
select 
 value,
 (select sum(p)from (select distinct substr(value,column1,1)s,p from(values(1),(2),(3),(4),(5)),qWeight where qWeight.s=substr(value,column1,1)) where (m is null or not m like '%'||s||'%')) 
from 
 (
	select 
	  replace(coalesce(max(s) filter (where o = '+' and ns=0),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=0)))),',','') p1,
	  replace(coalesce(max(s) filter (where o = '+' and ns=1),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=1)))),',','') p2,
	  replace(coalesce(max(s) filter (where o = '+' and ns=2),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=2)))),',','') p3,
	  replace(coalesce(max(s) filter (where o = '+' and ns=3),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=3)))),',','') p4,
	  replace(coalesce(max(s) filter (where o = '+' and ns=4),'^'||group_concat(distinct s) filter (where (o='-'or(o=' 'and ns=4)))),',','') p5,
	  replace(group_concat(distinct s) filter (where o=' '),',','') m, 
	  count(distinct s) filter (where o=' ') c
	from
	 (
		select  
		 k.column1 ns, substr(t.column1, k.column1 * 2 + 2, 1) s, substr(t.column1, k.column1 * 2 + 1, 1) o
		from (values (0),(1),(2),(3),(4)) k, (values 		 
		 ('-р-а-н-е-ц'),('-и с-т о-к') --Попытки
		) as t
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
order by 2 desc
```
