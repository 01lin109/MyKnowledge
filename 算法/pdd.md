# 1.📝 知识库专栏：特定三元组计数问题

### 📌 题目描述

给定一个长度为 $n$ 的整数数组 `nums`（$-10^9 \le nums[i] \le 10^9$），寻找满足以下条件的三元组 $(i, j, k)$ 的数量：

1. **位置关系**：$0 \le i < j < k < n$
    
2. **数值等式**：$(nums[i] - nums[j]) \times 2 = nums[k]$
    

- **数据规模**：$t \le 10, \; n \le 5000$
    
- **返回值**：结果对 $10^9+7$ 取模。
    

### 💡 核心算法思路：定中点，控两端

- **破局关键**：直接三重循环复杂度为 $O(n^3)$ 必定超时。因为 $n \le 5000$，我们需要一个 **$O(n^2)$** 的解法。
    
- **核心技巧**：移项变形得 $nums[k] = (nums[i] - nums[j]) \times 2$。我们选择**固定中间的 $j$**，将 $i$ 和 $k$ 完全隔离在 $j$ 的左右两侧。
    
- **数据结构**：
    
    - `leftMap`：动态维护当前 $j$ **左侧**（即历史 $i$）每个数字出现的频次。
        
    - `rightMap`：动态维护当前 $j$ **右侧**（即未来 $k$）每个数字出现的频次。
        
- **状态转移（指针滑动）**：当 $j$ 向右移动一位时，过去的 $j$ 自动加入 `leftMap`，而新的 $j$ 从 `rightMap` 中移除。通过“**左侧补票，右侧退票**”实现动态平衡，避免重复建表。
    

### 🛠️ 生产级 Java 实现代码

Java

```
import java.io.*;
import java.util.*;

public class Main {
    private static final int MOD = 1000000007;

    public static void main(String[] args) throws IOException {
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        int T = Integer.parseInt(br.readLine().trim());
        StringBuilder sb = new StringBuilder();
        
        while (T-- > 0) {
            String line = br.readLine();
            while (line != null && line.trim().isEmpty()) line = br.readLine();
            if (line == null) break;
            
            int n = Integer.parseInt(line.trim());
            long[] nums = new long[n];
            if (n > 0) {
                StringTokenizer st = new StringTokenizer(br.readLine());
                for (int i = 0; i < n; i++) nums[i] = Long.parseLong(st.nextToken());
            }
            if (n < 3) { sb.append(0).append("\n"); continue; }

            // 1. 初始化：预将所有元素（除第一个外）放入右侧集
            Map<Long, Integer> rightMap = new HashMap<>();
            for (int i = 1; i < n; i++) rightMap.put(nums[i], rightMap.getOrDefault(nums[i], 0) + 1);

            // 2. 隔离带初始化：j=1 准备就位，左侧纳入 nums[0]，右侧剔除 nums[1]
            Map<Long, Integer> leftMap = new HashMap<>();
            leftMap.put(nums[0], 1);
            rightMap.put(nums[1], rightMap.get(nums[1]) - 1);

            long totalTriplets = 0;

            // 3. 线性扫描中间点 j
            for (int j = 1; j < n - 1; j++) {
                long currentJVal = nums[j];

                // 优化：只遍历左侧已去重的特征值，利用哈希表进行组合计数
                for (Map.Entry<Long, Integer> entry : leftMap.entrySet()) {
                    long iVal = entry.getKey();
                    long targetKVal = (iVal - currentJVal) * 2; // 防溢出公式

                    int kCount = rightMap.getOrDefault(targetKVal, 0);
                    if (kCount > 0) {
                        totalTriplets = (totalTriplets + (long) entry.getValue() * kCount) % MOD;
                    }
                }

                // 4. 状态滚动转移
                leftMap.put(currentJVal, leftMap.getOrDefault(currentJVal, 0) + 1);
                long nextJVal = nums[j + 1];
                rightMap.put(nextJVal, rightMap.get(nextJVal) - 1);
            }
            sb.append(totalTriplets).append("\n");
        }
        System.out.print(sb.toString());
    }
}
```

### ⚠️ 避坑指南（复盘防错）

1. **防止整型溢出**：题目中数值可达 $\pm10^9$，两数相减再乘以 2 必定打破 `int` 范围。代码中统一强制升级为 **`long`** 进行矩阵与哈希表 Key 运算。
    
2. **去重提速**：内层没有直接双重遍历数组，而是去迭代 `leftMap.entrySet()`。由于大厂测试数据中通常包含大量重复元素，去重后迭代能让实际执行时间大打折扣。
    
3. **空间换时间**：巧妙利用“两表滑动维护”将时间复杂度从 $O(n^3)$ 彻底降维至标准的 **$O(n^2)$**，空间复杂度仅为稳定 $O(n)$。


# 2.📝 知识库专栏：去绝对值转化为前缀最值问题

### 📌 题目描述

给定一个长度为 $n$ 的数组 $A$（$1 \le A_i \le 10^9$），定义任意两个元素对儿 $(i, j)$ 的 FST 距离为：

$$dist(i,j) = |i^2 - j^2| + |A_i^2 - A_j^2|$$

求整个数组中所有元素对儿中的**最大 FST 距离**。

- **数据规模**：$n \le 10^5$
    
- **核心难点**：直接双重循环为 $O(n^2)$ 必定超时。必须利用数学性质将时间复杂度降到 **$O(n)$**。
    

### 💡 核心算法思路：去绝对值与状态分离

由于最大距离与 $i$ 和 $j$ 的先后顺序无关，我们不妨直接规定 **$i < j$**。

1. **消去第一项绝对值**：因为 $i < j$，所以 $i^2 < j^2$，第一项直接去绝对值变成：$j^2 - i^2$。
    
2. **拆分第二项绝对值**：根据 $A_i^2$ 和 $A_j^2$ 的大小关系，拆分成两种情况，并**将 $i$ 和 $j$ 的项完全分离开**：
    
    - **情况 1 ($A_i^2 \ge A_j^2$)**：
        
        $$dist = (j^2 - i^2) + (A_i^2 - A_j^2) = \mathbf{(j^2 - A_j^2)} + \mathbf{(A_i^2 - i^2)}$$
        
    - **情况 2 ($A_i^2 < A_j^2$)**：
        
        $$dist = (j^2 - i^2) + (A_j^2 - A_i^2) = \mathbf{(j^2 + A_j^2)} - \mathbf{(i^2 + A_i^2)}$$
        
3. **动态规划/动态维护最值**：
    
    当我们从左到右遍历 $j$ 时，要让距离最大，只需要在 $j$ 之前（即历史已遍历过的 $i$ 中）找到：
    
    - 情况 1：最大的 $(A_i^2 - i^2)$
        
    - 情况 2：最小的 $(i^2 + A_i^2)$
        
    
    我们无需开辟完整的 DP 数组，只需要用两个变量滚动维护这两个前缀最值即可，成功将复杂度降至 $O(n)$。
    

### 🛠️ 生产级 Java 实现代码

Java

```
import java.io.BufferedReader;
import java.io.InputStreamReader;
import java.io.IOException;
import java.util.StringTokenizer;

public class Main {
    public static void main(String[] args) throws IOException {
        // 使用 BufferedReader 加快输入速度，防止输入成为瓶颈
        BufferedReader br = new BufferedReader(new InputStreamReader(System.in));
        
        String line = br.readLine();
        if (line == null || line.trim().isEmpty()) return;
        int n = Integer.parseInt(line.trim());
        
        long[] a = new long[n + 1]; // 采用 1-based 索引
        StringTokenizer st = new StringTokenizer(br.readLine());
        for (int i = 1; i <= n; i++) {
            a[i] = Long.parseLong(st.nextToken());
        }

        long maxDist = 0;

        // 初始化边界：将第一个元素作为初始的 i 归档到历史最值中
        // maxPart1 对应情况 1：维护前面最大的 (A_i^2 - i^2)
        // minPart2 对应情况 2：维护前面最小的 (i^2 + A_i^2)
        long maxPart1 = (a[1] * a[1]) - (1L * 1);
        long minPart2 = (1L * 1) + (a[1] * a[1]);

        // 从第 2 个元素开始遍历作为当前的 j
        for (int j = 2; j <= n; j++) {
            long jsq = (long) j * j;
            long ajsq = a[j] * a[j];

            // 1. 对应情况 1 拆解公式：(j^2 - A_j^2) + 历史最大的 (A_i^2 - i^2)
            long dist1 = (jsq - ajsq) + maxPart1;
            
            // 2. 对应情况 2 拆解公式：(j^2 + A_j^2) - 历史最小的 (i^2 + A_i^2)
            long dist2 = (jsq + ajsq) - minPart2;

            // 3. 更新全局最大距离
            maxDist = Math.max(maxDist, Math.max(dist1, dist2));

            // 4. 状态滚动更新：将当前的 j 作为新的历史档案，去竞争最值
            maxPart1 = Math.max(maxPart1, ajsq - jsq);
            minPart2 = Math.min(minPart2, jsq + ajsq);
        }

        System.out.println(maxDist);
    }
}
```

### ⚠️ 避坑指南（复盘防错）

1. **彻底解决整型溢出**：特征值 $A_i \le 10^9$，其平方最大可达 $10^{18}$，远超 `int` 的 21 亿限制。代码中不仅数组类型采用了 **`long[]`**，在计算索引平方时也使用了 `1L * 1` 或强转 `(long) j * j` 来实施自动类型提权。
    
2. **拒绝绝对值嵌套**：本题不超时的唯一方法就是利用数学变形**把 $i$ 和 $j$ 从绝对值里松绑**。只要能将它们划分到加号或减号的两端，就可以用类似动态规划前缀最值的思想把双重循环降成单层循环。