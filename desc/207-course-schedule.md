```
class Solution {
public:
    bool canFinish(int numCourses, vector<vector<int>>& prerequisites) {
        // 1. 建图：邻接表 + 入度数组
        vector<vector<int>> graph(numCourses);   // graph[i] 表示学完i后能解锁哪些课
        vector<int> inDegree(numCourses, 0);      // 每门课还需要先修几门
        
        for (auto& pre : prerequisites) 
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