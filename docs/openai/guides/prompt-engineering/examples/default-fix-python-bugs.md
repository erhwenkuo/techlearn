# Python bug fixer

![](./assets/default-fix-python-bugs.png)

尋找並修復原始程式碼中的錯誤。

Find and fix bugs in source code.


**Prompt**

|||
|-------|------|
|**SYSTEM**|您將獲得一段 Python 程式碼，您的任務是尋找並修復其中的錯誤。<br/><br/>You will be provided with a piece of Python code, and your task is to find and fix bugs in it.|
|**USER**|<pre lang="python"><code>import Random<br/>    a = random.randint(1,12)<br/>    b = random.randint(1,12)<br/>    for i in range(10):<br/>        question = "What is "+a+" x "+b+"? "<br/>        answer = input(question)<br/>        if answer = a*b<br/>            print (Well done!)<br/>        else:<br/>            print(\"No.\")</code></pre>|

**Sample response**

````
There are several bugs in the provided code:

1. The import statement is incorrect. Python is case-sensitive, so "Random" should be "random".
2. Variables `a` and `b` are defined outside the loop, so they will not change for each iteration. They should be inside the loop if you want to generate different questions.
3. In the question string, `a` and `b` are integers, so you need to convert them to strings using the `str()` function to concatenate them.
4. In the if statement, you are using a single equals sign `=` which is an assignment operator. You should use the double equals sign `==` for comparison.
5. The print statement "Well done!" is not enclosed in quotes. In Python, strings must be enclosed in quotes.
6. The input from the user is a string, so you need to convert it to an integer using the `int()` function before comparing it with the product of `a` and `b`.

Here is the corrected code:

```python
import random
for i in range(10):
    a = random.randint(1,12)
    b = random.randint(1,12)
    question = "What is "+ str(a) +" x "+ str(b) +"? "
    answer = int(input(question))
    if answer == a*b:
        print ("Well done!")
    else:
        print("No.")
```
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
      "content": "You will be provided with a piece of Python code, and your task is to find and fix bugs in it."
    },
    {
      "role": "user",
      "content": "import Random\n    a = random.randint(1,12)\n    b = random.randint(1,12)\n    for i in range(10):\n        question = \"What is \"+a+\" x \"+b+\"? \"\n        answer = input(question)\n        if answer = a*b\n            print (Well done!)\n        else:\n            print(\"No.\")"
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
