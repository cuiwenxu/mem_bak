二分查找

二分查找是在数组nums的某个范围内进行的，初始范围包括整个数组。每次二分查找都选取位于当前查找范围中间的下标为mid的值，然后比较nums[mid]和目标值t。如果nums[mid]大于或等于t，那么接着比较它的前一个数字nums[mid-1]和t。如果同时满足nums[mid]≥t并且nums[mid-1]＜t，那么mid就是符合条件的位置，返回mid即可。如果nums[mid]≥t并且nums[mid-1]≥t，那么符合条件的位置一定位于mid的前面，接下来在当前范围的前半部分查找。如果nums[mid]小于t，则意味着符合条件的位置一定位于mid的后面，接下来在当前范围的后半部分查找。有两种情况需要特别注意。第1种情况是当mid等于0时如果nums[mid]依然大于目标值t，则意味着数组中的所有数字都比目标值大，应该返回0。第2种情况是当数组中不存在大于或等于目标值t的数字时，那么t应该添加到数组所有值的后面，即返回数组的长度

```java
/**
 * @ClassName BinSearch
 * @Author cui
 * @Date 2025/1/27 12:25
 **/
public class BinSearch {

    public int binSearch(int[] arr, int target) {
        int left = 0;
        int right = arr.length - 1;

        while (left <= right) {
            int mid = (left + right) / 2;
            if (arr[mid] >= target) {
                if (mid == 0 || arr[mid - 1] < target) {
                    return mid;
                }
                right = mid - 1;
            } else {
                left = mid + 1;
            }
        }
        return arr.length;
    }


    public static void main(String[] args) {
        int[] arr={1,3,6,7,9};
        BinSearch binSearch = new BinSearch();
        System.out.println(binSearch.binSearch(arr,10));
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
思路，按照uid分区，使用sum over遇到广告事件则+1，产出一个tag，然后按照tag分组
select uid,sum(case when event='ad' then 1 else 0 end) over(partition by uid 
                                          order by ts desc,event_rn desc
                                          rows between UNBOUNDED preceding and current row) as gp_tag
from 
(
select uid,event,ad_id,ts,1 as event_rn
from log
where event='ad'

union all

select uid,event,'' as ad_id,ts,0 as event_rn
from log
where event='ord'
)

select 
from 
``` 
 