---
layout: post
title: "재미로 만들어보는 Task Local Object"
date: 2017-03-16 01:51:00 +0900
categories: python
tags: ['python', 'thread-local', 'task-local', 'weakref']
use_mathjax: false
comments: true
---

파이썬의 `threading` 라이브러리에서는 `local` 이라고불리는, 하나의 객체로 선언되지만 서로 다른 쓰레드에서 서로 다른 객체로서 접근할 수 있는 thread local 객체를 제공한다. 각각의 쓰레드마다 고유한 컨텍스트를 저장할 공간이 필요할 때 참 유용한 객체다.

``` python
import threading

# 1. 직접 threading.local() 호출하기
local_obj = threading.local()
local_obj.x = 42
print(local_obj.x)

# 2. threading.local을 상속하는 클래스 사용하기
class ThreadLocalObject(threading.local)
    pass

local_obj2 = ThreadLocalObject()
local_obj2.x = 42
print(local_obj2.x)
```

위의 `local_obj` 와 `local_obj2` 는 서로 다른 쓰레드에서 서로 다른 객체로 프록시된다. 이는 파이썬의 동적 특징을 교묘하게 이용한 것인데, 오늘은 우리가 한 번 직접 thread-local, 아니 task-local object를 만들어 보자. (thread local은 이미 만들어져 있으니까!)

근데 왜 갑자기 뜬금없이 task가 나왔지?  먼저 `Task` 가 뭔지 간단히 짚고 넘어가보자. 정확히는 파이썬 3.4부터 지원하는 `asyncio` 라이브러리의 `Task` 객체를 짚고 넘어가 보자. `Task` 는 쓰레드랑 비슷한데, 쉽게 말해 하나의 쓰레드가 처리하고 있는 여러개의 작업 중 하나를 의미한다. `asyncio` 의 `Task` 는 실제로는 하나의 코루틴을 실행하는 작업인데, 하나의 `event_loop` 에는 수많은 코루틴 `Task` 들이 돌아가게 된다. 아 그리고 하나의 `event_loop` 는 하나의 쓰레드에서 돌아가게 되고.

``` python
import asyncio

loop = asyncio.get_event_loop()

# hello_world()는 코루틴이다!
async def hello_world():
    await asyncio.sleep(1)
    print('hello, world!')

# 코루틴을 실행하는 Task를 생성한다
task = asyncio.ensure_future(hello_world(), loop=loop)

# Task를 event_loop에서 실행한다
loop.run_until_complete(task)
```

대충 감이 잡혔는가? 그렇다면 바로 task-local object를 만들어보도록 하자! 구현은 `threading.local` 를 참고하도록 하겠다.

먼저 우리가 만들고 싶은 클래스인 `TaskLocal` 은 서로 다른 `Task` 에 대해서 다른 attribute 값을 가지고 있어야 하므로 `__getattribute__` , `__setattr__` , `__delattr__` 세 개의 메소드를 오버라이딩 해야 한다. 일단 다음과 같이!

``` python
class TaskLocal(object):
    def __getattribute__(self, name):
        """Returns attribute of current task"""
        pass
    def __setattr__(self, name, value):
        """Set attribute of current task"""
        pass
    def __delattr__(self, name):
        """Delete attribute of current task"""
        pass
```

그럼 이제 현재 실행중인 `Task` 를 알 수 있는 방법이 필요한데, 다행히도 `Task.current_task` 라는 클래스 함수가 존재한다.

``` python
import asyncio

def current_task():
    return asyncio.Task.current_task()
```

이제 실제로 각 `Task` 마다 attribute들을 저장할 방법이 필요하다. 여기서 중요한 것은 `Task` 가 완료되고 GC되는 경우에는 해당 `Task` 의 attribute들을 다 지워줘야 한다는 것이다. 그렇지 않다면 메모리에 계속 task-local 데이터가 쌓이게 되고 결국엔 메모리가 부족해져 프로그램이 더이상 실행되지 못할 것이다. 가장 간단한 방법은 `Task` 객체에 직접 attribute 데이터를 설정하는 것이다.

``` python
    import asyncio
    
    def my_ensure_future(coroutine):
        task = asyncio.ensure_future(coroutine)
        task.__my_data = {}
```

음 확실히 이러면 task가 삭제될 경우 `__my_data` 도 같이 GC될 것이다. 하지만 이렇게 남이 구현해놓은 객체에 함부로 손을 대서는 아주 잘못될 수가 있다. 우리는 `Task` 객체랑 별도로 데이터를 관리하는게 좋다. 여기서 좋은 방법이 하나 있는데, weak reference를 사용하면 `Task` 객체를 조작하지 않고도 `Task` 가완료되면 task-local 데이터랑 바이바이를 할 수 있다!

``` python
import asyncio
import weakref

task = asyncio.ensure_future(asyncio.sleep(1))
dic = weakref.WeakKeyDictionary()
dic[task] = 42  # task가 GC되면 dic[task]도 사라진다
```

자 이제 재료들이 모였으니 조리를 시작해보자. 먼저 task local 객체를 최대한 pure하게 만들기 위해 실제 task-local attribute 데이터는 별도의 `TaskLocalImpl` 이라는 클래스를 두어 수행하자.

``` python
import weakref

class TaskLocalImpl(object):
    def __init__(self):
        self.data = weakref.WeakKeyDictionary()

    def get(self, task):
        """task의 attribute map data를 가져온다."""
        if task not in self.data:
            self.data[task] = {}
        return self.data[task]
```

그리고 `TaskLocal` 객체는 `Impl` 을 이용하여 요렇게 구현한다

``` python
class TaskLocal(object):
    def __init__(self):
        self.__impl = TaskLocalImpl()
    def __getattribute__(self, name):
        return self.__impl.get(asyncio.Task.current_task()).get(name)
    def __setattr__(self, name, value):
        return self.__impl.get(asyncio.Task.current_task()).put(name, value)
    def __delattr__(self, name):
        self.__impl.get(asyncio.Task.current_task()).pop(name)
```

어어.. 이렇게하면 안된다. 우리는 감히 `__getattribute__` 등의 특별한 함수들을 수정중이기 때문에, `self.__impl` 과 같은 무엄한 접근은 infinite recursion을 초래할 뿐이다. 직접 super class의 함수들을 호출해줘야 한다. `TaskLocal` 은 `object` 를 상속하므로 `object` 의 함수들을 호출하면 된다.

``` python
class TaskLocal(object):
    def __init__(self):
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl())
    def __getattribute__(self, name):
        return object.__getattribute__(self, '_TaskLocal__impl').get(asyncio.Task.current_task()).get(name)
    def __setattr__(self, name, value):
        return object.__getattribute__(self, '_TaskLocal__impl').get(asyncio.Task.current_task()).put(name, value)
    def __delattr__(self, name):
        object.__getattribute__(self, '_TaskLocal__impl').get(asyncio.Task.current_task()).pop(name)
```

한편 각 메소드에서 직접 dictionary를 get하기보다는, `TaskLocal.__dict__` 를 매 호출시마다 task local data로 덮어씌워주는편이 자연스럽다. 비록 attribute 접근 메소드들을 전부 재구현하지만 실제로 `TaskLocal` 객체가 해당 attribute들을 갖게 되는거니까.

``` python
import asyncio

def patch(local):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    attrs = impl.get(asyncio.Task.current_task())
    object.__setattr__(local, '__dict__', attrs)

class TaskLocal(object):
    def __init__(self):
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl())
    def __getattribute__(self, name):
        patch(self)
        return object.__getattribute__(self, name)
    def __setattr__(self, name, value):
        patch(self)
        return object.__setattr__(self, name, value)
    def __delattr__(self, name):
        patch(self)
        return object.__delattr__(self, name)
```

엇 여기 문제가 하나 있다. 일반적으로 `__getattribute__` , `__setattr__` , `__delattr__` 메소드들은 atomic하지만 이렇게 오버라이딩을 하는 경우에는 atomicity가 보장되지 않는다. 락 등의 synchronization 메커니즘이 필요한데, `threading` 에서 제공하는 `Lock` 을 사용해보도록 하자.

`threading.Lock` 은 `acquire()` 와 `release()` 를 호출해줘야 하는데 파이썬의 with 구문을 사용하면 좀더 우아하게 함수를 구현할 수 있다.

``` python
import threading

thread_lock = threading.Lock()
with thread_lock:
    # do something
    pass
```

이 `Lock` 을 `ThreadLocalImpl` 에 넣어두고 사용하도록 하자.
참고로 `threading.local` 은 `Lock` 이 아니라 같은 쓰레드에서는 락 내부로 진입할 수 있는 `threading.RLock` (Reenterable Lock) 을 사용했는데, 이는 thread local은 같은 쓰레드간에는 락이 필요 없기 때문이다. 우리는 같은 쓰레드라도 서로 다른 `Task` 를 수행중일 수 있기 때문에 `Lock` 을 사용해야 한다.

``` python
import weakref
import threading

class TaskLocalImpl(object):
    def __init__(self):
        self.data = weakref.WeakKeyDictionary()
        self.lock = threading.Lock()
```

한편 `contextlib` 라이브러리에서 제공하는 `contextmanager` 데코레이션은 with 구문을 쉽게 활용할 수 있는 유틸리티이다. 다음과 같이 `patch` 함수를 변경하면

``` python
import asyncio
import contextlib

@contextlib.contextmanager
def patch(local):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    attrs = impl.get(asyncio.Task.current_task())
    with impl.lock:
        object.__setattr__(local, '__dict__', attrs)
        yield
```

그리고 다음과 같이 `TaskLocal` 클래스를 변경하면 `impl.lock` 의 with 구문을 `patch(local)` 으로 사용할 수 있다.

``` python
class TaskLocal(object):
    def __init__(self):
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl())
    def __getattribute__(self, name):
        with patch(self):
            return object.__getattribute__(self, name)
    def __setattr__(self, name, value):
        with patch(self):
            return object.__setattr__(self, name, value)
    def __delattr__(self, name):
        with patch(self):
            return object.__delattr__(self, name)
```

음 아직도 버그가 있다. `_TaskLocal__impl` 이 `__dict__` 에 저장되기 때문에 `local.__dict__` 를 새 dictionary로 set할 경우 `_TaskLocal__impl` 이 날아가버린다. 여러 방법이 있지만, `threading.local` 에서 사용하는 꽤 괜찮은 방법은 `__slots__` 를 이용하는 것이다. `__slots__` 에 등록된 변수명은 `__dict__` 에 포함되지 않는다. 또한 `__slots__` 필드가 있는 클래스는 `__dict__` 를 자동으로 만들지 않는데, 이는 `__slots__` 에 `'``__dict__'` 를 포함시킴으로서 해결할 수 있다.

``` python
class TaskLocal(object):
    __slots__ = ['_TaskLocal__impl', '__dict__']
    ...
```

지금까지 꽤 많은 것을 구현했는데 잠깐 쉬는 시간을 갖자. 화장실이 급한 사람은 다녀와도 좋다.

# Part 2

이제 다듬기를 할 차례이다. 지금까지 완성된 코드를 보면 다음과 같다.

``` python
import asyncio
import contextlib
import threading
import weakref

class TaskLocalImpl(object):
    def __init__(self):
        self.data = weakref.WeakKeyDictionary()
        self.lock = threading.Lock()

    def get(self, task):
        """task의 attribute map data를 가져온다."""
        if task not in self.data:
            self.data[task] = {}
        return self.data[task]

@contextlib.contextmanager
def patch(local):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    attrs = impl.get(asyncio.Task.current_task())
    with impl.lock:
        object.__setattr__(local, '__dict__', attrs)
        yield
        
class TaskLocal(object):
    __slots__ = ['_TaskLocal__impl', '__dict__']

    def __init__(self):
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl())
    def __getattribute__(self, name):
        with patch(self):
            return object.__getattribute__(self, name)
    def __setattr__(self, name, value):
        with patch(self):
            return object.__setattr__(self, name, value)
    def __delattr__(self, name):
        with patch(self):
            return object.__delattr__(self, name)
```

여기까지 구현한 것으로 다음과 같이 task local object를 만들 수 있다

``` python
import asyncio

local_obj = TaskLocal()

async def some_coroutine():
    await asyncio.sleep(local_obj.sleep_seconds)
```

하지만 `threading.local` 에는 있지만, 아직 구현되지 않은 부분이 있는데, 바로 `TaskLocal` 객체를 상속한 클래스도 task local object가 되도록 하는 것이다. 이를 위해서는 `TaskLocal` 의 `__init__` 함수 내용을 `__new__` 로 옮겨야 한다.

``` python
class TaskLocal(object):
    def __new__(cls, *args, **kwargs):
        self = object.__new__(cls)
        local_args = (args, kwargs)
        # 매 태스크마다 __init__(*args, **kwargs)을 호출해야 하므로 인자들을 저장해둔다.
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl(local_args))
        return self
```

물론 `TaskLocalImpl` 에는 `local_args` 를 저장해두어야 하고 `patch` 를 호출할 때도 적절히 `__init__` 을 호출해 주어야 한다

``` python
import threading
import weakref

class TaskLocalImpl(object):
    def __init__(self, local_args):
        self.local_args = local_args  # Save local_args
        self.data = weakref.WeakKeyDictionary()
        self.lock = threading.Lock()
    
    def get(self, task):
        """task의 attribute map data를 가져온다."""
        if task not in self.data:
            self.data[task] = {}
            return self.data[task], True  # 새로 생성된 객체인지 여부도 반환
        return self.data[task], False

def patch(local):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    attrs, is_new = impl.get(asyncio.Task.current_task())  
    if is_new:
        args, kwargs = impl.local_args
        self.__init__(*args, **kwargs)
    with impl.lock:
        object.__setattr__(local, '__dict__', attrs)
        yield
```

여기서 `self.__init__` 을 그냥 불러도 될까? 잘 생각해보면 `__getattribute__` 의 루프에 빠지지 않고 `__init__` 함수를 잘 가져옴을 알 수 있다. 또한 `self.__init__` 를 호출할 때 새로 만든 attribute data가 (즉 `__dict__` 가) 적용되므로 `self.__init__(*args, **kwargs)` 가 `object.__setattr__(local,` `'``__dict__', attrs)` 이전에 불려도 상관이 없다. 호출 순서를 그려보면 다음과 같다

``` python
# call my_local.x first time in new thread
my_local.__getattribute__('x')
    ⤷ with patch(my_local):
    ⤷ attrs, is_new = impl.get(current_task())  # is_new is True
        self.__init__(*args, **kwargs)
        ⤷ my_local.__getattribute__('__init__')
            ⤷ with patch(my_local):
                ⤷ attrs, is_new = impl.get(current_task())  # is_new is False, same attrs
                object.__setattr__(local, '__dict__', attrs)
        ⤷ my_local.__init__(*args, **kwargs)  # happy ending
```

마지막으로, 사용자가 엄하게 사용하는 것을 방어하는 코드를 추가해 보자

``` python
class TaskLocal(object):
    def __new__(cls, *args, **kwargs):
        # TaskLocal()에는 인자가 들어가면 안된다
        if (args or kwargs) and (cls.__init__ == object.__init__):
            raise TypeError('No args for TaskLocal(); try subclassing')
        self = object.__new__(cls)
        local_args = (args, kwargs)
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl(local_args))
        return self

    def __setattr__(self, name, value):
        # __dict__를 바꾸는 것을 방지
        if name == '__dict__':
            raise AttributeError('__dict__ is readonly')
        ...
        
    def __delattr__(self, name):
        # __dict__를 지우는 것을 방지
        if name == '__dict__':
            raise AttributeError('__dict__ is readonly')
        ...
```

와 이제 정말 끝났다! 최종 코드를 확인해보자

``` python
import asyncio
import contextlib
import threading
import weakref

class TaskLocalImpl(object):
    def __init__(self, local_args):
        self.local_args = local_args
        self.data = weakref.WeakKeyDictionary()
        self.lock = threading.Lock()

    def get(self, task):
        """task의 attribute map data를 가져온다."""
        if task not in self.data:
            self.data[task] = {}
            return self.data[task], True
        return self.data[task], False

@contextlib.contextmanager
def patch(local):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    attrs, is_new = impl.get(asyncio.Task.current_task())  
    if is_new:
        args, kwargs = impl.local_args
        self.__init__(*args, **kwargs)
    with impl.lock:
        object.__setattr__(local, '__dict__', attrs)
        yield
        
class TaskLocal(object):
    __slots__ = ['_TaskLocal__impl', '__dict__']

    def __new__(cls, *args, **kwargs):
        if (args or kwargs) and (cls.__init__ == object.__init__):
            raise TypeError('No args for TaskLocal(); try subclassing')
        self = object.__new__(cls)
        local_args = (args, kwargs)
        object.__setattr__(self, '_TaskLocal__impl', TaskLocalImpl(local_args))
        return self
        
    def __getattribute__(self, name):
        with patch(self):
            return object.__getattribute__(self, name)
            
    def __setattr__(self, name, value):
        if name == '__dict__':
            raise AttributeError('__dict__ is readonly')
        with patch(self):
            return object.__setattr__(self, name, value)
            
    def __delattr__(self, name):
        if name == '__dict__':
            raise AttributeError('__dict__ is readonly')
        with patch(self):
            return object.__delattr__(self, name)
```

# Bonus

우리가 열심히 만든 task local 객체는 현재 실행중인 `Task` 에 대한 로컬 객체를 가져온다. 이에 다음과 같은 기능을 추가해 볼 수 있을까?

``` python
local_obj = TaskLocal()
local_obj[task].x = x  # 특정 task에 대한 local을 직접 수정
```

일단 `patch` 함수에서 `asyncio.Task.current_task` 대신 직접 task를 인자로 받도록 수정해보자.

``` python
import asyncio
import contextlib

@contextlib.contextmanager
def patch(local, task=None):
    impl = object.__getattribute__(local, '_TaskLocal__impl')
    if task is None:  # 추가된 부분
        task = asyncio.Task.current_task()
    attrs, is_new = impl.get(task)  
    if is_new:
        args, kwargs = impl.local_args
        self.__init__(*args, **kwargs)
    with impl.lock:
        object.__setattr__(local, '__dict__', attrs)
        yield
```

프록시 클래스를 이용하는 방법을 사용하겠다. `TaskLocal` 의 `__getitem__` 은 프록시 클래스를 리턴해야 한다.

``` python
class TaskLocal(object):
    ...
    def __getitem__(self, task):
        return TaskLocalProxy(self, task)
```

그러면 `TaskLocalProxy` 는 `TaskLocal` 객체와 `Task` 객체를 모두 가지고 있으므로 우리가 원하던 대로 `patch` 에 `Task` 를 인자로 넣어서 프록시할 수 있다.

``` python
class TaskLocalProxy(object):
    __slots__ = ['_TaskLocalProxy__local', '_TaskLocalProxy__task']
    
    def __init__(self, local, task):
        object.__setattr__(self, '_TaskLocalProxy__local', local)
        object.__setattr__(self, '_TaskLocalProxy__task', task)
    
    def __getattribute__(self, name):
        with patch(object.__getattribute__(self, '_TaskLocalProxy__local'),
                    object.__getattribute__(self, '_TaskLocalProxy__task')):
            return object.__getattribute__(self, name)
    
    def __setattr__(self, name, value):
        with patch(object.__getattribute__(self, '_TaskLocalProxy__local'),
                    object.__getattribute__(self, '_TaskLocalProxy__task')):
            return object.__setattr__(self, name, value)
    
    def __delattr__(self, name):
        with patch(object.__getattribute__(self, '_TaskLocalProxy__local'),
                    object.__getattribute__(self, '_TaskLocalProxy__task')):
            return object.__delattr__(self, name)
```

# 마치며

이상으로 `weakref` 와 `__dict__` 를 이용한 동적 attribute 바인딩을 통해 local 객체를 만드는 방법을 살펴보았다. 위의 코드에 대한 저작권은 일차적으로 `threading.local` 를 참조했으므로 PSF (Python Software Foundation) 라이센스가 있을 것도 같은데, 만약에 내게 저작권이 있다면 MIT 라이센스로 배포하는 바이다.


    MIT License
    
    Copyright (c) 2017 Park Jong Bin
    
    Permission is hereby granted, free of charge, to any person obtaining a copy
    of this software and associated documentation files (the "Software"), to deal
    in the Software without restriction, including without limitation the rights
    to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
    copies of the Software, and to permit persons to whom the Software is
    furnished to do so, subject to the following conditions:
    
    The above copyright notice and this permission notice shall be included in all
    copies or substantial portions of the Software.
    
    THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
    IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
    FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
    AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
    LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
    OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
    SOFTWARE.
