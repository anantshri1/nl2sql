# Natural Language-to-SQL Agentic Pipeline

A from-scratch end-to-end agentic NL→SQL pipeline built with `LangChain`, `Claude`, and `Voyage AI` — evaluated on the BIRD Mini-Dev benchmark.

---
## Manual NL→SQL chain

### Core Agent on [Chinook Database](https://github.com/lerocha/chinook-database)

Dependencies installed: `langchain, langchain-anthropic, langchain-community, sqlalchemy`.

The original plan was to download a prebuilt `Chinook.sqlite` file from a Google Cloud Storage bucket referenced in LangChain's own tutorial, but that asset no longer exists (confirmed via a `NoSuchKey` XML error after inspecting the tiny downloaded file). The fix was to build the database differently and more durably: fetch the actual SQL script (`CREATE TABLE` and `INSERT` statements as plain text) from the actively maintained `lerocha/chinook-database` GitHub repository, execute it into an in-memory `SQLite` connection via Python's `sqlite3` module, and wrap that connection in a `SQLAlchemy` engine using StaticPool with a creator lambda. This was necessary because an in-memory SQLite database is tied to a single connection object — without `StaticPool` forcing reuse of that exact connection, the data would vanish after the first query. The resulting engine was wrapped in LangChain's `SQLDatabase` utility, which exposes the schema (`table names, columns, sample rows`) as a text string via `get_table_info` — this is how the LLM later learns what's in the database.

The first real NL-to-SQL chain was then built using `LangChain` Expression Language: 
* a `ChatPromptTemplate` piped into the LLM (`prompt | llm`), with the database schema and dialect injected into the prompt and the natural-language question as the human message;
* a `clean_sql` helper function was added to strip markdown code fences, since LLMs sometimes wrap SQL in triple-backtick fences despite explicit instructions not to;
* an `ask_db` function combined these: build the schema, invoke the chain, clean the output, execute it via `db.run`, and return the result. 

> Tested successfully on "How many tracks are there in total?", returning a correct count.

---
### Self-Correction Loop

The correction logic was built and tested in isolation before being wired into a full loop, specifically because Claude at `temperature 0` reliably succeeds on Chinook's small schema on the first attempt — meaning a full retry loop run on normal questions might never actually exercise the correction path. To force a test of the correction path, a deliberately broken query (`SELECT COUNT(*) FROM Tracks`, using the wrong plural table name) plus its real `SQLite error` message were manually fed into a correction chain.

With the correction logic validated, it was wired into `ask_db_with_retry`: generate an initial query, then loop up to `max_retries` times, attempting execution via `db.run` inside a try block; on any exception, the error message is fed into the correction chain to produce a new query for the next attempt. A successful run returns immediately from inside the loop; exhausting all retries raises a RuntimeError rather than silently returning a wrong answer. Tested successfully on Chinook, succeeding on the first attempt with no correction triggered (expected, given Chinook's simplicity).

---



---
## References
* **Can LLM Already Serve as A Database Interface? A BIg Bench for Large-Scale Database Grounded Text-to-SQLs**, Jinyang Li, Binyuan Hui, Ge Qu, Jiaxi Yang, Binhua Li, Bowen Li, Bailin Wang, Bowen Qin, Rongyu Cao, Ruiying Geng, Nan Huo, Xuanhe Zhou, Chenhao Ma, Guoliang Li, Kevin C.C. Chang, Fei Huang, Reynold Cheng, Yongbin Li. (2023). arXiv: [2305.03111](https://arxiv.org/abs/2305.03111)
* **A Survey of Text-to-SQL in the Era of LLMs: Where are we, and where are we going?**, Xinyu Liu, Shuyu Shen, Boyan Li, Peixian Ma, Runzhi Jiang, Yuxin Zhang, Ju Fan, Guoliang Li, Nan Tang, Yuyu Luo. (2024). arXiv: [2408.05109v6](https://arxiv.org/abs/2408.05109v6)
* **From Natural Language to SQL: Review of LLM-based Text-to-SQL Systems**, Ali Mohammadjafari, Anthony S. Maida, Raju Gottumukkala. (2024). arXiv: [2410.01066v1](https://arxiv.org/abs/2410.01066v1)
