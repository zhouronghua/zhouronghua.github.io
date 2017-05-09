---
layout: post
title:  "Bash源码分析"
date:   2017-05-09 22:32:23 +0800
categories: InsideBashSourceCode
---
Bash源码分析
====================================

周荣华

作者简介：10年通讯底层研发经验，熟悉linux/vxworks等实时操作系统的内核原理和实现，在虚拟化的openstack，kubernetes，docker等领域也初有涉猎。

摘要：本文讲述当下留下的linux的bash的源代码，通过代码分析和单步调试解析bash的运行流程，适合喜欢研究linux原理的高级用户。分析的源代码来自gnu的开源项目https://git.savannah.gnu.org/git/bash.git，例子是作者自己编写，可以随意引用。



#1 引言
Bash这个程序作为一个linux的用户，用的实在太频繁了，但一般局限于会用就结束了，一直没机会研究bash本身的原理。因工作需要，调试一个bash的cpu冲高问题，趁此机会对bash的源码做了一些研究，希望能对大家有点帮助。

#2 linux的各种主流shell介绍
现在一般使用的shell有sh，bash和csh这几种，我们这里主要说的是bash，其他shell的源代码逻辑也差不多。

#3 bash使用到的主要数据结构介绍
##3.1 COMMAND

{% highlight c linenos %}
/* What a command looks like. */
typedef struct command {
  enum command_type type;	/* FOR CASE WHILE IF CONNECTION or SIMPLE. */
  int flags;			/* Flags controlling execution environment. */
  int line;			/* line number the command starts on */
  REDIRECT *redirects;		/* Special redirects for FOR CASE, etc. */
  union {
    struct for_com *For;
    struct case_com *Case;
    struct while_com *While;
    struct if_com *If;
    struct connection *Connection;
    struct simple_com *Simple;
    struct function_def *Function_def;
    struct group_com *Group;
#if defined (SELECT_COMMAND)
    struct select_com *Select;
#endif
#if defined (DPAREN_ARITHMETIC)
    struct arith_com *Arith;
#endif
#if defined (COND_COMMAND)
    struct cond_com *Cond;
#endif
#if defined (ARITH_FOR_COMMAND)
    struct arith_for_com *ArithFor;
#endif
    struct subshell_com *Subshell;
    struct coproc_com *Coproc;
  } value;
} COMMAND;
{% endhighlight %}

COMMAND是所有数据结构的纲，从这里可以看出一个bash实际能执行的语句有14种，分别位for，case，while，fi，connection，simple_com，function，group，select，arith，cond，arith_for，subshell，coproc，其中select，arith，cond，arith_for这4个命令需要打开对应的编译开关之后才能执行。
除了下面的这个union外，另外几个属性分别对应命令类型，行号和执行环境控制参数。其中控制参数有很多，每个控制参数占用一个bit位，包括是否启动子shell，是否忽略exit值等。

{% highlight c linenos %}
/* Possible values for command->flags. */
#define CMD_WANT_SUBSHELL  0x01	/* User wants a subshell: ( command ) */
#define CMD_FORCE_SUBSHELL 0x02	/* Shell needs to force a subshell. */
#define CMD_INVERT_RETURN  0x04	/* Invert the exit value. */
#define CMD_IGNORE_RETURN  0x08	/* Ignore the exit value.  For set -e. */
#define CMD_NO_FUNCTIONS   0x10 /* Ignore functions during command lookup. */
#define CMD_INHIBIT_EXPANSION 0x20 /* Do not expand the command words. */
#define CMD_NO_FORK	   0x40	/* Don't fork; just call execve */
#define CMD_TIME_PIPELINE  0x80 /* Time a pipeline */
#define CMD_TIME_POSIX	   0x100 /* time -p; use POSIX.2 time output spec. */
#define CMD_AMPERSAND	   0x200 /* command & */
#define CMD_STDIN_REDIR	   0x400 /* async command needs implicit </dev/null */
#define CMD_COMMAND_BUILTIN 0x0800 /* command executed by `command' builtin */
#define CMD_COPROC_SUBSHELL 0x1000
#define CMD_LASTPIPE	    0x2000
#define CMD_STDPATH	    0x4000	/* use standard path for command lookup */
{% endhighlight %}

这些flag可以在bash启动shell脚本时设置，或者在shell脚本内部调用set指令来设置，一般用户不怎么关注，高阶用户可以看看：
![Alt text](/assets/InsideBashSourceCode906.png "bash")

##3.2 FOR_COM
{% highlight c linenos %}
/* FOR command. */
typedef struct for_com {
  int flags;		/* See description of CMD flags. */
  int line;		/* line number the `for' keyword appears on */
  WORD_DESC *name;	/* The variable name to get mapped over. */
  WORD_LIST *map_list;	/* The things to map over.  This is never NULL. */
  COMMAND *action;	/* The action to execute.
			   During execution, NAME is bound to successive
			   members of MAP_LIST. */
} FOR_COM;
{% endhighlight %}
FOR_COM对应的shell语句是for name in map_list; do action; done
从结构体定义可以看出，除了和COMMAND相同的flags和行号外，for语句是有一个变量名，一个列表和一个递归的COMMAND组成的，实际for循环执行过程中也是将列表中的每个元素拿出来赋值给变量名，并执行action中的脚本段。
从这里的flags，可以看出，每条命令的flags是可以单独设置的，本条命令设置的控制参数可以不影响其他命令的控制参数。

##3.3 CASE_COM
{% highlight c linenos %}
/* The CASE command. */
typedef struct case_com {
  int flags;			/* See description of CMD flags. */
  int line;			/* line number the `case' keyword appears on */
  WORD_DESC *word;		/* The thing to test. */
  PATTERN_LIST *clauses;	/* The clauses to test against, or NULL. */
} CASE_COM;
{% endhighlight %}
对照下面的脚本，可以看出，先判断一个变量，变量判断晚走到复合语句clauses，注意clause最终实现的时候是一个单向链表，链表中每个元素由一个样式的列表和一个执行体action来组成。
{% highlight bash linenos %}
case word in
    pattern1|pattern2|pattern3 )
        action1 ;;
    pattern4 )
        action4 ;;
    * )
    	actionx;;
esac
{% endhighlight %}

##3.4 WHILE_COM
{% highlight c linenos %}
/* WHILE command. */
typedef struct while_com {
  int flags;			/* See description of CMD flags. */
  COMMAND *test;		/* Thing to test. */
  COMMAND *action;		/* Thing to do while test is non-zero. */
} WHILE_COM;

{% endhighlight %}
WHILE_COM比较简单，主体有两部分组成，判断条件和执行体。上篇文章调试过程中导致CPU冲高到100%的例子就是用的WHILE_COM，不过例子中的WHILE_COM的判断条件留空，相当于永远为true。
{% highlight bash linenos %}
while :; do i=$((i+1)); done
{% endhighlight %}

##3.5 IF_COM

{% highlight c linenos %}
/* IF command. */
typedef struct if_com {
  int flags;			/* See description of CMD flags. */
  COMMAND *test;		/* Thing to test. */
  COMMAND *true_case;		/* What to do if the test returned non-zero. */
  COMMAND *false_case;		/* What to do if the test returned zero. */
} IF_COM;
{% endhighlight %}
IF_COM由3段组成，条件判断，条件为true时的执行体和条件为false时的执行体，其中false情况下的执行体可以为空。
{% highlight bash linenos %}
if [ -z $1]; then
	echo "hello true"
else
	echo "hello false"
fi
{% endhighlight %}

从IF_COM的定义看，3部分都可以是复杂的COMMAND结构，所以嵌套起来也可以做的非常复杂，例如可以在test部分通过执行脚本，依靠脚本的返回值来判断是应该执行true_case还是false_case。

##3.6 CONNECTION
{% highlight c linenos %}
/* Structure used to represent the CONNECTION type. */
typedef struct connection {
  int ignore;			/* Unused; simplifies make_command (). */
  COMMAND *first;		/* Pointer to the first command. */
  COMMAND *second;		/* Pointer to the second command. */
  int connector;		/* What separates this command from others. */
} CONNECTION;
{% endhighlight %}
CONNECTION由4个属性组成：ignore字段对应其他命令结构种的flags，但对连接命令实际上没有用first对应第一条命令，second对应第二条命令，connector对应两条命令之间的连接符。难道只能两个命令一起用，不能多余两个命令一起调用？显然不是，一个CONNECTION对应一个连接符连接起来的两段命令，每段命令又可以是一个CONNECTION，这样就形成了级联的效果。
CONNECTION有3种：AND_AND对应“&&”，表示first执行返回结果为0的时候执行second；OR_OR对应“||”，表示first执行返回结果为非0的时候执行second；分号对应的connector还是分号，表示无论first执行结果是0还是非0，都执行second。是不是有点像C语言里面的&&，||和;？
{% highlight bash linenos %}
cd $dir && rm -fr *
cd $dir || rm -fr *
cd $dir ; rm -fr *
{% endhighlight %}

上面的三个例子别真的执行，后果很严重(:))。第一条表示删除$dir对应值的目录中的所有文件；第二条表示$dir不存在的时候删除当前目录下面的所有文件；第三条表示，如果$dir存在就删除$dir目录下的所有文件，如果不存在就删除当前目录下面的所有文件。

##3.7 SIMPLE_COM
{% highlight c linenos %}
/* The "simple" command.  Just a collection of words and redirects. */
typedef struct simple_com {
  int flags;			/* See description of CMD flags. */
  int line;			/* line number the command starts on */
  WORD_LIST *words;		/* The program name, the arguments,
				   variable assignments, etc. */
  REDIRECT *redirects;		/* Redirections to perform. */
} SIMPLE_COM;
{% endhighlight %}
SIMPLE_COM按字面意思就是简单命令，结构体由四部分组成，通用的flags，行号line，命令队列WORD_LIST，重定向队列REDIRECT。 
WORD_LIST队列比较容易理解。一堆命令的集合：
{% highlight c linenos %}
/* A linked list of words. */
typedef struct word_list {
  struct word_list *next;
  WORD_DESC *word;
} WORD_LIST;
{% endhighlight %}
其中单个命令还可以独立设置flags：
{% highlight c linenos %}
/* A structure which represents a word. */
typedef struct word_desc {
  char *word;		/* Zero terminated string. */
  int flags;		/* Flags associated with this word. */
} WORD_DESC;
{% endhighlight %}
REDIRECT就是重定向的意思，这里也有很多种重定向，结构体的各个属性的含义：next，组成重定向链表的指针；redirector，重定向的源；rflags，重定向时使用的私有flags；flags，打开重定向目标文件时的flags；instruction，重定向的实际功能指令，这个又有很多种，下面会详细描述；redirectee，重定向的目的文件描述符或者文件名；here_doc_eof，本地文件。

{% highlight c linenos %}
/* Structure describing a redirection.  If REDIRECTOR is negative, the parser
   (or translator in redir.c) encountered an out-of-range file descriptor. */
typedef struct redirect {
  struct redirect *next;	/* Next element, or NULL. */
  REDIRECTEE redirector;	/* Descriptor or varname to be redirected. */
  int rflags;			/* Private flags for this redirection */
  int flags;			/* Flag value for `open'. */
  enum r_instruction  instruction; /* What to do with the information. */
  REDIRECTEE redirectee;	/* File descriptor or filename */
  char *here_doc_eof;		/* The word that appeared in <<foo. */
} REDIRECT;
{% endhighlight %}
从REDIRECTEE的定义看，它既可以是一个文件描述符，例如0表示标准输入，1表示标准输出，2表示错误输出，也可以是一个文件名。
{% highlight c linenos %}
typedef union {
  int dest;			/* Place to redirect REDIRECTOR to, or ... */
  WORD_DESC *filename;		/* filename to redirect to. */
} REDIRECTEE;
{% endhighlight %}
Bash支持二十种不同的重定向，后面会根据bash的源代码来一一解释一下具体内容（bash源代码的注释对重定向的含义理解也有很多帮助）：
{% highlight c linenos %}
/* Instructions describing what kind of thing to do for a redirection. */
enum r_instruction {
  r_output_direction, r_input_direction, r_inputa_direction,
  r_appending_to, r_reading_until, r_reading_string,
  r_duplicating_input, r_duplicating_output, r_deblank_reading_until,
  r_close_this, r_err_and_out, r_input_output, r_output_force,
  r_duplicating_input_word, r_duplicating_output_word,
  r_move_input, r_move_output, r_move_input_word, r_move_output_word,
  r_append_err_and_out
};
{% endhighlight %}
先来五种输出的重定向：普通输出，强制输出，错误和标准输出，叠加输出，错误和标准叠加输出。统一说一下几个概念，标准输出就是2级stdout，错误输出就是3级sdterr，强制的意思是文件存在的情况下会被先清空，再增加，叠加输出的意思是原有内容后面再增加。

{% highlight c linenos %}

    case r_output_direction:		/* >foo */
    case r_output_force:		/* >| foo */
    case r_err_and_out:			/* &>filename */
      temp->flags = O_TRUNC | O_WRONLY | O_CREAT;
      break;

    case r_appending_to:		/* >>foo */
    case r_append_err_and_out:		/* &>> filename */
      temp->flags = O_APPEND | O_WRONLY | O_CREAT;
      break;

    case r_input_direction:		/* <foo */
    case r_inputa_direction:		/* foo & makes this. */
      temp->flags = O_RDONLY;
      break;

    case r_input_output:		/* <>foo */
      temp->flags = O_RDWR | O_CREAT;
      break;

    case r_deblank_reading_until: 	/* <<-foo */
    case r_reading_until:		/* << foo */
    case r_reading_string:		/* <<< foo */
    case r_close_this:			/* <&- */
    case r_duplicating_input:		/* 1<&2 */
    case r_duplicating_output:		/* 1>&2 */
      break;

    /* the parser doesn't pass these. */
    case r_move_input:			/* 1<&2- */
    case r_move_output:			/* 1>&2- */
    case r_move_input_word:		/* 1<&$foo- */
    case r_move_output_word:		/* 1>&$foo- */
      break;

    /* The way the lexer works we have to do this here. */
    case r_duplicating_input_word:	/* 1<&$foo */
    case r_duplicating_output_word:	/* 1>&$foo */
      w = dest_and_filename.filename;
      wlen = strlen (w->word) - 1;
      if (w->word[wlen] == '-')		/* Yuck */
{% endhighlight %}
再来九种输入和输出重定向：普通输入重定向，后台执行，输入和输出同时重定向，去掉空格的输入重定向，输入重定向，字符串作为输入，关闭重定向源（怎么还有这种应用场景？），复制输入，复制输出。
紧接着六种输入输出的重定向分别为输入剪切，输出剪切，字符指向的输入剪切和字符指向的输出剪切，字符指向的输入复制和字符指向的输出复制。

Bash的重定向真是博大精深！！

##3.8 FUNCTION_DEF

{% highlight c linenos %}
/* The "function definition" command. */
typedef struct function_def {
  int flags;			/* See description of CMD flags. */
  int line;			/* Line number the function def starts on. */
  WORD_DESC *name;		/* The name of the function. */
  COMMAND *command;		/* The parsed execution tree. */
  char *source_file;		/* file in which function was defined, if any */
} FUNCTION_DEF;
{% endhighlight %}
FUNCTION_DEF由五部分组成，通用flags，起始行号，函数名，解析之后的函数执行体，如果函数定义在文件中，最后会有文件名。函数的定义也可以有入参，入参的提取和文件执行时类似的，都是走$1,$2类似的形式获得的。

{% highlight bash linenos %}
function e {
	echo $1
}
{% endhighlight %}


##3.9 GROUP_COM

{% highlight c linenos %}
/* A command that is `grouped' allows pipes and redirections to affect all
   commands in the group. */
typedef struct group_com {
  int ignore;			/* See description of CMD flags. */
  COMMAND *command;
} GROUP_COM;
{% endhighlight %}
GROUP_COM是个什么鬼？通过分析group的处理函数，发现group原来就是多个命令组成的命令段，一般用{}包围起来，从group_command_nesting变量的变化看，group是支持多层嵌套的。
{% highlight c linenos %}

static void
print_group_command (group_command)
     GROUP_COM *group_command;
{
  group_command_nesting++;
  cprintf ("{ ");

  if (inside_function_def == 0)
    skip_this_indent++;
  else
    {
      /* This is a group command { ... } inside of a function
	 definition, and should be printed as a multiline group
	 command, using the current indentation. */
      cprintf ("\n");
      indentation += indentation_amount;
    }

  make_command_string_internal (group_command->command);
  PRINT_DEFERRED_HEREDOCS ("");

  if (inside_function_def)
    {
      cprintf ("\n");
      indentation -= indentation_amount;
      indent (indentation);
    }
  else
    {
      semicolon ();
      cprintf (" ");
    }

  cprintf ("}");

  group_command_nesting--;
}
{% endhighlight %}

##3.10 SELECT_COM

{% highlight c linenos %}
#if defined (SELECT_COMMAND)
/* KSH SELECT command. */
typedef struct select_com {
  int flags;		/* See description of CMD flags. */
  int line;		/* line number the `select' keyword appears on */
  WORD_DESC *name;	/* The variable name to get mapped over. */
  WORD_LIST *map_list;	/* The things to map over.  This is never NULL. */
  COMMAND *action;	/* The action to execute.
			   During execution, NAME is bound to the member of
			   MAP_LIST chosen by the user. */
} SELECT_COM;
#endif /* SELECT_COMMAND */
{% endhighlight %}
SELECT_COM并不是每个版本的bash都存在，可以通过在bash里面敲help来确定其是否存在。下面这个bash 4.2.46的版本中是打开了select的开关的：

![Alt text](/assets/InsideBashSourceCode3262.png "bash")

SELECT_COM的作用是为了生成一个简单的菜单，用户通过选择菜单来让系统执行对应的命令，常见的SELECT_COM是时区配置时使用的。

![Alt text](/assets/InsideBashSourceCode3334.png "bash")
具体分析/usr/bin/tzselect源码时发现，为了做到各个shell之间的兼容，这个脚本写的比想象中要复杂的多。
首先要是一下版本号的记录：

![Alt text](/assets/InsideBashSourceCode3411.png "bash")
跳过紧接着的注释，然后是版本兼容性判断，使用帮助：

![Alt text](/assets/InsideBashSourceCode3439.png "bash")
然后很无聊的定义了一个完全不用的变量IFS，但用来定义IFS的newline后面倒是用过。
为了规避bug，还要把PS3清空。

![Alt text](/assets/InsideBashSourceCode3505.png "bash")
终于进正题了，先选择大洲或者大洋：

![Alt text](/assets/InsideBashSourceCode3525.png "bash")
根据大洲或者大洋，通过awk汇总对应的国家列表：

![Alt text](/assets/InsideBashSourceCode3552.png "bash")
二级select，选择国家：

![Alt text](/assets/InsideBashSourceCode3570.png "bash")
再次祭出awk，通过国家汇总时区列表：

![Alt text](/assets/InsideBashSourceCode3592.png "bash")
第三重select，选择时区：

![Alt text](/assets/InsideBashSourceCode3610.png "bash")
计算好时区之后，出现第四重select，确认是否要修改：

![Alt text](/assets/InsideBashSourceCode3641.png "bash")
还要判断一下当前是cshell还是其他shell，指导用户将当前的时区改到shell的启动脚本里面去（老大你写了这么多代码，不能自动把这句加进去么？还是要手动加:)）。

![Alt text](/assets/InsideBashSourceCode3727.png "bash")
##3.11 ARITH_COM

{% highlight c linenos %}
#if defined (DPAREN_ARITHMETIC)
/* The arithmetic evaluation command, ((...)).  Just a set of flags and
   a WORD_LIST, of which the first element is the only one used, for the
   time being. */
typedef struct arith_com {
  int flags;
  int line;
  WORD_LIST *exp;
} ARITH_COM;
#endif /* DPAREN_ARITHMETIC */
{% endhighlight %}
ARITH_COM也是需要开关打开的，不过当前默认用的bash都是支持的，这个命令的意思是算术表达式，算术表达式要用(())包起来，要不然bash会不知道你想当做算术表达式使用，如果执行“echo 1+1”会怎么样？bash认为它是一个文本，直接将文本本身显示出来了。

![Alt text](/assets/InsideBashSourceCode3882.png "bash")
加上(())之后echo还是失败的：

![Alt text](/assets/InsideBashSourceCode3903.png "bash")
再加一个$之后终于正常了：

![Alt text](/assets/InsideBashSourceCode3919.png "bash")
实际操作的时候，发现[]包起来的算术表达式也能用，但不加$的时候不会报错，加了会触发求值：

![Alt text](/assets/InsideBashSourceCode3967.png "bash")
通过查看代码，发现，只有(())是算术表达式：


{% highlight c linenos %}

#if defined (DPAREN_ARITHMETIC)
static int
execute_arith_command (arith_command)
     ARITH_COM *arith_command;
{
  int expok, save_line_number, retval;
  intmax_t expresult;
  WORD_LIST *new;
  char *exp;

  expresult = 0;

  save_line_number = line_number;
  this_command_name = "((";	/* )) */
  line_number = arith_command->line;
  /* If we're in a function, update the line number information. */
  if (variable_context && interactive_shell)
    {
      line_number -= function_line_number;
      if (line_number < 0)
	line_number = 0;
    }      

{% endhighlight %}
而[]只是为了兼容posix.2d9的一种算术替换规则，也就是说$[1+1]是直接替换成了2，而(())还需要走到算术表达式求值过程（是不是有点饶）。

{% highlight c linenos %}

    /* Do POSIX.2d9-style arithmetic substitution.  This will probably go
       away in a future bash release. */
    case '[':
      /* Extract the contents of this arithmetic substitution. */
      t_index = zindex + 1;
      temp = extract_arithmetic_subst (string, &t_index);
      zindex = t_index;
      if (temp == 0)
	{
	  temp = savestring (string);
	  if (expanded_something)
	    *expanded_something = 0;
	  goto return0;
	}	  

       /* Do initial variable expansion. */
      temp1 = expand_arith_string (temp, Q_DOUBLE_QUOTES|Q_ARITH);

      goto arithsub;

{% endhighlight %}

##3.12 COND_COM
{% highlight c linenos %}
typedef struct cond_com {
  int flags;
  int line;
  int type;
  WORD_DESC *op;
  struct cond_com *left, *right;
} COND_COM;
{% endhighlight %}

COND_COM由六个属性组成，通用的flags和line。Type有6种，分别是与、或、一元、二元、最小单元、表达式。这样分类起始让人有点疑惑，其实所有表达式只有一元、二元、三元等等参数个数的区分，bash源代码为了yacc解析方便，把其中的与表达式、或表达式、不带任何表达式的变量或者常量和其他表达式区分开来识别。
{% highlight c linenos %}
/* The conditional command, [[...]].  This is a binary tree -- we slipped
   a recursive-descent parser into the YACC grammar to parse it. */
#define COND_AND	1
#define COND_OR		2
#define COND_UNARY	3
#define COND_BINARY	4
#define COND_TERM	5
#define COND_EXPR	6
{% endhighlight %}
条件命令使用的一元表达式有26个，区分大小写（真的很多，居然只用了一半，没有把26*2=52个字母全部用光，设计这个的老大还真是拼啊），下面简单过一下。
a/e判断文件是否存在；r/w/x判断文件是否可读或者可写或者可执行；o表示当前用户是否拥有该文件；G表示当前用户的组是否拥有该文件；N表示文件存在而且有新内容（从你上次读，文件被修改过）；f表示常规文件（设备文件返回false）。
{% highlight c linenos %}

int
unary_test (op, arg)
     char *op, *arg;
{
  intmax_t r;
  struct stat stat_buf;
  SHELL_VAR *v;
     
  switch (op[1])
    {
    case 'a':			/* file exists in the file system? */
    case 'e':
      return (sh_stat (arg, &stat_buf) == 0);

    case 'r':			/* file is readable? */
      return (sh_eaccess (arg, R_OK) == 0);

    case 'w':			/* File is writeable? */
      return (sh_eaccess (arg, W_OK) == 0);

    case 'x':			/* File is executable? */
      return (sh_eaccess (arg, X_OK) == 0);

    case 'O':			/* File is owned by you? */
      return (sh_stat (arg, &stat_buf) == 0 &&
	      (uid_t) current_user.euid == (uid_t) stat_buf.st_uid);

    case 'G':			/* File is owned by your group? */
      return (sh_stat (arg, &stat_buf) == 0 &&
	      (gid_t) current_user.egid == (gid_t) stat_buf.st_gid);

    case 'N':
      return (sh_stat (arg, &stat_buf) == 0 &&
	      stat_buf.st_atime <= stat_buf.st_mtime);

    case 'f':			/* File is a file? */
      if (sh_stat (arg, &stat_buf) < 0)
	return (FALSE);

      /* -f is true if the given file exists and is a regular file. */
#if defined (S_IFMT)
      return (S_ISREG (stat_buf.st_mode) || (stat_buf.st_mode & S_IFMT) == 0);
#else
      return (S_ISREG (stat_buf.st_mode));
#endif /* !S_IFMT */

    case 'd':			/* File is a directory? */
      return (sh_stat (arg, &stat_buf) == 0 && (S_ISDIR (stat_buf.st_mode)));

    case 's':			/* File has something in it? */
      return (sh_stat (arg, &stat_buf) == 0 && stat_buf.st_size > (off_t) 0);

    case 'S':			/* File is a socket? */
#if !defined (S_ISSOCK)
      return (FALSE);
#else
      return (sh_stat (arg, &stat_buf) == 0 && S_ISSOCK (stat_buf.st_mode));
#endif /* S_ISSOCK */

    case 'c':			/* File is character special? */
      return (sh_stat (arg, &stat_buf) == 0 && S_ISCHR (stat_buf.st_mode));

    case 'b':			/* File is block special? */
      return (sh_stat (arg, &stat_buf) == 0 && S_ISBLK (stat_buf.st_mode));

    case 'p':			/* File is a named pipe? */
#ifndef S_ISFIFO
      return (FALSE);
#else
      return (sh_stat (arg, &stat_buf) == 0 && S_ISFIFO (stat_buf.st_mode));
#endif /* S_ISFIFO */

    case 'L':			/* Same as -h  */
    case 'h':			/* File is a symbolic link? */
#if !defined (S_ISLNK) || !defined (HAVE_LSTAT)
      return (FALSE);
#else
      return ((arg[0] != '\0') &&
	      (lstat (arg, &stat_buf) == 0) && S_ISLNK (stat_buf.st_mode));
#endif /* S_IFLNK && HAVE_LSTAT */

    case 'u':			/* File is setuid? */
      return (sh_stat (arg, &stat_buf) == 0 && (stat_buf.st_mode & S_ISUID) != 0);

    case 'g':			/* File is setgid? */
      return (sh_stat (arg, &stat_buf) == 0 && (stat_buf.st_mode & S_ISGID) != 0);

    case 'k':			/* File has sticky bit set? */
#if !defined (S_ISVTX)
      /* This is not Posix, and is not defined on some Posix systems. */
      return (FALSE);
#else
      return (sh_stat (arg, &stat_buf) == 0 && (stat_buf.st_mode & S_ISVTX) != 0);
#endif

    case 't':	/* File fd is a terminal? */
      if (legal_number (arg, &r) == 0)
	return (FALSE);
      return ((r == (int)r) && isatty ((int)r));

    case 'n':			/* True if arg has some length. */
      return (arg[0] != '\0');

    case 'z':			/* True if arg has no length. */
      return (arg[0] == '\0');

    case 'o':			/* True if option `arg' is set. */
      return (minus_o_option_value (arg) == 1);

    case 'v':
      v = find_variable (arg);
#if defined (ARRAY_VARS)
      if (v == 0 && valid_array_reference (arg, 0))
	{
	  char *t;
	  t = array_value (arg, 0, 0, (int *)0, (arrayind_t *)0);
	  return (t ? TRUE : FALSE);
	}
     else if (v && invisible_p (v) == 0 && array_p (v))
	{
	  char *t;
	  /* [[ -v foo ]] == [[ -v foo[0] ]] */
	  t = array_reference (array_cell (v), 0);
	  return (t ? TRUE : FALSE);
	}
      else if (v && invisible_p (v) == 0 && assoc_p (v))
	{
	  char *t;
	  t = assoc_reference (assoc_cell (v), "0");
	  return (t ? TRUE : FALSE);
	}
#endif
      return (v && invisible_p (v) == 0 && var_isset (v) ? TRUE : FALSE);

    case 'R':
      v = find_variable_noref (arg);
      return ((v && invisible_p (v) == 0 && var_isset (v) && nameref_p (v)) ? TRUE : FALSE);
    }

  /* We can't actually get here, but this shuts up gcc. */
  return (FALSE);
}

{% endhighlight %}
d表示目录，s表示文件大小大于0，S表示socket，c表示是字符设备，b表示块设备，p表示有名管道，L/h表示符号链接：

u表示文件被设置了uid，g表示设置了组id，k表示sticky文件（对目录表示任何人都可以在该目录创建文件，但只能删除自己创建的文件，对可执行程序表示程序执行结束之后还会在内存待一阵子，方便下次再次执行的时候，不用从硬盘读到内存里面），t表示终端。
n表示脚本执行时存在至少一个参数，z表示没有带任何参数，o表示arg选项设置了，v表示变量存在。最后还有一个R，表示是否在命名表里面有索引。

二元表达式比一元表达式少了不少，总共
=，>=，<=，>，<，!=，这六个望文生义就知道什么意思了。
另外四个其他语言也经常会见到：-nt（newer than，表示文件修改时间更新），-ot（older than，表示文件修改时间更老），-lt（less than，更小），-gt（greater than，更大）。
{% highlight c linenos %}

static int
binary_operator ()
{
  int value;
  char *w;

  w = argv[pos + 1];
  if ((w[0] == '=' && (w[1] == '\0' || (w[1] == '=' && w[2] == '\0'))) || /* =, == */
      ((w[0] == '>' || w[0] == '<') && w[1] == '\0') ||		/* <, > */
      (w[0] == '!' && w[1] == '=' && w[2] == '\0'))		/* != */
    {
      value = binary_test (w, argv[pos], argv[pos + 2], 0);
      pos += 3;
      return (value);
    }

#if defined (PATTERN_MATCHING)
  if ((w[0] == '=' || w[0] == '!') && w[1] == '~' && w[2] == '\0')
    {
      value = patcomp (argv[pos], argv[pos + 2], w[0] == '=' ? EQ : NE);
      pos += 3;
      return (value);
    }
#endif

  if ((w[0] != '-' || w[3] != '\0') || test_binop (w) == 0)
    {
      test_syntax_error (_("%s: binary operator expected"), w);
      /* NOTREACHED */
      return (FALSE);
    }

  value = binary_test (w, argv[pos], argv[pos + 2], 0);
  pos += 3;
  return value;
}

{% endhighlight %}
还有五个：-ef（equal file，同一文件，不只是文件内容完全一样，而且需要文件指向的inode节点也完全一样），-eq（equal，算术相等），-ne（not equal，算术不相等），-ge（greater or equal，算术大于等于），-le（算术小于等于）。上面的算术计算，对字符串也试用。


##3.13 ARITH_FOR_COM
{% highlight c linenos %}
#if defined (ARITH_FOR_COMMAND)
typedef struct arith_for_com {
  int flags;
  int line;	/* generally used for error messages */
  WORD_LIST *init;
  WORD_LIST *test;
  WORD_LIST *step;
  COMMAND *action;
} ARITH_FOR_COM;
#endif
{% endhighlight %}
ARITH_FOR_COM有点像C语言里面的for循环，几个属性中有初始化，测试边界命令，步进命令和实际的执行体。除了算术运算要求的两层小括号和do，done关键字，完整代码和C语言里面的for是不是没啥明显差别？

{% highlight bash linenos %}

for ((i=0;i<3;i++))
do
	echo $i
done
{% endhighlight %}

##3.14 SUBSHELL_COM
{% highlight c linenos %}
typedef struct subshell_com {
  int flags;
  COMMAND *command;
} SUBSHELL_COM;
{% endhighlight %}
SUBSHELL_COM的结构体定义比较简单，就是一个命令属性，看来关键的复杂度还是在写shell脚本本身上，不过对bash本身而言，就是把脚本文件读进来，后面一行一行执行的时候，和其他普通命令没有差别。

##3.15 COPROC_COM

{% highlight c linenos %}
typedef struct coproc_com {
  int flags;
  char *name;
  COMMAND *command;
} COPROC_COM;
{% endhighlight %}
COPROC_COM的全称是coprocess，翻译成中文应该是协程的意思，不过shell里面的协程没有像高级语言那么复杂，对bash而言，执行结果和执行命令后面加一个&类似，不过可以制定协程的名字。

#4 bash源代码的目录结构
现在才说源代码的目录结构是不是晚了点。
前面分析数据结构的时候，基本上把每个命令的执行过程也简单过了一下，这样大家读代码的时候先有一个概貌，不至于一叶障目不见泰山。
用tree命令打印出来的目录结构如下，其中builtins目录里面是大多数内置命令（例如cd，pwd等）的实现，但没有看到ls命令，难道部分复杂命令还另外建了git库来实现？
Cross-build是交叉编译的设置。CWRU是修改记录，很多人在修改记录里面还留了自己的邮箱，有兴趣的读者是否可以和他们聊聊。Doc是文档目录，包括html格式的和pdf格式的文档。Examples目录是各种脚本的例子，不知道怎么写复杂bash的读者有福了。Include目录里面是一些实现bash过程中经查会用到的一些结构体或者宏的定义，一般都是写和bash不直接联系的，直接要用到的定义都在最顶级目录下面。
Lib是实现bash种要用到的各种公共库，这些库的实现本身和bash的解析没有直接关系，统一放在lib目录。
M4是GUN的一种编程语言，类似宏，m4目录下面是用m4语言写的timespec结构体的定义相关头文件校验和时间统计相关的头文件校验宏。

{% highlight bash linenos %}
├─builtins
├─cross-build
├─CWRU
│  └─misc
├─doc
├─examples
│  ├─complete
│  ├─functions
│  ├─loadables
│  │  └─perl
│  ├─misc
│  ├─scripts
│  └─startup-files
├─include
├─lib
│  ├─glob
│  │  └─doc
│  ├─intl
│  ├─malloc
│  ├─readline
│  │  ├─doc
│  │  └─examples
│  ├─sh
│  ├─termcap
│  └─tilde
├─m4
├─po
├─support
└─tests
    └─misc
{% endhighlight %}
Po又是GNU的一种编程语言，字面意思是可扩展组件，bash主要用它来实现多语种的扩展，每个语种都有自己的一个po文件，分别负责将代码里面的打印字符串转换成对应的语言。例如zh_CN.po里面定义了中文的各种字符串：

{% highlight po linenos %}

#: arrayfunc.c:54
msgid "bad array subscript"
msgstr "数组下标不正确"

#: arrayfunc.c:368 builtins/declare.def:574 variables.c:2092 variables.c:2118
#: variables.c:2730
#, c-format
msgid "%s: removing nameref attribute"
msgstr ""

#: arrayfunc.c:393 builtins/declare.def:780
#, c-format
msgid "%s: cannot convert indexed to associative array"
msgstr "%s: 无法将索引数组转化为关联数组"

{% endhighlight %}
Support目录负责bash的手册html页面的生成。
Tests是很多自测用例。
最重要的代码都放在最突出的位置，顶级目录下面有44个c文件，其中最重要的有四个：shell.c，脚本的解析；make_cmd.c，命令生成；execute_cmd.c，命令执行；copy_cmd.c命令拷贝。前面对语法的具体用法代码多数都来自execute_cmd.c文件。

{% highlight bash linenos %}

ABOUT-NLS     aclocal.m4   bash.PS             config-top.h   expr.c     lib             pcomplete.c  support
AUTHORS       alias.c      bash.SearchResults  config.h.in    externs.h  list.c          pcomplete.h  syntax.h
CHANGES       alias.h      bash.WK3            configure      findcmd.c  locale.c        pcomplib.c   test.c
COMPAT        array.c      bashansi.h          configure.ac   findcmd.h  m4              po           test.h
COPYING       array.h      bashhist.c          configure.in   flags.c    mailcheck.c     print_cmd.c  tests
CWRU          arrayfunc.c  bashhist.h          conftypes.h    flags.h    mailcheck.h     quit.h       trap.c
ChangeLog     arrayfunc.h  bashintl.h          copy_cmd.c     general.c  make_cmd.c      redir.c      trap.h
INSTALL       assoc.c      bashjmp.h           cross-build    general.h  make_cmd.h      redir.h      unwind_prot.c
MANIFEST      assoc.h      bashline.c          dispose_cmd.c  hashcmd.c  mksyntax.c      shell.c      unwind_prot.h
MANIFEST.doc  bash.IAB     bashline.h          dispose_cmd.h  hashcmd.h  nojobs.c        shell.h      variables.c
Makefile.in   bash.IAD     bashtypes.h         doc            hashlib.c  parse.y         sig.c        variables.h
NEWS          bash.IMB     bracecomp.c         error.c        hashlib.h  parser-built    sig.h        version.c
NOTES         bash.IMD     braces.c            error.h        include    parser.h        siglist.c    xmalloc.c
POSIX         bash.PFI     builtins            eval.c         input.c    patchlevel.h    siglist.h    xmalloc.h
RBASH         bash.PO      builtins.h          examples       input.h    pathexp.c       stringlib.c  y.tab.c
README        bash.PR      command.h           execute_cmd.c  jobs.c     pathexp.h       subst.c      y.tab.h
Y2K           bash.PRI     config-bot.h        execute_cmd.h  jobs.h     pathnames.h.in  subst.h

{% endhighlight %}
redir.c主要是各种重定向的定义和执行，前面讲到SIMPLE_COM的时候，曾经详细讲解过。
subst.c主要是对[]表达式的值替换。值得一提的是，这里对$开头的变量做了完整的说明。首先$0到$9分别对应脚本文件名，第一个入参，…，第九个入参。

{% highlight c linenos %}

/* Expand a single ${xxx} expansion.  The braces are optional.  When
   the braces are used, parameter_brace_expand() does the work,
   possibly calling param_expand recursively. */
static WORD_DESC *
param_expand (string, sindex, quoted, expanded_something,
	      contains_dollar_at, quoted_dollar_at_p, had_quoted_null_p,
	      pflags)
     char *string;
     int *sindex, quoted, *expanded_something, *contains_dollar_at;
     int *quoted_dollar_at_p, *had_quoted_null_p, pflags;
{
  char *temp, *temp1, uerror[3], *savecmd;
  int zindex, t_index, expok;
  unsigned char c;
  intmax_t number;
  SHELL_VAR *var;
  WORD_LIST *list;
  WORD_DESC *tdesc, *ret;
  int tflag;

/*itrace("param_expand: `%s' pflags = %d", string+*sindex, pflags);*/
  zindex = *sindex;
  c = string[++zindex];

  temp = (char *)NULL;
  ret = tdesc = (WORD_DESC *)NULL;
  tflag = 0;

  /* Do simple cases first. Switch on what follows '$'. */
  switch (c)
    {
    /* $0 .. $9? */
    case '0':
    case '1':
    case '2':
    case '3':
    case '4':
    case '5':
    case '6':
    case '7':
    case '8':
    case '9':
      temp1 = dollar_vars[TODIGIT (c)];
      if (unbound_vars_is_error && temp1 == (char *)NULL)
	{
	  uerror[0] = '$';
	  uerror[1] = c;
	  uerror[2] = '\0';
	  last_command_exit_value = EXECUTION_FAILURE;
	  err_unboundvar (uerror);
	  return (interactive_shell ? &expand_wdesc_error : &expand_wdesc_fatal);
	}
      if (temp1)
	temp = (*temp1 && (quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES)))
		  ? quote_string (temp1)
		  : quote_escapes (temp1);
      else
	temp = (char *)NULL;

      break;

    /* $$ -- pid of the invoking shell. */
    case '$':
      temp = itos (dollar_dollar_pid);
      break;

    /* $# -- number of positional parameters. */
    case '#':
      temp = itos (number_of_args ());
      break;

    /* $? -- return value of the last synchronous command. */
    case '?':
      temp = itos (last_command_exit_value);
      break;

    /* $- -- flags supplied to the shell on invocation or by `set'. */
    case '-':
      temp = which_set_flags ();
      break;

      /* $! -- Pid of the last asynchronous command. */
    case '!':
      /* If no asynchronous pids have been created, expand to nothing.
	 If `set -u' has been executed, and no async processes have
	 been created, this is an expansion error. */
      if (last_asynchronous_pid == NO_PID)
	{
	  if (expanded_something)
	    *expanded_something = 0;
	  temp = (char *)NULL;
	  if (unbound_vars_is_error)
	    {
	      uerror[0] = '$';
	      uerror[1] = c;
	      uerror[2] = '\0';
	      last_command_exit_value = EXECUTION_FAILURE;
	      err_unboundvar (uerror);
	      return (interactive_shell ? &expand_wdesc_error : &expand_wdesc_fatal);
	    }
	}
      else
	temp = itos (last_asynchronous_pid);
      break;

    /* The only difference between this and $@ is when the arg is quoted. */
    case '*':		/* `$*' */
      list = list_rest_of_args ();

#if 0
      /* According to austin-group posix proposal by Geoff Clare in
	 <20090505091501.GA10097@squonk.masqnet> of 5 May 2009:

 	"The shell shall write a message to standard error and
 	 immediately exit when it tries to expand an unset parameter
 	 other than the '@' and '*' special parameters."
      */

      if (list == 0 && unbound_vars_is_error && (pflags & PF_IGNUNBOUND) == 0)
	{
	  uerror[0] = '$';
	  uerror[1] = '*';
	  uerror[2] = '\0';
	  last_command_exit_value = EXECUTION_FAILURE;
	  err_unboundvar (uerror);
	  return (interactive_shell ? &expand_wdesc_error : &expand_wdesc_fatal);
	}
#endif

      /* If there are no command-line arguments, this should just
	 disappear if there are other characters in the expansion,
	 even if it's quoted. */
      if ((quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES)) && list == 0)
	temp = (char *)NULL;
      else if (quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES|Q_PATQUOTE))
	{
	  /* If we have "$*" we want to make a string of the positional
	     parameters, separated by the first character of $IFS, and
	     quote the whole string, including the separators.  If IFS
	     is unset, the parameters are separated by ' '; if $IFS is
	     null, the parameters are concatenated. */
	  temp = (quoted & (Q_DOUBLE_QUOTES|Q_PATQUOTE)) ? string_list_dollar_star (list) : string_list (list);
	  if (temp)
	    {
	      temp1 = (quoted & Q_DOUBLE_QUOTES) ? quote_string (temp) : temp;
	      if (*temp == 0)
		tflag |= W_HASQUOTEDNULL;
	      if (temp != temp1)
		free (temp);
	      temp = temp1;
	    }
	}
      else
	{
	  /* We check whether or not we're eventually going to split $* here,
	     for example when IFS is empty and we are processing the rhs of
	     an assignment statement.  In that case, we don't separate the
	     arguments at all.  Otherwise, if the $* is not quoted it is
	     identical to $@ */
#  if defined (HANDLE_MULTIBYTE)
	  if (expand_no_split_dollar_star && ifs_firstc[0] == 0)
#  else
	  if (expand_no_split_dollar_star && ifs_firstc == 0)
#  endif
	    temp = string_list_dollar_star (list);
	  else
	    {
	      temp = string_list_dollar_at (list, quoted, 0);
	      if (quoted == 0 && (ifs_is_set == 0 || ifs_is_null))
		tflag |= W_SPLITSPACE;
	      /* If we're not quoted but we still don't want word splitting, make
		 we quote the IFS characters to protect them from splitting (e.g.,
		 when $@ is in the string as well). */
	      else if (temp && quoted == 0 && ifs_is_set && (pflags & PF_ASSIGNRHS))
		{
		  temp1 = quote_string (temp);
		  free (temp);
		  temp = temp1;
		}
	    }

	  if (expand_no_split_dollar_star == 0 && contains_dollar_at)
	    *contains_dollar_at = 1;
	}

      dispose_words (list);
      break;

    /* When we have "$@" what we want is "$1" "$2" "$3" ... This
       means that we have to turn quoting off after we split into
       the individually quoted arguments so that the final split
       on the first character of $IFS is still done.  */
    case '@':		/* `$@' */
      list = list_rest_of_args ();

#if 0
      /* According to austin-group posix proposal by Geoff Clare in
	 <20090505091501.GA10097@squonk.masqnet> of 5 May 2009:

 	"The shell shall write a message to standard error and
 	 immediately exit when it tries to expand an unset parameter
 	 other than the '@' and '*' special parameters."
      */

      if (list == 0 && unbound_vars_is_error && (pflags & PF_IGNUNBOUND) == 0)
	{
	  uerror[0] = '$';
	  uerror[1] = '@';
	  uerror[2] = '\0';
	  last_command_exit_value = EXECUTION_FAILURE;
	  err_unboundvar (uerror);
	  return (interactive_shell ? &expand_wdesc_error : &expand_wdesc_fatal);
	}
#endif

      /* We want to flag the fact that we saw this.  We can't turn
	 off quoting entirely, because other characters in the
	 string might need it (consider "\"$@\""), but we need some
	 way to signal that the final split on the first character
	 of $IFS should be done, even though QUOTED is 1. */
      /* XXX - should this test include Q_PATQUOTE? */
      if (quoted_dollar_at_p && (quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES)))
	*quoted_dollar_at_p = 1;
      if (contains_dollar_at)
	*contains_dollar_at = 1;

      /* We want to separate the positional parameters with the first
	 character of $IFS in case $IFS is something other than a space.
	 We also want to make sure that splitting is done no matter what --
	 according to POSIX.2, this expands to a list of the positional
	 parameters no matter what IFS is set to. */
      /* XXX - what to do when in a context where word splitting is not
	 performed? Even when IFS is not the default, posix seems to imply
	 that we behave like unquoted $* ?  Maybe we should use PF_NOSPLIT2
	 here. */
      /* XXX - bash-4.4/bash-5.0 passing PFLAGS */
      temp = string_list_dollar_at (list, (pflags & PF_ASSIGNRHS) ? (quoted|Q_DOUBLE_QUOTES) : quoted, pflags);

      tflag |= W_DOLLARAT;
      dispose_words (list);
      break;

    case LBRACE:
      tdesc = parameter_brace_expand (string, &zindex, quoted, pflags,
				      quoted_dollar_at_p,
				      contains_dollar_at);

      if (tdesc == &expand_wdesc_error || tdesc == &expand_wdesc_fatal)
	return (tdesc);
      temp = tdesc ? tdesc->word : (char *)0;

      /* XXX */
      /* Quoted nulls should be removed if there is anything else
	 in the string. */
      /* Note that we saw the quoted null so we can add one back at
	 the end of this function if there are no other characters
	 in the string, discard TEMP, and go on.  The exception to
	 this is when we have "${@}" and $1 is '', since $@ needs
	 special handling. */
      if (tdesc && tdesc->word && (tdesc->flags & W_HASQUOTEDNULL) && QUOTED_NULL (temp))
	{
	  if (had_quoted_null_p)
	    *had_quoted_null_p = 1;
	  if (*quoted_dollar_at_p == 0)
	    {
	      free (temp);
	      tdesc->word = temp = (char *)NULL;
	    }
	    
	}

      ret = tdesc;
      goto return0;

    /* Do command or arithmetic substitution. */
    case LPAREN:
      /* We have to extract the contents of this paren substitution. */
      t_index = zindex + 1;
      /* XXX - might want to check for string[t_index+2] == LPAREN and parse
	 as arithmetic substitution immediately. */
      temp = extract_command_subst (string, &t_index, (pflags&PF_COMPLETE) ? SX_COMPLETE : 0);
      zindex = t_index;

      /* For Posix.2-style `$(( ))' arithmetic substitution,
	 extract the expression and pass it to the evaluator. */
      if (temp && *temp == LPAREN)
	{
	  char *temp2;
	  temp1 = temp + 1;
	  temp2 = savestring (temp1);
	  t_index = strlen (temp2) - 1;

	  if (temp2[t_index] != RPAREN)
	    {
	      free (temp2);
	      goto comsub;
	    }

	  /* Cut off ending `)' */
	  temp2[t_index] = '\0';

	  if (chk_arithsub (temp2, t_index) == 0)
	    {
	      free (temp2);
#if 0
	      internal_warning (_("future versions of the shell will force evaluation as an arithmetic substitution"));
#endif
	      goto comsub;
	    }

	  /* Expand variables found inside the expression. */
	  temp1 = expand_arith_string (temp2, Q_DOUBLE_QUOTES|Q_ARITH);
	  free (temp2);

arithsub:
	  /* No error messages. */
	  savecmd = this_command_name;
	  this_command_name = (char *)NULL;
	  number = evalexp (temp1, &expok);
	  this_command_name = savecmd;
	  free (temp);
	  free (temp1);
	  if (expok == 0)
	    {
	      if (interactive_shell == 0 && posixly_correct)
		{
		  last_command_exit_value = EXECUTION_FAILURE;
		  return (&expand_wdesc_fatal);
		}
	      else
		return (&expand_wdesc_error);
	    }
	  temp = itos (number);
	  break;
	}

comsub:
      if (pflags & PF_NOCOMSUB)
	/* we need zindex+1 because string[zindex] == RPAREN */
	temp1 = substring (string, *sindex, zindex+1);
      else
	{
	  tdesc = command_substitute (temp, quoted);
	  temp1 = tdesc ? tdesc->word : (char *)NULL;
	  if (tdesc)
	    dispose_word_desc (tdesc);
	}
      FREE (temp);
      temp = temp1;
      break;

    /* Do POSIX.2d9-style arithmetic substitution.  This will probably go
       away in a future bash release. */
    case '[':
      /* Extract the contents of this arithmetic substitution. */
      t_index = zindex + 1;
      temp = extract_arithmetic_subst (string, &t_index);
      zindex = t_index;
      if (temp == 0)
	{
	  temp = savestring (string);
	  if (expanded_something)
	    *expanded_something = 0;
	  goto return0;
	}	  

       /* Do initial variable expansion. */
      temp1 = expand_arith_string (temp, Q_DOUBLE_QUOTES|Q_ARITH);

      goto arithsub;

    default:
      /* Find the variable in VARIABLE_LIST. */
      temp = (char *)NULL;

      for (t_index = zindex; (c = string[zindex]) && legal_variable_char (c); zindex++)
	;
      temp1 = (zindex > t_index) ? substring (string, t_index, zindex) : (char *)NULL;

      /* If this isn't a variable name, then just output the `$'. */
      if (temp1 == 0 || *temp1 == '\0')
	{
	  FREE (temp1);
	  temp = (char *)xmalloc (2);
	  temp[0] = '$';
	  temp[1] = '\0';
	  if (expanded_something)
	    *expanded_something = 0;
	  goto return0;
	}

      /* If the variable exists, return its value cell. */
      var = find_variable (temp1);

      if (var && invisible_p (var) == 0 && var_isset (var))
	{
#if defined (ARRAY_VARS)
	  if (assoc_p (var) || array_p (var))
	    {
	      temp = array_p (var) ? array_reference (array_cell (var), 0)
				   : assoc_reference (assoc_cell (var), "0");
	      if (temp)
		temp = (*temp && (quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES)))
			  ? quote_string (temp)
			  : quote_escapes (temp);
	      else if (unbound_vars_is_error)
		goto unbound_variable;
	    }
	  else
#endif
	    {
	      temp = value_cell (var);

	      temp = (*temp && (quoted & (Q_HERE_DOCUMENT|Q_DOUBLE_QUOTES)))
			? quote_string (temp)
			: quote_escapes (temp);
	    }

	  free (temp1);

	  goto return0;
	}
      else if (var && (invisible_p (var) || var_isset (var) == 0))
	temp = (char *)NULL;
      else if ((var = find_variable_last_nameref (temp1, 0)) && var_isset (var) && invisible_p (var) == 0)
	{
	  temp = nameref_cell (var);
#if defined (ARRAY_VARS)
	  if (temp && *temp && valid_array_reference (temp, 0))
	    {
	      tdesc = parameter_brace_expand_word (temp, SPECIAL_VAR (temp, 0), quoted, pflags, (arrayind_t *)NULL);
	      if (tdesc == &expand_wdesc_error || tdesc == &expand_wdesc_fatal)
		return (tdesc);
	      ret = tdesc;
	      goto return0;
	    }
	  else
#endif
	  /* y=2 ; typeset -n x=y; echo $x is not the same as echo $2 in ksh */
	  if (temp && *temp && legal_identifier (temp) == 0)
	    {
	      last_command_exit_value = EXECUTION_FAILURE;
	      report_error (_("%s: invalid variable name for name reference"), temp);
	      return (&expand_wdesc_error);	/* XXX */
	    }
	  else
	    temp = (char *)NULL;
	}

      temp = (char *)NULL;

unbound_variable:
      if (unbound_vars_is_error)
	{
	  last_command_exit_value = EXECUTION_FAILURE;
	  err_unboundvar (temp1);
	}
      else
	{
	  free (temp1);
	  goto return0;
	}

      free (temp1);
      last_command_exit_value = EXECUTION_FAILURE;
      return ((unbound_vars_is_error && interactive_shell == 0)
		? &expand_wdesc_fatal
		: &expand_wdesc_error);
    }

  if (string[zindex])
    zindex++;

return0:
  *sindex = zindex;

  if (ret == 0)
    {
      ret = alloc_word_desc ();
      ret->flags = tflag;	/* XXX */
      ret->word = temp;
    }
  return ret;
}
{% endhighlight %}
$$执行shell的pid；$#执行脚本时传入的参数总数；$?上一次同步命令的执行结果，同步命令的意思是这个命令不执行完，就无法继续下面的执行，如果命令执行的过程中加了&，那$?无法得到其执行结果；$-脚本执行时的flags；$!，和$?相对，$!指的是上一个异步命令执行的结果。
$*和$@都是打印出剩余所有参数，按代码的说法，$*和$@的差别就是是否将带引号的参数去掉引号，实测结果打印出来的结果似乎是一样的。


实测效果：
![Alt text](/assets/InsideBashSourceCode6563.png "bash")




#5 bash脚本的执行过程分析
一个环境上多个sh的cpu占用达到99%，但实际通过ps看，这个sh并没有带任何参数，如果想要知道这个sh在干什么活，为何会一直冲高，还是gdb调试一下比较靠谱（还有一种可选的方法是不断的敲 cat /proc/*/stack 来反复查看堆栈，多敲敲之后总能抓到几次上下文，其中*换成对应进程的pid）。
通过下面的调试，可以看到当前执行是一个简单命令(cm_simple)，通过p *command-＞value-＞Simple-＞words-＞word 看到当前执行的简单命令在字符串是true。
{% highlight bash linenos %}
Breakpoint 1, execute_command (command=0x10946f0) at execute_cmd.c:386
386       result = execute_command_internal (command, 0, NO_PIPE, NO_PIPE, bitmap);
(gdb) p *command
$9 = {type = cm_simple, flags = 8, line = 0, redirects = 0x0, value = {For = 0x1094760, Case = 0x1094760, While = 0x1094760, 
    If = 0x1094760, Connection = 0x1094760, Simple = 0x1094760, Function_def = 0x1094760, Group = 0x1094760, Select = 0x1094760, 
    Arith = 0x1094760, Cond = 0x1094760, ArithFor = 0x1094760, Subshell = 0x1094760, Coproc = 0x1094760}}
(gdb) p *command-＞value-＞Simple 
$10 = {flags = 8, line = 3, words = 0x1094740, redirects = 0x0}
(gdb) p *command-＞value-＞Simple-＞words-＞word 
$11 = {word = 0x10946d0 "true", flags = 0}
(gdb) c
Continuing.
{% endhighlight %}
重新运行，可以看到又跑到了一个简单命令，其命令是"i=i+1"（汗）。
{% highlight bash linenos %}
Breakpoint 1, execute_command (command=0x10947b0) at execute_cmd.c:386
386       result = execute_command_internal (command, 0, NO_PIPE, NO_PIPE, bitmap);
(gdb) p *command
$12 = {type = cm_simple, flags = 0, line = 0, redirects = 0x0, value = {For = 0x1094ad0, Case = 0x1094ad0, While = 0x1094ad0, 
    If = 0x1094ad0, Connection = 0x1094ad0, Simple = 0x1094ad0, Function_def = 0x1094ad0, Group = 0x1094ad0, Select = 0x1094ad0, 
    Arith = 0x1094ad0, Cond = 0x1094ad0, ArithFor = 0x1094ad0, Subshell = 0x1094ad0, Coproc = 0x1094ad0}}
(gdb) p *command-＞value-＞Simple-＞words-＞word 
$13 = {word = 0x1094ab0 "i=i+1", flags = 20}
(gdb) c
Continuing.
{% endhighlight %}
通过shell的进程号，查询进程的上下文，发现是从另外一个虚机链接过来的ssh，咨询环境负责人，该虚机是跑测试用例的，之前跑的测试用例不知道为何没有正常停止，测试用例确实就是简单的一行命令：

{% highlight bash linenos %}
cat /proc/27538/environ 
XDG_SESSION_ID=24620SHELL=/bin/bashSSH_CLIENT=192.168.9.13 45494 22USER=rootPATH=/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/binMAIL=/var/mail/rootPWD=/rootHOME=/rootSHLVL=2LOGNAME=rootSSH_CONNECTION=192.168.9.13 45494 192.168.9.196 22XDG_RUNTIME_DIR=/run/user/0_=/bin/sh-bash-4.2# 

{% endhighlight %}

#6 结束语
Bash的源码解析到这里就结束了，更详细的内容，需要各位读者在实际使用时一一对应bash的源代码来得到更详细的解读，源码都来自于GNU的社区：https://git.savannah.gnu.org/git/bash.git，欢迎感兴趣的同事一起讨论分析。


