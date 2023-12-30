# Lesson plan writer

![](./assets/default-lesson-plan-writer.png)

為特定主題產生課程計劃。

Generate a lesson plan for a specific topic.

**Prompt**

|||
|-------|------|
|**USER**|為代數入門課編寫課程計畫。課程計畫應涵蓋分配律，特別是它在涉及正數和負數混合的簡單情況下的工作原理。舉出一些例子來說明學生常見的錯誤。<br/><br/>Write a lesson plan for an introductory algebra class. The lesson plan should cover the distributive law, in particular how it works in simple cases involving mixes of positive and negative numbers. Come up with some examples that show common student errors.|

**Sample response**

```
Lesson Plan: Introduction to the Distributive Law

Objective: 
By the end of the lesson, students will be able to understand and apply the distributive law in algebra, particularly in cases involving positive and negative numbers.

Materials:
- Whiteboard and markers
- Student notebooks and pencils
- Worksheets for practice problems

Introduction (10 minutes):
1. Begin the lesson by reviewing the basic concepts of algebra, focusing on the operations of addition, subtraction, multiplication, and division.
2. Introduce the distributive law as a rule in algebra that allows us to multiply a single term by two or more terms inside a set of parentheses. Write the general form of the distributive law on the board: a(b + c) = ab + ac.

Direct Instruction (15 minutes):
1. Explain that the distributive law applies to both positive and negative numbers. 
2. Demonstrate the distributive law with a simple example involving positive numbers, such as 3(4 + 2) = 3*4 + 3*2.
3. Show how the distributive law works with negative numbers, using an example like -2(5 - 3) = -2*5 - (-2*3).
4. Emphasize that when distributing a negative number, the sign of the number inside the parentheses changes.

Guided Practice (15 minutes):
1. Provide students with a set of practice problems that involve the distributive law with both positive and negative numbers. 
2. Work through the first few problems as a class, encouraging students to explain their steps and reasoning.
3. Allow students to complete the remaining problems in pairs or small groups, offering guidance and clarification as needed.

Common Student Errors (10 minutes):
1. Discuss common errors students make when applying the distributive law, such as forgetting to distribute to all terms inside the parentheses, or not changing the sign when distributing a negative number.
2. Show examples of these errors, such as incorrectly simplifying -3(2 - 4) as -6 - 4 instead of -6 - (-12).
3. Reinforce the correct application of the distributive law and the importance of careful attention to signs.

Independent Practice (15 minutes):
1. Distribute a worksheet with additional problems for students to complete independently. These problems should include a mix of positive and negative numbers.
2. Circulate the room to answer questions and provide assistance as needed.

Closure (5 minutes):
1. Review the key points of the lesson, emphasizing the distributive law and its application to positive and negative numbers.
2. Ask students to summarize what they learned in their own words.
3. Collect the worksheets for assessment.

Homework:
Assign additional problems for students to practice applying the distributive law at home.

Assessment:
Assess student understanding through their participation in class discussions, their performance on the practice problems, and their completion of the independent practice worksheet.
```

**API request**

```bash
curl https://api.openai.com/v1/chat/completions \
  -H "Content-Type: application/json" \
  -H "Authorization: Bearer $OPENAI_API_KEY" \
  -d '{
  "model": "gpt-4",
  "messages": [
    {
      "role": "user",
      "content": "Write a lesson plan for an introductory algebra class. The lesson plan should cover the distributive law, in particular how it works in simple cases involving mixes of positive and negative numbers. Come up with some examples that show common student errors."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
