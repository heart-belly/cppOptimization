# 利用输入数据集的已知特性
## 输入数据集已经排序完成或是几乎排序完成，插入排序算法在这些数据上的性能很棒，达到了 O(n)。
## Timsort 是一种相对较新的混合型排序算法，它在输入数据集已经排序完成或是几乎排序完成时，性能也能达到 O(n)；而对于其他情况，最优性能则是 O(n log2n)。Timsort 现在已经成为 Python 语言的标准排序算法了。
## 最近还出现了一种称为内省排序（introsort）的算法，它是快速排序和堆排序的混合形式。内省排序首先以快速排序算法开始进行排序，但是当输入数据集导致快速排序的递归深度太深时，会切换为堆排序。内省排序可以确保在最差情况下的时间开销是 O(n log2n) 的同时，利用了快速排序算法的高效实现来减少平均情况下的运行时间。自 C++11 开始，内省排序已经成为了 std::sort() 的优先实现。
## 另外一种最近非常流行的算法是 Flash Sort。对于抽取自某种概率分布的数据，它的性能非常棒，达到了 O(n)。Flash Sort 是与基数排序类似，都是基于概率分布的百分位将数据排序至桶中。Flash Sort 的一个简单的适用场景是当数据元素均匀分布时。

```
// timsort算法的实现（插入+归并）
#include <vector>
#include <algorithm>
#include <stack>
#include <utility>

template<typename T>
class TimSort {
private:
    // 最小run长度
    static const int MIN_MERGE = 32;

    // 计算最小run长度
    static int minRunLength(int n) {
        int r = 0;
        while (n >= MIN_MERGE) {
            r |= (n & 1);
            n >>= 1;
        }
        return n + r;
    }

    // 对长度小于minRun的run进行插入排序
    static void insertionSort(std::vector<T>& arr, int left, int right) {
        for (int i = left + 1; i <= right; i++) {
            T temp = arr[i];
            int j = i - 1;
            while (j >= left && arr[j] > temp) {
                arr[j + 1] = arr[j];
                j--;
            }
            arr[j + 1] = temp;
        }
    }

    // 归并两个有序的run
    static void merge(std::vector<T>& arr, int left, int mid, int right) {
        int len1 = mid - left + 1;
        int len2 = right - mid;
        std::vector<T> leftArr(len1);
        std::vector<T> rightArr(len2);

        for (int i = 0; i < len1; i++)
            leftArr[i] = arr[left + i];
        for (int j = 0; j < len2; j++)
            rightArr[j] = arr[mid + 1 + j];

        int i = 0, j = 0, k = left;
        while (i < len1 && j < len2) {
            if (leftArr[i] <= rightArr[j]) {
                arr[k] = leftArr[i];
                i++;
            } else {
                arr[k] = rightArr[j];
                j++;
            }
            k++;
        }

        while (i < len1) {
            arr[k] = leftArr[i];
            i++;
            k++;
        }

        while (j < len2) {
            arr[k] = rightArr[j];
            j++;
            k++;
        }
    }

    // 检查并合并栈中违反条件的run
    static void mergeCollapse(std::vector<T>& arr, std::stack<std::pair<int, int>>& runs) {
        while (runs.size() >= 2) {
            std::pair<int, int> run2 = runs.top();
            runs.pop();
            std::pair<int, int> run1 = runs.top();
            runs.pop();

            if (run1.first == 0 && run2.first == 0) {
                runs.push(run1);
                runs.push(run2);
                break;
            }

            if (run1.second <= run2.second) {
                merge(arr, run1.first, run1.first + run1.second - 1, run1.first + run1.second + run2.second - 1);
                runs.push({run1.first, run1.second + run2.second});
            } else {
                runs.push(run1);
                runs.push(run2);
                break;
            }
        }
    }

public:
    static void sort(std::vector<T>& arr) {
        int n = arr.size();
        if (n <= 1) return;

        int minRun = minRunLength(n);
        std::stack<std::pair<int, int>> runs;

        for (int i = 0; i < n; ) {
            int runLength = 1;
            if (i + runLength < n && arr[i + runLength] >= arr[i + runLength - 1]) {
                while (i + runLength < n && arr[i + runLength] >= arr[i + runLength - 1])
                    runLength++;
            } else {
                while (i + runLength < n && arr[i + runLength] < arr[i + runLength - 1])
                    runLength++;
                std::reverse(arr.begin() + i, arr.begin() + i + runLength);
            }

            if (runLength < minRun) {
                int force = std::min(minRun, n - i);
                insertionSort(arr, i, i + force - 1);
                runLength = force;
            }

            runs.push({i, runLength});
            mergeCollapse(arr, runs);
            i += runLength;
        }

        while (runs.size() > 1) {
            std::pair<int, int> run2 = runs.top();
            runs.pop();
            std::pair<int, int> run1 = runs.top();
            runs.pop();
            merge(arr, run1.first, run1.first + run1.second - 1, run1.first + run1.second + run2.second - 1);
            runs.push({run1.first, run1.second + run2.second});
        }
    }
};

// 使用示例
int main() {
    std::vector<int> arr = {5, 2, 8, 1, 9, 3, 7, 4, 6};
    TimSort<int>::sort(arr);
    return 0;
}    
```
