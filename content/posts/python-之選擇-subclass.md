---
title: python 之選擇 subclass
date: 2020-10-10 22:00:00
description: 如何動態決定要使用哪一個 subclass？
categories:
    - [coding, python]
tags:
    - python
    - coding
    - class
mathjax: false
katex: false
draft: true
---

# 選擇 subclass？
相較於宣告的時候就確定要用哪一個 subclass，反而希望是在宣告時的再藉由 *變數* 決定是哪一個 subclass。

以下面的例子為例：
```python
class Base():
    ......

class subclassA(Base):
    ......

class subclassB(Base):
    ......

class subclassC(Base):
    ......
```

相較於已經知道需要何種 subclass 的
```python
# 需要 subclassA
a = subclassA()
```

反而是 **想達成**
```python
# 變數 subclassType 決定 subclass 的種類
subclass = Base.create(subclassType)
```
類似的形式，動態的決定所需要的 subclass。

# 方法
## 直接判斷需要的種類
利用 mapping 直接將 *變數對應到相應的 subclass*
```python
class Base():
    @classmethod
    def create(cls, subclassType, params):
        type2subclass = {
            'A': subclassA,
            'B': subclassB,
            'C': subclassC
        }

        if subclassType not in type2subclass:
            raise Exception
        
        return type2subclass[subclassType](params)
```
直接在 Base 中，建立一個對應的 dictionary，當給進變數時便能回傳其所對應的 subclass，如：
```python
# 欲得到 subclassB
subclassType = 'B'
sub = Base().create(subclassType, params)
```
便可由 subclassType 和 Base 取得相應的 subclass

**然而**，
此情況下，想要新增一個新的 subclass 時，除了定義新的 subclass，還需更動 `Base` 的部分，將其加至 `type2subclass`，否則便無法以動態的方式去取得。因此，此方法雖然可行，但在程式開發的過程中，容易因忘記而無法正確的取得 subclass。

## register subclasses
為避免上述的問題發生，利用 python 的 decorators 將 subclasses 自動加到 mapping 中。在定義 subclass 的時候，將其自身加到 mapping 中。

利用 parrameterized decorator 操作
```python
class Base():
    subclasses = {}

    @classmethod
    def register_subclass(cls, subclassType):
        def decorator(subclass):
            cls.subclasses[subclassType] = subclass
            return subclass
        reutrn decorator

    @classmethod
    def create(cls, subclassType, params):
        if subclassType not in cls.subclasses:
            raise Exception
        return cls.subclasses[subclasstype](params)

@Base.register_subclass('A')
class subclassA(Base):
    ......

......
```
如此一來增加或減少 subclass 時，便不需要更動到 Base 部分的程式碼，只需要在定義新的 subclass 前，加上 `@Base.register_subclass(相對應的字串)` 便可自動加至 mapping 中了。

## init subclass hook
在 python 3.6 以上的版本中，可以直接利用**特殊函式**`__init_subclass__`達到相同的目標。
```python
class Base():
    subclasses = {}

    def __init_subclass__(cls, **kwargs):
        super().__init_subclass__(**kwargs)
        cls.subclasses[cls._SUBCLASS_TYPE] = cls

    @classmethod
    def create(cls, subclassType, params):
        if subclassType not in cls.subclasses:
            raise Exception
        return cls.subclasses[subclasstype](params)

class subclassA(Base):
    _SUBCLASS_TYPE = 'A'
    ......

......
```

# 結論
以程式的結構、開發而言，[register subclasses](#register-subclasses) 和 [init subclass hook](#init-subclass-hook) 會較自己定義手動定義 mapping 並需要每增加、減少一個 subclass 都需要回去更動 Base class 而言，來的更加方便且不容易會有遺漏。然而，對於不熟悉 *decorator* 和沒有接觸過 *__init_subclass__* 的人而言，其可能是一個較容易接受、理解的一種方法。

# reference
- [Python: choosing subclass](https://medium.com/@vadimpushtaev/python-choosing-subclass-cf5b1b67c696)
