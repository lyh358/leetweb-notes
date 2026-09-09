class Solution {
public:
    int maxScore(vector<int>& cardPoints, int k) {
        int maxPoints = 0;
        int n = cardPoints.size();
        int sum =0;

        for(int i=0;i<k;i++)
        {
            sum += cardPoints[n-1-i];
        }
        maxPoints = sum;
        for(int i=0;i<k;i++)
        {
            sum = sum - cardPoints[n-k+i] + cardPoints[i];
            maxPoints = max(maxPoints,sum);
        }
        return maxPoints;
    }
};
