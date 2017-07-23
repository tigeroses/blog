---
title: Unix-Linux编程实践教程-chapter09-shell
date: 2016-07-31 23:11:11
tags: [Linux,C]
category: [programming]
---

## 第９章　可编程的shell,shell变量和环境：编写自己的shell

Unix shell 运行一种成为脚本的程序．一个shell脚本可以运行程序，接受
用户输入，使用变量和使用复杂的控制逻辑

if..then 语句依赖于下属惯例：Unix程序返回０以表示成功．shell使用
wait来得到程序的退出状态

shell编程语言包括变量．这些变量存储字符串，他们可以在任何命令中使用．shell
变量是脚本的局部变量

每个程序都从调用它的进程中继承一个字符串列表，这个列表被称为环境．
环境用来保存会话(session)的全局设置和某个程序的参数设置，shell允许
用户查看和修改环境

shell是有个编程语言解释器，这个解释器解释从键盘输入的命令，也解释
存储在脚本中的命令序列

shell包括两类变量：局部变量和环境变量
对变量的操作：
赋值    var=value
引用    $var
删除    unset var
输入    read var
列出变量    set
全局化  export var

## code

``` c
/*
 * smsh4.c small-shell version 4
 *
 * small shell that supports command line parsing
 * and if..then..else.fi logic(by calling process())
 */
#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include "smsh.h"

#define DFL_PROMPT ">"

int main()
{
    char * cmdline, *prompt, ** arglist;
    int result;
    void setup();

    prompt = DFL_PROMPT;
    setup();

    while ((cmdline = next_cmd(prompt, stdin)) != NULL)
    {
        if ((arglist = splitline(cmdline)) != NULL)
        {
            result = process(arglist);
            freelist(arglist);
        }
        free(cmdline);
    }
    return 0;
}

void setup()
/*
 * purpose: initialize shell
 * returns: nothing. calls fatal() if trouble
 */
{
    extern char ** environ;

    VLenviron2table(environ);
    signal(SIGINT, SIG_IGN);
    signal(SIGQUIT, SIG_IGN);
}

void fatal(char *s1, char *s2, int n)
{
    fprintf(stderr, "Error: %s, %s\n", s1, s2);
    exit(n);
}
```

``` c
// smsh.h

#define YES 1
#define NO 0

char *next_cmd();
char ** splitline(char *);
void freelist(char **);
void * emalloc(size_t);
void * erealloc(void *, size_t);
int execute(char **);
void fatal(char *, char *, int);
```

``` c

/*
 * builtin.c
 * contains the switch and the functions for builtin commands
 */

#include <stdio.h>
#include <string.h>
#include <ctype.h>
#include "smsh.h"
#include "varlib.h"

int assign(char *);
int okname(char *);

int builtin_command(char **args, int *resultp)
/*
 * purpose: run a builtin command
 * returns: 1 if args[0] is builtin, 0 if not
 * details: test args[0] against all known built-ins. Call functions
 */
{
    int rv = 0;

    if (strcmp(args[0], "set") == 0)
    {
        VLlist();
        *resultp = 0;
        rv = 1;
    }
    else if (strchr(args[0], '=') != NULL)
    {
        *resultp = assign(args[0]);
        if (*resultp != -1)
            rv = 1;
    }
    else if (strcmp(args[0], "export") == 0)
    {
        if (args[1] != NULL && okname(args[1]))
        {
            *resultp = VLexport(args[1]);
        }
        else
            *resultp = 1;
        rv = 1;
    }
    return rv;
}

int assign(char *str)
/*
 * purpose: execute name = val AND ensure that name is legal
 * returns: -1 for illegal lval, or result of VLstore
 * warning: modifies the string, but restores it to normal
 */
{
    char *cp;
    int rv;

    cp = strchr(str, '=');
    *cp = '\0';
    rv = (okname(str) ? VLstore(str, cp+1) : -1);
    *cp = '=';
    return rv;
}

int okname(char *str)
/*
 * purpose: determines if a string is a legal variable name
 * returns: 0 for no, 1 for yes
 */
{
    char *cp;

    for (cp = str; *cp; cp++)
    {
        if ((isdigit(*cp) && cp == str) || !(isalnum(*cp)) || *cp == '_')
            return 0;
    }

    return (cp != str);
}
```

``` c
/*
 * controlflow.c
 *
 * "if" processing is done with two state variables
 * if_state and if_result
 */
#include <stdio.h>
#include "smsh.h"

enum states {NEUTRAL, WANT_THEN, THEN_BLOCK};
enum results {SUCESS, FAIL};

static int if_state = NEUTRAL;
static int if_result = SUCESS;
static int last_stat = 0;

int syn_err(char *);

int ok_to_execute()
/*
 * purpose: determine the shell should execute a command
 * returns: 1 for yes, 0 for no
 * details: if in THEN_BLOCK and if_result was SUCESS then yes
 *          if in THEN_BLOCK and if_result was FAIL then no
 *          if in WANT_THEN then syntax error(sh is different)
 */
{
    int rv = 1;

    if (if_state == WANT_THEN)
    {
        syn_err("then expected");
        rv = 0;
    }
    else if (if_state == THEN_BLOCK && if_result == SUCESS)
        rv = 1;
    else if (if_state == THEN_BLOCK && if_result == FAIL)
        rv = 0;
    return rv;
}

int is_control_command(char *s)
/*
 * purpose: boolean to report if the command is a shell control command
 * returns: 0 or 1
 */
{
    return (strcmp(s, "if") == 0 || strcmp(s, "then") == 0 ||
            strcmp(s, "fi") == 0);
}

int do_control_command(char **args)
/*
 * purpose: Porcess "if", "then", "fi" - change state or detect error
 * returns: 0 if ok, -1 for syntax error
 */
{
    char * cmd = args[0];
    int rv = -1;

    if (strcmp(cmd, "if") == 0)
    {
        if (if_state != NEUTRAL)
            rv = syn_err("if unexpected");
        else
        {
            last_stat = process(args+1);
            if_result = (last_stat == 0 ? SUCESS: FAIL);
            if_state = WANT_THEN;
            rv = 0;
        }
    }
    else if (strcmp(cmd, "then") == 0)
    {
        if (if_state != WANT_THEN)
            rv = syn_err("then unexpected");
        else
        {
            if_state = THEN_BLOCK;
            rv = 0;
        }
    }
    else if (strcmp(cmd, "fi") == 0)
    {
        if (if_state != THEN_BLOCK)
            rv = syn_err("fi unexpected");
        else
        {
            if_state = NEUTRAL;
            rv = 0;
        }
    }
    else
        fatal("internal error processing:", cmd, 2);

    return rv;
}

int syn_err(char * msg)
/*
 * purpose: handles syntax errors in control structures
 * details: resets state to NEUTRAL
 * returns: -1 in interactive mode. Should call fatal in scripts
 */
{
    if_state = NEUTRAL;
    fprintf(stderr, "syntax error: %s\n", msg);
    return -1;
}
```

``` c
// execute.c - code used by small shell to execute commands

#include <stdio.h>
#include <stdlib.h>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>

int execute(char *argv[])
/*
 * purpose: run a program passing it arguments
 * returns: status returned via wait, or -1 on error
 * errors: -1 on fork() or wait() errors
 */
{
    int pid;
    int child_info = -1;
    extern char ** environ;

    if (argv[0] == NULL)
        return 0;

    if ((pid = fork()) == -1)
        perror("fork");
    else if (pid == 0)
    {
        environ = VLtable2environ();
        signal(SIGINT, SIG_DFL);
        signal(SIGQUIT, SIG_DFL);
        execvp(argv[0], argv);
        perror("cannot execute command");
        exit(1);
    }
    else
    {
        if (wait(&child_info) == -1)
            perror("wait");
    }
    return child_info;
}
```

``` c
/*
 * process.c
 * command processing layer
 *
 * The process(char ** arglist) function is called by the main loop
 * It sits in front of the execute() function. This layer handles
 * two main classes of processing.
 * a) built-in functions(e.g. exit(), set, =, read,..)
 * b) control structures(e.g. if, while, for)
 */

#include <stdio.h>
#include "smsh.h"

int is_control_command(char *);
int do_control_command(char **);
int ok_to_execute();

int process(char **args)
/*
 * purpose: process user command
 * returns: result of proceessing command
 * details: if a built-in then call arrprporiate function, if not
 *          execute()
 * errors: arise from subroutines, handled there
 */
{
    int rv = 0;

    if (args[0] == NULL)
        rv = 0;
    else if (is_control_command(args[0]))
        rv = do_control_command(args);
    else if (ok_to_execute())
    {
        if (! builtin_command(args, &rv))
            rv = execute(args);
    }
    return rv;
}
```

``` c
/*
 * splitline.c - command reading and parsing functions for smsh
 *
 * char * next_cmd(char *prompt, FILE *fp) - get next command
 * char ** splitline(char *str) - parse a string
 */
#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "smsh.h"

char * next_cmd(char * prompt, FILE * fp)
/*
 * purpose: read next command line from fp
 * returns: dynamically allocated string holding command line
 * errors: NULL at EOF (not really an error)
 *          calls fatal from emalloc()
 * notes: allocates space in BUFSIZ chunks
 */
{
    char *buf;      // the buffer
    int bufspace = 0;  // total size
    int pos = 0;        // current position
    int c;              // input char

    printf("%s", prompt);
    while ((c = getc(fp)) != EOF)
    {
        if (pos + 1 >= bufspace)
        {
            if (bufspace == 0)
                buf = emalloc(BUFSIZ);
            else
                buf = erealloc(buf, bufspace+BUFSIZ);
            bufspace += BUFSIZ;
        }

        if (c == '\n')
            break;

        buf[pos++] = c;
    }
    if (c == EOF && pos == 0)
        return NULL;
    buf[pos] = '\0';
    return buf;
}

/*
 * splitline (parse a line into an array of strings)
 */
char ** splitline(char * line)
/*
 * purpose: split a line into array of white - space separated tokens
 * returns: a NULL-terminated array of pointers to copies of the
 *          tokens or NULL if line if no tokens on the line
 * action: travers the array, locate strings, make copies
 * note: strtok() could work, but we may want to add quotes later
 */
{
    char * newstr();
    char **args;
    int spots = 0;      // spots in table
    int bufspace = 0;   // bytes in table
    int argnum = 0;     // slots used
    char *cp = line;    // pos in string
    char *start;
    int len;

    if (line == NULL)
        return NULL;

    args = emalloc(BUFSIZ);
    bufspace = BUFSIZ;
    spots = BUFSIZ / sizeof(char *);

    while (*cp != '\0')
    {
        while (isspace(*cp))       // skip leading spaces
            cp++;
        if (*cp == "\0")
            break;

        // make sure the array has room(+1 for NULL)
        if (argnum + 1 >= spots)
        {
            args = erealloc(args, bufspace+BUFSIZ);
            bufspace += BUFSIZ;
            spots += (BUFSIZ/sizeof(char *));
        }
        // mark start, then find end of word
        start = cp;
        len = 1;
        while (*++cp != '\0' && !(isspace(*cp)))
        {
            len++;
        }
        args[argnum++] = newstr(start, len);
    }

    args[argnum] = NULL;
    return args;
}

/*
 * purpose: constructor for strings
 * returns: a string, never NULL
 */
char * newstr(char *s, int l)
{
    char * rv = emalloc(l + 1);

    rv[l] = '\0';
    strncpy(rv, s, l);
    return rv;
}

void freelist(char ** list)
/*
 * purpose: free the list returned by splitline
 * returns: nothing
 * action: free all strings in list and then free the list
 */
{
    char ** cp = list;
    while (*cp)
        free(*cp++);
    free(list);
}

void * emalloc(size_t n)
{
    void * rv;
    if ((rv = malloc(n)) == NULL)
        fatal("out of memory", "", 1);
    return rv;
}

void * erealloc(void *p, size_t n)
{
    void *rv;
    if ((rv = realloc(p, n)) == NULL)
        fatal("realloc() failed", "", 1);
    return rv;
}
```

``` c
// varlib.h

int VLstore(char *name, char *val);
char * VLlookup(char * name);
int VLexport(char *name);
void VLlist();
int VLenviron2table(char *env[]);
char ** VLtable2environ();
```

``` c
/*
 * varlib.c
 *
 * a simple storage system to store name=value pairs
 * with facility to mark items as part of the environment
 *
 * interface:
 *  VLstore(name, value)    return 1 for ok, 0 for no
 *  VLlookup(name)          return string or NULL if not there
 *  VLlist()                prints out current table
 *
 * environment-related functions
 *  VLexport(name)          adds name to list of env vars
 *  VLtable2environ()       copy from table to environ
 *  VLenviron2table()       copy from environ to table
 *
 * details:
 * the table is stored as an array of structs that 
 * contain a flag for global and a single string of
 * the form name=value. This allows EZ addition to the
 * environment. It makes searching pretty easy, as
 * long as you search for "name="
 *
 */

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include "varlib.h"

#define MAXVARS 200         // a linked list would be nicer

struct var
{
    char *str;
    int global;
};

static struct var tab[MAXVARS];     // the table

static char *new_string(char *, char *);
static struct var * find_item(char *, int);

int VLstore(char *name, char *val)
/*
 * traverse list, if found, replace it, else add at end
 * since there is no delete, a blank on is a free one
 * return 1 if trouble, 0 if ok(like a command)
 */
{
    struct var * itemp;
    char *s;
    int rv = 1;

    // find spot to put it and make new string
    if ((itemp = find_item(name, 1)) != NULL &&
            (s = new_string(name, val)) != NULL)
    {
        if (itemp->str)
            free(itemp->str);
        itemp->str = s;
        rv = 0;
    }

    return rv;
}

static char * new_string(char *name, char *val)
/*
 * returns new string of form name=value or NULL on error
 */
{
    char * retval;

    retval = malloc(strlen(name) + strlen(val) + 2);
    if (retval != NULL)
        sprintf(retval, "%s = %s", name, val);
    return retval;
}

char * VLlookup(char * name)
/*
 * returns value of var or empty string if not there
 */
{
    struct var * itemp;

    if ((itemp = find_item(name, 0)) != NULL)
        return itemp->str + 1 + strlen(name);
    return "";
}

int VLexport(char *name)
/*
 * marks a var for export, adds it if not there
 * returns 1 for no, 0 for ok
 */
{
    struct var * itemp;
    int rv = 1;

    if ((itemp = find_item(name, 0)) != NULL)
    {
        itemp->global = 1;
        rv = 0;
    }
    else if (VLstore(name, "") == 1)
    {
        rv = VLexport(name);
    }

    return rv;
}

static struct var * find_item(char *name, int first_blank)
/*
 * searches table for an item
 * returns ptr to struct or NULL if not found
 * OR if (first_blank) then ptr to first blank one
 */
{
    int i;
    int len = strlen(name);
    char *s;

    for (i = 0; i < MAXVARS && tab[i].str != NULL; ++i)
    {
        s = tab[i].str;
        if (strncmp(s, name, len) == 0 && s[len] == '=')
        {
            // printf("%s %d\n", name, len);
            return &tab[i];
        }
    }
    if (i < MAXVARS && first_blank)
        return &tab[i];

    return NULL;
}

void VLlist()
/*
 * performs the shell's set command
 * Lists the contents of the variable table, marking each
 * exported variable with the symbol '*'
 */
{
    int i;
    for (i = 0; i < MAXVARS && tab[i].str != NULL; i++)
    {
        if (tab[i].global)
            printf(" * %s\n", tab[i].str);
        else
            printf(" %s\n", tab[i].str);
    }
}

int VLenviron2table(char *env[])
/*
 * initialize the variable table by loading array of strings
 * return 1 for ok, 0 for not ok
 */
{
    int i;
    char *newstring;

    for (i = 0; env[i] != NULL; i++)
    {
        if (i == MAXVARS)
            return 0;
        newstring = malloc(1+strlen(env[i]));
        if (newstring == NULL)
            return 0;
        strcpy(newstring, env[i]);
        tab[i].str = newstring;
        tab[i].global = 1;
    }
    while (i<MAXVARS)
    {
        tab[i].str = NULL;
        tab[i++].global = 0;
    }
    return 1;
}

char ** VLtable2environ()
/*
 * build an array of pointers suitable for making a new environment
 * note: you need to free() this when done to avoid memory leaks
 */
{
    int i,
        j,
        n = 0;
    char **envtab;

    // first, count the number of global variables
    for (i = 0; i < MAXVARS && tab[i].str != NULL; i++)
        if (tab[i].global == 1)
            n++;

    // then, allocate space for that many variables
    envtab = (char **)malloc((n+1) * sizeof(char *));
    if (envtab == NULL)
        return NULL;

    // then, load the array with pointers
    for (i = 0, j = 0; i < MAXVARS & tab[i].str != NULL; i++)
        if (tab[i].global == 1)
        {
            envtab[j++] = tab[i].str;
            // printf("%d %s\n",i, tab[i].str);
        }
    envtab[j] = NULL;

    return envtab;
}
```


