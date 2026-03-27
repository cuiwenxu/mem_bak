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
有日志uid event(ad=广告点击，ord=订单事件) ts ad_id，需要将订单归因到广告点击事件，
产出结果是uid event(ord) ad(归因广告) ts

需要注意的是一笔订单可能对应多个广告事件，需要归属到最近的那一条
```java
思路
1.按照uid分区，使用sum over遇到广告事件则+1，产出一个tag
2.根据该uid,tag做关联，tag差一个为相关联事件

with tmp_log as (
select 1 as uid,'ord' as event_type,10 as ts,100 as cz_amt,'' as ad_id
union all
select 1 as uid,'ad' as event_type,9 as ts,0 as cz_amt,'a' as ad_id
union all
select 1 as uid,'ad' as event_type,8 as ts,0 as cz_amt,'b' as ad_id
),

开窗打标
with tmp_tag as (
select *,sum(case when event_type='ad' then 1 else 0 end) over(partition by uid 
                                          order by ts desc,event_rn desc
                                          rows between UNBOUNDED preceding and current row) as gp_tag
from 
(
select uid,event_type,ad_id,ts,1 as event_rn
from tmp_log
where event_type='ad'

union all

select uid,event_type,ad_id,ts,0 as event_rn
from tmp_log
where event_type='ord'
) a 
),

select a.*,b.ad_id
from
(
select *
from tmp_tag
where event_type='ord'
) a 
left join 
(
select *
from tmp_tag
where event_type='ad'
) b 
on a.uid=b.uid
and a.gp_tag=(b.gp_tag-1)
``` 
总结，a事件要归因到b上，第一步打标（使用sum(case when event_type='b' then 1 else 0 end) over(partition by uid order by ts desc,event_type desc) as tag)
第二步，自关联，gp_tag差一个为同一组


 