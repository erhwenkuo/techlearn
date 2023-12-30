# Memo writer

![](./assets/default-memo-writer.png)

根據提供的要點產生公司備忘錄。

Generate a company memo based on provided points.

**Prompt**

|||
|-------|------|
|**USER**|<pre>Draft a company memo to be distributed to all employees. The memo should cover the following specific points without deviating from the topics mentioned and not writing any fact which is not present here:<br/>    <br/>    Introduction: Remind employees about the upcoming quarterly review scheduled for the last week of April.<br/>    <br/>    Performance Metrics: Clearly state the three key performance indicators (KPIs) that will be assessed during the review: sales targets, customer satisfaction (measured by net promoter score), and process efficiency (measured by average project completion time).<br/>    <br/>    Project Updates: Provide a brief update on the status of the three ongoing company projects:<br/>    <br/>    a. Project Alpha: 75% complete, expected completion by May 30th.<br/>    b. Project Beta: 50% complete, expected completion by June 15th.<br/>    c. Project Gamma: 30% complete, expected completion by July 31st.<br/>    <br/>    Team Recognition: Announce that the Sales Team was the top-performing team of the past quarter and congratulate them for achieving 120% of their target.<br/>    <br/>    Training Opportunities: Inform employees about the upcoming training workshops that will be held in May, including \"Advanced Customer Service\" on May 10th and \"Project Management Essentials\" on May 25th.</pre>|

**Sample response**

```
Dear Team,

I hope this message finds you well. I am writing to remind everyone about the upcoming quarterly review scheduled for the last week of April. This is an important time for us to reflect on our performance and set goals for the next quarter.

During the review, we will be focusing on three key performance indicators (KPIs): sales targets, customer satisfaction (measured by net promoter score), and process efficiency (measured by average project completion time). These metrics are crucial in assessing our overall performance and identifying areas for improvement.

I would also like to provide a brief update on the status of our ongoing company projects:

1. Project Alpha is currently 75% complete, with an expected completion date of May 30th.
2. Project Beta is 50% complete, and we anticipate it to be finished by June 15th.
3. Project Gamma is 30% complete, with a projected completion date of July 31st.

I am pleased to announce that the Sales Team was the top-performing team of the past quarter, achieving 120% of their target. Congratulations to the entire team for their exceptional work!

Lastly, I would like to inform everyone about the upcoming training workshops that will be held in May. "Advanced Customer Service" will be on May 10th and "Project Management Essentials" on May 25th. These workshops are excellent opportunities for professional development and skill enhancement.

Thank you for your continued dedication and hard work. Let's keep pushing forward and make the next quarter even more successful.

Best Regards,

[Your Name]
[Your Position]
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
      "content": "Draft a company memo to be distributed to all employees. The memo should cover the following specific points without deviating from the topics mentioned and not writing any fact which is not present here:\n    \n    Introduction: Remind employees about the upcoming quarterly review scheduled for the last week of April.\n    \n    Performance Metrics: Clearly state the three key performance indicators (KPIs) that will be assessed during the review: sales targets, customer satisfaction (measured by net promoter score), and process efficiency (measured by average project completion time).\n    \n    Project Updates: Provide a brief update on the status of the three ongoing company projects:\n    \n    a. Project Alpha: 75% complete, expected completion by May 30th.\n    b. Project Beta: 50% complete, expected completion by June 15th.\n    c. Project Gamma: 30% complete, expected completion by July 31st.\n    \n    Team Recognition: Announce that the Sales Team was the top-performing team of the past quarter and congratulate them for achieving 120% of their target.\n    \n    Training Opportunities: Inform employees about the upcoming training workshops that will be held in May, including \"Advanced Customer Service\" on May 10th and \"Project Management Essentials\" on May 25th."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
