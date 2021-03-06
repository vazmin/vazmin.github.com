---
title:  "加权随机"
date:   2019-12-22 23:11:00 +0800
categories:
  - utils
tags:
  - weight random
---

从N个元素中随机获取一个元素，通常的做法是得到[0, N)区间内的一个随机数作为索引值，来获取相应的元素。从权重的角度上看，每个元素的权重都相等，总权重为N。

当对部分元素的随机权重加权时，若总权重为W,则在N个元素中随机取一个元素就变成了在[0, W)取一个随机数r。第一个元素到第N个元素可根据总权重W划分随机范围分别为[0, W1), [W1, W2), ..., [Wn-1, Wn)。随机数r所在的随机范围对应本次随机的元素。

### 例
> 在字符数组A ['a', 'b', 'c']随机取一个字符，随机权重分别为1,3,6，总权重为10。

| 元素 | 权重 | 随机范围 |
|:----:|-----:|:---------|
|  a   |    1 | [0, 1)   |
|  b   |    3 | [1, 4)   |
|  c   |    6 | [4, 10)  |


若在[0, 10)取得随机数为2，1 < 2 < 4，则本次随机的元素为b。

表格中的随机范围只是为了方便理解。
**遍历元素权重列表，与随机数相减的差充当下一次循环的随机数，当该值小于0时即命中随机范围。**

```java
// 元素列表 
char[] elements = new char[]{'a', 'b', 'c'};
// 元素权重列表
int[] weights = new int[]{1, 3, 6};
// 权重总和
int totalWeight = Arrays.stream(weights).sum();
// 基于权重总和选择一个随机数
int offset = ThreadLocalRandom.current()
        .nextInt(totalWeight);
// 根据随机数取得命中目标
for (int i = 0; i < weights.length; i++) {
    offset -= weights[i];
    if (offset < 0) {
        // 命中的目标
        System.out.println("random target: " + elements[i]);
    }
}
```

### 实现


以下为参考[dubbo的集群负载均衡加权随机](https://github.com/apache/dubbo/blob/master/dubbo-cluster/src/main/java/org/apache/dubbo/rpc/cluster/loadbalance/RandomLoadBalance.java)实现
```java
public static <V> V random(List<Weight<V>> weightList) {
        // Number of Weight
        int length = weightList.size();
        // Has the same weight?
        boolean sameWeight = true;
        // the weight of every one
        int[] weights = new int[length];
        // the first Weight's weight
        int firstWeight = weightList.get(0).getWeight();
        weights[0] = firstWeight;
        // The sum of weights
        int totalWeight = firstWeight;
        for (int i = 1; i < length; i++) {
            int weight = weightList.get(i).getWeight();
            // save for later use
            weights[i] = weight;
            // Sum
            totalWeight += weight;
            if (sameWeight && weight != firstWeight) {
                sameWeight = false;
            }
        }
        if (totalWeight > 0 && !sameWeight) {
            // If (not every Weight has the same weight 
            // & at least one Weight's weight>0), 
            // select randomly based on totalWeight.
            int offset = ThreadLocalRandom.current()
                  .nextInt(totalWeight);
            // Return a Weight based on the random value.
            for (int i = 0; i < length; i++) {
                offset -= weights[i];
                if (offset < 0) {
                    return weightList.get(i).getTarget();
                }
            }
        }
        // If all Weight have the same weight value 
        // or totalWeight=0, return evenly.
        return weightList.get(
          ThreadLocalRandom.current().nextInt(length))
          .getTarget();
    }
```
权重对象
```java
public class Weight<T> {
    // 权重
    private Integer weight;
    // 元素
    private T target;

    public Weight(){

    }

    public Weight(Integer weight, T target) {
        this.weight = weight;
        this.target = target;
    }

    public Integer getWeight() {
        return weight;
    }

    public void setWeight(Integer weight) {
        this.weight = weight;
    }

    public T getTarget() {
        return target;
    }

    public void setTarget(T target) {
        this.target = target;
    }
}

```