title: 基数排序(radix sort)
date: 2014-11-14 15:20:14
categories: 算法
tags: 算法
---

基数排序(radix sor)的C语言实现。

<!--more-->

[基数排序](http://zh.wikipedia.org/wiki/%E5%9F%BA%E6%95%B0%E6%8E%92%E5%BA%8F)，是一种非比较性的正整数排序方法，其算法很简单：

1. 找出所有数字的最大位数，如42就是2位
2. 所有数字从最低有效位开始，每个有效位进行一次排序，数位较短的补0
3. 排序完成之后就得到正确的排序结果

时间复杂度为: `O(kN)`，`k`为最大位数。

下面是一个C语言的实现，

```c
#include <stdio.h>
#include <stdlib.h> // rand()
#include <math.h>   // pow()

#define SIZE 5000000
#define VERBOSE_DEBUG 0

static int *get_rand_num_array(int size) {
    int *p = (int*)malloc(sizeof(int)*size);
    if(p == NULL)
        printf("error happened when malloc.");
    int i;
    for(i = 0; i < size; i++) {
        p[i] = rand();
    }
    return p;
}

// 是否从小到大排序
static int test_sorted_nums(int *nums, int size) {
    int i;
    for(i = 0; i < size-1; i++) {
        if(nums[i] > nums[i+1]) {
            printf("numbers not sorted.\n");
            return -1;
        }
    }
    printf("numbers sorted.\n");
    return 0;
}

static int get_max_length(int *nums, int size) {
    int max = 1, i;
    for(i = 0; i < size; i++) {
        while(nums[i] >= (int)pow(10, max)) max++;
    }
    return max;
}

void radix_sort(int *nums, int size) {
    int max = get_max_length(nums, size);
    int *tmp = (int*)malloc(sizeof(int)*size);
    int *bucket = (int*)malloc(sizeof(int)*10); // ten numbers, 0...9
    int i, j, radix = 1, *p;
        
    for(i = 1; i <= max+1; i++) {
#if VERBOSE_DEBUG
        for(j = 0; j < size; j++)
            printf(" %d ", nums[j]);
        printf("\n");
#endif
        // 快速清空bucket数组
        memset(bucket, 0, sizeof(int)*10);

        // 统计每个数字(0...9)出现的次数
        for(j = 0; j < size; j++)
            bucket[(nums[j]/radix)%10]++;
        
        // 现在每个bucket[i]中保存的是该数字对应的整数在tmp中
        // 的结尾位置(当然还要加一)
        for(j = 1; j < 10; j++) {
            bucket[j] += bucket[j-1];
        }
        
        // 映射: nums[i] => tmp[j]，bucket中保存着该数字在tmp中的位置
        // 注意倒序，因为bucket[i]中保存的是结尾位置
        for(j = size-1; j >= 0; j--) {
            tmp[(--bucket[(nums[j]/radix)%10])] = nums[j];
        }
        
        // 最后交换两个数组，注意新的tmp数组不用清空
        p = nums;
        nums = tmp;
        tmp = p;
        
        // 进行下一个基数的排序
        radix *= 10;
    }
}

int main()
{
    int *nums = get_rand_num_array(SIZE);
    radix_sort(nums, SIZE);
    test_sorted_nums(nums, SIZE);
    free(nums);
}
```

(over)