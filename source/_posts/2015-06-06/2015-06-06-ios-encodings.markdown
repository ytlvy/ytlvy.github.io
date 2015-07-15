---
layout: post
title: "ios encodings"
date: 2015-06-06 20:48:05 +0800
comments: true
categories: IOS
---

## Type Encodings

 Code           | Meaning   
 ---------------|---------------------                                       
 c              | A char                                           
 i              | An int                                           
 s              | A short                                          
 l              | A longl is treated as a 32-bit quantity on 64-bit programs.
 q              | A long long                                      
 C              | An unsigned char                                 
 I              | An unsigned int                                  
 S              | An unsigned short                                
 L              | An unsigned long                                 
 Q              | An unsigned long long                            
 f              | A float                                          
 d              | A double                                         
 B              | A C++ bool or a C99 _Bool                        
 v              | A void                                           
 *              | A character string (char *)                      
 @              | An object (whether statically typed or typed id) 
 #              | A class object (Class)                           
 :              | A method selector (SEL)                          
 [array type]   | An array                                         
 {name=type...} | A structure                                      
 (name=type...) | A union                                          
 bnum           | A bit field of num bits                          
 ^type          | A pointer to type                                
 ?              | An unknown type (used for function pointers)     

<!--more-->

```
NSLog(@"int         : %s", @encode(int));                       //i
NSLog(@"float       : %s", @encode(float));                     //f
NSLog(@"float *     : %s", @encode(float*));                    //^f
NSLog(@"char        : %s", @encode(char));                      //c
NSLog(@"char *      : %s", @encode(char *));                    //*
NSLog(@"BOOL        : %s", @encode(BOOL));                      //c
NSLog(@"void        : %s", @encode(void));                      //v
NSLog(@"void *      : %s", @encode(void *));                    //^v

NSLog(@"NSObject *  : %s", @encode(NSObject *));                //@
NSLog(@"NSObject    : %s", @encode(NSObject));                  //#
NSLog(@"[NSObject class]  : %s", @encode(typeof([NSObject class])));  //{NSObject=#}
NSLog(@"NSError **  : %s", @encode(typeof(NSError **)));        //^@

int intArray[5] = {1, 2, 3, 4, 5};
NSLog(@"int[]       : %s", @encode(typeof(intArray)));          //[5i]

float floatArray[3] = {0.1f, 0.2f, 0.3f}; 
NSLog(@"float[]     : %s", @encode(typeof(floatArray)));        //[3f]

typedef struct _struct {
    short a;
    long long b;
    unsigned long long c;
} Struct;
NSLog(@"struct     : %s", @encode(typeof(Struct)));             //{_struct=sqQ}
```

There are some interesting takeaways from this:
1. Whereas the standard encoding for pointers is a preceding ^, char * gets its own code: *. This makes sense conceptually, since C strings are thought to be entities in and of themselves, rather than a pointer to something else.

2. BOOL is c, rather than i, as one might expect. Reason being, char is smaller than an int, and when Objective-C was originally designed in the 80's, bits (much like the US Dollar) were more valuable than they are today.

3. Passing NSObject directly yields #. However, passing [NSObject class] yields a struct named NSObject with a single class field. That is, of course, the isa field, which all NSObject instances have to signify their type.

## Method Encodings
type qualifiers for methods declared in a protocol
 Code    | Meaning  
 --------|--------
 r       | const    
 n       | in       
 N       | inout    
 o       | out      
 O       | bycopy   
 R       | byref    
 V       | oneway   