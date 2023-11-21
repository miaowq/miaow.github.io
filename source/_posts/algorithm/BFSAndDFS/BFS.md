---
title: BFS
categories:
  - BFS&DFS
tags:
  - BFS&DFS
abbrlink: d81e6871
date: 2023-11-20 23:00:00
---

> 示例代码为：Golang

# 从BFS说起
BFS本质就是让你在一幅「图」中找到从起点 `start` 到终点 `target` 的最近距离

举个例子：

  1. 走迷宫，有的格子是围墙不能走，从起点到终点的最短距离是多少？如果这个迷宫带「传送门」可以瞬间传送呢？
  2. 两个单词，要求你通过某些替换，把其中一个变成另一个，每次只能替换一个字符，最少要替换几次？
  3. 连连看游戏，两个方块消除的条件不仅仅是图案相同，还得保证两个方块之间的最短连线不能多于两个拐点。

<!-- more -->

## 伪代码
```Golang
func bfs(start, target *Node) {
  queue := []*Node{start}
  visited := map[*Node]bool{}
  step := 0

  visited[start] = true

  for len(queue) > 0 {
    size := len(queue)

    for i := 0; i < size; i++ {
      cur := queue[0]
      queue = queue[1:]

      // 树形结构中，没有子节点到父节点到指针，visited就不需要了
      for 一个条件 && !visited[cur] {
        visited[cur] = true
        queue = append(queue, cur)
      }
    }

    step++
  }
}
```


## 举个栗子

[二叉树的最小深度](https://leetcode.cn/problems/minimum-depth-of-binary-tree/)

明确start和target：

- start：root
- target：节点到达叶子结点，也就是
```Golang
if cur.Left == nil && cur.Right == nil{
	// 到达叶子节点
}
```

套入`模版`：
```Golang
// 二叉树的最小深度
func minDepth(root *TreeNode) int {
  if root == nil {
    return 0
  }

  queue := []*TreeNode{root}
  depth := 1

  for len(queue) > 0 {
    size := len(queue)

    for i := 0; i < size; i++ {
      cur := queue[0]
      queue = queue[1:]

      for cur.Left == nil && cur.Right == nil {
        return depth + 1
      }

      if cur.Left != nil {
        queue = append(queue, cur.Left)
      }

      if cur.Right != nil {
        queue = append(queue, cur.Right)
      }
    }

    depth++
  }

  return depth
}
```


## 再来个栗子
[打开转盘锁](https://leetcode.cn/problems/open-the-lock/)

### 聊聊思路

> `原题包含了deadends死亡数字和target限制，此处先抛开它们`

分析来看：一共4个位置，每个位置可以向上转，也可以向下转，也就是有 8 种可能

即：从 “0000”开始，转一次，可以穷举出"1000", "9000", "0100", "0900"... 共 8 种密码。然后，再以这 8 种密码作为基础，对每个密码再转一下，穷举出所有可能…

仔细想想，这就可以抽象成一幅图，每个节点有 8 个相邻的节点，求最短距离，这不就典型的 BFS

```Golang
// 伪代码
func bfs(target string) int {
  queue := []string{"0000"}
  step := 0

  for len(queue) > 0 {
    size := len(queue)

    for i := 0; i < size; i++ {
      cur = queue[0]
      queue = queue[1:]

      if cur = target {
        return step
      }

      for j := 0; j < 4; j++ {
        up := upOne(cur, j)
        down := downOne(cur, j)
        queue = append(queue, up)
        queue = append(queue, down)
      }
    }

    step++
  }

  return step
}

func upOne(s string, idx int) string {
  bs := []byte(s)

  if bs[idx] == '9' {
    bs[idx] = '0'
  } else {
    bs[idx] += '1'
  }

  return string(bs)
}

func downOne(s string, idx int) string {
  bs := []byte(s)

  if bs[idx] == '0' {
    bs[idx] = '9'
  } else {
    bs[idx] -= '1'
  }

  return string(bs)
}
```
* 注意：上述代码有诸多问题，只是思路的展示

### 思路中的几个问题
1、会走回头路。比如说我们从 `"0000"` 拨到 `"1000"`，但是等从队列拿出 `"1000"` 时，还会拨出一个 `"0000"`，这样的话会产生`死循环`。

2、没有终止条件，按照题目要求，我们找到 target 就应该结束并返回拨动的次数。

3、没有对 deadends 的处理，按道理这些「死亡密码」是不能出现的，也就是说你遇到这些密码的时候需要跳过。

### 最终实现

针对上述问题，作出调整：
```Golang
func openLock(deadends []string, target string) int {
	queue := []string{"0000"}
	visited := map[string]bool{}
	step := 0

	for i := range deadends {
		visited[deadends[i]] = true
	}

	for len(queue) > 0 {
		size := len(queue)

		for i := 0; i < size; i++ {
			cur := queue[0]
			queue = queue[1:]

			if cur == target {
				return step
			}

			if visited[cur] {
				continue
			}

			visited[cur] = true

			for j := 0; j < 4; j++ {
				num := int(cur[j] - '0')
				up := (num + 1) % 10
				if !visited[strconv.Itoa(up)] {
					queue = append(queue, cur[:j]+strconv.Itoa(up)+cur[j+1:])
				}

				down := (num + 9) % 10
				if !visited[strconv.Itoa(down)] {
					queue = append(queue, cur[:j]+strconv.Itoa(down)+cur[j+1:])
				}
			}
		}

		step++
	}

	return -1
}
```

## 几个问题
> **为什么 BFS 可以找到最短距离，DFS 不行吗？**

首先，你看 BFS 的逻辑，depth 每增加一次，队列中的所有节点都向前迈一步，这保证了第一次到达终点的时候，走的步数是最少的。

DFS 不能找最短路径吗？
其实也是可以的，但是时间复杂度相对高很多：`DFS` 实际上是靠`递归`的堆栈记录走过的路径，你要找到最短路径，`肯定得把二叉树中所有树杈都探索完`才能对比出最短的路径有多长

> **为什么还需要DFS？**

`BFS` 可以找到`最短距离`，但是空间复杂度高，而 `DFS` 的`空间复杂度较低`。

假设有一颗节点数为N的满二叉树

对于`DFS`来说，最坏情况下顶多就是树的高度，也就是 `O(logN)`

对于`BFS`来说，队列中每次都会储存着二叉树一层的节点，这样的话最坏情况下空间复杂度应该是树的最底层节点的数量，也就是 `N/2`，也就是 `O(N)`

# 双向BFS

本质区别：

- `传统的 BFS `框架就是从`起点`开始向四周扩散，遇到`终点`时停止
- `双向 BFS` 则是从`起点和终点`同时开始扩散，当两边有`交集`的时候停止

为什么这样能够能够提升效率呢？其实从 Big O 表示法分析算法复杂度的话，它俩的最坏复杂度都是 O(N)，但是实际上双向 BFS 确实会快一些，上图：

![](/images/algorithm/bfs/bfs0.png)

![](/images/algorithm/bfs/bfs1.png)

图示中的树形结构，如果终点在最底部：

- `传统 BFS` 算法的策略，会把整棵树的节点都搜索一遍，最后找到 `target`
- `双向 BFS` 其实只遍历了半棵树就出现了交集，也就是找到了最短距离

## 局限性

**必须知道终点在哪里**

## 举个栗子

[打开转盘锁](https://leetcode.cn/problems/open-the-lock/)题目可以用`双向BFS`作出优化：

```Golang
func openLock(deadends []string, target string) int {
	visited := map[string]bool{}
	beginQeueu := map[string]bool{
		"0000": true,
	}
	endQueue := map[string]bool{
		target: true,
	}
	step := 0

	for _, s := range deadends {
		visited[s] = true
	}

	for len(beginQeueu) > 0 && len(endQueue) > 0 {
		if len(beginQeueu) > len(endQueue) {
			beginQeueu, endQueue = endQueue, beginQeueu
		}

		newQueue := map[string]bool{}

		for cur, _ := range beginQeueu {
			if endQueue[cur] {
				return step
			}

			if visited[cur] {
				continue
			}

			visited[cur] = true

			for j := 0; j < 4; j++ {
				num := int(cur[j] - '0')
				up := cur[:j] + strconv.Itoa((num+1)%10) + cur[j+1:]
				down := cur[:j] + strconv.Itoa((num+9)%10) + cur[j+1:]

				if !visited[up] {
					newQueue[up] = true
				}

				if !visited[down] {
					newQueue[down] = true
				}
			}
		}

		step++
		beginQeueu = newQueue
	}

	return -1
}
```

双向BFS`不再使用队列，而是使用 map 方便快速判断两个集合是否有交集`，如下图：

![](/images/algorithm/bfs/bfs2.png)

也就是代码中`if len(beginQeueu) > len(endQueue) {...}`交换来体现这个交集，这算是一个优化：因为按照 BFS 的逻辑，队列（集合）中的元素越多，扩散之后新的队列（集合）中的元素就越多；在双向 BFS 算法中，如果我们每次都选择一个较小的集合进行扩散，那么占用的空间增长速度就会慢一些，效率就会高一些。