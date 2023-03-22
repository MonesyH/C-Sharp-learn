# 背景
在做单元测试的时候使用Contains来判断两个对象是否相同，debug过程中发现是有两个相同值的对象，可是出来的结果都是false。

所以猜测这个Contains是对比两个对象的地址而不是值。

# 看源码证实一下

```
public bool Contains(T item) {
    if (item == null) {
        for (int i = 0; i < _size; i++)
            if (_items[i] == null)
                return true;
        return false;
    }
    else {
        EqualityComparer<T> c = EqualityComparer<T>.Default;
        for (int i = 0; i < _size; i++) {
            if (c.Equals(_items[i], item))
                return true;
        }
        return false;
    }
}
```

重点就是那个Equals，对比的就是地址，所以在做对象对比时就无法为True；

# 解决方案
假如对象长这样：
```
public class Person {
    public string Name { get; set; }
    public int Age { get; set; }
}
```
## 1. 重写Equals（个人不推荐）
```
public class Person {
    public string Name { get; set; }
    public int Age { get; set; }

    public override bool Equals(object obj) {
        if (obj == null || !(obj is Person)) {
            return false;
        }
        Person other = (Person)obj;
        return this.Name == other.Name && this.Age == other.Age;
    }
}


List<Person> persons = new List<Person>();
persons.Add(new Person { Name = "Tom", Age = 20 });
persons.Add(new Person { Name = "John", Age = 25 });

Person p = new Person { Name = "Tom", Age = 20 };
bool exists = persons.Contains(p); 
```
## 2. 用Any方法
```
bool exists = persons.Any(person => "Tom".Equals(person.Name) && person.Age == 20);
```
## 3. 用FirstOrDefault方法(配合shouldy库来断言)
```
persons.FirstOrDefault(person => "Tom".Equals(person.Name) && person.Age == 20)
       .ShouldNotBeNull();
```
