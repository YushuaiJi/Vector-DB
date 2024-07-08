# WhuDatabase

# WhuDatabase是什么

可部署在Android的移动端时空向量数据库
100%离线、可移植且独立于SQLite。

# 什么时候需要它？
计划存储多种模态的向量并进行查询更新操作。
在Android设备上部署、收集、处理和快速查询大量与时空相关的向量数据，已经大量由向量点生成的几何数据（点、折线、多边形、多多边形等）时。
当大量数据集需要查询或插入更新数据集速度快时。

# 入门
代码行文方式与SQLite基本相同，继承SQL语句进行建表，插入数据，以及查询。

## 示例1
向量数据的更新（以插入为例）和查询，以向量长度为10的时候为例。
### 创建表和插入数据

```java
// 创建表
CREATE TABLE ten_dim_points (
    id INTEGER PRIMARY KEY,
    dim1 REAL,
    dim2 REAL,
    dim3 REAL,
    dim4 REAL,
    dim5 REAL,
    dim6 REAL,
    dim7 REAL,
    dim8 REAL,
    dim9 REAL,
    dim10 REAL
);

// 插入数据

面向海量数据插入时，面向GPU参与的移动端设备，开放GPU权限会使WhuDatabase调用批量数据插入算法，提高插入效率。
面向海量数据插入时，面向CPU参与的移动端设备，WhuDatabase会采用多线程插入的方式并采取替罪羊策略自动化平衡索引。
INSERT INTO ten_dim_points (dim1, dim2, dim3, dim4, dim5, dim6, dim7, dim8, dim9, dim10) 
VALUES (1.0, 2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0);
INSERT INTO ten_dim_points (dim1, dim2, dim3, dim4, dim5, dim6, dim7, dim8, dim9, dim10) 
VALUES (2.0, 3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0, 11.0);
INSERT INTO ten_dim_points (dim1, dim2, dim3, dim4, dim5, dim6, dim7, dim8, dim9, dim10) 
VALUES (3.0, 4.0, 5.0, 6.0, 7.0, 8.0, 9.0, 10.0, 11.0, 12.0);
```
### 数据查询
查询所有数据

```java
SELECT * FROM ten_dim_points;
```

查询特定条件的数据

```java
SELECT * FROM ten_dim_points WHERE dim1 = 1.0 AND dim2 = 2.0;
```

范围查询
面向海量查询时，WhuDatabase在所设计的索引上会训练小型自动化选择搜寻策略模型，选择最优的查询策略并并行处理提高查询效率。
单次查询服从SQLite查询方法。
```java
SELECT * FROM ten_dim_points WHERE dim1 BETWEEN 1.0 AND 2.0 AND dim2 BETWEEN 2.0 AND 3.0;
```

k-最近邻

面对k-最近邻查询，需自身定义查询的距离计算公式（如欧式距离），并进行查询。当面对小数据集时，采用遍历的方式寻找数据点。当面对大数据且稳定的k近邻查询的时，我们构建多叉平衡KD树改进查询效率。

```java
// 求距离(3, 3, 3, 3, 3, 3, 3, 3, 3, 3)最近的点
WITH distances AS (
    SELECT 
        id, 
        dim1, dim2, dim3, dim4, dim5, dim6, dim7, dim8, dim9, dim10,
        SQRT(
            (dim1 - 3.0) * (dim1 - 3.0) + 
            (dim2 - 3.0) * (dim2 - 3.0) + 
            (dim3 - 3.0) * (dim3 - 3.0) + 
            (dim4 - 3.0) * (dim4 - 3.0) + 
            (dim5 - 3.0) * (dim5 - 3.0) + 
            (dim6 - 3.0) * (dim6 - 3.0) + 
            (dim7 - 3.0) * (dim7 - 3.0) + 
            (dim8 - 3.0) * (dim8 - 3.0) + 
            (dim9 - 3.0) * (dim9 - 3.0) + 
            (dim10 - 3.0) * (dim10 - 3.0)
        ) AS distance
    FROM 
        ten_dim_points
)
SELECT 
    id, 
    dim1, dim2, dim3, dim4, dim5, dim6, dim7, dim8, dim9, dim10, 
    distance
FROM 
    distances
ORDER BY 
    distance
LIMIT 2; // 返回最近的两个点

```
聚类分析，基于 dim1 和 dim2 的分组和聚合来分析不同的簇

```java
// 假设每10个单位为一个簇
SELECT
    CAST(dim1 / 10 AS INTEGER) * 10 AS x_cluster,
    CAST(dim2 / 10 AS INTEGER) * 10 AS y_cluster,
    COUNT(*) AS count
FROM ten_dim_points
GROUP BY x_cluster, y_cluster;
```
自定义排序

```java
// dim1 升序，dim2 降序排序
SELECT *
FROM ten_dim_points
ORDER BY dim1 ASC, dim2 DESC;
```

## 示例二
多维向量的处理与计算，以三维向量为例。
### 创建表和插入数据

```java
// 创建表格
CREATE TABLE vectors (
    id INTEGER PRIMARY KEY,
    x REAL,
    y REAL,
    z REAL
);

// 插入数据
INSERT INTO vectors (x, y, z) VALUES 
(1.0, 2.0, 3.0),
(4.0, 5.0, 6.0),
(7.0, 8.0, 9.0);
```
### 查询向量数据
```java
// 查询所有向量
SELECT * FROM vectors;

// 查询特定条件的向量
SELECT * FROM vectors WHERE x > 4;
```
### 计算
计算向量的模（长度）

```java
SELECT id, SQRT(x*x + y*y + z*z) AS magnitude FROM vectors;
```
计算两个向量点积（假设有两个与vectors定义相同的向量表vector1和vector2）

```java
SELECT v1.id, 
       (v1.x * v2.x + v1.y * v2.y + v1.z * v2.z) AS dot_product
FROM vectors1 v1
JOIN vectors2 v2 ON v1.id = v2.id;
```
计算向量距离（欧几里得距离），继续使用vector1和vector2

```java
SELECT v1.id, 
       SQRT((v1.x - v2.x) * (v1.x - v2.x) + 
            (v1.y - v2.y) * (v1.y - v2.y) + 
            (v1.z - v2.z) * (v1.z - v2.z)) AS euclidean_distance
FROM vectors1 v1
JOIN vectors2 v2 ON v1.id = v2.id;
```

## 示例三
数据的统计与聚合，以商品的售价为例。
### 创建表和插入数据

```java
// 创建表格
CREATE TABLE sales (
    id INTEGER PRIMARY KEY,
    product_id INTEGER,
    amount REAL
);

// 插入数据
INSERT INTO sales (product_id, amount) VALUES 
(1, 100.0),
(2, 150.0),
(1, 200.0),
(2, 250.0),
(1, 300.0);
```
### 数据处理并返回

```java
SELECT 
    product_id, 
    COUNT(*) AS sales_count, 
    SUM(amount) AS total_sales, 
    AVG(amount) AS average_sales
FROM 
    sales
GROUP BY 
    product_id;
```

## 示例四
面向时空短向量生成的几何对象的查询和处理，WhuDatabase数据库支持自定义的查询
### 创建表和插入数据
```java
// 创建表格
CREATE TABLE places (
    id INTEGER PRIMARY KEY,
    name TEXT,
    geom GEOMETRY
);

// 插入点数据
INSERT INTO places (name, geom) VALUES ('Place A', GeomFromText('POINT(1 1)', 4326));
INSERT INTO places (name, geom) VALUES ('Place B', GeomFromText('POINT(2 2)', 4326));

// 插入线数据
INSERT INTO places (name, geom) VALUES ('Line A', GeomFromText('LINESTRING(0 0, 2 2)', 4326));
INSERT INTO places (name, geom) VALUES ('Line B', GeomFromText('LINESTRING(0 2, 2 0)', 4326));

// 插入多边形数据
INSERT INTO places (name, geom) VALUES ('Polygon A', GeomFromText('POLYGON((0 0, 0 3, 3 3, 3 0, 0 0))', 4326));
```
### 基本查询

查询所有数据并显示几何对象的文本表示
```java
SELECT id, name, AsText(geom) FROM places;
```
### 空间查询

查询包含特定点的几何对象
```java
SELECT name FROM places WHERE Contains(geom, GeomFromText('POINT(1 1)', 4326));
```
查询相交的几何对象

```java
// 两条线是否相交
SELECT Intersects(
    (SELECT geom FROM places WHERE name = 'Line A'),
    (SELECT geom FROM places WHERE name = 'Line B')
) AS do_intersect;

// 与一条线相交的几何对象
SELECT name FROM places WHERE Intersects(geom, GeomFromText('LINESTRING(0 0, 2 2)', 4326));
```

查询距离

```java
// 两点之间的距离
SELECT Distance(
    (SELECT geom FROM places WHERE name = 'Place A'),
    (SELECT geom FROM places WHERE name = 'Place B')
) AS distance;

// 距离特定点一定范围内的几何对象
SELECT name FROM places WHERE Distance(geom, GeomFromText('POINT(1 1)', 4326)) < 2.0;
```
### 几何操作
加快数据分析，计算，和机器学习模型训练运算效率

```java
SELECT id, name, AsText(Buffer(geom, 1.0)) AS buffered_geom FROM places;
```
计算几何对象的凸包

```java
SELECT id, name, AsText(ConvexHull(geom)) AS convex_hull_geom FROM places;
```
# 其他常见问题
## 什么是 WhuDatabase？
简单来说：WhuDatabase= 查询效率改进的SQLite + 高效大规模的向量搜索引擎 + 高级地理空间支持。
## 它使用 JDBC 吗？
否，它使用cursors - 建议使用轻量级方法来访问 Android 平台中使用的 SQL，而不是更重的 JDBC。
