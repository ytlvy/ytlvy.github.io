---
author: null
categories:
- Sqlite
date: '2015-07-28'
description: null
photos: null
tags:
- Sqlite
title: sqlite insert replace
---

## Sqlite insert && replace

Assuming 3 columns in the table.. ID, NAME, ROLE

### BAD: This will insert or replace all columns with new values for ID=1:
```
INSERT OR REPLACE INTO Employee (id, name, role) 
  VALUES (1, 'John Foo', 'CEO');
```

### BAD: This will insert or replace 2 of the columns... the NAME column will be set to NULL or the default value:
```
INSERT OR REPLACE INTO Employee (id, role) 
  VALUES (1, 'code monkey');
```

### GOOD: This will update 2 of the columns. When ID=1 exists, the NAME will be unaffected. When ID=1 does not exist, the name will be default (NULL).
```
INSERT OR REPLACE INTO Employee (id, role, name) 
  VALUES (  1, 
            'code monkey',
            (SELECT name FROM Employee WHERE id = 1)
          );
```

### GOOD: This will update 2 of the columns. When ID=1 exists, the ROLE will be unaffected. When ID=1 does not exist, the role will be set to 'Benchwarmer' instead of the default value.
```
INSERT OR REPLACE INTO Employee (id, name, role) 
  VALUES (  1, 
            'Susan Bar',
            COALESCE((SELECT role FROM Employee WHERE id = 1), 'Benchwarmer')
          );
```
