# CakePHP 3 ORM By Example

Quick set of examples about query building in CakePHP 3 ORM

We're using https://github.com/lorenzo/cakephp3-advanced-examples/blob/master/config/Schema/employees.sql.gz

## Load employees table using any of the available methods:

- **Recommended** Using the ModelAwareTrait
- Using available Table associations
- **Last chance** Using the TableRegistry class

## Preconditions

- Let $employees be a `\App\Model\Table\EmployeesTable` instance
- All PHP snippets will return a `\Cake\ORM\Query`
- For simplicity, sql select `*` will be used instead of the full list of columns, even if the framework will use the columns names instead

## Using the console to test the examples

- Create a new project
- Import the dump
- Copy this file (for global options) https://github.com/bobthecow/psysh/wiki/Sample-config into `~/.config/psysh/config.php` (or `C:\Users\{USER}\AppData\Roaming\PsySH\config.php` on Windows)
- Create a new file in `~/.config/psysh/include/bootstrap.php`
  - This is a great place to setup some `class_alias`, or utility methods for your console experience
- Open your CakePHP Console `bin/cake/console`
- Get a reference to employees table using

```php
$employees = \Cake\ORM\TableRegistry::getTableLocator()->get('Employees');
```

# Simple queries

---

```sql
SELECT * FROM employees;
```

```php
$employees->find();
```

---

```sql
SELECT first_name, last_name FROM employees;
```

```php
$employees->find()
    ->select(['first_name', 'last_name']);
```

# Associations and contain, hasMany

```sql
SELECT * FROM employees Employees LIMIT 1;
SELECT * FROM salaries Salaries WHERE Salaries.employee_id in (10001);
```

```php
$employees->hasMany('Salaries');
$employees->find()
    ->contain('Salaries')
    ->limit(1);
```

- Note CakePHP generates 2 queries for the hasMany association by default

---

```sql
SELECT * FROM employees Employees LIMIT 1;
SELECT * FROM salaries Salaries 
    INNER JOIN (SELECT (Employees.id) FROM employees Employees GROUP BY Employees.id  LIMIT 1) Employees 
        ON Salaries.employee_id = (Employees.id);
```

```php
$employees->hasMany('Salaries')
    ->setStrategy(\Cake\ORM\Association\HasMany::STRATEGY_SUBQUERY);
$employees->find()
    ->contain('Salaries')
    ->limit(1);
```

- Note CakePHP generates 2 queries for the hasMany association, this strategy could improve performance in some cases

# Filtering, top down

```sql
SELECT * FROM employees Employees LIMIT 1;
SELECT * FROM salaries Salaries WHERE (Salaries.employee_id in (10001) AND salary >= 1000);
```

```php
$employees->hasMany('Salaries');
$employees->find()
    ->contain([
        'Salaries' => function(\Cake\ORM\Query $q) {
            return $q->where([
                'salary >=' => 1000
            ]);
        }
    ])
    ->limit(1);
```

# Filtering, bottom up

```sql
SELECT * FROM employees Employees
    INNER JOIN salaries Salaries ON (
        salary >= 1000 AND
        Employees.id = (Salaries.employee_id)
    );
```

```php
$employees->find()
    ->matching('Salaries', function(\Cake\ORM\Query $q) {
        return $q->where([
            'salary >=' => 1000
        ]);
    })->limit(1)
```

# Expressions and functions

```sql
SELECT * FROM employees
    WHERE (CONCAT(first_name, ' ', last_name) LIKE 'luft aron%') OR
(CONCAT(last_name, ' ', first_name) LIKE 'luft aron%');
```

- Expressions based
  - Note we use `$query->identifier()` to mark column names (or 'literal' value in array syntax)
  - Function code is portable vs harder to read
  - Resulting code is reusable, specific complex expressions/functions could be extracted
  - Carefully naming the query parts helps reading the final query

```php
$query = $employees->find();
$concatFirstLast = $query->func()->concat([
    $query->identifier('first_name'),
    "' '" => 'literal',
    $query->identifier('last_name')
]);
$concatLastFirst = $query->func()->concat([
    $query->identifier('last_name'),
    "' '" => 'literal',
    $query->identifier('first_name')
]);
$likeFirstLast = $query->newExpr()->like($concatFirstLast, 'luft aron%');
$likeLastFirst = $query->newExpr()->like($concatLastFirst, 'luft aron%');
$query
    ->where($query->newExpr()->or_([
        $likeFirstLast,
        $likeLastFirst
    ]));
```

- Array based, mysql (**not portable**)

```php
$employees->find()
    ->where([
        'OR' => [
            "CONCAT(first_name, ' ', last_name) LIKE" => 'luft aron%',
            "CONCAT(last_name, ' ', first_name) LIKE" => 'luft aron%',
        ]
    ]);
```
```sql
SELECT
  e.first_name,
  s.from_date,
  s.salary
FROM
  employees AS e
  INNER JOIN salaries AS s ON e.id = (s.employee_id)
WHERE (
    s.from_date BETWEEN '1985-12-01'
    AND '1987-12-01'
    AND s.salary > 60000
);
```

```php
$employees->hasMany('Salaries');
$query = $employees->find();
$query
    ->select([
        'Employees.first_name',
        'Salaries.from_date',
        'Salaries.salary',
    ])
    ->innerJoinWith('Salaries');
    $between = $query->newExpr()->between(
        'Salaries.from_date',
        \Cake\I18n\Time::create(1985, 12, 01),
        \Cake\I18n\Time::create(1987, 12, 01)
    );
    $greaterThan = $query->newExpr()
        ->gt('Salaries.salary', 60000);
    $query->where([
        $between,
        $greaterThan
    ]);
```
