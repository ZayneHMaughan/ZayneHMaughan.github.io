# NLP_FINAL
## Supreme Court Retrival Augmented Generation Tool
This research guide was designed to parse through various legal topics to 
better understand the Supreme Court's decision-making.

## Installation
You can clone this repository to your own local machine 
```bash
git clone https://github.com/ZayneHMaughan/NLP_FINAL
```
Install Required dependencies
```bash
pip install -r requirements.txt
```
After doing so, you can run the chainlit app by running
```bash
chainlit chainlit_app.py
```

This is intended to be a modular system following a similar system as a Streamlit app.

### Data
1. Pre-Processed data, and PDFs are found in the documents folder
2. Argument Transcripts are found from the Supreme Court website.
3. Metadata and embeddings for document retrieval are pre-generated for offline use.

## USE CASES 

### Example 1: Querying a Source

#### Query
```
What did Source one say about the Sixth Amendment?
```
#### Response:
```
Source 1 discusses the Sixth Amendment rights of a defendant, 
particularly in relation to the opportunity for defense counsel
to have important conversations with their client. 
It suggests that if a defendant's testimony were conducted 
continuously without a break, the opportunity for such discussions, 
including those about potential perjury, would not arise. 
The source indicates that the notion of Sixth Amendment rights 
turning on the "fortuity of that recess" strikes them as far-fetched [Source 1]. 
Justice Sotomayor notes that this is indeed what was done,
regardless of whether it is liked [Source 1].
```
### Example 2: Understanding Themes

#### Query
```
What were the main themes of the cases the Supreme Court saw?
```
#### Response:
```
Based on the provided context, the main themes of the cases the
 Supreme Court heard revolve around the Sixth Amendment rights 
 of defendants and the assistance of counsel. Specifically,
  the discussions touch upon:
The ability of defense counsel to have crucial conversations
 with their clients: Source 1 and Source 2 highlight the 
 importance of these discussions, suggesting that they are 
 basic and necessary for competent legal representation 
 guaranteed by the Sixth Amendment.
The impact of recesses on these conversations: Source 1 
discusses the idea that the opportunity for such discussions 
might arise "fortuitously" due to recesses. The court appears 
to be grappling with whether Sixth Amendment rights should 
depend on such chance occurrences.
The necessity of discussing testimony: Source 2 emphasizes that 
defendants and their counsel cannot have these vital conversations 
if they are not allowed to discuss the defendant's testimony. 
This includes situations where counsel might need to advise a defendant to 
accept a plea agreement if their testimony has gone poorly.
The context also indicates that these discussions are part of oral arguments 
for specific cases, as seen in the case titles and appearances listed in Source 3 
(Dwayne Barrett v. United States), Source 4 (Holsey Ellingburg, Jr. v. United States), 
and Source 5 (Donald J. Trump, President of the United States, et al. v. Rebecca Kelly Slaughter, et al.).
```
### Example 3: RAG Document Query

#### Query
```
Which Justices asked question in case 24-5774


Used
query_rag
```
#### Response
```
Answer:
In case 24-5774, the Justices who asked questions were Justice Kavanaugh, Chief Justice Roberts, and Justice Jackson (Source 3).
📚 Sources: 24-5774_6jf7, 24-557_5ie6
📊 Retrieved: 10 chunks → Reranked: 5 → Used: 5
```
### Example 4: Web Search

#### Query:

```
web: how many justices went to harvard?
```
#### Response:
```
Used
search_web

1. [PDF] The Legal Education of the Justices from Holmes to Kagan
2. Where Supreme Court Justices Earned Law Degrees - USNews.com
3. A Conversation with Six Justices of the U.S. Supreme Court - YouTube
```
### Database 

#### Query:
```
db:
db: was the most recent case a split vote?
Avatar for query_db_nl
```
#### Response:
```
Used
query_db_nl
```
```json

{
  "success": true,
  "sql": "SELECT\n  CASE WHEN T2.splitVote = 1 THEN 'Yes' ELSE 'No' END\nFROM Cases AS T1\nINNER JOIN Votes AS T2\n  ON T1.caseId = T2.caseId\nORDER BY\n  T1.dateDecision DESC\nLIMIT 1;",
  "data": [
    {
      "CASE WHEN T2.splitVote = 1 THEN 'Yes' ELSE 'No' END": "Yes"
    }
  ],
  "columns": [
    "CASE WHEN T2.splitVote = 1 THEN 'Yes' ELSE 'No' END"
  ],
  "row_count": 1,
  "error": null
}
```

### References
Data was from Washington University Law http://scdb.wustl.edu/data.php
Documents were the Argument Transcripts found here: https://www.supremecourt.gov/oral_arguments/argument_transcript/2025