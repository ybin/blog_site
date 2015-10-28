title: 一个简单的Lisp实现
date: 2015-01-12 10:08:00
categories: 编程
tags: [lisp, c]
---

一个简易的Lisp实现，in C。

<!-- more -->

这个Lisp实现是基于John McCarthy的论文完成的，原文在[这里][nakkaya]，此处只是把lisp.c这个文件抽取、整理。

### 编译
`gcc -o lisp lisp.c && ./lisp test.lisp`

### 代码

```c lisp.c
#include <stdio.h>
#include <stdlib.h>
#include <ctype.h>
#include <string.h>

//
// data structure
//

#define car(X)        (((cons_object *) (X))->car)
#define cdr(X)        (((cons_object *) (X))->cdr)

enum type {
    CONS, ATOM, FUNC, LAMBDA
};

typedef struct {
    enum type type;
} object;

typedef struct {
    enum type type;
    char *name;
} atom_object;

typedef struct {
    enum type type;
    object *car;
    object *cdr;
} cons_object;

typedef struct {
    enum type type;
    object* (*fn)(object*, object*);
} func_object;

typedef struct {
    enum type type;
    object* args;
    object* sexp;
} lambda_object;

object *tee, *nil;

object *eval(object *sexp, object *env);

//
// internal functions
//

static char *name(object *o) {
    if (o->type != ATOM)
        exit(1);
    return ((atom_object*) o)->name;
}

static object *atom(char *n) {
    atom_object *ptr = (atom_object *) malloc(sizeof(atom_object));
    ptr->type = ATOM;
    char *name;
    name = malloc(strlen(n) + 1);
    strcpy(name, n);
    ptr->name = name;
    return (object *) ptr;
}

static object *cons(object *first, object *second) {
    cons_object *ptr = (cons_object *) malloc(sizeof(cons_object));
    ptr->type = CONS;
    ptr->car = first;
    ptr->cdr = second;
    return (object *) ptr;
}

static object *func(object* (*fn)(object*, object*)) {
    func_object *ptr = (func_object *) malloc(sizeof(func_object));
    ptr->type = FUNC;
    ptr->fn = fn;
    return (object *) ptr;
}

static void append(object *list, object *obj) {
    object *ptr;
    for (ptr = list; cdr(ptr) != NULL; ptr = cdr(ptr))
        ;
    cdr(ptr) = cons(obj, NULL);
}

static object *lambda(object *args, object *sexp) {
    lambda_object *ptr = (lambda_object *) malloc(sizeof(lambda_object));
    ptr->type = LAMBDA;
    ptr->args = args;
    ptr->sexp = sexp;
    return (object *) ptr;
}

static object *interleave(object *c1, object *c2) {
    object *list = cons(cons(car(c1), cons(car(c2), NULL)), NULL);
    c1 = cdr(c1);
    c2 = cdr(c2);

    while (c1 != NULL && c1->type == CONS) {
        append(list, cons(car(c1), cons(car(c2), NULL)));
        c1 = cdr(c1);
        c2 = cdr(c2);
    }

    return list;
}

static object *replace_atom(object *sexp, object *with) {
    if (sexp->type == CONS) {
        object *list = cons(replace_atom(car(sexp), with), NULL);
        sexp = cdr(sexp);

        while (sexp != NULL && sexp->type == CONS) {
            append(list, replace_atom(car(sexp), with));
            sexp = cdr(sexp);
        }
        return list;
    } else {
        object* tmp = with;

        while (tmp != NULL && tmp->type == CONS) {
            object *item = car(tmp);
            object *atom = car(item);
            object *replacement = car(cdr(item));

            if (strcmp(name(atom), name(sexp)) == 0)
                return replacement;
            tmp = cdr(tmp);
        }
        return sexp;
    }
}

static object* lookup(char* n, object *env) {
    object *tmp = env;

    while (tmp != NULL && tmp->type == CONS) {
        object *item = car(tmp);
        object *nm = car(item);
        object *val = car(cdr(item));

        if (strcmp(name(nm), n) == 0)
            return val;
        tmp = cdr(tmp);
    }
    return NULL;
}

//
// buildin functions
//

object *fn_car(object *args, object *env) {
    return car(car(args));
}

object *fn_cdr(object *args, object *env) {
    return cdr(car(args));
}

object *fn_quote(object *args, object *env) {
    return car(args);
}

object *fn_cons(object *args, object *env) {
    object *list = cons(car(args), NULL);
    args = car(cdr(args));
    while (args != NULL && args->type == CONS) {
        append(list, car(args));
        args = cdr(args);
    }
    return list;
}

object *fn_equal(object *args, object *env) {
    object *first = car(args);
    object *second = car(cdr(args));
    if (strcmp(name(first), name(second)) == 0)
        return tee;
    else
        return nil;
}

object *fn_atom(object *args, object *env) {
    if (car(args)->type == ATOM)
        return tee;
    else
        return nil;
}

object *fn_cond(object *args, object *env) {
    while (args != NULL && args->type == CONS) {
        object *list = car(args);
        object *pred = nil;
        if (car(list) != nil)
            pred = eval(car(list), env);

        object *ret = car(cdr(list));

        if (pred != nil)
            return eval(ret, env);
        args = cdr(args);
    }
    return nil;
}

object *fn_lambda(object *args, object *env) {
    object *lambda = car(args);
    args = cdr(args);

    object *list = interleave((((lambda_object *) (lambda))->args), args);
    object* sexp = replace_atom((((lambda_object *) (lambda))->sexp), list);
    return eval(sexp, env);
}

object *fn_label(object *args, object *env) {
    append(env, cons(atom(name(car(args))), cons(car(cdr(args)), NULL)));
    return tee;
}

object* init_env() {
    object *env = cons(cons(atom("QUOTE"), cons(func(&fn_quote), NULL)), NULL);
    append(env, cons(atom("CAR"), cons(func(&fn_car), NULL)));
    append(env, cons(atom("CDR"), cons(func(&fn_cdr), NULL)));
    append(env, cons(atom("CONS"), cons(func(&fn_cons), NULL)));
    append(env, cons(atom("EQUAL"), cons(func(&fn_equal), NULL)));
    append(env, cons(atom("ATOM"), cons(func(&fn_atom), NULL)));
    append(env, cons(atom("COND"), cons(func(&fn_cond), NULL)));
    append(env, cons(atom("LAMBDA"), cons(func(&fn_lambda), NULL)));
    append(env, cons(atom("LABEL"), cons(func(&fn_label), NULL)));

    tee = atom("#T");
    nil = cons(NULL, NULL);

    return env;
}

//
// I/O
//

void print(object *sexp) {
    if (sexp == NULL)
        return;

    if (sexp->type == CONS) {
        printf("(");
        print(car(sexp));
        sexp = cdr(sexp);
        while (sexp != NULL && sexp->type == CONS) {
            printf(" ");
            print(car(sexp));
            sexp = cdr(sexp);
        }
        printf(")");
    } else if (sexp->type == ATOM) {
        printf("%s", name(sexp));
    } else if (sexp->type == LAMBDA) {
        printf("#");
        print((((lambda_object *) (sexp))->args));
        print((((lambda_object *) (sexp))->sexp));
    } else
        printf("Error.");
}

object *next_token(FILE *in) {
    int ch = getc(in);
    while (isspace(ch))
        ch = getc(in);
    if (ch == '\n')
        ch = getc(in);
    if (ch == EOF)
        exit(0);
    if (ch == ')')
        return atom(")");
    if (ch == '(')
        return atom("(");
    char buffer[128];
    int index = 0;
    while (!isspace(ch) && ch != ')') {
        buffer[index++] = ch;
        ch = getc(in);
    }
    buffer[index++] = '\0';
    if (ch == ')')
        ungetc(ch, in);

    return atom(buffer);
}

object *read_tail(FILE *in) {
    object *token = next_token(in);

    if (strcmp(name(token), ")") == 0)
        return NULL;
    else if (strcmp(name(token), "(") == 0) {
        object *first = read_tail(in);
        object *second = read_tail(in);
        return cons(first, second);
    } else {
        object *first = token;
        object *second = read_tail(in);
        return cons(first, second);
    }
}

object *read(FILE *in) {
    object *token = next_token(in);
    if (strcmp(name(token), "(") == 0)
        return read_tail(in);
    return token;
}

//
// REPL
//

object *eval_fn(object *sexp, object *env) {
    object *symbol = car(sexp);
    object *args = cdr(sexp);

    if (symbol->type == LAMBDA)
        return fn_lambda(sexp, env);
    else if (symbol->type == FUNC)
        return (((func_object *) (symbol))->fn)(args, env);
    else
        return sexp;
}

object *eval(object *sexp, object *env) {
    if (sexp == NULL)
        return nil;

    if (sexp->type == CONS) {
        if (car(sexp)->type == ATOM && strcmp(name(car(sexp)), "LAMBDA") == 0) {
            object* largs = car(cdr(sexp));
            object* lsexp = car(cdr(cdr(sexp)));
            return lambda(largs, lsexp);
        } else {
            object *accum = cons(eval(car(sexp), env), NULL);
            sexp = cdr(sexp);
            while (sexp != NULL && sexp->type == CONS) {
                append(accum, eval(car(sexp), env));
                sexp = cdr(sexp);
            }
            return eval_fn(accum, env);
        }
    } else {
        object *val = lookup(name(sexp), env);
        if (val == NULL)
            return sexp;
        else
            return val;
    }
}

//
// main
//

int main(int argc, char *argv[]) {
    object *env = init_env();
    FILE* in;

    if (argc > 1)
        in = fopen(argv[1], "r");
    else
        in = stdin;

    do {
        printf("> ");
        print(eval(read(in), env));
        printf("\n");
    } while (1);

    return 0;
}

```

测试文件，

```lisp test.lisp
(QUOTE A)
(QUOTE (A B C))
(CAR (QUOTE (A B C)))
(CDR (QUOTE (A B C)))
(CONS (QUOTE A) (QUOTE (B C)))
(EQUAL (CAR (QUOTE (A B))) (QUOTE A))
(EQUAL (CAR (CDR (QUOTE (A B)))) (QUOTE A))
(ATOM (QUOTE A))
(COND ((ATOM (QUOTE A)) (QUOTE B)) ((QUOTE T) (QUOTE C)))
((LAMBDA (X Y) (CONS (CAR X) Y)) (QUOTE (A B)) (CDR (QUOTE (C D))))
(LABEL FF (LAMBDA (X Y) (CONS (CAR X) Y)))
(FF (QUOTE (A B)) (CDR (QUOTE (C D))))
(LABEL XX (QUOTE (A B)))
(CAR XX)
```

### 论文中的Lisp实现
Lisp就像几何一样，整个大厦建立在几个公理基础之上，Lisp要求实现如下几个函数，
其他都可以由他们推演出来，

1. `quote`
2. `car`
3. `cdr`
4. `cons`
5. `equal`
6. `atom`
7. `cond`
8. An atom, regarded as a variable, may have a value
9. `lambda`
10. `label`，即给lambda函数命名，函数也是一个变量的值

John McCarthy在其论文中用Lisp实现了Lisp，下面是代码及自己的注释，整个就是一个`eval`函数而已，
`lambda`的实现很精彩啊！

```lisp
; e - the expression to be evaluted
; a - the associate table, every element is a (variable, value) pair

(label eval
       (lambda (e a)
               (cond
                ; e is an atom
                ((atom e)
                 (cond ((eq e nil) nil)
                       ((eq e T) T)
                       ; e is an atom otherwise nil or T,
                       ; so, find it in assoc table if exists
                       (T (cdr ((label
                                 assoc
                                 (lambda (e a)
                                         (cond ((null a) nil)
                                               ((eq e (caar a)) (car a))
                                               (T (assoc e (cdr a))))))
                                e a)))))
                
                ; e is not atom, so it must be list
                ; the first element of e is an atom
                ((atom (car e))
                 (cond ((eq (car e) (quote quote))   ; call quote
                        (cadr e))
                       ((eq (car e) (quote car))     ; call car
                        (car (eval (cadr e) a)))
                       ((eq (car e) (quote cdr))     ; call cdr
                        (cdr (eval (cadr e) a)))
                       ((eq (car e) (quote cadr))    ; call cadr
                        (cadr (eval (cadr e) a)))
                       ((eq (car e) (quote caddr))   ; call caddr
                        (caddr (eval (cadr e) a)))
                       ((eq (car e) (quote caar))    ; call caar
                        (caadr (eval (cadr e) a)))
                       ((eq (car e) (quote cadar))   ; call cadar
                        (cadar (eval (cadr e) a)))
                       ((eq (car e) (quote caddar))  ; call caddar
                        (caddar (eval (cadr e) a)))
                       ((eq (car e) (quote atom))    ; call atom
                        (atom (eval (cadr e) a)))
                       ((eq (car e) (quote null))    ; call null
                        (null (eval (cadr e) a)))
                       ((eq (car e) (quote cons))    ; call cons
                        (cons (eval (cadr e) a) (eval (caddr e) a)))
                       ((eq (car e) (quote eq))      ; call eq
                        (eq (eval (cadr e) a) (eval (caddr e) a)))
                       ((eq (car e) (quote cond))    ; call cond
                        ((label evcond
                                (lambda (v a) (cond ((eval (caar v) a)
                                                     (eval (cadar v) a))
                                                    (T (evcond (cdr v) a)))))
                         (cdr e) a))
                       (T (eval (cons (cdr ((label   ; call the 'value' of first element
                                             assoc
                                             (lambda (e a)
                                                     (cond ((null a) nil)
                                                           ((eq e (caar a)) (car a))
                                                           (T (assoc e (cdr a))))))
                                            (car e) a))
                                      (cdr e))
                                a))))
                ; the first element of e is not atom, so it must be a list,
                ; the first element of e is lambda expression
                ((eq (caar e) (quote lambda))
                 (eval (caddar e)        ; 取得lambda表达式的内容，下面准备参数，注意下面三个步骤的顺序，三层嵌套函数
                       ((label ffappend  ; 3.将参数的值对儿添加到assoc table中去
                               (lambda (u v)
                                       (cond ((null u) v)
                                             (T (cons (car u)
                                                      (ffappend (cdr u) v))))))
                        ((label pairup   ; 2.将参数与真实值绑定成对
                                (lambda (u v)
                                        (cond ((null u) nil)
                                              (T (cons (cons (car u) (car v))
                                                       (pairup (cdr u) (cdr v)))))))
                         (cadar e)
                         ((label evlis   ; 1.计算参数的值
                                 (lambda (u a)
                                         (cond ((null u) nil)
                                               (T (cons (eval (car u) a)
                                                        (evlis (cdr u) a))))))
                          (cdr e) a))
                        a)))
                ; the first element of e is a label expression
                ((eq (caar e) (quote label))
                 (eval (cons (caddar e) (cdr e)) ; construct the content of label expression(e.g. a lambda expression) and the arguments as final expression
                       (cons (cons (cadar e) (car e)) a)))))) ; add (label name, label content) pair to a(the assoc table)
```

`(eval expr assoc-table)`函数(label)分为四部分：

1. expr是一个atom
2. `(car expr)`是一个atom
3. `(car expr)`是一个lambda表达式
4. `(car expr)`是一个lable表达式(即函数定义)

### 疑问
你可能注意到函数参数的可见性问题了，是的，两个函数的形参如果同名的话，会不会产生干扰呢？
不会的。
assoc table，也就是env，是一个list，它是一个顺序链表，当执行函数`func1(u, v)`的时候，u, v
的值对儿加入assoc table，注意它们是被加到表格的head的，当它们执行完成，assoc table又恢复原状了，
再次执行`func2(u, v)`时，并没有func1的形参的干扰，assoc table其实就像一个stack，`cons`函数
可以保证这一点。

另外，向clojure的变量不可变性，更加保证了这一点。

(over)

[nakkaya]: https://github.com/nakkaya/nakkaya.com/blob/master/resources/posts/2010-08-24-a-micro-manual-for-lisp-implemented-in-c.org