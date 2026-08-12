<div align="center">

# 🐬 MySQL — From Zero to Query Master

### A hands-on, example-driven guide to writing real-world SQL in MySQL

**🎥 Learn it step-by-step on YouTube → [Watch the Full MySQL Playlist](https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9)**

<a href="https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9" title="Watch the full MySQL course on YouTube">
  <img src="https://img.youtube.com/vi/h2Bf3IvN8gw/maxresdefault.jpg" alt="MySQL Complete Course — click to watch the full playlist on YouTube" width="640">
</a>

[![Watch on YouTube](https://img.shields.io/badge/▶%20Watch%20the%20Course-YouTube-FF0000?style=for-the-badge&logo=youtube&logoColor=white)](https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9)
[![MySQL](https://img.shields.io/badge/MySQL-8.0+-4479A1?style=for-the-badge&logo=mysql&logoColor=white)](https://dev.mysql.com/)
[![License: MIT](https://img.shields.io/badge/License-MIT-green?style=for-the-badge)](LICENSE)

</div>

---

> ### ⭐ Enjoying this guide?
> **[Subscribe on YouTube](https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9)** for the complete video walkthrough of every topic below, and **give this repo a ⭐** so more learners can find it. New lessons drop regularly — hit the bell so you never miss one.

---

## 📖 About This Repository

This is a **structured, beginner-to-advanced MySQL reference** built to pair with the video course. Every guide is written as a standalone lesson with clear explanations, copy-paste-ready SQL, and worked examples against a single, consistent sample database — so concepts build naturally from one part to the next.

Whether you're preparing for an interview, leveling up at work, or just starting out, work through the parts in order or jump straight to the topic you need.

## 🚀 Getting Started

1. **Set up the sample database** — start with [Part 0: Sample Database](docs/00-sample-database.md) and run the schema + seed data in your MySQL instance.
2. **Follow along with the video** — each guide maps to a lesson in the [YouTube playlist](https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9).
3. **Practice** — every query in these docs runs against the sample database, so you can experiment as you learn.

```bash
# Clone the repo
git clone https://github.com/vivekpandey76/Mysql-course.git
cd Mysql-course

# Load the sample database into MySQL, then start with docs/00-sample-database.md
```

## 📚 Course Contents

| # | Topic | Guide |
|---|-------|-------|
| 0 | Sample Database (schema & seed data) | [Sample Database](docs/00-sample-database.md) |
| 1 | Filtering Data (`WHERE`, operators, patterns) | [Filtering Data](docs/01-filtering-data.md) |
| 2 | Aggregate Functions, `GROUP BY` & `HAVING` | [Aggregate Functions & GROUP BY](docs/02-aggregate-functions-and-group-by.md) |
| 3 | JOINs (INNER, LEFT, RIGHT, SELF) | [Joins](docs/03-joins.md) |
| 4 | Subqueries | [Subqueries](docs/04-subqueries.md) |
| 5 | Window Functions | [Window Functions](docs/05-window-functions.md) |
| 6 | Common Table Expressions (CTEs) | [Common Table Expressions](docs/06-common-table-expressions.md) |
| 7 | String, Date & `CASE` Functions | [String, Date & CASE Functions](docs/07-string-date-and-case-functions.md) |
| 8 | Views (virtual tables) | [Views](docs/08-views.md) |
| 9 | Stored Procedures | [Stored Procedures](docs/09-stored-procedures.md) |
| 10 | Triggers (automatic `INSERT`/`UPDATE`/`DELETE` actions) | [Triggers](docs/10-triggers.md) |
| 11 | `UNION` & `UNION ALL` (stacking result sets) | [UNION & UNION ALL](docs/11-union-and-union-all.md) |
| 12 | Recursive CTEs (hierarchies & sequence generation) | [Recursive CTE](docs/12-recursive-cte.md) |
| 13 | Indexes (single, composite, unique, prefix & `EXPLAIN`) | [Indexes](docs/13-indexes.md) |

## 🧩 Interview Practice

Real-world SQL problems with datasets, expected output, and fully explained solutions — great for interview prep and self-testing.

| Set | Focus | Link |
|-----|-------|------|
| 1 | Anti-joins, delivery times, running balances, distinct users, cancellation rates | [Problem Set 1](interview-questions/problem-set-01.md) |
| 2 | Self-joins & interval overlaps, top-N per group, `HAVING`, `LEAD()`, conditional aggregation | [Problem Set 2](interview-questions/problem-set-02.md) |
| 3 | Screening-round classics: Nth-highest salary, find & delete duplicates, manager/dept comparisons, month-over-month growth | [Problem Set 3](interview-questions/problem-set-03.md) |
| 4 | Multi-table joins, median (overall & per-group), gaps-and-islands streaks (2 methods), self-`UNION` win-percentage | [Problem Set 4](interview-questions/problem-set-04.md) |
| 5 | String parsing with `SUBSTRING_INDEX`, longest win streak, pivot & unpivot reshaping, first/last order per customer | [Problem Set 5](interview-questions/problem-set-05.md) |
| 6 | Consecutive-day orders, "bought all products", missing IDs, `NOT EXISTS` + theory: `GROUP BY` rules, `DELETE`/`TRUNCATE`/`DROP`, `IFNULL` vs `COALESCE` | [Problem Set 6](interview-questions/problem-set-06.md) |

## 🏆 LeetCode SQL 50

Full walkthroughs of the [LeetCode SQL 50](https://leetcode.com/studyplan/top-sql-50/) study plan — the fastest way to get interview-ready. Each problem includes the **original LeetCode link**, the schema and sample data, real verified output, a **"How to approach it"** breakdown, and the trade-offs behind the solution.

| Set | Problems | Focus | Link |
|-----|----------|-------|------|
| 1 | 1–10 | `WHERE` logic, `NULL` traps, `INNER` vs `LEFT JOIN`, anti-joins, `LAG()`, self-joins | [Set 1 — Select & Basic Joins](leetcode-problems/set-01-easy-basics.md) |
| 2 | 11–20 | `CROSS JOIN` scaffolding, `COUNT(*)` vs `COUNT(col)`, `HAVING`, conditional aggregation, rates & percentages, range joins | [Set 2 — Joins & Aggregation](leetcode-problems/set-02-joins-and-aggregation.md) |
| 3 | 21–30 | First-row-per-group with `ROW_NUMBER()`, `DENSE_RANK()` on ties, `LEAD()` retention, `COUNT(DISTINCT)`, relational division, inclusive date windows | [Set 3 — Window Functions & First-Row Logic](leetcode-problems/set-03-window-functions-and-first-rows.md) |

## 🗂️ Repository Structure

```
mysql/
├── docs/                 # Lesson guides (start with 00, follow in order)
│   ├── 00-sample-database.md
│   ├── 01-filtering-data.md
│   ├── 02-aggregate-functions-and-group-by.md
│   ├── 03-joins.md
│   ├── 04-subqueries.md
│   ├── 05-window-functions.md
│   ├── 06-common-table-expressions.md
│   ├── 07-string-date-and-case-functions.md
│   ├── 08-views.md
│   ├── 09-stored-procedures.md
│   ├── 10-triggers.md
│   ├── 11-union-and-union-all.md
│   ├── 12-recursive-cte.md
│   └── 13-indexes.md
├── interview-questions/  # Practice problem sets with solutions
│   ├── problem-set-01.md
│   ├── problem-set-02.md
│   ├── problem-set-03.md
│   ├── problem-set-04.md
│   ├── problem-set-05.md
│   └── problem-set-06.md
├── leetcode-problems/    # LeetCode SQL 50 walkthroughs
│   ├── set-01-easy-basics.md
│   ├── set-02-joins-and-aggregation.md
│   └── set-03-window-functions-and-first-rows.md
├── assets/               # Diagrams & images
│   ├── Indexing.png
│   └── joins.png
├── LICENSE
└── README.md
```

## 🎯 Who This Is For

- **Beginners** learning SQL for the first time
- **Developers & analysts** brushing up on intermediate/advanced querying
- **Interview candidates** revising joins, subqueries, window functions, and CTEs
- **LeetCode grinders** who want the *reasoning* behind each solution, not just an accepted answer

## 🤝 Contributing

Found a typo, a bug in a query, or have a clearer example? Contributions are welcome — open an issue or submit a pull request.

## 📜 License

Released under the [MIT License](LICENSE). Free to use, share, and learn from — attribution appreciated.

---

<div align="center">

**If this helped you, please ⭐ the repo and [subscribe on YouTube](https://www.youtube.com/playlist?list=PLkFShEMrLia1jn4NLHAEI8gX3lIWW6kH9). It genuinely helps! 🙌**

</div>
