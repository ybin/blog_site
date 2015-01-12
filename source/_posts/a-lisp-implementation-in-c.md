title: 一个简单的Lisp实现
date: 2015-01-12 10:08:00
categories: 编程
tags: [lisp, c]
description: lisp实现 lisp implementation c source code
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

(over)

[nakkaya]: https://github.com/nakkaya/nakkaya.com/blob/master/resources/posts/2010-08-24-a-micro-manual-for-lisp-implemented-in-c.org