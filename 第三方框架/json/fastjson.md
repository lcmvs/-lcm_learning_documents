# fastjson

### 1.处理数组

```java
    public static void main(String[] args){
        String json="{\"return_code\":0,\"return_msg\":\"查询成功\",\"return_data\":{\"province\":\"GD\",\"city\":\"GD_GZ\",\"hphm\":\"粤AD06597\",\"hpzl\":\"52\",\"lists\":[{\"date\":\"2018-09-18 12:17:00\",\"area\":\"利通广场\",\"act\":\"机动车违反禁令标志指示的\",\"code\":\"13441\",\"fen\":\"3\",\"wzcity\":\"mock\",\"money\":\"200\",\"handled\":\"0\",\"archiveno\":\"mock\"},{\"date\":\"2019-01-18 12:17:00\",\"area\":\"test利通广场\",\"act\":\"test机动车违反禁令标志指示的\",\"code\":\"13441\",\"fen\":\"3\",\"wzcity\":\"mock\",\"money\":\"200\",\"handled\":\"0\",\"archiveno\":\"mock\"}]}}";
//        Map map=Json2Map(json);
//        List list=(List)map.get("return_data");
//        System.out.println(list);
     String  array=JSONObject.parseObject(JSONObject.parseObject(json).getString("return_data")).getString("lists");
        List<ViolationInfo> list = JSONArray.parseArray(array).toJavaList(ViolationInfo.class);
        System.out.println(list);
    }
```

# 复杂处理

[fastjson json字符串和JavaBean、List、Map及复杂集合类型的相互转换](https://www.jianshu.com/p/b82761a64581)

本文主要示例两部分内容：

- JavaBean、List、Map、复杂集合 转换成 json字符串；
- json字符串 转换成 JavaBean、List、Map、复杂集合；

定义POJO：

```tsx
public class A {

    private String usename;
    private String password;

    public A() {
    }

    public A(String usename, String password) {
        this.usename = usename;
        this.password = password;
    }

    public String getUsename() {
        return usename;
    }

    public void setUsename(String usename) {
        this.usename = usename;
    }

    public String getPassword() {
        return password;
    }

    public void setPassword(String password) {
        this.password = password;
    }
}
```
测试
```dart
public static void main(String[] args) {

        //javaBean 转 json字符串
        A a1 = new A("wei.hu", "123456");

        String a1Json = JSON.toJSONString(a1);
        System.out.println(a1Json);



        //List<JavaBean> 转 json字符串
        A a2 = new A("mengna.shi", "123456");
        A a3 = new A("ming.li", "567890");

        List<A> aList = Lists.newArrayList(a1, a2, a3);
        String aListJson = JSON.toJSONString(aList);
        System.out.println(aListJson);

        //List<String> 转 json字符串
        List<String> stringList = Lists.newArrayList("wei.hu", "mengna.shi", "fastJson");

        String stringListJson = JSON.toJSONString(stringList);
        System.out.println(stringListJson);

        //List<Integer> 转 json字符串
        List<Integer> integerList = Lists.newArrayList(10, 9, 8, 7);

        String integerListJson = JSON.toJSONString(integerList);
        System.out.println(integerListJson);



        //Map<String, A> 转 json字符串
        Map<String, A> aMap = Maps.newHashMap();
        aMap.put("a1", a1);
        aMap.put("a2", a2);
        aMap.put("a3", a3);

        String aMapJson = JSON.toJSONString(aMap);
        System.out.println(aMapJson);

        //Map<String, String> 转 json字符串
        Map<String, String> stringMap = Maps.newHashMap();
        stringMap.put("name", "wei.hu");
        stringMap.put("gender", "man");
        stringMap.put("age", "18");

        String stringMapJson = JSON.toJSONString(stringMap);
        System.out.println(stringMapJson);

        //Map<String, Integer> 转 json字符串
        Map<String, Integer> integerMap = Maps.newHashMap();
        integerMap.put("int1",18);
        integerMap.put("int2", 19);
        integerMap.put("int3", 20);

        String integerMapJson = JSON.toJSONString(integerMap);
        System.out.println(integerMapJson);

        //Map<String, Object> 转 json字符串
        Map<String, Object> objectMap = Maps.newHashMap();
        objectMap.put("name", "wei.hu");
        objectMap.put("gender", "man");
        objectMap.put("age", 18);

        String objectMapJson = JSON.toJSONString(objectMap);
        System.out.println(objectMapJson);



        //List<Map<String,Object>> 转 json字符串
        Map<String, A> aMap1 = Maps.newHashMap();
        aMap1.put("a1", a1);
        aMap1.put("a2", a2);

        List<Map<String, A>> aList1 = Lists.newArrayList();
        aList1.add(aMap);
        aList1.add(aMap1);

        String complexJson1 = JSON.toJSONString(aList1);
        System.out.println(complexJson1);

        //Map<String, List<JavaBean>> 转 json字符串
        List<A> aList2 = Lists.newArrayList(a1, a2);
        List<A> aList3 = Lists.newArrayList(a2, a3);

        Map<String, List<A>> listMap = Maps.newHashMap();
        listMap.put("key1", aList2);
        listMap.put("key2", aList3);

        String complexJson2 = JSON.toJSONString(listMap);
        System.out.println(complexJson2);



        //json字符串 转 javaBean
        String jsonString1 = "{\"password\":\"123456\",\"usename\":\"wei.hu\"}";

        A aa1 = JSON.parseObject(jsonString1, A.class);
        System.out.println(aa1.getUsename() + " / " + aa1.getPassword());


        //json字符串 转 List<JavaBean>
        String jsonString2 = "[{\"password\":\"123456\",\"usename\":\"wei.hu\"},{\"password\":\"123456\",\"usename\":\"mengna.shi\"},{\"password\":\"567890\",\"usename\":\"ming.li\"}]";

        List<A> aList4 = JSON.parseArray(jsonString2, A.class);
        System.out.println(aList4.size());


        //json字符串 转 List<String>
        String jsonString3 = "[\"wei.hu\",\"mengna.shi\",\"fastJson\"]";

        List<String> stringList1 = JSON.parseArray(jsonString3, String.class);
        System.out.println(stringList1.size());


        //json字符串 转 List<Integer>
        String jsonString4 = "[10,9,8,7]";

        List<Integer> integerList1 = JSON.parseArray(jsonString4, Integer.class);
        System.out.println(integerList1.size());


        //json字符串转 Map<String, Object>
        String jsonString5 = "{\"a1\":{\"password\":\"123456\",\"usename\":\"wei.hu\"},\"a2\":{\"password\":\"123456\",\"usename\":\"mengna.shi\"},\"a3\":{\"password\":\"567890\",\"usename\":\"ming.li\"}}";

        Map<String, Object> stringObjectMap = JSON.parseObject(jsonString5, Map.class);
        System.out.println(stringObjectMap.size());



        //json字符串 转 List<Map<String, A>>
        String jsonString6 = "[{\"a1\":{\"password\":\"123456\",\"usename\":\"wei.hu\"},\"a2\":{\"password\":\"123456\",\"usename\":\"mengna.shi\"},\"a3\":{\"password\":\"567890\",\"usename\":\"ming.li\"}},{\"a1\":{\"$ref\":\"$[0].a1\"},\"a2\":{\"$ref\":\"$[0].a2\"}}]";

        List<Map<String, A>> mapList = JSON.parseObject(jsonString6, new TypeReference<List<Map<String, A>>>(){});
        System.out.println("mapList.size():" + mapList.size());

        //json字符串 转 Map<String, List<A>>
        String jsonString7 = "{\"key1\":[{\"password\":\"123456\",\"usename\":\"wei.hu\"},{\"password\":\"123456\",\"usename\":\"mengna.shi\"}],\"key2\":[{\"$ref\":\"$.key1[1]\"},{\"password\":\"567890\",\"usename\":\"ming.li\"}]}";

        Map<String, List<A>> listMap1 = JSON.parseObject(jsonString7, new TypeReference<Map<String, List<A>>>() {});
        System.out.println("listMap1.size():" + listMap1.size());


        //json字符串 转 List<Map<String, List<A>>>
        A aaa1 = new A("wei.hu", "123456");
        A aaa2 = new A("mengna.shi", "abcdef");
        A aaa3 = new A("admin", "098765");
        List<A> myAList1 = Lists.newArrayList(aaa1, aaa2, aaa3);

        A aaa4 = new A("song.xu", "28");
        A aaa5 = new A("jielun.zhou", "36");
        List<A> myAList2 = Lists.newArrayList(aaa4, aaa5);

        Map<String, List<A>> myMap1 = Maps.newHashMap();
        myMap1.put("myAList1", myAList1);
        myMap1.put("myAList2", myAList2);


        A aaa6 = new A("junjie.lin", "61");
        A aaa7 = new A("jian.xiao", "31");
        A aaa8 = new A("xi.ben", "32");
        List<A> myAList3 = Lists.newArrayList(aaa6, aaa7, aaa8);

        Map<String, List<A>> myMap2 = Maps.newHashMap();
        myMap2.put("myAList3", myAList3);


        A aaa9 = new A("xing.qun", "33");
        A aaa10 = new A("datong.fang", "34");
        A aaa11 = new A("dun.tong", "35");
        List<A> myAList4 = Lists.newArrayList(aaa9, aaa10, aaa11);

        Map<String, List<A>> myMap3 = Maps.newHashMap();
        myMap3.put("myAList4", myAList4);

        List<Map<String, List<A>>> list = Lists.newArrayList(myMap1, myMap2, myMap3);

        System.out.println(JSON.toJSONString(list));

        List<Map<String, List<A>>> newList = JSON.parseObject(JSON.toJSONString(list), new TypeReference<List<Map<String, List<A>>>>() {});
        System.out.println("newList.size():" + newList.size());


        //josn字符串 转 Map<String, List<Map<String, A>>>
        A objectA1 = new A("1", "1");
        A objectA2 = new A("2", "2");
        A objectA3 = new A("3", "3");

        Map<String, A> newMap1 = Maps.newHashMap();
        newMap1.put("objectA1", objectA1);
        newMap1.put("objectA2", objectA2);
        newMap1.put("objectA3", objectA3);


        A objectA4 = new A("4", "4");
        A objectA5 = new A("5", "5");
        A objectA6 = new A("6", "6");

        Map<String, A> newMap2 = Maps.newHashMap();
        newMap2.put("objectA4", objectA4);
        newMap2.put("objectA5", objectA5);
        newMap2.put("objectA6", objectA6);

        List<Map<String, A>> newList1 = Lists.newArrayList(newMap1, newMap2);


        A objectA7 = new A("7", "7");
        A objectA8 = new A("8", "8");

        Map<String, A> newMap3 = Maps.newHashMap();
        newMap3.put("objectA7", objectA7);
        newMap3.put("objectA8", objectA8);

        List<Map<String, A>> newList2 = Lists.newArrayList(newMap3);


        A objectA9 = new A("9", "9");
        A objectA0 = new A("0", "0");

        Map<String, A> newMap4 = Maps.newHashMap();
        newMap4.put("objectA9", objectA9);
        newMap4.put("objectA0", objectA0);

        List<Map<String, A>> newList3 = Lists.newArrayList(newMap4);

        Map<String, List<Map<String, A>>> map = Maps.newHashMap();
        map.put("newList1", newList1);
        map.put("newList2", newList2);
        map.put("newList3", newList3);

        System.out.println(JSON.toJSONString(map));


        Map<String, List<Map<String, A>>> newMap = JSON.parseObject(JSON.toJSONString(map), new TypeReference<Map<String, List<Map<String, A>>>>() {});
        System.out.println("newMap.size()" + newMap.size());
    }
```

输出

```csharp
{"password":"123456","usename":"wei.hu"}
[{"password":"123456","usename":"wei.hu"},{"password":"123456","usename":"mengna.shi"},{"password":"567890","usename":"ming.li"}]
["wei.hu","mengna.shi","fastJson"]
[10,9,8,7]
{"a1":{"password":"123456","usename":"wei.hu"},"a2":{"password":"123456","usename":"mengna.shi"},"a3":{"password":"567890","usename":"ming.li"}}
{"gender":"man","name":"wei.hu","age":"18"}
{"int2":19,"int1":18,"int3":20}
{"gender":"man","name":"wei.hu","age":18}
[{"a1":{"password":"123456","usename":"wei.hu"},"a2":{"password":"123456","usename":"mengna.shi"},"a3":{"password":"567890","usename":"ming.li"}},{"a1":{"$ref":"$[0].a1"},"a2":{"$ref":"$[0].a2"}}]
{"key1":[{"password":"123456","usename":"wei.hu"},{"password":"123456","usename":"mengna.shi"}],"key2":[{"$ref":"$.key1[1]"},{"password":"567890","usename":"ming.li"}]}
wei.hu / 123456
3
3
4
3
mapList.size():2
listMap1.size():2
[{"myAList2":[{"password":"28","usename":"song.xu"},{"password":"36","usename":"jielun.zhou"}],"myAList1":[{"password":"123456","usename":"wei.hu"},{"password":"abcdef","usename":"mengna.shi"},{"password":"098765","usename":"admin"}]},{"myAList3":[{"password":"61","usename":"junjie.lin"},{"password":"31","usename":"jian.xiao"},{"password":"32","usename":"xi.ben"}]},{"myAList4":[{"password":"33","usename":"xing.qun"},{"password":"34","usename":"datong.fang"},{"password":"35","usename":"dun.tong"}]}]
newList.size():3
{"newList3":[{"objectA9":{"password":"9","usename":"9"},"objectA0":{"password":"0","usename":"0"}}],"newList2":[{"objectA7":{"password":"7","usename":"7"},"objectA8":{"password":"8","usename":"8"}}],"newList1":[{"objectA2":{"password":"2","usename":"2"},"objectA3":{"password":"3","usename":"3"},"objectA1":{"password":"1","usename":"1"}},{"objectA6":{"password":"6","usename":"6"},"objectA4":{"password":"4","usename":"4"},"objectA5":{"password":"5","usename":"5"}}]}
newMap.size()3
```

# TypeReference