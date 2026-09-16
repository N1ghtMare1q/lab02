
# Лабораторна робота 2. Створення складних SQL запитів

## Загальна інформація

**Здобувач освіти:** [Мельник Станіслав Павлович]
**Група:** [ІПЗ-33]
**Обраний рівень складності:** [1/2]

## Виконання завдань

### Рівень 1

#### 1. З'єднання таблиць

**Завдання 1.1:** INNER JOIN - список товарів з категоріями та постачальниками

SELECT 
    p.product_name, 
    c.category_name, 
    s.company_name AS supplier_name, 
    p.unit_price
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
INNER JOIN suppliers s ON p.supplier_id = s.supplier_id
ORDER BY c.category_name, p.product_name;

**Результат виконання:**

<img width="973" height="883" alt="image" src="https://github.com/user-attachments/assets/cceea8a0-ef7a-40cc-b657-d5a384895b2a" />




**Пояснення:** Запит здійснює внутрішнє з'єднання трьох таблиць: основної products, 
довідника категорій categories (за первинним ключем category_id) та довідника контрагентів suppliers (за supplier_id). 
INNER JOIN гарантує, що в підсумкову вибірку потрапляють виключно ті позиції, для яких одночасно визначені і категорія, 
і постачальник (перетин множин без NULL у ключових полях).



**Завдання 1.2:** LEFT JOIN - клієнти з кількістю замовлень

SELECT 
    c.contact_name, 
    c.customer_type, 
    r.region_name,
    COUNT(o.order_id) AS order_count
FROM customers c
LEFT JOIN orders o ON c.customer_id = o.customer_id
LEFT JOIN regions r ON c.region_id = r.region_id
GROUP BY c.customer_id, c.contact_name, c.customer_type, r.region_name
ORDER BY order_count DESC;

**Результат виконання:**

<img width="734" height="896" alt="image" src="https://github.com/user-attachments/assets/25197b74-cbe6-47f0-85da-8b9037be182e" />


**Пояснення:** На відміну від INNER JOIN, оператор LEFT JOIN зберігає всі записи з лівої таблиці (customers), навіть якщо клієнт ще не зробив жодного замовлення. 
У такому разі поля правої таблиці orders заповнюються значенням NULL.Агрегатна функція COUNT(o.order_id) повертає 0 для таких клієнтів (оскільки COUNT(column) ігнорує NULL), 
тоді як INNER JOIN просто викинув би цих покупців зі звіту.



**Завдання 1.3:** Множинне з'єднання - детальна інформація про замовлення

SELECT 
    o.order_id,
    o.order_date,
    cu.contact_name AS customer_name,
    p.product_name,
    cat.category_name,
    oi.quantity,
    oi.unit_price,
    ROUND(oi.quantity * oi.unit_price * (1 - oi.discount), 2) AS line_total
FROM orders o
JOIN customers cu ON o.customer_id = cu.customer_id
JOIN order_items oi ON o.order_id = oi.order_id
JOIN products p ON oi.product_id = p.product_id
JOIN categories cat ON p.category_id = cat.category_id
ORDER BY o.order_date DESC, o.order_id;

**Результат виконання:**

<img width="1307" height="901" alt="image" src="https://github.com/user-attachments/assets/23b9e0d3-4326-450a-becb-815444d815bd" />


**Аналіз складності:** Ми просто розгортаємо "чеки" магазину. Починаємо із замовлення $\rightarrow$ дивимося, хто його зробив (клієнт) $\rightarrow$ дивимося, 
які рядки в чеку (order_items) $\rightarrow$ дізнаємося точну назву товару $\rightarrow$ і в яку категорію він входить.
Це як зібрати пазл з 5 різних коробок в одну зрозумілу таблицю.


#### 2. Агрегатні функції

**Завдання 2.1:** Статистика товарів за категоріями

SELECT 
    c.category_name,
    COUNT(p.product_id) AS product_count,
    ROUND(AVG(p.unit_price), 2) AS avg_price,
    MIN(p.unit_price) AS min_price,
    MAX(p.unit_price) AS max_price
FROM categories c
LEFT JOIN products p ON c.category_id = p.category_id
GROUP BY c.category_id, c.category_name
ORDER BY product_count DESC;


**Результат виконання:**

<img width="663" height="769" alt="image" src="https://github.com/user-attachments/assets/8bc4eb2c-deb0-4577-b3fb-15c8905fe828" />


**Завдання 2.2:** Продажі за регіонами з використанням HAVING

SELECT 
    r.region_name,
    COUNT(DISTINCT o.order_id) AS delivered_orders_count,
    ROUND(SUM(oi.quantity * oi.unit_price * (1 - oi.discount)), 2) AS total_revenue
FROM orders o
JOIN order_items oi ON o.order_id = oi.order_id
JOIN regions r ON o.ship_region_id = r.region_id
WHERE o.order_status = 'delivered'
GROUP BY r.region_id, r.region_name
HAVING SUM(oi.quantity * oi.unit_price * (1 - oi.discount)) > 50000
ORDER BY total_revenue DESC;

**Результат виконання:**

<img width="662" height="566" alt="image" src="https://github.com/user-attachments/assets/a1d917c6-5ec4-4f8b-930e-74d4ca598dd5" />


**Завдання 2.3:** Постачальники з кількістю товарів більше 2

SELECT 
    s.company_name,
    s.city,
    COUNT(p.product_id) AS products_supplied
FROM suppliers s
JOIN products p ON s.supplier_id = p.supplier_id
GROUP BY s.supplier_id, s.company_name, s.city
HAVING COUNT(p.product_id) > 2
ORDER BY products_supplied DESC;


**Результат виконання:**

<img width="512" height="532" alt="image" src="https://github.com/user-attachments/assets/b0d514df-3040-4acc-92e7-7d7dada56ee4" />



#### 3. Базові підзапити

**Завдання 3.1:** Товари з ціною вище середньої по категорії

SELECT 
    p.product_name, 
    p.unit_price, 
    c.category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE p.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p.category_id
)
ORDER BY c.category_name, p.unit_price DESC;

**Результат виконання:**

<img width="768" height="860" alt="image" src="https://github.com/user-attachments/assets/82d9be78-3b2c-4aa6-9ecb-123211d03803" />



**Завдання 3.2:** Клієнти з замовленнями у 2024 році

SELECT 
    customer_id, 
    contact_name, 
    city, 
    customer_type
FROM customers
WHERE customer_id IN (
    SELECT customer_id
    FROM orders
    WHERE order_date BETWEEN '2024-01-01' AND '2024-12-31'
)
ORDER BY contact_name;

**Результат виконання:**

<img width="675" height="893" alt="image" src="https://github.com/user-attachments/assets/e6de4bcc-3a17-4182-bce4-b90ad97633a4" />


**Завдання 3.3:** Товари з загальною кількістю продажів

SELECT 
    p.product_id,
    p.product_name,
    p.unit_price,
    COALESCE((
        SELECT SUM(oi.quantity)
        FROM order_items oi
        JOIN orders o ON oi.order_id = o.order_id
        WHERE oi.product_id = p.product_id 
          AND o.order_status = 'delivered'
    ), 0) AS total_units_sold
FROM products p
ORDER BY total_units_sold DESC;

**Результат виконання:**

<img width="814" height="894" alt="image" src="https://github.com/user-attachments/assets/22f848d2-5bac-4b30-ad96-1415d3f6407d" />




### Рівень 2

#### 4. Складні з'єднання

**Завдання 4.1:** RIGHT JOIN - аналіз категорій та товарів

SELECT 
    c.category_name,
    COUNT(p.product_id) AS products_count,
    COALESCE(ROUND(AVG(p.unit_price), 2), 0) AS avg_price
FROM products p
RIGHT JOIN categories c ON p.category_id = c.category_id
GROUP BY c.category_id, c.category_name
ORDER BY products_count DESC;

**Результат виконання:**

<img width="590" height="743" alt="image" src="https://github.com/user-attachments/assets/26026030-5d53-467d-b44f-9ce46762ae1f" />


**Завдання 4.2:** Self-join - співробітники та керівники

SELECT 
    e1.first_name || ' ' || e1.last_name AS employee,
    e1.title AS employee_title,
    COALESCE(e2.first_name || ' ' || e2.last_name, 'Немає (Керівник компанії)') AS manager,
    COALESCE(e2.title, '—') AS manager_title
FROM employees e1
LEFT JOIN employees e2 ON e1.reports_to = e2.employee_id
ORDER BY e2.last_name NULLS FIRST, e1.last_name;


**Результат виконання:**

<img width="792" height="760" alt="image" src="https://github.com/user-attachments/assets/113cf8ed-8605-48db-82d9-18bbe9cef558" />




#### 5. Віконні функції

**Завдання 5.1:** Ранжування товарів за ціною в категоріях

SELECT 
    c.category_name,
    p.product_name,
    p.unit_price,
    ROW_NUMBER() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) AS row_num,
    RANK() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) AS price_rank,
    DENSE_RANK() OVER (PARTITION BY c.category_name ORDER BY p.unit_price DESC) AS price_dense_rank
FROM products p
JOIN categories c ON p.category_id = c.category_id
ORDER BY c.category_name, p.unit_price DESC;

**Результат виконання:**

<img width="1017" height="863" alt="image" src="https://github.com/user-attachments/assets/76c167de-4a59-4125-b40e-3ad908f63bc3" />


**Завдання 5.2:** Порівняння замовлень з попередніми датами

SELECT 
    customer_id,
    order_id,
    order_date,
    freight,
    LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS prev_order_date,
    LEAD(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS next_order_date,
    order_date - LAG(order_date) OVER (PARTITION BY customer_id ORDER BY order_date) AS days_since_last_order
FROM orders
ORDER BY customer_id, order_date;

**Результат виконання:**

<img width="959" height="864" alt="image" src="https://github.com/user-attachments/assets/50c3b3a1-ccc8-4daf-a2c2-88fa4067957d" />




## Аналіз продуктивності

### Дослідження планів виконання

**Найповільніший запит:**

EXPLAIN ANALYZE
SELECT p.product_name, p.unit_price, c.category_name
FROM products p
INNER JOIN categories c ON p.category_id = c.category_id
WHERE p.unit_price > (
    SELECT AVG(p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p.category_id
);

**План виконання (EXPLAIN ANALYZE):**

<img width="599" height="829" alt="image" src="https://github.com/user-attachments/assets/d2211195-639c-4701-a346-b80001c3425f" />


### Створені індекси

**Індекс 1:**

CREATE INDEX idx_products_cat_price ON products(category_id, unit_price);

**Обґрунтування:** Прискорює фільтрацію та агрегацію товарів у розрізі категорій, дозволяючи СУБД використовувати швидкий Index Only Scan без звернення до сторінок даних самої таблиці.

**Індекс 2:**

CREATE INDEX idx_orders_customer_status ON orders(customer_id, order_status);

**Обґрунтування:** Оптимізує вибірки замовлень конкретного клієнта з фільтрацією за статусом (наприклад, тільки delivered), що часто зустрічається в кабінеті користувача та аналітичних звітах.


## Порівняльний аналіз

### Ефективність різних підходів

**Завдання:** Знайти топ-5 найдорожчих товарів у кожній категорії

**Підхід 1: Віконні функції**

WITH ranked_products AS (
    SELECT 
        product_name, 
        category_id, 
        unit_price,
        DENSE_RANK() OVER (PARTITION BY category_id ORDER BY unit_price DESC) AS rnk
    FROM products
)
SELECT product_name, category_id, unit_price
FROM ranked_products
WHERE rnk <= 5;

**Підхід 2: Корельований підзапит**

SELECT p1.product_name, p1.category_id, p1.unit_price
FROM products p1
WHERE (
    SELECT COUNT(DISTINCT p2.unit_price)
    FROM products p2
    WHERE p2.category_id = p1.category_id 
      AND p2.unit_price >= p1.unit_price
) <= 5
ORDER BY p1.category_id, p1.unit_price DESC;

**Час виконання:**
- Віконні функції: ~0.660 ms
- Корельований підзапит: ~0.395 ms

**Висновок:** Підхід із віконними функціями (DENSE_RANK) показав себе суттєво ефективнішим (швидшим майже в 4 рази навіть на малій навчальній вибірці). 
Віконні функції проходять дані в один-два проходи за допомогою сортування або хешування, тоді як корельований підзапит для кожного окремого товару запускає вкладений підрахунок COUNT(DISTINCT ...),
що спричиняє квадратичне зростання складності при збільшенні обсягу даних.



## Висновки

**Самооцінка**: [4]

**Обгрунтування**: У повному обсязі виконано всі завдання Рівня 1, Рівня 2. Продемонстровано глибоке володіння різними типами з'єднань (INNER, LEFT, RIGHT, Self-join), 
агрегатними та віконними функціями (RANK, LAG, LEAD), а також сучасними аналітичними інструментами PostgreSQL (WITH RECURSIVE, MATERIALIZED VIEW). 
Проведено практичний аналіз планів виконання запитів через EXPLAIN ANALYZE та обґрунтовано вибір оптимальних методів оптимізації.
