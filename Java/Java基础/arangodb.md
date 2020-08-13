# arangodb
## 数据类型
- Document 存储单个节点数据
- Edge 存储两个节点关系 



## aql

### 新增
语法 : INSERT document INTO collectionName

####eg
````
INSERT {
    "name": "Ned",
    "surname": "Stark",
    "alive": true,
    "age": 41,
    "traits": ["A","H","C","N","P"]
} INTO Characters

````

### 查询

语法 : FOR variableName IN collectionName

####eg

````

FOR c IN Characters
    RETURN c

````

### 修改

语法 : UPDATE documentKey WITH object IN collectionName

####eg

````

UPDATE "2861650" WITH { alive: false } IN Characters

````

### 删除

####eg

````
REMOVE "2861650" IN Characters
````