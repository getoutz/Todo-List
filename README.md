
# 📜 คู่มือการใช้งานสคริปต์ PostgreSQL สำหรับวิเคราะห์ข้อมูลลูกค้า
เอกสารนี้เป็นคู่มือสำหรับใช้งานสคริปต์ SQL เพื่อสร้างฐานข้อมูลจำลองของร้านค้า E-Commerce, เพิ่มข้อมูลตัวอย่าง, และรัน Query ที่ซับซ้อนเพื่อวิเคราะห์หา **ลูกค้า 3 อันดับแรกที่มียอดใช้จ่ายสูงสุดในปี 2024** พร้อมกับ **หมวดหมู่สินค้าที่พวกเขาชื่นชอบที่สุด**

---

## ⚙️ สิ่งที่ต้องมีก่อนเริ่ม (Prerequisites)
1.  **PostgreSQL Server**: คุณต้องมี PostgreSQL ติดตั้งบนเครื่องคอมพิวเตอร์ของคุณ
2.  **Client Tool**: คุณต้องมีโปรแกรมสำหรับเชื่อมต่อกับฐานข้อมูล PostgreSQL เช่น:
    * `psql`: โปรแกรม Command-line ที่มาพร้อมกับการติดตั้ง PostgreSQL
    * **pgAdmin**: โปรแกรมแบบ GUI (Graphical User Interface) ที่นิยมใช้กัน

---

## 🚀 ขั้นตอนการใช้งาน

### ขั้นตอนที่ 1: บันทึกสคริปต์
คัดลอกโค้ดทั้งหมดด้านล่างนี้ และนำไปใช้ได้เลยครับ

```sql
-- ========= Section 1: CREATE TABLES =========
-- ล้างตารางเก่า (ถ้ามี) เพื่อเริ่มต้นใหม่
DROP TABLE IF EXISTS Order_Items, Orders, Products, Categories, Customers CASCADE;

-- 1.1 ตารางลูกค้า
CREATE TABLE Customers (
    customer_id SERIAL PRIMARY KEY,
    first_name VARCHAR(100) NOT NULL,
    last_name VARCHAR(100) NOT NULL,
    registration_date DATE NOT NULL DEFAULT CURRENT_DATE
);

-- 1.2 ตารางหมวดหมู่สินค้า
CREATE TABLE Categories (
    category_id INT PRIMARY KEY,
    category_name VARCHAR(255) NOT NULL UNIQUE
);

-- 1.3 ตารางสินค้า
CREATE TABLE Products (
    product_id INT PRIMARY KEY,
    product_name VARCHAR(255) NOT NULL,
    category_id INT NOT NULL,
    price DECIMAL(10, 2) NOT NULL CHECK (price >= 0),
    CONSTRAINT fk_category
        FOREIGN KEY(category_id)
        REFERENCES Categories(category_id)
);

-- 1.4 ตารางคำสั่งซื้อ
CREATE TABLE Orders (
    order_id INT PRIMARY KEY,
    customer_id INT NOT NULL,
    order_date DATE NOT NULL,
    status VARCHAR(50),
    CONSTRAINT fk_customer
        FOREIGN KEY(customer_id)
        REFERENCES Customers(customer_id)
);

-- 1.5 ตารางรายละเอียดคำสั่งซื้อ
CREATE TABLE Order_Items (
    order_item_id SERIAL PRIMARY KEY,
    order_id INT NOT NULL,
    product_id INT NOT NULL,
    quantity INT NOT NULL CHECK (quantity > 0),
    price_per_unit DECIMAL(10, 2) NOT NULL,
    CONSTRAINT fk_order
        FOREIGN KEY(order_id)
        REFERENCES Orders(order_id),
    CONSTRAINT fk_product
        FOREIGN KEY(product_id)
        REFERENCES Products(product_id)
);

-- ========= Section 2: INSERT DATA =========
-- เพิ่มข้อมูลลงในตาราง
INSERT INTO Customers (customer_id, first_name, last_name, registration_date) VALUES
(1, 'Somchai', 'Jaidee', '2023-01-15'),
(2, 'Somsri', 'Rakdee', '2023-02-20'),
(3, 'Mana', 'Petch', '2023-03-10'),
(4, 'Malee', 'Klinhom', '2024-01-05');

INSERT INTO Categories (category_id, category_name) VALUES
(101, 'Electronics'), (102, 'Books'), (103, 'Clothing'), (104, 'Home & Kitchen');

INSERT INTO Products (product_id, product_name, category_id, price) VALUES
(1, 'Laptop', 101, 35000.00), (2, 'Smartphone', 101, 22000.00), (3, 'SQL Guide', 102, 750.00),
(4, 'T-Shirt', 103, 400.00), (5, 'Coffee Maker', 104, 2500.00);

INSERT INTO Orders (order_id, customer_id, order_date, status) VALUES
(1001, 1, '2024-02-10', 'completed'), (1002, 2, '2024-03-15', 'completed'),
(1003, 1, '2024-04-20', 'completed'), (1004, 3, '2024-05-01', 'completed'),
(1005, 2, '2024-06-12', 'completed'), (1006, 4, '2024-07-22', 'completed'),
(1007, 1, '2025-01-05', 'completed'); -- Order นี้อยู่นอกปี 2024 เพื่อทดสอบการกรอง

INSERT INTO Order_Items (order_id, product_id, quantity, price_per_unit) VALUES
(1001, 1, 1, 35000.00), (1001, 3, 2, 750.00), (1002, 2, 1, 22000.00),
(1002, 4, 5, 400.00), (1003, 4, 10, 400.00), (1004, 5, 1, 2500.00),
(1005, 1, 1, 35000.00), (1006, 3, 3, 750.00);

-- ปรับค่า sequence ของ SERIAL ให้ถูกต้องหลังจาก manual insert
SELECT setval('customers_customer_id_seq', (SELECT MAX(customer_id) FROM Customers));
SELECT setval('order_items_order_item_id_seq', (SELECT MAX(order_item_id) FROM Order_Items));

-- ========= Section 3: CREATE INDEXES =========
-- สร้าง Index บน Foreign Keys และคอลัมน์ที่ใช้บ่อยในการค้นหา/JOIN เพื่อเพิ่มความเร็ว
CREATE INDEX idx_products_category_id ON Products(category_id);
CREATE INDEX idx_orders_customer_id ON Orders(customer_id);
CREATE INDEX idx_orders_order_date ON Orders(order_date); -- สำคัญมากสำหรับ WHERE clause
CREATE INDEX idx_order_items_order_id ON Order_Items(order_id);
CREATE INDEX idx_order_items_product_id ON Order_Items(product_id);

-- ========= Section 4: THE HEAVY QUERY =========
-- Query หลักสำหรับวิเคราะห์ข้อมูล
WITH
  SalesIn2024 AS (
    SELECT o.customer_id, p.category_id, oi.quantity * oi.price_per_unit AS total_item_price
    FROM Order_Items AS oi
    JOIN Orders AS o ON oi.order_id = o.order_id
    JOIN Products AS p ON oi.product_id = p.product_id
    WHERE EXTRACT(YEAR FROM o.order_date) = 2024
  ),
  CustomerTotalSpending AS (
    SELECT customer_id, SUM(total_item_price) AS total_spent_2024
    FROM SalesIn2024
    GROUP BY customer_id
  ),
  CategorySpendingPerCustomer AS (
    SELECT s.customer_id, c.category_name, SUM(s.total_item_price) AS category_spent
    FROM SalesIn2024 AS s
    JOIN Categories AS c ON s.category_id = c.category_id
    GROUP BY s.customer_id, c.category_name
  ),
  RankedCategorySpending AS (
    SELECT customer_id, category_name, category_spent,
           RANK() OVER (PARTITION BY customer_id ORDER BY category_spent DESC) AS category_rank
    FROM CategorySpendingPerCustomer
  )
SELECT
  c.first_name, c.last_name,
  cts.total_spent_2024,
  rcs.category_name AS top_category,
  rcs.category_spent AS spent_in_top_category
FROM RankedCategorySpending AS rcs
JOIN CustomerTotalSpending AS cts ON rcs.customer_id = cts.customer_id
JOIN Customers AS c ON rcs.customer_id = c.customer_id
WHERE rcs.category_rank = 1
ORDER BY cts.total_spent_2024 DESC
LIMIT 3;
