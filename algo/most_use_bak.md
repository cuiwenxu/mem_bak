二分查找

二分查找是在有序数组 nums 的某个范围内进行的，初始范围包括整个数组。每次都取中间位置 mid，然后判断 `arr[mid]` 和目标值 target 的关系。

这里的目标不是“找到某个等于 target 的值”，而是“找到第一个大于等于 target 的位置”，也就是 lower_bound，所以代码看起来像是“只判断一边”。实际上不是少判断了，而是因为数组有序，另一边已经可以直接排除：

1. 如果 `arr[mid] >= target`，说明答案只可能是 `mid` 或者在 `mid` 左边，不可能在右边。
2. 这时再额外判断一下 `arr[mid - 1]`：
    - 如果 `mid == 0`，说明已经到数组开头了，那 `mid` 就是答案；
    - 如果 `arr[mid - 1] < target`，说明 `mid` 恰好是第一个大于等于 target 的位置；
    - 否则说明左边还有更早满足条件的位置，继续去左半边找。
3. 如果 `arr[mid] < target`，说明 `mid` 以及它左边的元素都不可能是答案，只能去右半边找。

所以二分查找的本质就是：每次利用有序性，直接排除一半区间，而不是两边都继续看。

有两种情况需要特别注意：

- 当 `mid == 0` 且 `arr[mid] >= target` 时，说明答案就是 0；
- 当数组中不存在大于或等于 `target` 的数字时，最终会跳出循环，此时应该返回数组长度 `arr.length`，表示 target 应该插入到数组末尾。

如果题目改成“精确查找 target 的位置”，那逻辑就会更直接：

1. 如果 `arr[mid] == target`，直接返回 `mid`；
2. 如果 `arr[mid] > target`，去左半边继续找；
3. 如果 `arr[mid] < target`，去右半边继续找；
4. 如果循环结束还没找到，说明数组里不存在这个值，返回 `null`。

注意：Java 的基本类型 `int` 不能返回 `null`，所以返回值类型要写成 `Integer`。

```java
/**
 * @ClassName BinSearch
 * @Author cui
 * @Date 2025/1/27 12:25
 **/
public class BinSearch {

    // 查找第一个 >= target 的位置
    public int lowerBound(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;

        while (left <= right) {
            int mid = (left + right) / 2;

            // arr[mid] 已经满足 >= target
            // 那么答案只可能在 mid 或 mid 左边，右边可以直接排除
            if (arr[mid] >= target) {

                // 如果 mid 已经是最左边，或者前一个数还小于 target
                // 说明 mid 就是“第一个 >= target 的位置”
                if (mid == 0 || arr[mid - 1] < target) {
                    return mid;
                }

                // 说明左边还有更早满足条件的位置，继续去左边找
                right = mid - 1;
            } else {

                // arr[mid] < target
                // 说明 mid 以及左边都不可能是答案，只能去右边找
                left = mid + 1;
            }
        }

        // 走到这里说明数组里所有元素都 < target
        // target 应该插入到数组末尾
        return arr.length;
    }


    // 精确查找 target 的位置
    // 找到返回下标，找不到返回 null
    public Integer binSearch(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;

        while (left <= right) {
            int mid = (left + right) / 2;

            if (arr[mid] == target) {
                return mid;
            } else if (arr[mid] > target) {
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }

        return null;
    }


    public static void main(String[] args) {
        int[] arr={1,3,6,7,9};
        BinSearch binSearch = new BinSearch();
        System.out.println(binSearch.lowerBound(arr,10));
        System.out.println(binSearch.binSearch(arr,7));
        System.out.println(binSearch.binSearch(arr,8));
    }

}
```

爬楼梯
```java
/**
 * @ClassName ClimbStair
 * @Author cui
 * @Date 2025/1/31 10:03
 **/
public class ClimbStair {

    /**
     * 递归函数的模板
     * f() {
     *     1.终止条件
     *     2.f(i)=f(i-1) + f(i-2) + x
     * }
     */
    public int helper(int[] cost, int i) {
        /**
         * f(i)= min(f(i-1)+f(i-2)) + cost[i]
         */
        if (i < 2) {
            return cost[i];
        }
        return Math.min(helper(cost, i - 1), helper(cost, i - 2)) + cost[i];
    }

    public int climb(int[] cost) {
        int length = cost.length;
        return Math.min(helper(cost, length - 2), helper(cost, length - 1));
    }

    public static void main(String[] args) {
        int[] cost = {1, 100, 3, 1, 100};
        System.out.println(new ClimbStair().climb(cost));
    }

}
```
# 订单归因到广告点击事件
有日志uid event(ad=广告点击，ord=订单事件) ts ad_id ord_id，需要将订单归因到广告点击事件，
产出结果是uid event(ord) ord_id ad_id(归因广告) ts(广告时间)

需要注意的是一笔订单可能对应多个广告事件，需要归属到最近的那一条， 帮我用hive sql写出来
```java
with tmp_ord_ad as (
select a.ord_id,a.ord_ts,ad_id,row_number() over(partition by ord_id order by ad_ts desc) as rn
from 
(
select ord_id,ts as ord_ts,uid
from table
where event='ord'
) a
left join 
(
select ad_id,ts as ad_ts,uid
from table
where event='ad'
) b 
on a.uid=b.uid
where a.ord_ts>b.ad_ts
),

select ord_id,ord_ts,ad_id 
from tmp_ord_ad 
where rn=1

``` 
如果是首次归因呢，写法就需要复杂一些，
比如用户的行为数据为：ad1,ad2,ord1,ad3,ad4,ord5
```sql
-- 先对订单取lag，把订单以及订单前一条的订单加工到一行
WITH ord_log AS (
    SELECT
        uid,
        ord_id,
        ts AS ord_ts,
        lag(ts) OVER (PARTITION BY uid ORDER BY ts, ord_id) AS prev_ord_ts
    FROM your_table
    WHERE event = 'ord'
),
ad_log AS (
    SELECT
        uid,
        ad_id,
        ts AS ad_ts
    FROM your_table
    WHERE event = 'ad'
),
matched AS (
    SELECT
        o.uid,
        'ord' AS event,
        o.ord_id,
        a.ad_id,
        a.ad_ts AS ts,
        row_number() OVER (
            PARTITION BY o.uid, o.ord_id
            ORDER BY a.ad_ts DESC
        ) AS rn
    FROM ord_log o
    LEFT JOIN ad_log a
      ON o.uid = a.uid
     AND a.ad_ts <= o.ord_ts
     AND (o.prev_ord_ts IS NULL OR a.ad_ts > o.prev_ord_ts) -- 取ad_ts < ord_ts 并且 (ad_ts > prev_ord_ts or prev_ord_ts is null)
)
SELECT
    uid,
    event,
    ord_id,
    ad_id,
    ts
FROM matched
WHERE rn = 1;

```
当然，末次归因也可以借助 1.取lag,加工出prev_ts 2.然后取ad_ts between prev和cur 之间来优化性能

 