# 基础应用

## 场景

### 只有几个固定值的场景

可以使用性能更好的match替代hashmap，match 会被编译器优化成跳转表或比较指令

```rust
impl Solution {
    #[inline]
    fn value(c: u8) -> i32 {
        match c {
            b'I' => 1,
            b'V' => 5,
            b'X' => 10,
            b'L' => 50,
            b'C' => 100,
            b'D' => 500,
            b'M' => 1000,
            _ => 0,
        }
    }

    pub fn roman_to_int(s: String) -> i32 {
        let s = s.as_bytes();
        let mut ans = 0;

        for i in 0..s.len() - 1 {
            let x = Self::value(s[i]);
            let y = Self::value(s[i + 1]);

            if x < y {
                ans -= x;
            } else {
                ans += x;
            }
        }

        ans + Self::value(s[s.len() - 1])
    }
}

// 如果使用map
let roman = HashMap::from([
    (b'I', 1),
    (b'V', 5),
    (b'X', 10),
    (b'L', 50),
    (b'C', 100),
    (b'D', 500),
    (b'M', 1000),
]);

// roman[&s[i]];

let roman: FxHashMap<u8, i32> = FxHashMap::from_iter([
    (b'I', 1),
    (b'V', 5),
    (b'X', 10),
    (b'L', 50),
    (b'C', 100),
    (b'D', 500),
    (b'M', 1000),
]);
```

