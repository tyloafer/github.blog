---
title: 深入PHP系列之PHP排序sort函数实现
tags:
  - PHP
categories:
  - PHP
date: '2018-11-17 14:30'
abbrlink: 30174
---

PHP的数组是个很强大的存在，而且使用PHP的数组的时候，只要你能想到，基本都已实现，所以也让我慢慢忘记了排序算法的存在。近期，有个人问我，PHP的sort函数是怎么实现的，尴尬，在重温了一遍算法之后，根据我大学依稀存在的C语言基础，尝试阅读了一下PHP的sort函数实现的源码，以下是以PHP7.3源码为例，个人技术水平有限，仅供参考

<!--more-->

# sort

> bool **sort**    ( array `&$array`   [, int `$sort_flags` = SORT_REGULAR  ] )



首先我们在 `php_srray.h` 中可以看到array 中 相关 排序函数的定义

~~~
PHP_FUNCTION(ksort);
PHP_FUNCTION(krsort);
PHP_FUNCTION(natsort);
PHP_FUNCTION(natcasesort);
PHP_FUNCTION(asort);
PHP_FUNCTION(arsort);
PHP_FUNCTION(sort);
PHP_FUNCTION(rsort);
PHP_FUNCTION(usort);
....
~~~

然后进入`array.c` 找到 `PHP_FUNCTION(sort)` 的实现

~~~
PHP_FUNCTION(sort)
{
    zval *array;
    zend_long sort_type = PHP_SORT_REGULAR;
    compare_func_t cmp;

    ZEND_PARSE_PARAMETERS_START(1, 2)
        Z_PARAM_ARRAY_EX(array, 0, 1)
        Z_PARAM_OPTIONAL
        Z_PARAM_LONG(sort_type)
    ZEND_PARSE_PARAMETERS_END_EX(RETURN_FALSE);

    cmp = php_get_data_compare_func(sort_type, 0);

    if (zend_hash_sort(Z_ARRVAL_P(array), cmp, 1) == FAILURE) {
        RETURN_FALSE;
    }
    RETURN_TRUE;
}
~~~

1. `ZEND_PARSE_PARAMETERS_START`  `ZEND_PARSE_PARAMETERS_END_EX` ，在 `zend_api.h` 中有定义及实现， 主要是进行参数的校验转换等操作
2. `php_get_data_compare_func` 设置比较函数， 也就是根据 sort_flag 来决定
3. `zend_hash_sort` 这里开始了正式的排序

# zend_hash_sort

这个定义在 `zend_hash.h` 中

~~~
#define zend_hash_sort(ht, compare_func, renumber) \
    zend_hash_sort_ex(ht, zend_sort, compare_func, renumber)
~~~

这里将 `zend_hash_sort `  的方法转到了 `zend_hash_sort_ex` 的方法，接下来继续查看 `zend_hash_sort_ex` 即可

# zend_hash_sort_ex

这个定义在 `zend_hash.h` 中，在`zend_hash.c` 中实现

~~~
ZEND_API int ZEND_FASTCALL zend_hash_sort_ex(HashTable *ht, sort_func_t sort, compare_func_t compar, zend_bool renumber)
{
    Bucket *p;
    uint32_t i, j;

    IS_CONSISTENT(ht);
    HT_ASSERT_RC1(ht);

    if (!(ht->nNumOfElements>1) && !(renumber && ht->nNumOfElements>0)) { /* Doesn't require sorting */
        return SUCCESS;
    }

    if (HT_IS_WITHOUT_HOLES(ht)) {
        i = ht->nNumUsed;
    } else {
        for (j = 0, i = 0; j < ht->nNumUsed; j++) {
            p = ht->arData + j;
            if (UNEXPECTED(Z_TYPE(p->val) == IS_UNDEF)) continue;
            if (i != j) {
                ht->arData[i] = *p;
            }
            i++;
        }
    }

    sort((void *)ht->arData, i, sizeof(Bucket), compar,
            (swap_func_t)(renumber? zend_hash_bucket_renum_swap :
                ((HT_FLAGS(ht) & HASH_FLAG_PACKED) ? zend_hash_bucket_packed_swap : zend_hash_bucket_swap)));

    ht->nNumUsed = i;
    ht->nInternalPointer = 0;

    if (renumber) {
        for (j = 0; j < i; j++) {
            p = ht->arData + j;
            p->h = j;
            if (p->key) {
                zend_string_release(p->key);
                p->key = NULL;
            }
        }

        ht->nNextFreeElement = i;
    }
    if (HT_FLAGS(ht) & HASH_FLAG_PACKED) {
        if (!renumber) {
            zend_hash_packed_to_hash(ht);
        }
    } else {
        if (renumber) {
            void *new_data, *old_data = HT_GET_DATA_ADDR(ht);
            Bucket *old_buckets = ht->arData;

            new_data = pemalloc(HT_SIZE_EX(ht->nTableSize, HT_MIN_MASK), (GC_FLAGS(ht) & IS_ARRAY_PERSISTENT));
            HT_FLAGS(ht) |= HASH_FLAG_PACKED | HASH_FLAG_STATIC_KEYS;
            ht->nTableMask = HT_MIN_MASK;
            HT_SET_DATA_ADDR(ht, new_data);
            memcpy(ht->arData, old_buckets, sizeof(Bucket) * ht->nNumUsed);
            pefree(old_data, GC_FLAGS(ht) & IS_ARRAY_PERSISTENT);
            HT_HASH_RESET_PACKED(ht);
        } else {
            zend_hash_rehash(ht);
        }
    }

    return SUCCESS;
}
~~~

1. 第13行开始，判断这个hash表是否有空洞， 如果有的话，遍历整个hash表并填补空洞
2. 使用sort方法来进行 这个hash表的排序， 这里的sort其实是个指针，由`zend_hash_sort_ex` 的第二个参数 **sort_func_t sort** 传递过来，根据上一部分 `zend_hash_sort` 的实现代码，可以看出，这里的sort，其实是指向了 **zend_sort** 这个方法
3. 30行以后就是排序完成后的操作了，不是主要，就不多废话了

# zend_sort

这个在`zend_sort.h`中定义，在 `zend_sort.c` 中实现

~~~
ZEND_API void zend_sort(void *base, size_t nmemb, size_t siz, compare_func_t cmp, swap_func_t swp)
{
	while (1) {
		if (nmemb <= 16) {
			zend_insert_sort(base, nmemb, siz, cmp, swp);
			return;
		} else {
			char *i, *j;
			char *start = (char *)base;
			char *end = start + (nmemb * siz);
			size_t offset = (nmemb >> Z_L(1));
			char *pivot = start + (offset * siz);

			if ((nmemb >> Z_L(10))) {
				size_t delta = (offset >> Z_L(1)) * siz;
				zend_sort_5(start, start + delta, pivot, pivot + delta, end - siz, cmp, swp);
			} else {
				zend_sort_3(start, pivot, end - siz, cmp, swp);
			}
			swp(start + siz, pivot);
			pivot = start + siz;
			i = pivot + siz;
			j = end - siz;
			while (1) {
				while (cmp(pivot, i) > 0) {
					i += siz;
					if (UNEXPECTED(i == j)) {
						goto done;
					}
				}
				j -= siz;
				if (UNEXPECTED(j == i)) {
					goto done;
				}
				while (cmp(j, pivot) > 0) {
					j -= siz;
					if (UNEXPECTED(j == i)) {
						goto done;
					}
				}
				swp(i, j);
				i += siz;
				if (UNEXPECTED(i == j)) {
					goto done;
				}
			}
done:
			swp(pivot, i - siz);
			if ((i - siz) - start < end - i) {
				zend_sort(start, (i - start)/siz - 1, siz, cmp, swp);
				base = i;
				nmemb = (end - i)/siz;
			} else {
				zend_sort(i, (end - i)/siz, siz, cmp, swp);
				nmemb = (i - start)/siz - 1;
			}
		}
	}
}
~~~

1. 4-7行进行了判断， 如果元素总数 <= 16 ，则进行插入排序，这是很正常的，因为插入排序在最好的情况下时O(n) 级的算法，而数据量小的情况下，数据有序的可能性就越高，也就符合了最好的情况
2. 后面就是 如果元素总数  > 16， 则开始了另外一种排序，首先确定了头元素和尾元素，并让元素总数右移一位作为基准， 这里涉及一个函数 `Z_L` ，后面做补充解释
3. 继续判断 `nmemb >> Z_L(10)` 如果 元素总数 右移 10位，依然大于 0的话，选取的偏移数 再向右偏移 1位 作为 delta， 然后  `zend_sort_5(start, start + delta, pivot, pivot + delta, end - siz, cmp, swp);` 进行交换，否则 执行 `zend_sort_3(start, pivot, end - siz, cmp, swp);` 执行排序
4. 下面就是标准的快速排序的操作思路了，只是一般我们会使用递归来处理，但是递归会消耗空间，所以 PHP源码里面选择了非递归的方式

# Z_L

这里在 `zend_long.h` 中有定义

~~~
# define Z_L(i) INT64_C(i)
~~~

然后继续看 `INT64_C` 

# INI64_C

这个在 `php_stdint.h` 中有定义

~~~
#ifndef INT64_C
# if SIZEOF_INT >= 8
#  define INT64_C(c) c
# elif SIZEOF_LONG >= 8
#  define INT64_C(c) c ## L
# elif SIZEOF_LONG_LONG >= 8
#  define INT64_C(c) c ## LL
# endif
#endif
~~~

# zend_sort_3

这个函数也是在 `zend_sort.c`中实现的，主要就是判断，然后交换

~~~
static inline void zend_sort_3(void *a, void *b, void *c, compare_func_t cmp, swap_func_t swp) /* {{{ */ {
	if (!(cmp(a, b) > 0)) {
		if (!(cmp(b, c) > 0)) {
			return;
		}
		swp(b, c);
		if (cmp(a, b) > 0) {
			swp(a, b);
		}
		return;
	}
	if (!(cmp(c, b) > 0)) {
		swp(a, c);
		return;
	}
	swp(a, b);
	if (cmp(b, c) > 0) {
		swp(b, c);
	}
}
~~~

# 总结

综上分析，php的sort函数在元素数 较小(16个及以下)的时候，使用插入排序，否则使用非递归的快速排序来进行排序。