而在`Python`3.7之后, `Python`的Dict使用了新的数据结构, 使`Python`新Dict的内存占用也比老的Dict少了, 同时新的Dict在遍历时是跟插入顺序是一致的, 具体的实现是, 初始化时会生成两个数组, 插入值时, 在数组二追加当前的数据, 并获得当前追加数据所在的下标A, 然后对key进行哈希取模算出下标B, 最后对数组下标B的值更新为A, 简单的演示如下:

```Python
# 初始的结构  
# -1代表还未插入数据  
array_1 = [-1, -1, -1, -1, -1, -1, -1, -1]  
array_2 = []  
  
  
# 插入值后, 他就会变为:  
array_1 = [-1, 0, -1, -1, -1, 1, -1, -1]  
array_2 = [  
 [123456, "key1", 1],  
 [234567, "key2", 2],  
]
```

可以看出, 数组2的容量跟当前放入的值相等的, 数组1虽然还会保持1/3的空闲标记位, 但他只保存数组二的下标, 占用空间也不多, 相比之前的方案会节省一些空间, 同时在遍历的时候可以直接遍历数组2, 这样`Python`的Dict就变为有序的了. 注: 为了保持操作高效, 在删除的时候, 只是把数组1的值设置为-2, 并把数组2对应的值设置为None, 而不去删除它, 在查找时会忽略掉数组1中值为-2的元素， 在遍历时会忽略掉值为None的元素.

```Python
from typing import Any, Iterable, List, Optional, Tuple  
  
  
class CustomerDict(object):  
  
    def __init__(self):  
        self._init_seed: int = 3  # 容量因子  
        self._init_length: int = 2 ** self._init_seed  # 初始化数组大小  
        self._load_factor: float = 2 / 3  # 扩容因子  
        self._index_array: List[int] = [-1 for _ in range(self._init_length)]  # 存放下标的数组  
        self._data_array: List[Optional[Tuple[int, Any, Any]]] = []  # 存放数据的数组  
        self._used_count: int = 0  # 目前用的量  
        self._delete_count: int = 0  # 被标记删除的量  
  
    def _create_new(self):  
        """扩容函数"""  
        self._init_seed += 1  # 增加容量因子  
        self._init_length = 2 ** self._init_seed  
        old_data_array: List[Tuple[int, Any, Any]] = self._data_array  
        self._index_array: List[int] = [-1 for _ in range(self._init_length)]  
        self._data_array: List[Tuple[int, Any, Any]] = []  
        self._used_count = 0  
        self._delete_count = 0  
  
        # 这里只是简单实现, 实际上只需要搬运一半的数据  
        for item in old_data_array:  
            if item is not None:  
                self.__setitem__(item[1], item[2])  
  
    def _get_next(self, index: int):  
        """计算冲突的下一跳，如果下标对应的值冲突了, 需要计算下一跳的下标"""  
        return ((5*index) + 1) % self._init_length  
  
    def _core(self, key: Any, default_value: Optional[Any] = None) -> Tuple[int, Any, int]:  
        """获取数据或者得到可以放新数据的方法, 返回值是index_array的索引, 数据, data_array的索引"""  
        index: int = hash(key) % (self._init_length - 1)  
        while True:  
            data_index: int = self._index_array[index]  
            # 如果是-1则代表没有数据  
            if data_index == -1:  
                break  
            # 如果是-2则代表之前有数据则不过被删除了  
            elif data_index == -2:  
                index = self._get_next(index)  
                continue  
  ``
            _, new_key, default_value = self._data_array[data_index]  
            # 判断是不是对应的key  
            if key != new_key:  
                index = self._get_next(index)  
            else:  
                break  
        return index, default_value, data_index  
  
    def __getitem__(self, key: Any) -> Any:  
        _, value, data_index = self._core(key)  
        if data_index == -1:  
            raise KeyError(key)  
        return value  
  
    def __setitem__(self, key: Any, value: Any) -> None:  
        if (self._used_count / self._init_length) > self._load_factor:  
            self._create_new()  
        index, _, _ = self._core(key)  
  
        self._index_array[index] = self._used_count  
        self._data_array.append((hash(key), key, value))  
        self._used_count += 1  
  
    def __delitem__(self, key: Any) -> None:  
        index, _, data_index = self._core(key)  
        if data_index == -1:  
            raise KeyError(key)  
        self._index_array[index] = -2  
        self._data_array[data_index] = None  
        self._delete_count += 1  
  
    def __len__(self) -> int:  
        return self._used_count - self._delete_count  
  
    def __iter__(self) -> Iterable:  
        return iter(self._data_array)  
      
    def __str__(self) -> str:  
        return str({item[1]: item[2] for item in self._data_array if item is not None})  
  
    def keys(self) -> List[Any]:  
        """模拟实现keys方法"""  
        return [item[1] for item in self._data_array if item is not None]  
  
    def values(self) -> List[Any]:  
        """模拟实现values方法"""  
        return [item[2] for item in self._data_array if item is not None]  
  
    def items(self) -> List[Tuple[Any, Any]]:  
        """模拟实现items方法"""  
        return [(item[1], item[2]) for item in self._data_array if item is not None]  
  
  
if __name__ == '__main__':  
    customer_dict: CustomerDict = CustomerDict()  
    customer_dict["demo_1"] = "a"  
    customer_dict["demo_2"] = "b"  
    assert len(customer_dict) == 2  
  
    del customer_dict["demo_1"]  
    del customer_dict["demo_2"]  
    assert len(customer_dict) == 0  
  
    for i in range(30):  
        customer_dict[i] = i  
    assert len(customer_dict) == 30  
  
    customer_dict_value_list: List[Any] = customer_dict.values()  
    for i in range(30):  
        assert i == customer_dict[i]  
  
    for i in range(30):  
        assert customer_dict[i] == i  
        del customer_dict[i]  
    assert len(customer_dict) == 0
```
