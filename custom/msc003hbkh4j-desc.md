```
// 快排核心：对 [l, r] 区间排序
void quickSort(int arr[], int l, int r) {
    if (l >= r) return; // 区间只有0/1个元素，终止

    int pivot = arr[r]; // 选最右元素作为基准
    int i = l;
    // 分区：把 <= pivot 的放左边
    for (int j = l; j < r; j++) {
        if (arr[j] <= pivot) {
            swap(arr[i], arr[j]);
            i++;
        }
    }
    swap(arr[i], arr[r]); // pivot放到分割点
    // 递归左右两部分
    quickSort(arr, l, i - 1);
    quickSort(arr, i + 1, r);
}
```
