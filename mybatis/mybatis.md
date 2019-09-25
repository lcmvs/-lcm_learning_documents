##### 1.Mybatis原理

https://blog.51cto.com/13714880/2112087

https://blog.51cto.com/13714880/2112520

##### 2.获取批量插入实体类获取自增id

https://www.cnblogs.com/chenmz1995/p/10300875.html

```xml
   <insert id="insertList" parameterType="java.util.List" keyProperty="vehicleGnssId" keyColumn="vehicle_gnss_id" useGeneratedKeys="true">
   INSERT INTO vehicle_gnss
(vehicle_info_id, device_id, connection_type, equipment_mfrs,device_type)
VALUES
       <foreach collection="list" item="item" index="index" separator=",">
         ( #{item.vehicleInfoId},
           #{item.deviceId},
           #{item.connectionType},
           #{item.equipmentMfrs},
           #{item.deviceType})
       </foreach>
   </insert>
```

##### 3.多参数，包含list

```java
List<ViolationTimes> selectListByVehicleInfoId(@Param("ids") List<Long> ids, @Param("date")Date date);
```

```xml
<select id="selectListByVehicleInfoId"
        resultType="cn.gdmcmc.iovs.module.violation.pojo.ViolationTimes">
      SELECT violation_times_id,vehicle_info_id, province, city, hphm, hpzl, remark, size,status
   FROM violation_times
   where create_time &gt;= #{date}
      and vehicle_info_id in
      <foreach collection="ids" index="index" item="item" open="(" separator="," close=")">
        #{item}
      </foreach>
</select>
```

##### 4.Free Mybatis plugin跳转插件

