---
layout: post
title: 利用guava美化代码
date: 2018-04-12 
tags: guava
---

### Google collection

	过滤

`按照条件过滤`
```
ImmutableList<String> names = ImmutableList.of("begin", "code", "Guava", "Java");
Iterable<String> fitered =Iterables.filter(names, Predicates.or(Predicates.equalTo("Guava"),Predicates.equalTo("Java")));
System.out.println(fitered);
```
`自定义过滤条件   使用自定义回调方法对Map的每个Value进行操作`  
```
ImmutableMap<String, Integer> map = ImmutableMap.of("begin", 12, "code", 15);
Map<String,Integer> m2 = Maps.transformValues(map,new Function<Integer, Integer>() {
            public Integer apply(Integer input) {
                if(input > 12){
                    return input;
                }else{
                    return input+1;
                }
            }
        });
System.out.println(m2);   //{begin=13, code=15}
```
	Set
```
HashSet setA = newHashSet(1, 2, 3, 4, 5);  
HashSet setB = newHashSet(4, 5, 6, 7, 8);  

SetView union = Sets.union(setA, setB);//setA, setB并集
System.out.println("union:");   //union:12345867

SetView difference = Sets.difference(setA, setB);//setA中有，setB中没有
System.out.println("difference:");   //difference:123

SetView intersection = Sets.intersection(setA, setB);  //交集
System.out.println("intersection:");   //intersection:45
```

	Map
```
MapDifference differenceMap = Maps.difference(mapA, mapB);  
MapDifference differenceMap=Maps.difference(mapA,mapB, Equivalence<? super V> valueEquivalence); 
differenceMap.areEqual();  //是否相等
Map<K, ValueDifference<V>> = differenceMap.entriesDiffering();//key相同，value不同
Map<K,V> entriesOnlyOnLeft = differenceMap.entriesOnlyOnLeft();//只有左边map
Map<K,V> entriesOnlyOnRight = differenceMap.entriesOnlyOnRight(); //只有右边map
Map<K,V> entriesInCommon = differenceMap.entriesInCommon();//相同map
```

	Multimap（没有实现 Map 的接口）  
　　就是一个 key 对应多个 value 的数据结构。看上去它很像 java.util.Map 的结构，但是 Muitimap 不是 Map，没有实现 Map 的接口。
设想你对 Map 调了 2 次参数 key 一样的 put 方法，结果就是第 2 次的 value 覆盖了第 1 次的 value。但是对 Multimap 来说这个 key 同时对应了 2 个 value。所以 Map 看上去是 : {k1=v1, k2=v2,...}，而 Multimap 是 :{k1=[v1, v2, v3], k2=[v7, v8],....}。

  Collection<V> get(K key);

Multimap 接口的主要实现类有：  
`HashMultimap`: key 放在 HashMap，而 value 放在 HashSet，即一个 key 对应的 value 不可重复  
`ArrayListMultimap`: key 放在 HashMap，而 value 放在 ArrayList，即一个 key 对应的 value 有顺序可重复  
`LinkedHashMultimap`: key 放在 LinkedHashMap，而 value 放在 LinkedHashSet，即一个 key 对应的 value 有顺序不可重复  
`TreeMultimap`: key 放在 TreeMap，而 value 放在 TreeSet，即一个 key 对应的 value 有排列顺序  
`create()`方法创建；

	Multiset: 把重复的元素放入集合(没有实现set接口)  
　　事实上，Multiset 并没有实现 Java.util.Set 接口，普通的 Set 就像这样 :[car, ship, bike]，而 Multiset 会是这样 : [car x 2, ship x 6, bike x 3]。
```      
HashMultiset<String> multiSet = HashMultiset.create();
multiSet.add("s");
multiSet.add("s");
multiSet.add("s");
multiSet.add("s");
System.out.println(multiSet.count("s"));//4
 ```
常用实现 Multiset 接口的类有：  
`HashMultiset`: 元素存放于 HashMap  
`LinkedHashMultiset`: 元素存放于 LinkedHashMap，即元素的排列顺序由第一次放入的顺序决定  
`TreeMultiset`:元素被排序存放于TreeMap  
`EnumMultiset`: 元素必须是 enum 类型  
`ImmutableMultiset`: 不可修改的 Mutiset  

	BiMap: 双向 Map（实现了 Map 的接口）  
　　它的特点是它的 value 和它 key 一样也是不可重复的，换句话说它的 key 和 value 是等价的。如果你往BiMap的value里面放了重复的元素，就会得到 IllegalArgumentException。
```
BiMap<K, V> inverse = biMap.inverse().get();
```
BiMap的常用实现有：  
`HashBiMap`: key 集合与 value 集合都有 HashMap 实现  
`EnumBiMap`: key 与 value 都必须是 enum 类型  
`ImmutableBiMap`: 不可修改的 BiMap  

警告：
"The two bimaps are backed by the same data，any changes to one will appear in the other."
```
BiMap<String,String> map = HashBiMap.create();
map.put("s","b");//{s=b}
System.out.println(map);
BiMap<String, String> inverse = map.inverse();
String r = inverse.get("b");//s
System.out.println(r);
inverse.put("ee","mm");
System.out.println(map);//{s=b, mm=ee}
```
	Table

　　当我们需要多个索引的数据结构的时候，通常情况下，我们只能用Map<FirstName, Map<LastName, Person>>来实现。
为此Guava提供了一个新的集合类型－Table集合类型，来支持这种数据结构的使用场景。Table支持"row"和"column"，而且提供多种视图。
Table：相当于有两个key的map


/**
*    A  1  A1
*    A  2  A2
*    A  3  A3
*    B  1  B1
*    B  2  B2
*    B  3  B3
*    C  1  C1
*    C  2  C2
*    C  3  C3
*
*/
```
public void TableTest(){
        Table<String, Integer, String> aTable = HashBasedTable.create();   
        for (char a = 'A'; a <= 'C'; ++a) {  
            for (Integer b = 1; b <= 3; ++b) {   
                aTable.put(Character.toString(a), b, String.format("%s%s", a, b));  
            }  
        }  
        System.out.println(aTable.column(2)); //{A=A2, B=B2, C=C2}
        System.out.println(aTable.row("B"));// {1=B1, 2=B2, 3=B3}  
        System.out.println(aTable.get("B", 2));//B2
        System.out.println(aTable.contains("D", 1));//false   
        System.out.println(aTable.containsColumn(3));// true  
        System.out.println(aTable.containsRow("C")); //true 
        System.out.println(aTable.columnMap());// {1={A=A1, B=B1, C=C1}, 2={A=A2, B=B2, C=C2}, 3={A=A3, B=B3, C=C3}}
        System.out.println(aTable.rowMap()); // {A={1=A1, 2=A2, 3=A3}, B={1=B1, 2=B2, 3=B3}, C={1=C1, 2=C2, 3=C3}} 
        System.out.println(aTable.remove("B", 3));// B3
    }
```
 Immutable(不可变)集合  
　　 不可变集合，顾名思义就是说集合是不可被修改的。集合的数据项是在创建的时候提供，并且在整个生命周期中都不可改变。  
　　 Immutable对象有以下的优点：
>* 线程安全的,immutable对象在多线程下安全，没有竞态条件;     
>* 不需要支持可变性, 可以尽量节省空间和时间的开销. 所有的不可变集合实现都比可变集合更加有效的利用内存;    
```
      在JDK中提供了Collections.unmodifiableXXX系列方法来实现不可变集合, 但是存在一些问题
      Set<String> set = new HashSet<String>();
      set.add("aa");
      Set<String> unmodSet = Collections.unmodifiableSet(set);
      set.add("vv");
      System.out.println(set);//aa,vv
      System.out.println(unmodSet);//aa,vv
```
Immutable集合使用方法：  
>* 1.用copyOf方法:  
     ImmutableSet.copyOf(set)
>* 2.使用of方法  
　　 ImmutableSet.of("a", "b", "c")  
　　 ImmutableMap.of("a", 1, "b", 2)  
>* 3.使用Builder类  
　　 ImmutableSet.Builder<String> builder =ImmutableSet.builder();   
　　 builder.add("1").build();


### 基本数据类型包装工具类
对于基本数据类型的包装类型：Bytes,Booleans,Chars,Doubles,Floats,Ints,Longs......    
>* int[] array2 = Ints.toArray(list);//list to array
>* Ints.asList(1,2,3,4,5,6); //arrays to list
>* int compare = Ints.compare(a, b);// 比较a,b
>* Ints.contains(array, a);//array 中包含a
>* Ints.indexOf(array, a);//a在arrays中位置
>* Ints.max(array);//array最大值
>* Ints.min(array);//array最小值
>* Ints.concat(array, array2);//连接n个数组

### 拆分器[Splitter]
```
 List<String> strings = Splitter.on(',')
                                .trimResults()
                                .omitEmptyStrings()
                                .splitToList("foo,bar,qux");
``` 
拆分器工厂  

|方法	                    |                描述  |
|Splitter.on(char)          |        按单个字符拆分|
|Splitter.on(String)|按字符串拆分|
|Splitter.on(Pattern);Splitter.onPattern(String)|按正则表达式拆分|
|Splitter.fixedLength(int)|按固定长度拆分；最后一段可能比给定长度短|

拆分器修饰符
方法	描述
omitEmptyStrings()
从结果中自动忽略空字符串
trimResults()
移除结果字符串的前导空白和尾部空白
limit(int)
限制拆分出的字符串数量
withKeyValueSeparator(String)	String拆分为map时key和value的连接符 eg：
Splitter.on("&").withKeyValueSeparator("=")
                   .split("a=1&b=2");//返回值为map




连接器[Joiner]
Joiner joiner = Joiner.on("_");
joiner.join("1","2","3");//1_2_3

  如果字符串序列中含有null:
Joiner joiner = Joiner.on("_").skipNulls();
joiner.join("1","2","3",null);//1_2_3

  替换null:
         			Joiner joiner = Joiner.on("_").useForNull("*");
joiner.join("1","2","3",null);//1_2_3_*

  拼接Map     			
                 Joiner.MapJoinerjoiner=Joiner.on("&")
                                           .withKeyValueSeparator("=");
        joiner.join(ImmutableMap.of("a", "1", "b", "2"));//a=1&b=2

 连接对象类型，在这种情况下，会把对象的toString()值连接起来。
Joiner.on(",").join(Arrays.asList(1,5,7));//1,5,7  

 joiner实例总是不可变的。用来定义joiner目标语义的配置方法总会返回一个新的                               joiner实例。

文件操作

Files.copy(from,to);  //复制文件
Files.write(content, file, Charsets.UTF_8);  //写入文件
Files.toByteArray(File file) //读取字节数组
Files.readLines(File file, Charset charset); //从文件中读取内容
Files.move(File from, File to); //移动文件
URL url = Resources.getResource("abc.xml"); //获取classpath下的文件url
还有许多方法可以用............
