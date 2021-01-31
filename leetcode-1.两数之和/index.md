# Leetcode 1.两数之和


## leetcode 1.两数之和

使用map存储已经遍历过的值，key为target - num，即后续需要配对的值，value为值下标。

```go
func twoSum(nums []int, target int) []int {
	// key: target - num  value: index
	ano := make(map[int]int)
	for idx, v := range nums {
		if _, ok := ano[v]; ok {
			return []int{ano[v], idx}
		} else {
			ano[target-v] = idx
		}
	}
	return []int{0, 0}
}
```


