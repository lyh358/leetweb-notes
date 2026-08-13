### 把课程和先修关系看成有向图，用 Kahn 算法（BFS 拓扑排序）判环。
### 数据结构：邻接表存图 + 入度数组记每门课的前置依赖数 + 队列存当前能修的课（入度为 0）
### 核心流程：入度为 0 的课入队 → 出队修完 → 它解锁的课入度减 1 → 入度变 0 再入队
### 判环：最后统计修完的课数，等于总课数则无环能修完，否则有循环依赖。
```
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        // 1. 建图：邻接表 + 入度数组
        vector<vector<int>> graph(numCourses);   // graph[i] 表示学完i后能解锁哪些课
        vector<int> inDegree(numCourses, 0);      // 每门课还需要先修几门
        
        for (auto& pre : prerequisites) //加个引用可以加速，不加也行
      {
            int course = pre[0];    // 要学的课
            int need = pre[1];      // 先修课
            graph[need].push_back(course);  // 学完need，可以解锁course
            inDegree[course]++;             // course的依赖数+1
        }
        
        // 2. 把所有没有依赖的课（入度为0）加入队列
        queue<int> q;
        for (int i = 0; i < numCourses; i++) {
            if (inDegree[i] == 0) {
                q.push(i);
            }
        }
        
        // 3. BFS：一门一门"修完"，消减后续课程的依赖
        int completed = 0;  // 记录已修完的课程数
        while (!q.empty()) {
            int cur = q.front();
            q.pop();
            completed++;
            
            // 修完cur，它解锁的所有课依赖数-1
            for (int next : graph[cur]) {
                inDegree[next]--;
                // 如果这门课的所有先修都学完了，就可以入队
                if (inDegree[next] == 0) {
                    q.push(next);
                }
            }
        }
        
        // 4. 如果所有课都修完了，说明无环
        return completed == numCourses;
    }
};
```