# Leetcode 48.旋转图像



## leetcode 48.旋转图像

![image-20200925165535541](https://tva1.sinaimg.cn/large/007S8ZIlly1gj3003uo4yj30fs06z75j.jpg)

重点要求原地旋转，即不可以使用额外矩阵来做中间存储再写回，因此直接循环赋值的做法放弃。

![image-20200925165708994](https://tva1.sinaimg.cn/large/007S8ZIlly1gj301khoy5j30730833yv.jpg)

要求整体顺时针旋转90度，此时我们只考虑单个元素的转移策略。由此三阶矩阵可以分为两层考虑，即最外面一层和5单独的一层，同理考虑二阶矩阵为一层，四阶也为两层，五阶为三层，可以推导公式，**level = (n + 1) / 2**

![IMG_0269](https://tva1.sinaimg.cn/large/007S8ZIlly1gj30gxim2dj31ct0u07mt.jpg)

所以我们需要每层做个循环，然后每层单独处理。

再考虑每个层内部的处理方法：![IMG_0270](https://tva1.sinaimg.cn/large/007S8ZIlly1gj30nu4jptj30rd0pln7v.jpg)

如图以四阶矩阵为例，旋转即为每个包含四个元素的组进行内部交换，蓝色为一组，红色为一组。

交换方式即：12换，13换，14换，此时顺序从原先1234变为2341（从左上开始顺时针依次为1234）

所以我们的问题现在转化为，找到这四个元素的坐标即可。

我们此时定义，矩阵的阶数为len，层级为lev，每个层内部循环变量为i（如蓝色为lev=0，i=0，红色为lev=0，i=1）

进而可知，四个点坐标分别为：

* 左上：(lev, lev + i)
* 右上：(lev + i, len - 1 - lev)
* 右下：(len - 1 - lev, len - 1 - lev - i)
* 左下：(len - 1 - lev - i, lev)

至此，此题思路完结，即一层循环，遍历level，二层遍历一个level里面的元素组，即i

算法时间复杂度O(n * n)

本题目使用go语言实现，其他语言可借鉴

```go
func rotate(matrix [][]int) {
	lens := len(matrix)
	levels := (lens + 1) / 2
	for lev := 0; lev < levels; lev++ {
		for i := 0; i < lens-1-lev*2; i++ {
			matrix[lev][i+lev], matrix[lev+i][lens-1-lev] = matrix[lev+i][lens-1-lev], matrix[lev][i+lev]
			matrix[lev][i+lev], matrix[lens-lev-1][lens-lev-i-1] = matrix[lens-lev-1][lens-lev-i-1], matrix[lev][i+lev]
			matrix[lev][i+lev], matrix[lens-lev-i-1][lev] = matrix[lens-lev-i-1][lev], matrix[lev][i+lev]
		}
	}
}
```

![image-20200925172914506](https://tva1.sinaimg.cn/large/007S8ZIlly1gj30yytr0dj30u5078go2.jpg)

![image-20200925172930617](https://tva1.sinaimg.cn/large/007S8ZIlly1gj30z8ldquj30dn02q74s.jpg)
