```
// 冒泡排序核心，升序排列
void bubbleSort(int arr[], int n) {
    // 外层：一共n-1轮
    for (int i = 0; i < n - 1; i++) {
        // 内层：每轮把最大的冒到末尾，每轮少比较i个
        for (int j = 0; j < n - 1 - i; j++) {
            if (arr[j] > arr[j + 1]) {
                // 交换
                int temp = arr[j];
                arr[j] = arr[j + 1];
                arr[j + 1] = temp;
            }
        }
    }
}
```
