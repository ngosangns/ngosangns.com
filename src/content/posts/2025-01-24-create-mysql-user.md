---
published: 2025-01-24
title: Create a MySQL user
---
```sql
CREATE USER '<username>'@'%' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON <database name>. * TO 'nsenglish'@'%';
CREATE USER '<username>'@'localhost' IDENTIFIED BY '<password>';
GRANT ALL PRIVILEGES ON <database name>. * TO '<username>'@'localhost';
```
