# MySQL Performance Tuning – Practice Repository

This repository documents my **learning journey**, **hands-on practice**, and **experiments**
while following the YouTube playlist:

🎥 **MySQL Performance Tuning**

🔗 Playlist Link:  
https://www.youtube.com/playlist?list=PLBrWqg4Ny6vXQZqsJ8qRLGRH9osEa45sw

---

## 🎯 Purpose of This Repository

The goal of this repository is to **deeply understand MySQL performance tuning**
through real experiments, large datasets, and practical query analysis.

This is **not theory only** — every concept is:

- Implemented
- Tested
- Measured
- Documented

---

## 📌 Learning Objectives

- Understand how MySQL executes queries internally
- Learn how the MySQL Optimizer works
- Analyze queries using `EXPLAIN` and `EXPLAIN ANALYZE`
- Practice indexing strategies and measure their impact
- Generate large datasets (1M / 5M rows) for performance testing
- Apply real-world performance tuning techniques

---

## 🛠 Initial Setup & Preparation

Before starting the actual performance analysis topics, I focused on setting up
a solid foundation that can be reused throughout the learning process.

---

### 📦 Database Preparation

- Created a reusable table structure to simulate a real-world users table
- The table includes:
  - User identity information
  - Location-related fields
  - Account-related attributes
- This table will be used in upcoming performance and indexing experiments

---

### 🧪 Data Volume Strategy

- Decided to work with large datasets to make performance behavior visible
- Planned data sizes:
  - 1,000,000 rows
  - 5,000,000 rows
- This approach helps reveal:
  - Differences between indexed and non-indexed queries
  - Full table scans vs index scans
  - Optimizer decision patterns

---

### ⚙️ Data Generation Approach

- Prepared a stored procedure to generate large volumes of test data
- Data is inserted in batches to balance performance and safety
- Generated data includes:
  - Unique user records
  - Different account types
  - Simulated geographic distribution
- This setup allows repeating experiments consistently

---

### 🧠 Learning Mindset

- Focus on understanding **why** a query behaves in a certain way
- Avoid premature optimization
- Rely on measurements and execution plans instead of assumptions

---

## 🧾 Initial SQL Queries & Setup Scripts

As part of the preparation phase, the following SQL queries and scripts were written
to set up a controlled environment for future performance testing.

These queries are not focused on optimization yet,
but are essential for creating realistic testing conditions.

---

### 🗄 Table Creation Query

The following query was used to create a reusable table structure
that simulates a real-world users table.

```sql
CREATE TABLE user_info (
  id              INT UNSIGNED NOT NULL AUTO_INCREMENT,
  name            VARCHAR(64)  NOT NULL DEFAULT '',
  email           VARCHAR(64)  NOT NULL DEFAULT '',
  password        VARCHAR(64)  NOT NULL DEFAULT '',
  dob             DATE DEFAULT NULL,
  address         VARCHAR(255) NOT NULL DEFAULT '',
  city            VARCHAR(64)  NOT NULL DEFAULT '',
  state_id        SMALLINT UNSIGNED NOT NULL DEFAULT '0',
  zip             VARCHAR(8)   NOT NULL DEFAULT '',
  country_id      SMALLINT UNSIGNED NOT NULL DEFAULT '0',
  account_type    VARCHAR(32)  NOT NULL DEFAULT '',
  closest_airport VARCHAR(3)   NOT NULL DEFAULT '',
  PRIMARY KEY (id),
  UNIQUE KEY email (email)
) ENGINE=InnoDB;
```

---

### 🔢 Creating the Helper Numbers Table (`seq_10000`)

To efficiently generate large datasets, a helper numbers table was created.

This table provides a sequence of numbers that can be reused in multiple
data generation and performance testing scenarios.

The main goal of this table is to avoid inserting rows one by one
and instead rely on set-based operations.

```sql

CREATE TABLE seq_10000 (
  n INT NOT NULL PRIMARY KEY
) ENGINE=InnoDB;

INSERT INTO seq_10000 (n)
SELECT
  (a.i + b.i * 10 + c.i * 100 + d.i * 1000) + 1 AS n
FROM
  (SELECT 0 i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
   UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) a
CROSS JOIN
  (SELECT 0 i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
   UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) b
CROSS JOIN
  (SELECT 0 i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
   UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) c
CROSS JOIN
  (SELECT 0 i UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3 UNION ALL SELECT 4
   UNION ALL SELECT 5 UNION ALL SELECT 6 UNION ALL SELECT 7 UNION ALL SELECT 8 UNION ALL SELECT 9) d;
```

---

### ⚙️ Stored Procedure for Bulk Data Generation

A stored procedure was created to generate large datasets
in a controlled and repeatable way.

This procedure supports:
- Inserting millions of rows
- Batch-based inserts
- Transaction control
- Repeatable test data generation

```sql
DELIMITER $$

CREATE PROCEDURE seed_user_info(IN p_total_rows INT, IN p_batch_size INT)
BEGIN
  DECLARE v_inserted INT DEFAULT 0;
  DECLARE v_take INT DEFAULT 0;

  IF p_total_rows <= 0 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'p_total_rows must be > 0';
  END IF;

  IF p_batch_size <= 0 OR p_batch_size > 10000 THEN
    SIGNAL SQLSTATE '45000'
      SET MESSAGE_TEXT = 'p_batch_size must be between 1 and 10000';
  END IF;

  SET autocommit = 0;

  WHILE v_inserted < p_total_rows DO
    SET v_take = LEAST(p_batch_size, p_total_rows - v_inserted);

    INSERT INTO user_info
      (name, email, password, dob, address, city, state_id, zip, country_id, account_type, closest_airport)
    SELECT
      CONCAT('User_', v_inserted + s.n),
      CONCAT('user', v_inserted + s.n, '@example.com'),
      CONCAT('pass_', v_inserted + s.n),
      DATE_ADD('1985-01-01', INTERVAL ((v_inserted + s.n) % 12000) DAY),
      CONCAT('Address ', v_inserted + s.n),
      CONCAT('City_', ((v_inserted + s.n) % 200)),
      ((v_inserted + s.n) % 50),
      LPAD(((v_inserted + s.n) % 99999), 5, '0'),
      ((v_inserted + s.n) % 250),
      ELT(((v_inserted + s.n) % 3) + 1, 'basic', 'premium', 'business'),
      ELT(((v_inserted + s.n) % 5) + 1, 'DXB', 'RUH', 'JED', 'AMM', 'DOH')
    FROM seq_10000 s
    WHERE s.n <= v_take;

    SET v_inserted = v_inserted + v_take;
    COMMIT;
  END WHILE;

  SET autocommit = 1;
END$$

DELIMITER ;
```

---

### 📊 Data Seeding & Verification Queries

After preparing the table structure and the data generation procedure,
the following queries were used to populate and verify the dataset.

```sql
TRUNCATE TABLE user_info;

-- Generate 5 million records
CALL seed_user_info(5000000, 10000);
```

Verification queries:
```sql
SELECT COUNT(*) FROM user_info;

SELECT *
FROM user_info
ORDER BY id DESC
LIMIT 5;
```