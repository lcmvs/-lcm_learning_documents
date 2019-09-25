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