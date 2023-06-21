## 2.1 - Compression

```py
import random


def count_bits(value_list):
    def bit_length(value):
        if value >= 0:
            binary = "{0:b}".format(value)
            return len(binary)
        else:
            return 32
    return sum(bit_length(value) for value in value_list)


def for_encoding(value_list, frame_size):
    encoded_values = []

    for i, value in enumerate(value_list):
        if i >= frame_size - 1:
            frame_sum = sum(values[i-frame_size:i])
            frame_avg = frame_sum / frame_size
            diff = value - frame_avg
            encoded_values.append(round(diff))
        else:
            encoded_values.append(value)

    return encoded_values


def delta_encoding(value_list):
    encoded_values = [value_list[0]]
    for i in range(1, len(value_list)):
        delta = value_list[i] - value_list[i-1]
        encoded_values.append(delta)
    return encoded_values


def dictionary_encoding(value_list):
    dictionary = {}
    encoded_values = []
    for value in value_list:
        if value in dictionary:
            encoded_values.append(dictionary[value])
        else:
            dictionary[value] = len(dictionary)
            encoded_values.append(dictionary[value])
    return encoded_values


def print_result(value_list, encoded_values):
    for i, value in enumerate(value_list):
        print(f'{value: <3}  {encoded_values[i]: >3}  {"{0:b}".format(encoded_values[i]): >6}')


def test(value_list):
    print(f"Needed Bits:")
    print(f"Source:                 {count_bits(value_list)}")
    for_encoded = for_encoding(value_list, 2)
    delta_encoded = delta_encoding(value_list)
    dict_encoded = dictionary_encoding(value_list)
    print(f"FOR(Frame Size = 2):    {count_bits(for_encoded)}")
    print_result(value_list, for_encoded)
    print(f"DELTA:                  {count_bits(delta_encoded)}")
    print_result(value_list, delta_encoded)
    print(f"DICT:                   {count_bits(dict_encoded)}")
    print_result(value_list, dict_encoded)


if __name__ == "__main__":
    values = [1, 3, 7, 12, 13, 13, 14, 17]
    shuffled_values = random.sample(values, len(values))
    test(values)
    test(shuffled_values)
```

Results:

```sh
Needed Bits:
Source:                 27
FOR(Frame Size = 2):    17
1      1       1
3      3      11
7      5     101
12     7     111
13     4     100
13     0       0
14     1       1
17     4     100
DELTA:                  14
1      1       1
3      2      10
7      4     100
12     5     101
13     1       1
13     0       0
14     1       1
17     3      11
DICT:                   18
1      0       0
3      1       1
7      2      10
12     3      11
13     4     100
13     4     100
14     5     101
17     6     110
Needed Bits:
Source:                 27
FOR(Frame Size = 2):    81
17    17   10001
13    13    1101
3      1       1
14     9    1001
12     2      10
13     0       0
1    -12   -1100
7     -6    -110
DELTA:                  141
17    17   10001
13    -4    -100
3    -10   -1010
14    11    1011
12    -2     -10
13     1       1
1    -12   -1100
7      6     110
DICT:                   16
17     0       0
13     1       1
3      2      10
14     3      11
12     4     100
13     1       1
1      5     101
7      6     110
```

Why does the output from the different algorithms differ between the sorted and the shuffled list?

- FOR: In a sorted list, the values tend to have a smaller difference from their preceding values since they are in ascending order. Therefore, the differences tend to be smaller, resulting in smaller encoded values.
- DELTA: In a sorted list, the differences tend to be smaller since the values are in ascending order. Therefore, the deltas are smaller, resulting in smaller encoded values.
- DICT: Dictionary encoding assigns a unique code to each distinct value encountered. The smaller the coded value assigned to the number of occurrences in the original data list, the shorter the number of bits of the coded result. However, this is random and has nothing to do with the order.

Why does Delta encoding needs fewer bits than Dictionary encoding? Is this always the case?

- Delta encoding generally requires fewer bits than dictionary encoding because it exploits the correlation between consecutive values in the input data. However, this is not always the case, and there can be situations where dictionary encoding performs better. For example in a shuffled case.

What is the optimal frame size for FOR (minimal number of needed bits)?

- The optimal frame size for FOR encoding, which results in the minimal number of needed bits, depends on the characteristics of the data being encoded. There is no universally optimal frame size that applies to all datasets.

Do we save memory by using those algorithms?

- I think so, since the data is save in a compact way.

What happens if negative numbers are part of the input data, and why does it happen?

- first result is that the needed bits for source are increased, sine we need 32 bits to represent negative numbers. The second result is that the needed bits for FOR and DELTA are increased, since the difference between the values is bigger. The needed bits for DICT are not affected, since the order of the values does not matter.

## 2.2 - Displacement strategies

```py
class CacheEntry:
    def __init__(self, key, value):
        self.__key = key
        self.__value = value

    def __eq__(self, other):
        return self.__key == other.__key

    def __str__(self):
        return f"({self.__key}: {self.__value})"

    @property
    def key(self):
        return self.__key

    @property
    def value(self):
        return self.__value


class LRUK:
    def __init__(self, capacity=8, k=2):
        self.__data = []
        self.__capacity = capacity
        self.__element_count = 0
        self.__k = k
        self.__ref_order = []

    def __str__(self):
        return f"[{', '.join(str(entry) for entry in self.__data)}]"

    def put(self, key, value):
        if self.__element_count < self.__capacity:
            self.__data.append(CacheEntry(key, value))
            self.__element_count += 1
            self.__ref_order.append(key)
        else:
            bt = {}
            for entry in self.__data:
                bt[entry.key] = self.__find_last_k_th(entry.key)

            evict_key = max(bt, key=bt.get)
            print(f'ref order before put {key}: {self.__ref_order} -> evict page: {evict_key}')

            for entry in self.__data:
                if entry.key == evict_key:
                    self.__data.remove(entry)
                    self.__data.insert(0, entry)
                    break

            self.__data = self.__data[1:]
            self.__data.append(CacheEntry(key, value))
            self.__ref_order.append(key)

    def __find_last_k_th(self, key):
        order_desc = self.__ref_order[::-1]
        try:
            index = -1
            for _ in range(self.__k):
                index = order_desc.index(key, index+1)
            return index + 1
        except ValueError:
            return 100

    def get(self, key):
        for idx in range(self.__element_count):
            if self.__data[idx].key == key:
                self.__ref_order.append(key)
                self.__data.append(self.__data.pop(idx))
                return self.__data[-1]
        return None


if __name__ == "__main__":
    cache = LRUK()
    for i in range(8):
        cache.put(i, i+1)
    print(f'initial:\n    {cache}')
    cache.get(0)
    cache.get(3)
    cache.get(4)
    cache.get(6)
    cache.get(5)
    cache.get(1)
    cache.get(2)
    cache.get(7)

    print(f'before put page 8:\n    {cache}')
    cache.put(8, 9)
    print(f'after put page 8:\n    {cache}')

    cache.get(8)

    print(f'before put page 9:\n    {cache}')
    cache.put(9, 10)
    print(f'after put page 9:\n    {cache}')
```

Results:

```sh
initial:
    [(0: 1), (1: 2), (2: 3), (3: 4), (4: 5), (5: 6), (6: 7), (7: 8)]
before put page 8:
    [(0: 1), (3: 4), (4: 5), (6: 7), (5: 6), (1: 2), (2: 3), (7: 8)]
ref order before put 8: [0, 1, 2, 3, 4, 5, 6, 7, 0, 3, 4, 6, 5, 1, 2, 7] -> evict page: 0
after put page 8:
    [(3: 4), (4: 5), (6: 7), (5: 6), (1: 2), (2: 3), (7: 8), (8: 9)]
before put page 9:
    [(3: 4), (4: 5), (6: 7), (5: 6), (1: 2), (2: 3), (7: 8), (8: 9)]
ref order before put 9: [0, 1, 2, 3, 4, 5, 6, 7, 0, 3, 4, 6, 5, 1, 2, 7, 8, 8] -> evict page: 1
after put page 9:
    [(3: 4), (4: 5), (6: 7), (5: 6), (2: 3), (7: 8), (8: 9), (9: 10)]
```

Why do we need caching in the first place?

- Caching makes programs run faster because they can access data from the cache rather than having to recompute a result or read from a slower data store.

What happens when k in LRU-K becomes bigger?

- As k increases, the cache needs to maintain a larger data structure to store the history of the last k accesses.
- can improved hit rate

In which scenario LIFO performs better than FIFO?

- LIFO performs better than FIFO when the most recently accessed items are the ones that are most likely to be accessed again.

## 3.1 - Compression in Postgres

Compression can only be enabled for separate tablespaces. To compress a tablespace, you should enable the compression option when creating this tablespace. For example:

```sql
postgres=# CREATE TABLESPACE tsc LOCATION '/Users/xingyuzhou/Desktop/DBDI/TPCH/tpch-dbgen/tmp/tsc' WITH (compression=true);
unrecognized parameter "compression"
```
