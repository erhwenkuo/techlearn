# Improve code efficiency

![](./assets/default-code-improvement.png)

提供提高 Python 程式碼效率的想法。

Provide ideas for efficiency improvements to Python code.

**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得一段Python程式碼，您的任務是提供提高效率的想法。<br/><br/>You will be provided with a piece of Python code, and your task is to provide ideas for efficiency improvements.|
|**USER**|<pre lang="python"><code>from typing import List<br/>                <br/>                <br/>    def has_sum_k(nums: List[int], k: int) -> bool:<br/>        """<br/>        Returns True if there are two distinct elements in nums such that their sum <br/>        is equal to k, and otherwise returns False.<br/>        """<br/>        n = len(nums)<br/>        for i in range(n):<br/>            for j in range(i+1, n):<br/>                if nums[i] + nums[j] == k:<br/>                    return True<br/>        return False</code></pre>|

**Sample response**

````
The current implementation has a time complexity of O(n^2) due to the nested for loops. This can be improved to O(n) by using a set to store the numbers we have seen so far.

Here is the improved code:

```python
from typing import List

def has_sum_k(nums: List[int], k: int) -> bool:
    """
    Returns True if there are two distinct elements in nums such that their sum 
    is equal to k, and otherwise returns False.
    """
    seen = set()
    for num in nums:
        if k - num in seen:
            return True
        seen.add(num)
    return False
```

In this code, we iterate through the list of numbers only once. For each number, we check if its complement (k - num) is in the set of numbers we have seen so far. If it is, we return True because we have found two numbers that sum to k. If it's not, we add the current number to the set and continue to the next number. If we finish iterating through the list without finding two numbers that sum to k, we return False.

This approach reduces the time complexity from O(n^2) to O(n), which is a significant improvement for large lists. The space complexity is also O(n) because in the worst case, we might need to store all the numbers in the set. However, this is a reasonable trade-off for the improved time complexity.
````

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-4",
  "messages": [
    {
      "role": "system",
      "content": "You will be provided with a piece of Python code, and your task is to provide ideas for efficiency improvements."
    },
    {
      "role": "user",
      "content": "from typing import List\n                \n                \n    def has_sum_k(nums: List[int], k: int) -> bool:\n        \"\"\"\n        Returns True if there are two distinct elements in nums such that their sum \n        is equal to k, and otherwise returns False.\n        \"\"\"\n        n = len(nums)\n        for i in range(n):\n            for j in range(i+1, n):\n                if nums[i] + nums[j] == k:\n                    return True\n        return False"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
