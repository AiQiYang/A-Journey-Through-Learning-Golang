# 哈希

## 基本操作
### 1. 创建与初始化
**1. 使用 `make` 创建空 `map`**
````golang
m := make(map[string]int) // 创建一个空 map
````
特点 ：
+ 动态分配内存，适合需要动态添加键值对的场景
+ 可指定初始容量以优化性能

**2. 使用字面量初始化**
````golang
m := map[string]int{
    "Alice": 25,
    "Bob":   30,
}
````
特点 ：
+ 在编译时确定初始内容，适合静态定义的场景

### 2. 插入与更新
**1. 添加键值对**
````golang
m["Carol"] = 28 // 添加新键值对
````
**2. 更新键值对**
````golang
m["Alice"] = 26 // 更新已存在的键
````

### 3. 查找与判断键是否存在
**1. 查找键值**  
通过键访问值，并判断键是否存在：
````golang
age, exists := m["Alice"]
if exists {
    fmt.Println("Alice's age:", age)
} else {
    fmt.Println("Alice not found")
}
````
**2. 忽略布尔值**  
若不关心键是否存在，可直接访问值：
````golang
age := m["Alice"] // 若键不存在，返回值类型的零值（如 int 类型为 0）
fmt.Println(age)
````

### 4. 删除键值对
使用 `delete` 函数删除指定键：  
````golang
delete(m, "Bob") // 删除键 "Bob"
````
特点 ：
+ 删除不存在的键不会报错，也不会影响其他键值对

### 5. 遍历
使用 `for range` 遍历所有键值对：
````golang
for key, value := range m {
    fmt.Printf("%s: %d\n", key, value)
}
````
注意 ：
+ 遍历顺序是随机的，Go 不保证键值对的顺序
+ 若需有序遍历，可将键提取到切片中并排序：
  ````golang
  keys := make([]string, 0, len(m))
  for key := range m {
      keys = append(keys, key)
  }
  sort.Strings(keys)
  for _, key := range keys {
      fmt.Printf("%s: %d\n", key, m[key])
  }
  ````  

指遍历键：  
````golang
for key := range m {
    // 处理键
}
````

### 6. 获取长度
使用 `len` 函数获取 `map` 中键值对的数量。  
特点 ：
时间复杂度为 O(1)，快速返回当前元素数量。

### 7. 进阶操作
**1. 获取键值集合：**  
+ `map.Keys`：返回 `map` 中所有键。
+ `map.Values`：返回 `map` 中所有值。

**2. 比较两个 `map`：**
+ `map.Equal`：判断两个 `map` 是否相等（键值对都相等）
+ `map.EqualFunc`：通过自定义比较函数来判断两个 `map` 是否相等

**3. 克隆与复制：**
+ `map.Clone`：深拷贝一个 `map`
+ `maps.Copy` ：将一个 `map` 的内容复制到另一个 `map`

**4. 过滤与转换：**
+ `maps.DeleteFunc`：根据条件删除键值对
+ `maps.Transform`：对 `map` 的值进行原地修改
