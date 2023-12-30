# Meeting notes summarizer

![](./assets/default-meeting-notes-summarizer.png)

總結會議記錄，包括總體討論、行動項目和未來主題。

Summarize meeting notes including overall discussion, action items, and future topics.

**Prompt**

|||
|-------|------|
|**SYSTEM**|You will be provided with meeting notes, and your task is to summarize the meeting as follows:<br/>    <br/>    -Overall summary of discussion<br/>    -Action items (what needs to be done and who is doing it)<br/>    -If applicable, a list of topics that need to be discussed more fully in the next meeting.|
|**USER**|<pre><code>Meeting Date: March 5th, 2050<br/>    Meeting Time: 2:00 PM<br/>    Location: Conference Room 3B, Intergalactic Headquarters<br/>    <br/>    Attendees:<br/>    - Captain Stardust<br/>    - Dr. Quasar<br/>    - Lady Nebula<br/>    - Sir Supernova<br/>    - Ms. Comet<br/>    <br/>    Meeting called to order by Captain Stardust at 2:05 PM<br/>    <br/>    1. Introductions and welcome to our newest team member, Ms. Comet<br/>    <br/>    2. Discussion of our recent mission to Planet Zog<br/>    - Captain Stardust: "Overall, a success, but communication with the Zogians was difficult. We need to improve our language skills."<br/>    - Dr. Quasar: "Agreed. I'll start working on a Zogian-English dictionary right away."<br/>    - Lady Nebula: "The Zogian food was out of this world, literally! We should consider having a Zogian food night on the ship."<br/>    <br/>    3. Addressing the space pirate issue in Sector 7<br/>    - Sir Supernova: "We need a better strategy for dealing with these pirates. They've already plundered three cargo ships this month."<br/>    - Captain Stardust: "I'll speak with Admiral Starbeam about increasing patrols in that area.<br/>    - Dr. Quasar: "I've been working on a new cloaking technology that could help our ships avoid detection by the pirates. I'll need a few more weeks to finalize the prototype."<br/>    <br/>    4. Review of the annual Intergalactic Bake-Off<br/>    - Lady Nebula: "I'm happy to report that our team placed second in the competition! Our Martian Mud Pie was a big hit!"<br/>    - Ms. Comet: "Let's aim for first place next year. I have a secret recipe for Jupiter Jello that I think could be a winner."<br/>    <br/>    5. Planning for the upcoming charity fundraiser<br/>    - Captain Stardust: "We need some creative ideas for our booth at the Intergalactic Charity Bazaar."<br/>    - Sir Supernova: "How about a 'Dunk the Alien' game? We can have people throw water balloons at a volunteer dressed as an alien."<br/>    - Dr. Quasar: "I can set up a 'Name That Star' trivia game with prizes for the winners."<br/>    - Lady Nebula: "Great ideas, everyone. Let's start gathering the supplies and preparing the games."<br/>    <br/>    6. Upcoming team-building retreat<br/>    - Ms. Comet: "I would like to propose a team-building retreat to the Moon Resort and Spa. It's a great opportunity to bond and relax after our recent missions."<br/>    - Captain Stardust: "Sounds like a fantastic idea. I'll check the budget and see if we can make it happen."<br/>    <br/>    7. Next meeting agenda items<br/>    - Update on the Zogian-English dictionary (Dr. Quasar)<br/>    - Progress report on the cloaking technology (Dr. Quasar)<br/>    - Results of increased patrols in Sector 7 (Captain Stardust)<br/>    - Final preparations for the Intergalactic Charity Bazaar (All)<br/>    <br/>    Meeting adjourned at 3:15 PM. Next meeting scheduled for March 19th, 2050 at 2:00 PM in Conference Room 3B, Intergalactic Headquarters.</code></pre>|

**Sample response**

```
Overall Summary of Discussion:
The team discussed the recent mission to Planet Zog, the space pirate issue in Sector 7, the annual Intergalactic Bake-Off, planning for the upcoming charity fundraiser, and a proposed team-building retreat. The mission to Planet Zog was successful, but communication with the Zogians was challenging. The team also discussed the need for a better strategy to deal with space pirates. The team placed second in the Intergalactic Bake-Off and is planning for the upcoming charity fundraiser. A team-building retreat to the Moon Resort and Spa was proposed.

Action Items:
1. Dr. Quasar to start working on a Zogian-English dictionary.
2. Captain Stardust to speak with Admiral Starbeam about increasing patrols in Sector 7.
3. Dr. Quasar to finalize the prototype of the new cloaking technology.
4. All team members to start gathering supplies and preparing games for the Intergalactic Charity Bazaar.
5. Captain Stardust to check the budget for the proposed team-building retreat to the Moon Resort and Spa.

Topics for Next Meeting:
1. Update on the Zogian-English dictionary.
2. Progress report on the cloaking technology.
3. Results of increased patrols in Sector 7.
4. Final preparations for the Intergalactic Charity Bazaar.
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
      "role": "system",
      "content": "You will be provided with meeting notes, and your task is to summarize the meeting as follows:\n    \n    -Overall summary of discussion\n    -Action items (what needs to be done and who is doing it)\n    -If applicable, a list of topics that need to be discussed more fully in the next meeting."
    },
    {
      "role": "user",
      "content": "Meeting Date: March 5th, 2050\n    Meeting Time: 2:00 PM\n    Location: Conference Room 3B, Intergalactic Headquarters\n    \n    Attendees:\n    - Captain Stardust\n    - Dr. Quasar\n    - Lady Nebula\n    - Sir Supernova\n    - Ms. Comet\n    \n    Meeting called to order by Captain Stardust at 2:05 PM\n    \n    1. Introductions and welcome to our newest team member, Ms. Comet\n    \n    2. Discussion of our recent mission to Planet Zog\n    - Captain Stardust: \"Overall, a success, but communication with the Zogians was difficult. We need to improve our language skills.\"\n    - Dr. Quasar: \"Agreed. I'\''ll start working on a Zogian-English dictionary right away.\"\n    - Lady Nebula: \"The Zogian food was out of this world, literally! We should consider having a Zogian food night on the ship.\"\n    \n    3. Addressing the space pirate issue in Sector 7\n    - Sir Supernova: \"We need a better strategy for dealing with these pirates. They'\''ve already plundered three cargo ships this month.\"\n    - Captain Stardust: \"I'\''ll speak with Admiral Starbeam about increasing patrols in that area.\n    - Dr. Quasar: \"I'\''ve been working on a new cloaking technology that could help our ships avoid detection by the pirates. I'\''ll need a few more weeks to finalize the prototype.\"\n    \n    4. Review of the annual Intergalactic Bake-Off\n    - Lady Nebula: \"I'\''m happy to report that our team placed second in the competition! Our Martian Mud Pie was a big hit!\"\n    - Ms. Comet: \"Let'\''s aim for first place next year. I have a secret recipe for Jupiter Jello that I think could be a winner.\"\n    \n    5. Planning for the upcoming charity fundraiser\n    - Captain Stardust: \"We need some creative ideas for our booth at the Intergalactic Charity Bazaar.\"\n    - Sir Supernova: \"How about a '\''Dunk the Alien'\'' game? We can have people throw water balloons at a volunteer dressed as an alien.\"\n    - Dr. Quasar: \"I can set up a '\''Name That Star'\'' trivia game with prizes for the winners.\"\n    - Lady Nebula: \"Great ideas, everyone. Let'\''s start gathering the supplies and preparing the games.\"\n    \n    6. Upcoming team-building retreat\n    - Ms. Comet: \"I would like to propose a team-building retreat to the Moon Resort and Spa. It'\''s a great opportunity to bond and relax after our recent missions.\"\n    - Captain Stardust: \"Sounds like a fantastic idea. I'\''ll check the budget and see if we can make it happen.\"\n    \n    7. Next meeting agenda items\n    - Update on the Zogian-English dictionary (Dr. Quasar)\n    - Progress report on the cloaking technology (Dr. Quasar)\n    - Results of increased patrols in Sector 7 (Captain Stardust)\n    - Final preparations for the Intergalactic Charity Bazaar (All)\n    \n    Meeting adjourned at 3:15 PM. Next meeting scheduled for March 19th, 2050 at 2:00 PM in Conference Room 3B, Intergalactic Headquarters."
    }
  ],
  "temperature": 0.7,
  "max_tokens": 64,
  "top_p": 1
}'
```
