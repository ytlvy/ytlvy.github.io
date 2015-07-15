---
layout: post
title: "How to Use Reflect to Set a Struct Field"
date: 2015-05-27 15:51:58 +0800
comments: true
categories: Golang
---
Golang reflect package is one of the hardest to understand.
Yet reflect is one of the most powerful because you can inspect a struct and it’s fields during runtime, and change it.
Go isn’t as simple to understand as other languages such as Python in using reflection. Just try reading their [laws of reflection](http://blog.golang.org/laws-of-reflection).
But setting a field of a struct by name could be a one liner. There is a nice answer in [Stack Overflow](http://stackoverflow.com/a/6396678/242682).
In short, if you want to set a struct ```foo``` object when you know the field name to set, this is the code:

```
func setFoo(foo *Foo, field string, value string) {
    v := reflect.ValueOf(foo).Elem().FieldByName(field)
    if v.IsValid() {
        v.SetString(value)
    }
}
```
If you want to print all the fields of `foo`, this is the code:

```
s := reflect.ValueOf(foo).Elem()
typeOfT := s.Type()
for i := 0; i < s.NumField(); i++ {
    f := s.Field(i)
    fmt.Printf("%d: %s %s = %v\n", i,
        typeOfT.Field(i).Name, f.Type(), f.Interface())
}
```

