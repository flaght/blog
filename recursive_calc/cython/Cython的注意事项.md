## Cython的注意事项.md

遇到了以下几点需要特别注意的点：
> 1 .pyx中用cdef定义的东西，除类以外对.py都是不可见的

> 2 .py中是不能操作C类型的，如果想在.py中操作C类型就要在.pyx中从python object转成C类型或者用含有set/get方法的C类型包裹类；

> 3 虽然Cython能对Python的str和C的“char *”之间进行自动类型转换，但是对于“char a[n]”这种固定长度的字符串是无法自动转换的。需要使用Cython的libc.string.strcpy进行显式拷贝；

> 4 回调函数需要用函数包裹，再通过C的“void *”强制转换后才能传入C函数。

### 1. .pyx中用cdef定义的类型，除类以外对.py都不可见

``` python
#file: invisible.pyx

cdef inline cdef_function():
	print('cdef_function')
	
def def_function():
	print('def_function')
	
cdef int cdef_value

def_value = 999

cdef class cdef_class:
	def __init__(self):
		self.value = 1
		
class def_class:
	def __init__(self):
	self.value = 1
```

``` python
#file: test_visible.py
import invisible

if __name__ == '__main__':
    print('invisible.__dict__', invisible.__dict__)
```

```
$ python invisible.py
{
'__builtins__': <module '__builtin__' (built-in)>, 
'def_class': <class invisible.def_class at 0x10feed1f0>, 
'__file__': '/git/EasonCodeShare/cython_tutorials/invisible-for-py/invisible.so', 
'call_all_in_pyx': <built-in function call_all_in_pyx>, 
'__pyx_unpickle_cdef_class': <built-in function __pyx_unpickle_cdef_class>, 
'__package__': None, 
'__test__': {}, 
'cdef_class': <type 'invisible.cdef_class'>, 
'__name__': 'invisible', 
'def_value': 999, 
'def_function': <built-in function def_function>, 
'__doc__': None}

```
我们在.pyx用cdef定义的函数cdef_function、变量cdef_value都看不到了，只有类cdef_class能可见。所以，使用过程中要注意可见性问题，不要错误的在.py中尝试使用不可见的模块成员


### 2. .py传递C结构体类型
Cython扩展C的能力仅限于.pyx脚本中，.py脚本还是只能用纯Python。如果你在C中定义了一个结构，要从Python脚本中传进来就只能在.pyx手工转换一次，或者用包裹类传进来。

``` C
/*file: person_info.h */
typedef struct person_info_t
{
    int age;
    char *gender;
}person_info;

void print_person_info(char *name, person_info *info);

```

``` C
//file: person_info.c
#include <stdio.h>
#include "person_info.h"

void print_person_info(char *name, person_info *info)
{
    printf("name: %s, age: %d, gender: %s\n",
            name, info->age, info->gender);
}

```

``` cython
#file: cython_person_info.pyx
cdef extern from "person_info.h":
	struct person_info_t:
		int age
		char *gender
	ctypedef person_info_t person_info
	void print_person_info(char *name, person_info *info)
	
def cyprint_person_info(name, info):
    cdef person_info pinfo
    pinfo.age = info.age
    pinfo.gender = info.gender
    print_person_info(name, &pinfo)
```
因为“cyprint\_person_info”的参数只能是python object，所以我们要在函数中手工编码转换一下类型再调用C函数。


``` python
#file: test_person_info.py
from cython_person_info import cyprint_person_info

class person_info(object):
    age = None
    gender = None

if __name__ == '__main__':
    info = person_info()
    info.age = 18
    info.gender = 'male'
    
    cyprint_person_info('handsome', info)
```

```
$ python test_person_info.py
name: handsome, age: 18, gender: male
```
能正常调用到C函数。可是，这样存在一个问题，如果我们C的结构体字段很多，我们每次从.py脚本调用C函数都要手工编码转换一次类型数据就会很麻烦。还有更好的一个办法就是给C的结构体提供一个包裹类。

``` cython
#file: cython_person_info.pyx
from libc.stdlib cimport malloc, free
cdef extern from "person_info.h":
    struct person_info_t:
        int age
        char *gender
    ctypedef person_info_t person_info

    void print_person_info(char *name, person_info *info)
    
def cyprint_person_info(name, person_info_wrap info):
    print_person_info(name, info.ptr)
    
cdef class person_info_warp(object):
	cdef person_info *ptr
	
	def __init__(self):
		self.ptr = <person_info *>malloc(sizeof(person_info))
	
	def __del__(self):
        free(self.ptr)
    
    @property
    def age(self):
        return self.ptr.age
    @age.setter
    def age(self, value):
        self.ptr.age = value
    
    @property
    def gender(self):
        return self.ptr.gender
    @gender.setter
    def gender(self, value):
        self.ptr.gender = value
	
```

我们定义了一个“person\_info”结构体的包裹类“person\_info\_wrap”，并提供了成员set/get方法，这样就可以在.py中直接赋值了。减少了在.pyx中转换数据类型的步骤，能有效的提高性能。

``` python
#file: test_person_info.py
from cython_person_info import cyprint_person_info, person_info_wrap

if __name__ == '__main__':
    info_wrap = person_info_wrap()
    info_wrap.age = 88
    info_wrap.gender = 'mmmale'
    
    cyprint_person_info('hhhandsome', info_wrap)
```

```
$ python test_person_info.py 
name: hhhandsome, age: 88, gender: mmmale
```

### 3. python的str传递给C固定长度字符串要用strcpy
正如在C语言中，字符串之间不能直接赋值拷贝，而要使用strcpy复制一样，python的str和C字符串之间也要用cython封装的libc.string.strcpy函数来拷贝。

``` C
/*file: person_info.h */
typedef struct person_info_t
{
    int age;
    char gender[16];
}person_info;
```

``` pyx
#file: cython_person_info.pyx
cdef extern from "person_info.h":
    struct person_info_t:
        int age
        char gender[16]
    ctypedef person_info_t person_info
    
cdef class person_info_wrap(object):
    cdef person_info *ptr
    …… ……
    @property
    def gender(self):
        return self.ptr.gender
    @gender.setter
    def gender(self, value):
        strcpy(self.ptr.gender, value)
```

```
$ make
$ python test_person_info.py 
name: hhhandsome, age: 88, gender: mmmale
```

### 4. 用回调函数作为参数的C函数封装
C中的回调函数比较特殊，用户传入回调函数来定制化的处理数据。Cython官方提供了封装带有回调函数参数的[例子](https://github.com/cython/cython/tree/master/Demos/callback)：

``` C
//file: cheesefinder.h
typedef void (*cheesefunc)(char *name, void *user_data);
void find_cheeses(cheesefunc user_func, void *user_data);
```

``` C
//file: cheesefinder.c
#include "cheesefinder.h"

static char *cheeses[] = {
  "cheddar",
  "camembert",
  "that runny one",
  0
};

void find_cheeses(cheesefunc user_func, void *user_data) {
  char **p = cheeses;
  while (*p) {
    user_func(*p, user_data);
    ++p;
  }
}
```

``` pyx
#file: cheese.pyx
cdef extern from "cheesefinder.h":
    ctypedef void (*cheesefunc)(char *name, void *user_data)
    void find_cheeses(cheesefunc user_func, void *user_data)

def find(f):
    find_cheeses(callback, <void*>f)

cdef void callback(char *name, void *f):
    (<object>f)(name.decode('utf-8'))
```

``` python
import cheese

def report_cheese(name):
    print("Found cheese: " + name)

cheese.find(report_cheese)
```

关键的步骤就是在.pyx中定义一个和C的回调函数相同的回调包裹函数，如上的“cdef void callback(char *name, void *f)”。之后，将.py中的函数作为参数传递给包裹函数，并在包裹函数中转换成函数对象进行调用。