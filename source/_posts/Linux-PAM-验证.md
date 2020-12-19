---
title: Linux PAM 验证
tags: PAM
categories: Linux
abbrlink: a0b678d2
date: 2017-12-06 02:33:51
copyright_author: Jitwxs
---

## 一、什么是PAM验证

Linux-PAM(linux可插入认证模块)是一套**共享库**，使本地系统管理员可以随意选择程序的认证方式。换句话说，不用重新编译一个包含PAM功能的应用程序，就可以改变它使用的认证机制。应用程序只需调用API就可方便的使用PAM提供的各种认证功能，而无需了解底层的实现。这种方式下,就算升级本地认证机制,也不用修改程序。

像我们使用su命令时，系统会提示你输入root用户的密码，这就是su命令通过调用PAM模块实现的。

## 二、PAM层次结构

PAM API是应用程序层与PAM服务模块之间联系的纽带，起着承上启下的作用。系统管理员通过PAM配置文件来制定不同应用程序的不同认证策略。

当应用程序调用PAM API时，应用接口层按照配置文件`pam.conf`的规定，加载相应的PAM服务模块。当PAM服务模块完成相应的认证操作之后，将结果返回给应用接口层。

![PAM层次结构](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206002942497.png)

## 三、PAM工作原理

1.用户调用某个应用程序，以得到某种服务

2.PAM应用程序调用后台的PAM库进行认证工作

3.PAM库到`/etc/pam.d/`目录查找有关程序配置来确定认证机制

4.PAM库装载所需的认证模块

5.上述装载的认证模块让PAM与应用程序中的会话函数进行通信

6.会话函数向用户要求有关信息

7.用户对这些要求做出回应，提供所需信息

8.PAM认证模块通过PAM库将认证信息提供给应用程序

9.认证完成后，应用程序做出两种选择：

- 认证成功：将所需权限赋予用户，并通知用户

- 认证失败：通知用户

![PAM工作原理](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206003847251.png)

---

## 四、PAM配置

### 4.1 PAM配置文件格式

PAM配置文件有两种写法，第一种是写在`/etc/pam.conf`文件中，但这种方法现在**已被废弃**，不再介绍。

第二种是在`/etc/pam.d`目录中，**使用应用程序名作为配置文件名：**

![pam.d](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206004553065.png)

配置格式形如：

```
auth required pam_stack.so service=system-auth
```

其中：

- auth 模块类型

- required 控制标记

- pam_stack.so 模块路径

- service=system-auth 模块参数

### 4.2 PAM的模块类型

Linux PAM有以下四种模块类型，分别代表四种不同的任务：

| 管理方式 | 说明|
|:-------------|:-------------|
| auth | 用来对用户的身份进行识别.如:提示用户输入密码,或判断用户是否为root等 |
| account  | 对帐号的各项属性进行检查.如:是否允许登录,是否达到最大用户数,或是root用户是否允许在这个终端登录等 |
| session | 这个模块用来定义用户登录前的,及用户退出后所要进行的操作.如:登录连接信息,用户数据的打开与关闭,挂载文件系统等 |
| password  | 使用用户信息来更新。如修改用户密码 |

### 4.3 PAM的控制标记

PAM使用控制标记来**处理和判断各个模块的返回值**：

| 控制标记 | 说明|
|:-------------|:-------------|
| required | 表示即使某个模块对用户的验证失败，也要等所有的模块都执行完毕后,PAM才返回错误信息。这样做是为了不让用户知道被哪个模块拒绝。如果对用户验证成功，所有的模块都会返回成功信息。 |
| requisite | 与required相似,但是如果这个模块返回失败，则立刻向应用程序返回失败，表示此类型失败。不再进行同类型后面的操作 |
| sufficient | 表示如果一个用户通过这个模块的验证，PAM结构就立刻返回验证成功信息（即使前面有模块fail了，也会把 fail结果忽略掉），把控制权交回应用程序。后面的层叠模块即使使用requisite或者required 控制标志，也不再执行。如果验证失败，sufficient 的作用和 optional 相同 |
| optional | 示即使本行指定的模块验证失败，也允许用户接受应用程序提供的服务，一般返回PAM_IGNORE(忽略) |

### 4.4 模块路径及模块参数

（1）模块路径

即要调用模块的位置。 对于64为Ubuntu，一般保存在`/lib/x86_64-linux-gnu/security`，如: `pam_unix.so`。

同一个模块，可以出现在不同的类型中。每个模块针对不同的模块类型，编制了不同的执行函数。

（2）模块参数

即传递给模块的参数.参数可以有多个,之间用**空格**分隔开。

（3）常用的PAM模块

![PAM模块](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206010523465.png)

---

## 五、PAM应用程序开发

### 5.1 预备知识

#### 5.1.1 头文件

```c
#include <security/pam_appl.h>
#include <security/pam_misc.h>
#include <security/pam_modules.h>
```

如果提示找不到头文件，需要先安装一下：

> sudo apt-get install libpam0g-dev

#### 5.1.2 struct pam_conv

这个结构体用于与PAM通信，因为其中比较复杂，我们只列出我们要用到的：

```c
static struct pam_conv conv = {
	misc_conv,
	NULL
};
```

#### 5.1.3 相关函数

以下函数具有相同的返回值，这里先说明下：

- **PAM_SUCCESS**: Successful completion

- **PAM_ABORT**: Genaral failure

- **PAM_SYSTEM_ERR**: System error

- **PAM_BUF_ERR**: Memort buffer error

```c

/*
 service_name: /etc/pam.d中注册的服务名
 user: 要验证的用户名
*/
int pam_start(const char *service_name, const char *user,
                     const struct pam_conv *pam_conversation,
                     pam_handle_t **pamh);

/*
 flags: 一般为0
*/
int pam_authenticate(pam_handle_t *pamh, int flags);

/*
 flags: 一般为0
*/
int pam_acct_mgmt(pam_handle_t *pamh, int flags);

/*
 pam_status：为验证函数的返回值，即pam_authenticate或pam_acct_mgmt的返回值
*/
int pam_end(pam_handle_t *pamh, int pam_status);

/*
 errnum: 错误值
*/
const char *pam_strerror(pam_handle_t *pamh, int errnum);
```

### 5.2 利用已有的PAM模块编写登陆验证

首先在`/etc/pam.d`中注册服务，服务名为`my_pam`，我们利用已有的模块`pam_unix.so`：

```
wxs@ubuntu:/etc/pam.d$ vim my_pam
wxs@ubuntu:/etc/pam.d$ cd /etc/pam.d/
wxs@ubuntu:/etc/pam.d$ sudo vim my_pam
wxs@ubuntu:/etc/pam.d$ 
wxs@ubuntu:/etc/pam.d$ cat my_pam 
auth required /lib/x86_64-linux-gnu/security/pam_unix.so
account required /lib/x86_64-linux-gnu/security/pam_unix.so
```

编写test.c文件来测试：

```c
#include <stdio.h>
#include <stdlib.h>
#include <security/pam_appl.h>
#include <security/pam_misc.h>
#include <security/pam_modules.h>

#define MAXSIZE 128
#define SERVICE_NAME "myPam"

int main(void) {
    int res;
	char user_name[MAXSIZE];
	pam_handle_t *handle;
	static struct pam_conv conv = {
		misc_conv,
		NULL
	};

	printf("UserName: ");
	scanf("%s", user_name);	

	/* 1.初始化PAM模块 */
	res = pam_start(SERVICE_NAME, user_name, &conv, &handle);
	if (res == PAM_SUCCESS)
		puts("pam service start!");
	else
		printf("%s\n",pam_strerror(handle, res));

	/* 2.PAM的auth类型验证接口 */
	res = pam_authenticate(handle, 0);
	if (res == PAM_SUCCESS)
		puts("passwd authenticate success!");
	else
		printf("%s\n",pam_strerror(handle, res));
	
	/* 3.PAM的account类型验证接口 */
	res = pam_acct_mgmt(handle, 0);
	if (res == PAM_SUCCESS)
		puts("account mgmt success!");
	else
		printf("%s\n",pam_strerror(handle, res));

	/* 4.PAM模块结束，释放资源 */
	res = pam_end(handle, res);
	if (res == PAM_SUCCESS)
		puts("pam service end!");
	else
		printf("%s\n",pam_strerror(handle, res));
    
	return 0;
}
```

生成可执行文件：

>gcc test.c -lpam -lpam_misc -ldl -o test

因为该程序要查询`/etc/passwd`和`/etc/shadow文件`，因此要么执行时加上sudo，要么`sudo chown root.root test`，不然可能会出现错误。运行结果如下：

![运行结果1](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206014820670.png)

### 5.3 自定义PAM模块编写登陆验证

引入头文件`pam_private.h`：

```c
/*
 * pam_private.h
 *
 * This is the Linux-PAM Library Private Header. It contains things
 * internal to the Linux-PAM library. Things not needed by either an
 * application or module.
 *
 * Please see end of file for copyright.
 *
 * Creator: Marc Ewing.
 * Maintained: CVS
 */

#ifndef _PAM_PRIVATE_H
#define _PAM_PRIVATE_H

//#include "config.h"

#include <syslog.h>

#include <security/pam_appl.h>
#include <security/pam_modules.h>
#include <security/pam_ext.h>

/* the Linux-PAM configuration file */

#define PAM_CONFIG    "/etc/pam.conf"
#define PAM_CONFIG_D  "/etc/pam.d"
#define PAM_CONFIG_DF "/etc/pam.d/%s"

#define PAM_DEFAULT_SERVICE        "other"     /* lower case */
#define PAM_DEFAULT_SERVICE_FILE   PAM_CONFIG_D "/" PAM_DEFAULT_SERVICE

#ifdef PAM_LOCKING
/*
 * the Linux-PAM lock file. If it exists Linux-PAM will abort. Use it
 * to block access to libpam
 */
#define PAM_LOCK_FILE "/var/lock/subsys/PAM"
#endif

/* components of the pam_handle structure */

#define _PAM_INVALID_RETVAL  -1    /* default value for cached_retval */

struct handler {
    int handler_type;
    int (*func)(pam_handle_t *pamh, int flags, int argc, char **argv);
    int actions[_PAM_RETURN_VALUES];
    /* set by authenticate, open_session, chauthtok(1st)
       consumed by setcred, close_session, chauthtok(2nd) */
    int cached_retval; int *cached_retval_p;
    int argc;
    char **argv;
    struct handler *next;
    char *mod_name;
    int stack_level;
};

#define PAM_HT_MODULE       0
#define PAM_HT_MUST_FAIL    1
#define PAM_HT_SUBSTACK     2
#define PAM_HT_SILENT_MODULE 3

struct loaded_module {
    char *name;
    int type; /* PAM_STATIC_MOD or PAM_DYNAMIC_MOD */
    void *dl_handle;
};

#define PAM_MT_DYNAMIC_MOD 0
#define PAM_MT_STATIC_MOD  1
#define PAM_MT_FAULTY_MOD 2

struct handlers {
    struct handler *authenticate;
    struct handler *setcred;
    struct handler *acct_mgmt;
    struct handler *open_session;
    struct handler *close_session;
    struct handler *chauthtok;
};

struct service {
    struct loaded_module *module; /* Array of modules */
    int modules_allocated;
    int modules_used;
    int handlers_loaded;

    struct handlers conf;        /* the configured handlers */
    struct handlers other;       /* the default handlers */
};

/*
 * Environment helper functions
 */

#define PAM_ENV_CHUNK         10 /* chunks of memory calloc()'d      *
				  * at once                          */

struct pam_environ {
    int entries;                 /* the number of pointers available */
    int requested;               /* the number of pointers used:     *
				  *     1 <= requested <= entries    */
    char **list;                 /* the environment storage (a list  *
				  * of pointers to malloc() memory)  */
};

#include <sys/time.h>

typedef enum { PAM_FALSE, PAM_TRUE } _pam_boolean;

struct _pam_fail_delay {
    _pam_boolean set;
    unsigned int delay;
    time_t begin;
    const void *delay_fn_ptr;
};

/* initial state in substack */
struct _pam_substack_state {
    int impression;
    int status;
};

struct _pam_former_state {
/* this is known and set by _pam_dispatch() */
    int choice;            /* which flavor of module function did we call? */

/* state info for the _pam_dispatch_aux() function */
    int depth;             /* how deep in the stack were we? */
    int impression;        /* the impression at that time */
    int status;            /* the status before returning incomplete */
    struct _pam_substack_state *substates; /* array of initial substack states */

/* state info used by pam_get_user() function */
    int fail_user;
    int want_user;
    char *prompt;          /* saved prompt information */

/* state info for the pam_chauthtok() function */
    _pam_boolean update;
};

struct pam_handle {
    char *authtok;
    unsigned caller_is;
    struct pam_conv *pam_conversation;
    char *oldauthtok;
    char *prompt;                /* for use by pam_get_user() */
    char *service_name;
    char *user;
    char *rhost;
    char *ruser;
    char *tty;
    char *xdisplay;
    char *authtok_type;          /* PAM_AUTHTOK_TYPE */
    struct pam_data *data;
    struct pam_environ *env;      /* structure to maintain environment list */
    struct _pam_fail_delay fail_delay;   /* helper function for easy delays */
    struct pam_xauth_data xauth;        /* auth info for X display */
    struct service handlers;
    struct _pam_former_state former;  /* library state - support for
					 event driven applications */
    const char *mod_name;	/* Name of the module currently executed */
    int mod_argc;               /* Number of module arguments */
    char **mod_argv;            /* module arguments */
    int choice;			/* Which function we call from the module */

#ifdef HAVE_LIBAUDIT
    int audit_state;             /* keep track of reported audit messages */
#endif
};

/* Values for select arg to _pam_dispatch() */
#define PAM_NOT_STACKED   0
#define PAM_AUTHENTICATE  1
#define PAM_SETCRED       2
#define PAM_ACCOUNT       3
#define PAM_OPEN_SESSION  4
#define PAM_CLOSE_SESSION 5
#define PAM_CHAUTHTOK     6

#define _PAM_ACTION_IS_JUMP(x)  ((x) > 0)
#define _PAM_ACTION_IGNORE      0
#define _PAM_ACTION_OK         -1
#define _PAM_ACTION_DONE       -2
#define _PAM_ACTION_BAD        -3
#define _PAM_ACTION_DIE        -4
#define _PAM_ACTION_RESET      -5
/* Add any new entries here.  Will need to change ..._UNDEF and then
 * need to change pam_tokens.h */
#define _PAM_ACTION_UNDEF      -6   /* this is treated as an error
				       ( = _PAM_ACTION_BAD) */

#define PAM_SUBSTACK_MAX_LEVEL 16   /* maximum level of substacks */

/* character tables for parsing config files */
extern const char * const _pam_token_actions[-_PAM_ACTION_UNDEF];
extern const char * const _pam_token_returns[_PAM_RETURN_VALUES+1];

/*
 * internally defined functions --- these should not be directly
 * called by applications or modules
 */
int _pam_dispatch(pam_handle_t *pamh, int flags, int choice);

/* Free various allocated structures and dlclose() the libs */
int _pam_free_handlers(pam_handle_t *pamh);

/* Parse config file, allocate handler structures, dlopen() */
int _pam_init_handlers(pam_handle_t *pamh);

/* Set all hander stuff to 0/NULL - called once from pam_start() */
void _pam_start_handlers(pam_handle_t *pamh);

/* environment helper functions */

/* create the environment structure */
int _pam_make_env(pam_handle_t *pamh);

/* delete the environment structure */
void _pam_drop_env(pam_handle_t *pamh);

/* these functions deal with failure delays as required by the
   authentication modules and application. Their *interface* is likely
   to remain the same although their function is hopefully going to
   improve */

/* reset the timer to no-delay */
void _pam_reset_timer(pam_handle_t *pamh);

/* this sets the clock ticking */
void _pam_start_timer(pam_handle_t *pamh);

/* this waits for the clock to stop ticking if status != PAM_SUCCESS */
void _pam_await_timer(pam_handle_t *pamh, int status);

typedef void (*voidfunc(void))(void);
typedef int (*servicefn)(pam_handle_t *, int, int, char **);

#ifdef PAM_STATIC
/* The next two in ../modules/_pam_static/pam_static.c */

/* Return pointer to data structure used to define a static module */
struct pam_module * _pam_open_static_handler (pam_handle_t *pamh,
					      const char *path);

/* Return pointer to function requested from static module */

voidfunc *_pam_get_static_sym(struct pam_module *mod, const char *symname);
#else
void *_pam_dlopen (const char *mod_path);
servicefn _pam_dlsym (void *handle, const char *symbol);
void _pam_dlclose (void *handle);
const char *_pam_dlerror (void);
#endif

/* For now we just use a stack and linear search for module data. */
/* If it becomes apparent that there is a lot of data, it should  */
/* changed to either a sorted list or a hash table.               */

struct pam_data {
     char *name;
     void *data;
     void (*cleanup)(pam_handle_t *pamh, void *data, int error_status);
     struct pam_data *next;
};

void _pam_free_data(pam_handle_t *pamh, int status);

char *_pam_StrTok(char *from, const char *format, char **next);

char *_pam_strdup(const char *s);

char *_pam_memdup(const char *s, int len);

int _pam_mkargv(char *s, char ***argv, int *argc);

void _pam_sanitize(pam_handle_t *pamh);

void _pam_set_default_control(int *control_array, int default_action);

void _pam_parse_control(int *control_array, char *tok);

#define _PAM_SYSTEM_LOG_PREFIX "PAM"

/*
 * XXX - Take care with this. It could confuse the logic of a trailing
 *       else
 */

#define IF_NO_PAMH(X,pamh,ERR)                    \
if ((pamh) == NULL) {                             \
    syslog(LOG_ERR, _PAM_SYSTEM_LOG_PREFIX " " X ": NULL pam handle passed"); \
    return ERR;                                   \
}

/*
 * include some helpful macros
 */

#include <security/_pam_macros.h>

/* used to work out where control currently resides (in an application
   or in a module) */

#define _PAM_CALLED_FROM_MODULE         1
#define _PAM_CALLED_FROM_APP            2

#define __PAM_FROM_MODULE(pamh)  ((pamh)->caller_is == _PAM_CALLED_FROM_MODULE)
#define __PAM_FROM_APP(pamh)     ((pamh)->caller_is == _PAM_CALLED_FROM_APP)
#define __PAM_TO_MODULE(pamh) \
        do { (pamh)->caller_is = _PAM_CALLED_FROM_MODULE; } while (0)
#define __PAM_TO_APP(pamh)    \
        do { (pamh)->caller_is = _PAM_CALLED_FROM_APP; } while (0)

#ifdef HAVE_LIBAUDIT
extern int _pam_auditlog(pam_handle_t *pamh, int action, int retval, int flags);
extern int _pam_audit_end(pam_handle_t *pamh, int pam_status);
#endif


#endif /* _PAM_PRIVATE_H_ */
```

编写验证模块`pam_test_auth.c`：

```c
#include "pam_private.h"
#include <string.h>
#include <stdlib.h>
#include <unistd.h>
#include <stdio.h>
#include <syslog.h>
#include <crypt.h>
#include <shadow.h>
#include <pwd.h>
#include <security/pam_modules.h>

#define MAXSIZE 128

PAM_EXTERN int pam_sm_authenticate(pam_handle_t *pamh, int flags, int argc, const char **argv ) {
	int i,j,count,lower_letter,upper_letter,digit;
	char *user_name, passwd[MAXSIZE], *my_crypt_passwd, salt[MAXSIZE];
	struct spwd *my_spwd;
	struct passwd *my_passwd;

	//get user name with pam_get_item function
	pam_get_item(pamh, PAM_USER, (const void **)&user_name);

	count = 0;
	do{
		// 输入超过三次退出
		if(count >= 3) {
			printf("[ERROR] Try over 3 times!\n");
			return PAM_ABORT;
		}

		// 输入密码
		printf("Passwd: ");
		scanf("%s", passwd);
		count++;

		// 打印用户名和密码
		printf("---Your Input---\n");
		printf("UserName: %s\n", user_name);
		printf("Password: %s\n", passwd);

		// 判断用户是否存在
		my_passwd = getpwnam(user_name);
		if (my_passwd == NULL) {
			printf("[ERROR] User not exist!\n");
			return PAM_ABORT;
		}

		// 密码长度不符合要求
		if (strlen(passwd) < 8) {
			printf("[ERROR] Your password length < 8\n");
			continue;
		}
		
		// 密码强度不符合要求
		lower_letter = upper_letter = digit = 0;
		for(i=0; i < strlen(passwd); i++) {
			if (passwd[i] >= 'a' && passwd[i] <= 'z')
				lower_letter++;
			else if (passwd[i] >= 'A' && passwd[i] <= 'Z')
				upper_letter++;
			else if (passwd[i] >= '0' && passwd[i] <= '9')
				digit++;
		}
		if (!lower_letter || !upper_letter || !digit) {
			printf("[ERROR] Your passwd must contains lower_letter,upper_letter and digit\n");
			continue;
		}

		//从/etc/shadow文件中取出密码信息
		my_spwd = getspnam(user_name);
		
		//得到加密salt
		for (i=0,j=0;i<strlen(my_spwd->sp_pwdp);i++) {
			if(my_spwd->sp_pwdp[i] == '$') {
				j++;
				if(j == 3)
					break;
			}
			salt[i] = my_spwd->sp_pwdp[i];
		}
		salt[i] = '\0';

		//计算出用户输入密码+salt后的密码
		my_crypt_passwd = crypt(passwd, salt);

		//比较密码是正确
		if(strcmp(my_spwd->sp_pwdp, my_crypt_passwd) == 0) {
			printf("Verify Success!\n");
			break;
		} else {
			printf("Verify Error!\n");
			return PAM_ABORT;
		}
	}while(1);
	
	//check the passwd
	
	return PAM_SUCCESS;
}

	PAM_EXTERN int
pam_sm_setcred (pam_handle_t *pamh, int flags ,int argc , const char **argv) {
	int retval;
	retval = PAM_SUCCESS;
	return retval;
}

#ifdef PAM_STATIC
struct pam_module _pam_unix_auth_modstruct = {
	"pam_unix_auth",
	pam_sm_authenticate,
	pam_sm_setcred,
	NULL,
	NULL,
	NULL,
	NULL
};
#endif
```

生成动态库：

>gcc -o pam_test_auth.so -shared -fPIC pam_test_auth.c -lpam -lcrypt

将其放到`/lib/x86_64-linux-gnu/security`中：

>sudo mv pam_test_auth.so /lib/x86_64-linux-gnu/security

因为这里我们只编写登陆验证，因此去掉账户有效性的验证，修改test.c代码如下：

```c
#include <stdio.h>
#include <stdlib.h>
#include <security/pam_appl.h>
#include <security/pam_misc.h>
#include <security/pam_modules.h>

#define MAXSIZE 128
#define SERVICE_NAME "my_pam"

int main(void) {
	int res;
	char user_name[MAXSIZE];
	pam_handle_t *handle;
	static struct pam_conv conv = {
		misc_conv,
		NULL
	};

	printf("UserName: ");
	scanf("%s", user_name);	

	res = pam_start(SERVICE_NAME, user_name, &conv, &handle);
	if (res == PAM_SUCCESS)
		puts("pam service start!");
	else
		printf("%s\n",pam_strerror(handle, res));

	res = pam_authenticate(handle, 0);
	if (res == PAM_SUCCESS)
		puts("passwd authenticate success!");
	else
		printf("%s\n",pam_strerror(handle, res));

	res = pam_end(handle, res);
	if (res == PAM_SUCCESS)
		puts("pam service end!");
	else
		printf("%s\n",pam_strerror(handle, res));

	return 0;
}
```

重新编译`test.c`：

>gcc test.c -lpam -lpam_misc -ldl -o test

别忘了`/etc/pam.d`中的服务配置信息还要修改为我们自己写的动态库：

```
wxs@ubuntu:~/my_pam$ sudo cat /etc/pam.d/my_pam 
auth required /lib/x86_64-linux-gnu/security/pam_test_auth.so
account required /lib/x86_64-linux-gnu/security/pam_test_auth.so
```

一切就绪，我们准备一个账号来测试：

```
用户名：mdzz
密码：mdzzMDZZ123
```

此时运行程序：

因为我们代码逻辑是先判断密码强度再验证正确（这个逻辑并不正确，只是为了演示使用），因此当密码强度不够时：

![运行结果2](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206021153072.png)

当尝试次数超过三次时：

![运行结果3](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206021630599.png)

正确输入时：

![运行结果4](https://cdn.jsdelivr.net/gh/jitwxs/cdn/blog/posts/20171206021729751.png)
