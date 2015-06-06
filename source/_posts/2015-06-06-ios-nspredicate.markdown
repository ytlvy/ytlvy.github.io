---
layout: post
title: "IOS NSPredicate"
date: 2015-06-06 20:56:36 +0800
comments: true
categories: IOS
---
### NSPredicate
NSPredicate is a Foundation class that specifies how data should be fetched or filtered.
```
NSPredicate *bobPredicate = [NSPredicate predicateWithFormat:@"firstName = 'Bob'"];
NSPredicate *thirtiesPredicate = [NSPredicate predicateWithFormat:@"age >= 30"];

[people filterdArrayUsingPredicate:bobPredicate];
```

<!--more-->

#### Predicate Syntax
1. %@ is a var arg substitution for an object value—often a string, number, or date
2. %K is a var arg substitution for a key path

```
NSPredicate *ageIs33Predicate = [NSPredicate predicateWithFormat: @"%K = %@", @"age", 33]
```

3. $VARIABLE_NAME is a value that can be substituted
```
NSPredicate *namesBeginningWithLetterPredicate = 
    [NSPredicate predicateWithFormat: @"(firstName BEGINSWITH[cd] $letter) OR (lastName BEGINWITH[cd] $letter)"];

NSLog(@"'A' Names: %@", 
            [people filterWithPredicate:
                [namesBeginningWithLetterPredicate predicateWithSubstitutionVariables:@{@"letter" : @"A"}]]);
```

#### Basic Comparisons
1. =, ==:   equal 
2. >=, =>:  greater than or equal
3. <=, =<:  less than or equal
4. >: T     greater t.
5. <:       less than.
6. !=, <>:  not equal to
7. BETWEEN  eg: 1 BETWEEN { 0 , 33 }  $INPUT BETWEEN { $LOWER, $UPPER }

#### Basic Compound Predicates
1. AND, &&
2. OR, ||
3. NOT, !

#### String Comparisons
String comparisons are case and diacritic sensitive. by default. One can modify an operator using the key characters c and d within square braces to specify case and diacritic insensitivity respectively. eg: firstName BEGINSWITH[cd] $FIRST_NAME.

1. BEGINSWITH
2. CONTAINS
3. ENDSWITH
4. LIKE         ? and * are allowed as wildcard characters
5. MATCHES      regex-style 

### Aggregate Operations
#### Relational Operations
1. ANY, SOME            eg: ANY children.age < 18
2. ALL                  eg: ALL children.age < 18
3. NONE                 eg: NONE children.age < 18
4. IN                   eg: name IN {'Ben', 'Melissa', 'Nick' }

#### Array Operations
1. array[index]         Specifies the element at the specified index in array
2. array[FIRST]
3. array[LAST]
4. array[SIZE]
5. Boolean
6. TRUEPREDICATE        A predicate that always evaluates to TRUE
7. FALSEPREDICATE       A predicate that always evaluates to FALSE

#### NSCompoundPredicate
```
[NSCompoundPredicate andPredicateWithSubpredicate:@[
    [NSPredicate predicateWithFormat: @"age > 25"],
    [NSPredicate predicateWithFormat: @"firstName = %@", @"Quentin"]
    ]];

[NSPredicate predicateWithFormat: @"(age > 25) AND (firstName = %@)", @"Quentin"];
```

#### NSComparisonPredicate
```
+ (NSPredicate *)predicateWithLeftExpression:
(NSExpression *)lhs
rightExpression:(NSExpression *)rhs
       modifier:(NSComparisonPredicateModifier)modifier
           type:(NSPredicateOperatorType)type
        options:(NSUInteger)options
```
- lhs: The left hand expression.
- rhs: The right hand expression.
- modifier: The modifier to apply. (ANY or ALL)
- type: The predicate operator type.
- options: The options to apply. For no options, pass 0.

#### Block Predicates
```
NSPredicate *shortNamePredicate = 
    [NSPredicate predicateWithBlock: ^BOOL(id evaluatedObject, NSDictionary *bindings){
        return [[evaluatedObject firstName] length] <= 5;
    } ]
```
***NSPredicates created with predicateWithBlock: cannot be used for Core Data fetch requests backed by a SQLite store***