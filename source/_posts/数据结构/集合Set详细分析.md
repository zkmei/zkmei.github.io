---
title: 集合Set详细分析
date: 2019-07-10 23:30:00
author: MZK
img: https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Set/java-set-bg.jpg
categories: 技术栈
tags:
  - 数据结构
  - 集合
---
# 集合——Set详细分析
## 简介
> Set用于存储无序（存入、去除顺序不一定相同）元素，值不重复。
> 对象相等性：引用到堆上同一个对象的两个引用是相等的。

如果对两个引用调用hashCode方法，会得到相同的结果，如果对象所属的类没有覆盖Object的hashCode方法的话，hashCode会返回每个对象特有的序号（java是依据对象的内存地址计算出的此序号），所以两个不同的对象的hashCode值是不可能相等的。如果想要让两个不同的对象视为相等的，就必须覆盖Object继下来的hashCode方法和equals方法，因为Object hashCode方法返回的是该对象的内存地址，所以必须重写hashCode方法，才能保证两个不同的对象具有相同的hashCode，同时也需要两个不同对象比较equals方法会返回true

## 继承Collection
![](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Set/set-01.png)

set集合添加元素并使用迭代器迭代元素。
```java
public class SetDemo {
	public static void main(String[] args) {
		//Set 集合存和取的顺序不一致。
		Set hs = new HashSet();
		hs.add("酸");
		hs.add("甜");
		hs.add("苦");
		hs.add("辣");
		System.out.println(hs);
		Iterator it = hs.iterator();
		while (it.hasNext()) {
			System.out.println(it.next());
		}
	}
}
```
## HashSet
> HashSet  线程不安全，存取速度快。底层是以哈希表实现的。
> HashSet不存入重复元素的规则.使用hashcode和equals

元素的哈希值是通过元素的hashcode方法 来获取的, HashSet首先判断两个元素的哈希值，如果哈希值一样，接着会比较equals方法 如果 equls结果为true ，HashSet就视为同一个元素。如果equals 为false就不是同一个元素。
哈希值相同equals为false的元素是怎么存储呢,就是在同样的哈希值下顺延（可以认为哈希值相同的元素放在一个哈希桶中）。也就是哈希一样的存一列。
![hashCode值不相同 hashCode值相同equals不相同](https://kaikaistar-1258907959.cos.ap-shanghai.myqcloud.com/Blog/Set/set-02.png)
HashSet：通过hashCode值来确定元素在内存中的位置。一个hashCode位置上可以存放多个元素。
当hashcode() 值相同equals() 返回为true 时,hashset 集合认为这两个元素是相同的元素.只存储一个（重复元素无法放入）。调用原理:先判断hashcode 方法的值,如果相同才会去判断equals 如果不相同,是不会调用equals方法的。
### HashSet判断两个元素重复
通过hashCode方法和equals方法来保证元素的唯一性，add()返回的是boolean类型

判断两个元素是否相同，先要判断元素的hashCode值是否一致，只有在该值一致的情况下，才会判断equals方法，如果存储在HashSet中的两个对象hashCode方法的值相同equals方法返回的结果是true，那么HashSet认为这两个元素是相同元素，只存储一个（重复元素无法存入）。
注意：HashSet集合在判断元素是否相同先判断hashCode方法，如果相同才会判断equals。如果不相同，是不会调用equals方法的。

HashSet 和ArrayList集合都有判断元素是否相同的方法，

boolean contains(Object o)；
HashSet使用hashCode和equals方法，ArrayList使用了equals方法
使用HashSet存储自定义对象，并尝试添加重复对象（对象的重复的判定）:
```java
public class HashSetDemo {
	public static void main(String[] args) {
		HashSet hs = new HashSet();
		hs.add(new Person("jack", 20));
		hs.add(new Person("rose", 20));
		hs.add(new Person("hmm", 20));
		hs.add(new Person("lilei", 20));
		hs.add(new Person("jack", 20));
 
		Iterator it = hs.iterator();
		while (it.hasNext()) {
			Object next = it.next();
			System.out.println(next);
		}
	}
}
 
class Person {
	private String name;
	private int age;
	Person() {
	}
	public Person(String name, int age) {
		this.name = name;
		this.age = age;
	}
	public String getName() {
		return name;
	}
	public void setName(String name) {
		this.name = name;
	}
	public int getAge() {
		return age;
	}
	public void setAge(int age) {
		this.age = age;
	}
	@Override
	public int hashCode() {
		System.out.println("hashCode:" + this.name);
		return this.name.hashCode() + age * 37;
	}
	@Override
	public boolean equals(Object obj) {
		System.out.println(this + "---equals---" + obj);
		if (obj instanceof Person) {
			Person p = (Person) obj;
			return this.name.equals(p.name) && this.age == p.age;
		} else {
			return false;
		}
	}
	@Override
	public String toString() {
		return "Person@name:" + this.name + " age:" + this.age;
	}
}
```

问题：现在有一批数据，要求不能重复存储元素，而且要排序。ArrayList 、 LinkedList不能去除重复数据。HashSet可以去除重复，但是是无序。
所以这时候就要使用TreeSet了
## TreeSet
> TreeSet 红-黑树的数据结构，默认对元素进行自然排序（String）。如果在比较的时候两个对象返回值为0，那么元素重复。
> TreeSet 属于Set集合，该集合的元素是不能重复的，TreeSet如何保证元素的唯一性

```java
public class TreeSetDemo {
	public static void main(String[] args) {
		TreeSet ts = new TreeSet();
		ts.add("ccc");
		ts.add("aaa");
		ts.add("ddd");
		ts.add("bbb");
		System.out.println(ts); // [aaa, bbb, ccc, ddd]
	}
}
```
TreeSet可以自然排序,TreeSet也可以指定排序规则（元素自身具备比较性和容器具备比较性）
### 指定排序规则
比较性要实现Comparable接口，重写该接口的compareTo方法
通过compareTo或者compare方法中的来保证元素的唯一性。
添加的元素必须要实现Comparable接口。当compareTo()函数返回值为0时，说明两个对象相等，此时该对象不会添加进来。
>1 ----| Comparable
>2      		compareTo(Object o)     元素自身具备比较性
>3 ----| Comparator
>4      		compare( Object o1, Object o2 )	给容器传入比较器

元素自身具备比较性：
元素自身具备比较性，需要元素实现Comparable接口，重写compareTo方法，也就是让元素自身具备比较性，这种方式叫做元素的自然排序也叫做默认排序。
```java
public class Demo1 {
	public static void main(String[] args) {
		TreeSet ts = new TreeSet();
		ts.add(new Person("aa", 20, "男"));
		ts.add(new Person("bb", 18, "女"));
		ts.add(new Person("cc", 17, "男"));
		ts.add(new Person("dd", 17, "女"));
		ts.add(new Person("dd", 15, "女"));
		ts.add(new Person("dd", 15, "女"));
		System.out.println(ts);
		System.out.println(ts.size()); // 5
	}
}

class Person implements Comparable {
	private String name;
	private int age;
	private String gender;
	public Person() {
	}
 
	public Person(String name, int age, String gender) {
 
		this.name = name;
		this.age = age;
		this.gender = gender;
	}
 
	public String getName() {
		return name;
	}
 
	public void setName(String name) {
		this.name = name;
	}
 
	public int getAge() {
		return age;
	}
 
	public void setAge(int age) {
		this.age = age;
	}
 
	public String getGender() {
		return gender;
	}
 
	public void setGender(String gender) {
		this.gender = gender;
	}
 
	@Override
	public int hashCode() {
		return name.hashCode() + age * 37;
	}
 
	public boolean equals(Object obj) {
		System.err.println(this + "equals :" + obj);
		if (!(obj instanceof Person)) {
			return false;
		}
		Person p = (Person) obj;
		return this.name.equals(p.name) && this.age == p.age; 
	}
 
	public String toString() {
		return "Person [name=" + name + ", age=" + age + ", gender=" + gender
				+ "]";
	}
 
	@Override
	public int compareTo(Object obj) {		
		Person p = (Person) obj;
		System.out.println(this+" compareTo:"+p);
		if (this.age > p.age) {
			return 1;
		}
		if (this.age < p.age) {
			return -1;
		}
		return this.name.compareTo(p.name);
	}
}
```
容器具备比较性：
当元素自身不具备比较性，或者自身具备的比较性不是所需要的。那么此时可以让容器自身具备。需要定义一个类实现接口Comparator，重写compare方法，并将该接口的子类实例对象作为参数传递给TreeMap集合的构造方法。

注意：当Comparable比较方式和Comparator比较方式同时存在时，以Comparator的比较方式为主；
```java
public class Demo5 {
	public static void main(String[] args) {
		TreeSet ts = new TreeSet(new MyComparator());
		ts.add(new Book("think in java", 100));
		ts.add(new Book("java 核心技术", 75));
		ts.add(new Book("现代操作系统", 50));
		ts.add(new Book("java就业教程", 35));
		ts.add(new Book("think in java", 100));
		ts.add(new Book("ccc in java", 100));
 
		System.out.println(ts); 
	}
}
 
class MyComparator implements Comparator {
 
	public int compare(Object o1, Object o2) {
		Book b1 = (Book) o1;
		Book b2 = (Book) o2;
		System.out.println(b1+" comparator "+b2);
		if (b1.getPrice() > b2.getPrice()) {
			return 1;
		}
		if (b1.getPrice() < b2.getPrice()) {
			return -1;
		}
		return b1.getName().compareTo(b2.getName());
	} 
}
 
class Book {
	private String name;
	private double price;
	public Book() {
	}
 
	public String getName() {
		return name;
	}
 
	public void setName(String name) {
		this.name = name;
	}
 
	public double getPrice() {
		return price;
	}
 
	public void setPrice(double price) {
		this.price = price;
	}
 
	public Book(String name, double price) {
		this.name = name;
		this.price = price;
	}

	@Override
	public String toString() {
		return "Book [name=" + name + ", price=" + price + "]";
	}
}
```
## LinkedHashSet
具有可预测的迭代顺序，也就是我们插入的顺序。
在元素的后面添加新的元素，整个过程就是LinkedHashSet在容器插入数据的过程。此过程主要由LinkedHashSet.class中重写超类的两个addEntry和createEntry 实现双向链表的结构。保证数据已我们录入的顺序遍历输出。

## 小结
看到array，就要想到角标。
看到link，就要想到first，last。
看到hash，就要想到hashCode,equals.
看到tree，就要想到两个接口。Comparable，Comparator。