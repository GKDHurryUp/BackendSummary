### `int[]`转`List<Integer>`

```java
int[] arr = new int[5];
List<Integer> list = Arrays.stream(arr).boxed().collect(Collectors.toList());
```

### `Integer[]`转`List<Integer>`

```java
Integer[] arr = new Integer[5];
List<Integer> list = Arrays.asList(arr);
```

### 求小于X的最大完全平方数

```
(int)Math.pow((int)Math.sqrt(X), 2)
```

### 求小于X的最大2^n数

```java
int n = (int)(Math.log(X) / Math.log(2));
int num = 1<< n;
```

### 二进制数的最低位的1

负数的存储是按位取反+1，

```java
int mask = n & (-n);
```

### 二维数组上下左右移动

```java
int[] dx = new int[]{0, 1, 0, -1};
int[] dy = new int[]{1, 0, -1, 0};
```

### 计算三角形面积

**鞋带公式**：

```java
public double calcArea(int[] x1, int[] x2, int[] x3){
    return  0.5 * Math.abs(x1[0]*x2[1] + x2[0]*x3[1] + x3[0]*x1[1]
                           - x2[0]*x1[1] - x2[1]*x3[0] - x3[1]*x1[0]);
}
```

### HashMap根据value排序

1. 使用list

```java
List<Map.Entry<String, Integer>> list = new ArrayList<>(map.entrySet());
Collections.sort(list, new Comparator<Map.Entry<String, Integer>>(){
    @Override
    public int compare(Map.Entry<String, Integer> o1, Map.Entry<String, Integer> o2){ 
        //按照value值，用compareTo()方法默认是从小到大排序
        return o1.getValue().compareTo(o2.getValue());
    }
});
```

### split转义符号：`\\`

