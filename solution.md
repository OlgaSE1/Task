--Задание 1

WITH cohorts AS (
    SELECT 
        u.id AS user_id,
        DATE_TRUNC('month', u.date_joined) AS cohort_month
    FROM users u
),
user_entries AS (
    SELECT 
        ue.user_id,
        c.cohort_month,
        EXTRACT(DAY FROM (ue.entry_at - c.cohort_month))::int AS day_number
    FROM userentry ue
    JOIN cohorts c ON ue.user_id = c.user_id
    WHERE ue.entry_at >= c.cohort_month
),
cohort_retention AS (
    SELECT 
        cohort_month,
        user_id,
        MAX(day_number) AS last_active_day
    FROM user_entries
    GROUP BY cohort_month, user_id
)
SELECT 
    cohort_month AS "Когорта",
    COUNT(user_id) AS "Всего пользователей",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 0 THEN 1 END) / COUNT(user_id), 2) AS "D0, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 1 THEN 1 END) / COUNT(user_id), 2) AS "D1, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 3 THEN 1 END) / COUNT(user_id), 2) AS "D3, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 7 THEN 1 END) / COUNT(user_id), 2) AS "D7, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 14 THEN 1 END) / COUNT(user_id), 2) AS "D14, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 30 THEN 1 END) / COUNT(user_id), 2) AS "D30, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 60 THEN 1 END) / COUNT(user_id), 2) AS "D60, %",
    ROUND(100.0 * COUNT(CASE WHEN last_active_day >= 90 THEN 1 END) / COUNT(user_id), 2) AS "D90, %"
FROM cohort_retention
GROUP BY cohort_month
ORDER BY cohort_month;

-- Выводы:
1. Оптимальный срок основного тарифа – 1 месяц. Он позволяет собрать платёж с максимального числа пользователей до того, как они потеряют интерес. 
2. Дополнительно стоит внедрить годовой тариф со скидкой для удержания лояльной аудитории и пробный период на 7–14 дней для привлечения новых клиентов.


--Задание 2

SELECT 
  ROUND(AVG(spending), 2) AS "Среднее списание коинов на пользователя",
  ROUND(AVG(earnings), 2) AS "Среднее начисление коинов на пользователя",
  ROUND(AVG(balance), 2) AS "Средний баланс коинов",
  PERCENTILE_CONT(0.5) WITHIN GROUP (ORDER BY balance) AS "Медианный баланс коинов"
FROM (
  SELECT 
    u.id AS user_id,
    COALESCE(SUM(CASE WHEN t.type_id IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END), 0) AS spending,
    COALESCE(SUM(CASE WHEN t.type_id NOT IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END), 0) AS earnings,
    COALESCE(
      SUM(CASE WHEN t.type_id NOT IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END) 
      - SUM(CASE WHEN t.type_id IN (1,23,24,25,26,27,28,30) THEN t.value ELSE 0 END), 
      0
    ) AS balance
  FROM users u
  LEFT JOIN transaction t ON u.id = t.user_id
  GROUP BY u.id
) AS user_transactions;

-- Выводы:
1. Медианный баланс (56) в 4,2 раза ниже среднего (238) – у половины пользователей почти нет запаса коинов. Это создаёт спрос на пополнение, значит, подписка с ежемесячным пакетом 50–70 коинов будет востребована у широкой аудитории.
2. Среднее списание (27) намного меньше среднего начисления (265) – пользователи получают слишком много бесплатных коинов и не мотивированы тратить. Чтобы подписка стала нужной, сократите бесплатные раздачи или привяжите их к действиям, иначе платить за коины не будут

-- Задание 3

WITH user_stats AS (
  SELECT 
    u.id,
    COUNT(DISTINCT cr.problem_id) AS solved,
    COUNT(DISTINCT ts.test_id) AS tested,
    COUNT(cr.id) AS runs,
    COUNT(cs.id) AS submits
  FROM users u
  LEFT JOIN coderun cr ON u.id = cr.user_id
  LEFT JOIN codesubmit cs ON u.id = cs.user_id
  LEFT JOIN teststart ts ON u.id = ts.user_id
  GROUP BY u.id
),
tx_stats AS (
  SELECT 
    COUNT(DISTINCT user_id) FILTER (WHERE type_id = 23) AS opened_tasks,
    COUNT(DISTINCT user_id) FILTER (WHERE type_id = 24) AS opened_tests,
    COUNT(DISTINCT user_id) FILTER (WHERE type_id = 25) AS opened_hints,
    COUNT(DISTINCT user_id) FILTER (WHERE type_id = 26) AS opened_solutions,
    COUNT(*) FILTER (WHERE type_id IN (23,24,25,26)) AS total_purchases,
    COUNT(DISTINCT user_id) FILTER (WHERE type_id IN (23,24,25,26)) AS bought_any,
    COUNT(DISTINCT user_id) AS any_tx
  FROM transaction
)
SELECT 
  ROUND(AVG(solved), 2) AS "Ср. задач решено на пользователя",
  ROUND(AVG(tested), 2) AS "Ср. тестов пройдено на пользователя",
  ROUND(COALESCE(SUM(runs)::numeric / NULLIF(SUM(solved), 0), 0), 2) AS "Ср. попыток на задачу",
  ROUND(COALESCE(SUM(submits)::numeric / NULLIF(SUM(tested), 0), 0), 2) AS "Ср. попыток на тест",
  ROUND(COUNT(CASE WHEN solved > 0 OR tested > 0 THEN 1 END) * 100.0 / COUNT(*), 2) AS "Доля активных пользователей, %",
  tx_stats.opened_tasks AS "Пользователей открывали задачи за коины",
  tx_stats.opened_tests AS "Пользователей открывали тесты за коины",
  tx_stats.opened_hints AS "Пользователей открывали подсказки за коины",
  tx_stats.opened_solutions AS "Пользователей открывали решения за коины",
  tx_stats.total_purchases AS "Всего покупок за коины",
  tx_stats.bought_any AS "Пользователей покупали что‑то за коины",
  tx_stats.any_tx AS "Пользователей с транзакциями"
FROM user_stats
CROSS JOIN tx_stats
GROUP BY 
  tx_stats.opened_tasks,
  tx_stats.opened_tests,
  tx_stats.opened_hints,
  tx_stats.opened_solutions,
  tx_stats.total_purchases,
  tx_stats.bought_any,
  tx_stats.any_tx;

--Выводы:
1. Безлимитное открытие задач (или квота > среднего потребления, например, 10–20 задач в месяц).
2. Безлимитные подсказки – сейчас их покупают, значит, ценят.
3. Доступ к тестам – можно сделать ограниченное число бесплатных, а в подписке – безлимит.
4. Эксклюзивные задачи и тесты – только для подписчиков (чтобы стимулировать переход).
5. Обучающий контент – разборы решений, теория, чтобы сократить попытки и повысить успешность.
5. Решения – можно оставить бесплатными, но добавить в подписку доступ к «правильным решениям с пояснениями» (если сейчас только код).

--Дополнительное задание

1. Churn Rate (отток за 30 дней)
Мне кажется, что для оценки эффективности удержания пользователей и понимания того, насколько быстро мы теряем аудиторию, ключевым показателем является Churn Rate за первый месяц.
Для этого я предлагаю посчитать долю пользователей, которые не возвращаются на платформу в течение 30 дней после регистрации (как дополнение к Retention D30).
Потому что:
- Показывает эффективность онбординга. Если отток высокий (например, >70%), значит, пользователи не видят ценности в первые недели – это сигнал к улучшению первого опыта, обучению или ранним «быстрым победам».
- Помогает рассчитать стоимость оттока. Зная, сколько мы тратим на привлечение одного пользователя (CAC), мы можем оценить, сколько денег «сгорает» из-за уходящих пользователей, и приоритизировать задачи по удержанию.
- Выявляет проблемные когорты. Сравнивая Churn по месяцам регистрации, мы видим, какие маркетинговые кампании или сезонные факторы приводят к худшему удержанию, и можем точечно корректировать стратегию привлечения.

2. Средний срок жизни пользователя (Average Lifetime)
Мне кажется, что для планирования долгосрочной монетизации и оценки окупаемости инвестиций в привлечение необходимо знать, сколько в среднем живёт пользователь на платформе.
Для этого я предлагаю посчитать среднее количество дней от регистрации до последней активности (max_days_active) по каждой когорте.
Потому что:
- Базовый ингредиент для LTV. Средний срок жизни умножается на средний доход с пользователя в месяц – так мы получаем прогноз пожизненной ценности клиента и можем обоснованно устанавливать бюджет на маркетинг (CAC не должен превышать LTV).
- Сравнивает каналы привлечения. Если пользователи из одного канала живут в 2 раза дольше, чем из другого, мы перераспределяем бюджет в пользу более качественного трафика, повышая общую эффективность.
- Помогает выбрать период подписки. Если средний срок жизни меньше 30 дней, годовая подписка не имеет смысла, а вот помесячная или даже недельная – в самый раз. Это напрямую влияет на дизайн тарифной сетки.

3. Доля «ядровых» пользователей (активны >90 дней)
Мне кажется, что для оценки устойчивости бизнес-модели и выявления самой ценной аудитории стоит выделить пользователей, которые остаются активными дольше 90 дней.
Для этого я предлагаю посчитать долю таких пользователей в каждой когорте (активны после 90 дней с регистрации).
Потому что:
- Показывает устойчивость продукта. Если доля ядра >5–10%, значит, продукт действительно решает долгосрочную проблему – на этих пользователей можно опираться при прогнозировании доходов и развитии новых функций.
- Основа для премиум‑тарифов и VIP‑программ. Именно эти 5–10% готовы платить больше за эксклюзивный контент, ускоренную поддержку или дополнительные возможности. Для них стоит разрабатывать годовые подписки с глубокой скидкой, чтобы зафиксировать их на долгий срок.
- Позволяет запустить реферальную программу. Лояльные пользователи охотнее приводят друзей – зная их долю, мы можем оценить потенциал вирального роста и окупаемость реферальных бонусов, а также настроить программу именно на эту аудиторию.

WITH cohorts AS (
  SELECT 
    id AS user_id,
    DATE_TRUNC('month', date_joined) AS cohort_month
  FROM users
),
user_activity AS (
  SELECT DISTINCT
    ue.user_id,
    c.cohort_month,
    (EXTRACT(EPOCH FROM (ue.entry_at - c.cohort_month)) / 86400)::int AS days_since_registration
  FROM userentry ue
  JOIN cohorts c ON ue.user_id = c.user_id
  WHERE ue.entry_at >= c.cohort_month
),
retention AS (
  SELECT
    cohort_month,
    user_id,
    MAX(days_since_registration) AS max_days_active
  FROM user_activity
  GROUP BY cohort_month, user_id
)
SELECT
  cohort_month AS "Когорта",
  COUNT(user_id) AS "Всего пользователей",
  ROUND(100.0 * COUNT(CASE WHEN max_days_active >= 30 THEN 1 END) / COUNT(user_id), 2) AS "Retention D30, %",
  ROUND(100.0 - (100.0 * COUNT(CASE WHEN max_days_active >= 30 THEN 1 END) / COUNT(user_id)), 2) AS "Churn Rate (отток за 30 дней), %",
  ROUND(AVG(max_days_active)::numeric, 2) AS "Средний срок жизни, дней",
  COUNT(CASE WHEN max_days_active >= 90 THEN 1 END) AS "Ядровых пользователей",
  ROUND(100.0 * COUNT(CASE WHEN max_days_active >= 90 THEN 1 END) / COUNT(user_id), 2) AS "Доля ядровых, %"
FROM retention
GROUP BY cohort_month
ORDER BY cohort_month;

--Итоговые выводы по смене модели монетизации

Я считаю, что вам нужно сократить пробный период с 30 до 14 дней (или вообще убрать бесплатный доступ), чтобы не терять потенциальный доход на пользователях, которые всё равно уходят до первого платежа, внедрить триггерные коммуникации на 7‑й и 14‑й день – напоминания, мотивационные письма, скидки на подписку, чтобы удержать пользователей в критической точке оттока, провести глубинные интервью с пользователями, которые ушли в феврале–апреле, чтобы понять, что именно их не устроило.

-- Дополнительное задание 2

Для выгрузки данных используем SQL-запрос:

SELECT
  EXTRACT(ISODOW FROM entry_at) AS day_of_week,   -- 1..7
  EXTRACT(HOUR FROM entry_at) AS hour_of_day,
  COUNT(*) AS activity_count
FROM userentry
GROUP BY day_of_week, hour_of_day
ORDER BY day_of_week, hour_of_day;

Для загрузки данных, построения аналога графика (таблицы), применяем SQL-запрос, т.к. Python не изучался на курсе, и используем такой код:
--Сводная таблица (тепловая карта) – активность по дням и часам
SELECT
  CASE EXTRACT(ISODOW FROM entry_at)
    WHEN 1 THEN 'Пн'
    WHEN 2 THEN 'Вт'
    WHEN 3 THEN 'Ср'
    WHEN 4 THEN 'Чт'
    WHEN 5 THEN 'Пт'
    WHEN 6 THEN 'Сб'
    WHEN 7 THEN 'Вс'
  END AS day_name,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 0)  AS h00,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 1)  AS h01,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 2)  AS h02,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 3)  AS h03,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 4)  AS h04,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 5)  AS h05,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 6)  AS h06,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 7)  AS h07,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 8)  AS h08,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 9)  AS h09,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 10) AS h10,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 11) AS h11,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 12) AS h12,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 13) AS h13,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 14) AS h14,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 15) AS h15,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 16) AS h16,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 17) AS h17,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 18) AS h18,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 19) AS h19,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 20) AS h20,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 21) AS h21,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 22) AS h22,
  COUNT(*) FILTER (WHERE EXTRACT(HOUR FROM entry_at) = 23) AS h23
FROM userentry
GROUP BY day_name
ORDER BY MIN(EXTRACT(ISODOW FROM entry_at));

--Агрегированная статистика
	--а) Суммарная активность по дням недели (для выбора лучшего дня релиза)
SELECT
  CASE EXTRACT(ISODOW FROM entry_at)
    WHEN 1 THEN 'Пн'
    WHEN 2 THEN 'Вт'
    WHEN 3 THEN 'Ср'
    WHEN 4 THEN 'Чт'
    WHEN 5 THEN 'Пт'
    WHEN 6 THEN 'Сб'
    WHEN 7 THEN 'Вс'
  END AS day_name,
  COUNT(*) AS total_activity
FROM userentry
GROUP BY day_name
ORDER BY MIN(EXTRACT(ISODOW FROM entry_at));
	--б) Суммарная активность по часам (для выбора лучшего времени)
SELECT
  EXTRACT(HOUR FROM entry_at) AS hour_of_day,
  COUNT(*) AS total_activity
FROM userentry
GROUP BY hour_of_day
ORDER BY hour_of_day;
	--в) День с наименьшей активностью (оптимальный для релиза)
SELECT
  CASE EXTRACT(ISODOW FROM entry_at)
    WHEN 1 THEN 'Пн'
    WHEN 2 THEN 'Вт'
    WHEN 3 THEN 'Ср'
    WHEN 4 THEN 'Чт'
    WHEN 5 THEN 'Пт'
    WHEN 6 THEN 'Сб'
    WHEN 7 THEN 'Вс'
  END AS day_name,
  COUNT(*) AS total_activity
FROM userentry
GROUP BY day_name
ORDER BY total_activity ASC
LIMIT 1;
	--г) Час с наименьшей активностью (оптимальное время суток)
SELECT
  EXTRACT(HOUR FROM entry_at) AS hour_of_day,
  COUNT(*) AS total_activity
FROM userentry
GROUP BY hour_of_day
ORDER BY total_activity ASC
LIMIT 1;
	--д) Часы пиковой активности (например, часы с активностью выше среднего)
WITH hourly AS (
  SELECT
    EXTRACT(HOUR FROM entry_at) AS hour_of_day,
    COUNT(*) AS cnt
  FROM userentry
  GROUP BY hour_of_day
),
avg_hour AS (
  SELECT AVG(cnt) AS avg_cnt FROM hourly
)
SELECT
  hour_of_day,
  cnt,
  ROUND(100.0 * cnt / (SELECT avg_cnt FROM avg_hour), 2) AS above_avg_pct
FROM hourly
WHERE cnt > (SELECT avg_cnt FROM avg_hour)
ORDER BY cnt DESC;

-- Выводы: Рекомендую проводить релизы в субботу в 1 час ночи, потому что в это время и день недели зафиксирована минимальная суммарная активность (на основе SQL-аналитики, Python не изучался на курсе)
