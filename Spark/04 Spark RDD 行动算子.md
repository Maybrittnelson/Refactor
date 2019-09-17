> rdd：{1, 2, 3, 3}

| 算子名           | 目的                               | 示例                              | 结果         |
| ---------------- | ---------------------------------- | --------------------------------- | ------------ |
| *collect()*      | 返回RDD中所有的元素                | ```rdd.collect()```               | {1, 2, 3, 3} |
| *take(num)*      | 从RDD中返回 num 个元素             | ```rdd.take(2)```                 | {1, 2}       |
| *first()*        | 从RDD中返回第一个元素              | ```rdd.first()```                 | {1}          |
| *reduce(func)*   | 并行整合RDD中所有数据（例如：sum） | ```rdd.reduce((x, y) => x + y)``` | 9            |
| *count()*        | RDD中的元素个数                    | ```rdd.count()```                 | 4            |
| *saveAsTextFile* | 将每个rdd元素转换成一行写入文件    | ```rdd.saveAsTextFile(path)```    | 1 2 3 3      |

