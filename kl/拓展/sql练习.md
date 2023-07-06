[toc]

# 连续登录问题

## 使用`join`

1. 通过`inner join` 来关联同一张表，找出第二张表中的日期在第一张表中的（`n天`之前，当天）；
2. 筛出基于当天的过去第`n天`的所有数据；

```sql
select t2.id
from (
	select t1.id, t1.dt
    from (
        select u1.dt, u1.id, [u2.dt /*之前日期的登录*/]
        from userTable/*基于dt,id去重*/ u1 inner join userTable u2 on u1.id = u2.id
        where u2.dt between date_sub(u1.dt, 2/*连续登录天数减一*/) and u1.dt
    	) t1
    group by t.id, t.dt having count([u2.dt|1]) = 3 /*连续登录天数*/
	) t2
group by t2.id
```

## 使用`row_number`

1. 对`id, dt`已经去重的数据，使用`row_number`的窗口函数；
2. 窗口函数基于`id`分区，`dt`排序，能找到每个用户的登录次数；
3. 如果对基于当前和登录次数做相减，那么可以找到最早登录的日期；
4. 通过对最早登录日期的聚合统计，即可以得到连续登录的用户。

```
select t2.id
from (
	select id, date_sub(dt, rn) dt_line
	from (
		select id, dt, 
			row_number() over(partition by id order by dt) rn
		from userTable
		) t1
	group by id, date_sub(dt,rn) having count(1) >= 3 /*连续登录天数*/
	) t2
group by t2.id
```

## 使用`lag`

1. 使用`lag`基于`id`分区，`dt`排序，得到最初的日期；
2. 如果当前日期和最初的日期相差**`连续登录 - 1`**，那么则说明这个用户是连续登录用。

```
select t2.id
from (
	select id, if(date_diff(dt, lag_dt) == 2 /*连续登录 - 1*/, 1, 0) flag
	from (
		select id, dt, 
			lag(dt, 2/*连续登录 - 1*/, '0000-00-00') over(partition by id, order by dt) lag_dt/*两天前*/
		from userTable;
		) t1
	) t2
where t2.flag = 1
group by t2.id
```

## 使用`lead`

1. 使用`lead`基于`id`分区，`dt`排序，得到最初的日期；
2. 如果当前日期和最初的日期相差**`连续登录 - 1`**，那么则说明这个用户是连续登录用。

```
select t2.id
from (
	select id, if(date_diff(lead_dt, dt) == 2 /*连续登录 - 1*/, 1, 0) flag
	from (
		select id, dt, 
			lead(dt, 2/*连续登录-1*/, '0000-00-00') over(partition by id, order by dt desc) lead_dt/*两天后*/
		from userTable;
		) t1
	) t2
where t2.flag = 1
group by t2.id
```

# 找出连续3天及以上减少碳排放量在100以上的用户

```
id		dt				lowcarbon
1001	2021-12-12		123
1002	2021-12-12		45
1001	2021-12-13		43
1001	2021-12-13		45
1001	2021-12-13		23
1002	2021-12-14		45
1001	2021-12-14		230
1002	2021-12-15		45
1001	2021-12-15		23
```

```
select id, flag, count(*) ct/*按照用户及Flag分组,求每个组有多少条数据,并找出大于等于3条的数据*/
from (
	select id, dt, lowcarbon, date_sub(dt, rk) flag /*每行数据中的日期减去Rank值*/
	from (
	   /*按照用户分组,同时按照时间排序,计算每条数据的Rank值*/
		select id, dt, lowcarbon,
			rank() over(partition by id order by dt) rk
		from (
			/*每日碳排放的数量*/
			select id, dt, sum(lowcarbon) lowcarbon
			from test1
			group by id, dt having lowcarbon > 100
			) t1
		) t2
	) t3
group by id, flag having ct >= 3/*按照用户及Flag分组,求每个组有多少条数据,并找出大于等于3条的数据*/
```

# 分组问题

```
id		ts            group
1001	17523641234
1001	17523641256
1002	17523641278
1001	17523641334
1002	17523641434
1001	17523641534
1001	17523641544
1002	17523641634
1001	17523641638
1001	17523641654

==>

时间间隔小于60秒，则分为同一个组
1001	17523641234		1
1001	17523641256		1
1001	17523641334		2
1001	17523641534		3
1001	17523641544		3
1001	17523641638		4
1001	17523641654		4
1002	17523641278		1
1002	17523641434		2
1002	17523641634		3
```

```
/*计算每个用户范围内从第一行到当前行tsdiff大于等于60的总个数(分组号)*/
select id, ts, sum(tsdiff>=60, 1, 0) over(partition by id order by ts) groupid
from (
	select id, ts, ts, ts-lagts tsdiff/*将当前行时间数据减去上一行时间数据*/
	from (
		/*将上一行时间数据下移*/
		select id, ts, lag(ts, 1, 0) over(partition by id order by ts) lagts
		from test2
		) t1
	) t2
```

# 间隔连续问题

```
id		dt
1001	2021-12-12
1001	2021-12-13
1001	2021-12-14
1001	2021-12-16
1001	2021-12-19
1001	2021-12-20

1002	2021-12-12
1002	2021-12-16
1002	2021-12-17
```

```
select id, max(days)+1 /*取连续登录天数的最大值*/
from (
	/*按照用户和flag分组,求最大时间减去最小时间并加上1*/
	select id, flag, datediff(max(dt),min(dt)) days
	from (
		select id, dt,
			/*按照用户分组,同时按照时间排序,计算从第一行到当前行大于2的数据的总条数(sum(if(flag>2,1,0)))*/
    		sum(if(flag>2,1,0)) over(partition by id order by dt) flag
		from (
			/*将当前行时间减去上一行时间数据(datediff(dt1,dt2))*/
			select id, dt, datediff(dt,lagdt) flag
			from (
				/*将上一行时间数据下移*/
				select id, dt, lag(dt,1,'1970-01-01') over(partition by id order by dt) lagdt
				from test3
				)t1
			)t2
		)t3
	group by id,flag
	)t4
group by id;
```

#  打折日期交叉问题

```
id 		stt 		edt
oppo	2021-06-05	2021-06-09
oppo	2021-06-11	2021-06-21

vivo	2021-06-05	2021-06-15
vivo	2021-06-09	2021-06-21

redmi	2021-06-05	2021-06-21
redmi	2021-06-09	2021-06-15
redmi	2021-06-17	2021-06-26

huawei	2021-06-05	2021-06-26
huawei	2021-06-09	2021-06-15
huawei	2021-06-17	2021-06-21
```



```
select id, sum(if(days>=0,days+1,0)) days /*按照品牌分组,计算每条数据加一的总和*/
from (
	select id, datediff(edt,stt) days /*将每行数据中的结束日期减去开始日期*/
	from (
		/*比较开始时间与移动下来的数据,如果开始时间大,则不需要操作,
		  反之则需要将移动下来的数据加一替换当前行的开始时间
		  如果是第一行数据,maxEDT为null,则不需要操作*/
		select id,if(maxEdt is null,stt,if(stt>maxEdt,stt,date_add(maxEdt,1))) stt, edt
		from (
			select id, stt, edt,
			/*将当前行以前的数据中最大的edt放置当前行*/
    max(edt) over(partition by id order by stt rows between UNBOUNDED PRECEDING and 1 PRECEDING) maxEdt
			from test4
			)t1
		)t2
	)t3
group by id;
```

#  同时在线问题

```
id		stt						edt
1001	2021-06-14 12:12:12		2021-06-14 18:12:12
1003	2021-06-14 13:12:12		2021-06-14 16:12:12
1004	2021-06-14 13:15:12		2021-06-14 20:12:12
1002	2021-06-14 15:12:12		2021-06-14 16:12:12
1005	2021-06-14 15:18:12		2021-06-14 20:12:12
1001	2021-06-14 20:12:12		2021-06-14 23:12:12
1006	2021-06-14 21:12:12		2021-06-14 23:15:12
1007	2021-06-14 22:12:12		2021-06-14 23:10:12
```

```
select max(sum_p)
from (
	/*按照时间排序,计算累加人数*/
	select id, dt, sum(p) over(order by dt) sum_p
	from (
		/*对数据分类,在开始数据后添加正1,表示有主播上线,同时在关播数据后添加-1,表示有主播下线*/
		select id,stt dt,1 p from test5
		union
		select id,edt dt,-1 p from test5
		)t1
	)t2
```

# 对城市的销售最好的10件商品

```
bill_id		good_name	city_name
1001	巧克力		   上海
1003	核桃		    北京
```

```
select city_name, good_name, ranks
from (
    select city_name, good_name,
        rank() over(partition by city_name order by bills desc) as ranks
    from (
        select city_name, good_name, count(1) bills
        from table
        group by city_name, good_name
        )
    )
where ranks <= 10
```

# 对每个学生得到最高分和最低分

```
st_id   clazz   score
1001    en       10	
1003	data     15
```

```
select st_id, clazz, score, ranks
	(case when ranks = 1 then clazz end) max_clazz,
	(case when ranks = 1 then score end) max_score,
	(case when ranks = 10 then clazz end) min_clazz,
	(case when ranks = 10 then score end) min_score
from (
	select st_id, clazz, score,
		row_number() over (partiton by st_id order by score desc) ranks
	from table
	)
group by st_id


select st_id, clazz, score,
	first_value(score) over(partition by st_id order by score) first_score,
	last_value(score) over(partition by st_id order by score 
							rows between unbounded preceding and unbounded following ) last_score
from table
```

# 求五分钟内点击5次的用户

```
dt                    id    url
2019-08-22 19:00:01    1    www.baidu.com
2019-08-22 19:01:01    1    www.baidu.com
2019-08-22 19:02:01    1    www.baidu.com
2019-08-22 19:03:01    2    www.baidu.com
2019-08-22 19:04:01    2    www.baidu.com
```

```
select id, dt
from (
	select id, dt,
		/*计算当前行访问时间和前五行数据的访问时间差*/
    	(unix_timestamp(dt,'yyyy-MM-dd HH:mm')-unix_timestamp(newDt,'yyyy-MM-dd HH:mm'))/60 diffDt
	from (
		/*按照用户分组开窗，使用lag函数将数据下移5行*/
		select id, dt, lag(dt,5,'') over(partition by id order by dt) newDt 
		from test6
		)t1
	)t2
where diffDt <= 5 /*判断时间差是否小于5分钟*/
```

