* Проект «Секреты Тёмнолесья»
 * Цель проекта: изучить влияние характеристик игроков и их игровых персонажей 
 * на покупку внутриигровой валюты «райские лепестки», а также оценить 
 * активность игроков при совершении внутриигровых покупок
 *  
 * Автор: Шрайер Людмила
 * Дата: 04.09.2025
*/

-- Часть 1. Исследовательский анализ данных
-- Задача 1. Исследование доли платящих игроков
-- 1.1. Доля платящих пользователей по всем данным:
SELECT COUNT(id) AS count_id, 
SUM(payer) AS count_pay,
SUM(payer)::float / COUNT(id) AS dola
FROM fantasy.users;

-- 1.2. Доля платящих пользователей в разрезе расы персонажа:
SELECT u.race_id, 
r.race,
SUM(u.payer) AS count_pay,
COUNT(u.id) AS count_id,
SUM(u.payer)::float / COUNT(u.id) AS dola
FROM fantasy.users AS u
LEFT JOIN fantasy.race AS r USING (race_id)
GROUP BY u.race_id, r.race;
-- Задача 2. Исследование внутриигровых покупок
-- 2.1. Статистические показатели по полю amount:
SELECT COUNT(transaction_id),
SUM(amount),
MIN(amount),
MAX(amount),
AVG(amount) AS avg_amount,
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median, 
STDDEV(amount) AS stand_dev
FROM fantasy.events -- С нулевыми покупками
UNION
SELECT COUNT(transaction_id),
SUM(amount),
MIN(amount),
MAX(amount),
AVG(amount) AS avg_amount,
PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY amount) AS median,
STDDEV(amount) AS stand_dev
FROM fantasy.events
WHERE amount > 0;   -- Без нулевых покупок (добавила сравнение)

-- 2.2: Аномальные нулевые покупки:
SELECT 
COUNT(amount) AS count_amount,
COUNT(amount)::real / (select COUNT(transaction_id) FROM fantasy.events)::real AS dola
FROM fantasy.events
WHERE amount = 0;    


-- 2.3: Популярные эпические предметы:
with 
dla_item as (select e.item_code,
i.game_items,
COUNT(e.item_code) as count_item
FROM fantasy.events AS e
LEFT JOIN fantasy.items AS i USING (item_code)
WHERE amount <> 0
group by e.item_code, i.game_items
),
dla_id as (select item_code,
COUNT(distinct id) as count_id
FROM fantasy.events
WHERE amount <> 0
group by item_code
)
SELECT distinct it.item_code,
it.game_items,     
it.count_item AS absolut,
it.count_item::float / (select COUNT(transaction_id) FROM fantasy.events WHERE amount <> 0) as otnosit,
i.count_id::float / (select COUNT(distinct id) FROM fantasy.events WHERE amount <> 0) as dola_id
FROM dla_item AS it
LEFT JOIN dla_id AS i USING (item_code)
ORDER BY absolut DESC;

-- Часть 2. Решение ad hoc-задачи
-- Задача: Зависимость активности игроков от расы персонажа:
WITH 
-- кол-во зарегистрированных
count_id AS (SELECT 
u.race_id,
r.race,
COUNT(u.id) AS count_u_id
FROM fantasy.users AS u
LEFT JOIN fantasy.race AS r USING (race_id)
GROUP BY u.race_id, r.race
),
--кол-во покупателей и плятящих
count_events AS (SELECT
u.race_id,
r.race,
COUNT(distinct e.id) as count_e_id,
COUNT(distinct e.id) FILTER (WHERE u.payer = 1) AS count_payer 
FROM fantasy.users AS u 
LEFT JOIN fantasy.race AS r USING (race_id)
LEFT JOIN fantasy.events AS e USING (id)
LEFT JOIN count_id AS ci USING (race)
WHERE amount > 0
GROUP BY u.race_id, r.race, ci.count_u_id
),
-- средние значения
agregat AS (SELECT
u.race_id,
r.race,
COUNT(e.transaction_id) / COUNT(distinct e.id)::float as avg_count,
SUM(e.amount) / COUNT(e.transaction_id) as avg_amount,
SUM(e.amount) / COUNT(distinct e.id) as avg_sum
FROM fantasy.users AS u
LEFT JOIN fantasy.race AS r USING (race_id)
LEFT JOIN fantasy.events AS e USING (id)
WHERE amount > 0
GROUP BY u.race_id, r.race
)
SELECT 
ci.race_id,
ci.race,
ci.count_u_id,--кол-во зарегистрированных 
ce.count_e_id, -- кол-во игроков с покупками
ROUND(ce.count_e_id::numeric / ci.count_u_id, 4) AS dola_id, -- доля игроков с покупками от общего количества зарегистрированных игроков
ROUND(ce.count_payer / ce.count_e_id::numeric, 2) AS dola_payer, -- доля платящих игроков среди игроков, которые совершили внутриигровые покупки
ROUND(a.avg_count::numeric, 2) AS avg_count, -- среднее количество покупок на игрока 
ROUND(a.avg_amount::numeric, 2) AS avg_amount, -- средняя стоимость покупки на игрока
ROUND(a.avg_sum::numeric, 2) AS avg_sum -- средняя суммарная стоимость всех покупок на игрока
FROM count_id AS ci 
FULL JOIN count_events AS ce USING (race_id)
FULL JOIN agregat AS a USING (race_id)
ORDER BY ci.count_u_id DESC;


Изучение влияния характеристик игроков и их игровых персонажей на покупку внутриигровой валюты «райские лепестки», а также оценка активности игроков при совершении внутриигровых покупок
