---
layout: blog
title: "使用PostgreSQL做LBS查询"
catalog: true
header-img: img/post-bg-1.jpg
subtitle: 使用PostgreSQL查询附近的人/附近的店
date: 2024-11-08
tags: [2024,GIS,Spring Boot,LBS]
---

## 使用PostgreSQL做LBS查询

PostgreSQL的PostGis模块，提供了`SFSQL`(Simple Features for SQL规范从 OGC 中定义而来，它定义了构成标准空间数据库的类型和函数。)的完整支。

下面是关于PostGis的简单介绍。
[https://postgis.net/workshops/zh_Hans/postgis-intro/introduction.html](https://postgis.net/workshops/zh_Hans/postgis-intro/introduction.html)

本项目相关demo，[github项目地址](https://github.com/RussXia/postgis-java)

## 安装PostGis并初始化数据

+ 使用Docker安装PostGis，当前镜像版本`17-master`。
```bash
$ docker run --name local-postgis -e POSTGRES_PASSWORD=admin -p15432:5432 -d postgis/postgis
```
+ 启动镜像后，进入容器内，并连接pgsql
```bash
$ docker exec -it local-postgis bash
$ psql -U postgres
```

+ 创建数据库，并加载PostGIS扩展
```bash
$ create database location;
$ \c location;
# 加载 PostGIS 扩展
$ CREATE EXTENSION postgis;
# 验证 PostGIS安装成功
SELECT postgis_full_version();
```
+ 创建表，并导入数据。建表语句如下，测试导入数据如下
    + [init.sql](https://raw.githubusercontent.com/RussXia/postgis-java/refs/heads/master/src/main/resources/sql/init.sql)
    + [city_info_data.sql](https://raw.githubusercontent.com/RussXia/postgis-java/refs/heads/master/src/main/resources/sql/city_info_data.sql)
```sql
CREATE TABLE city_info (
    id SERIAL PRIMARY KEY,
    province VARCHAR(50) NOT NULL,       -- 省份名称
    city VARCHAR(50) NOT NULL,           -- 城市名称
    geom GEOMETRY(Point, 4326) NOT NULL, -- 几何列，使用WGS 84坐标系
    CONSTRAINT unique_province_city UNIQUE (province, city) -- 唯一性约束，确保省份和城市组合唯一
);
```

+ 测试查询验证数据
```sql
select * from city_info where province = '江苏省';
```

## 关键代码编写

+ 项目依赖

```xml
<!-- spring boo版本 -->
<parent>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-parent</artifactId>
    <version>3.3.5</version>
    <relativePath/>
</parent>

<!-- dependencies -->
<dependencies>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-web</artifactId>
    </dependency>
    <dependency>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-data-jpa</artifactId>
    </dependency>
    <dependency>
        <groupId>com.alibaba.fastjson2</groupId>
        <artifactId>fastjson2</artifactId>
        <version>2.0.53</version>
    </dependency>
    <dependency>
        <groupId>org.hibernate</groupId>
        <artifactId>hibernate-spatial</artifactId>
        <version>6.5.3.Final</version>
    </dependency>
    <dependency>
        <groupId>org.postgresql</groupId>
        <artifactId>postgresql</artifactId>
        <scope>runtime</scope>
    </dependency>
    <dependency>
        <groupId>org.projectlombok</groupId>
        <artifactId>lombok</artifactId>
        <optional>true</optional>
    </dependency>
</dependencies>
```

+ spring配置
```poperties
spring.application.name=postgis-java

## default connection pool
spring.datasource.hikari.connectionTimeout=2000
spring.datasource.hikari.maximumPoolSize=5

## PostgreSQL
spring.datasource.url=jdbc:postgresql://localhost:15432/postgres
spring.datasource.username=postgres
spring.datasource.password=admin
spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
```

+ Repository接口
```Java
public interface CityInfoRepository extends JpaRepository<CityInfo, Long> {

    CityInfo findByCity(String city);

    List<CityInfo> findByProvince(String province);

    @Query(value = """
            SELECT id,province,city,ST_DistanceSphere(geom, ST_SetSRID(ST_MakePoint(:longitude, :latitude), 4326)) AS distance
                                    FROM CityInfo
                                    ORDER BY distance asc
                                    LIMIT :limit
            """)
    List<Object[]> findNearestCities(@Param("longitude") BigDecimal longitude,
                                            @Param("latitude") BigDecimal latitude,
                                            @Param("limit") int limit);


    @Query(value = """
            SELECT ST_DistanceSphere(
                       ST_SetSRID(ST_MakePoint(:longitude1, :latitude1), 4326),
                       ST_SetSRID(ST_MakePoint(:longitude2, :latitude2), 4326)
                   ) AS distance
            """, nativeQuery = true)
    BigDecimal calcDistance(@Param("longitude1") BigDecimal longitude1,
                            @Param("latitude1") BigDecimal latitude1,
                            @Param("longitude2") BigDecimal longitude2,
                            @Param("latitude2") BigDecimal latitude2);
}
```
