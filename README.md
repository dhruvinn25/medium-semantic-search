# Medium Semantic Search

Homework 11 for SJSU DATA 226, by Dhruvin Patel. This project runs an Airflow DAG in Docker to download a Medium article dataset, create sentence embeddings, store them in Pinecone, and search for related articles.

Repository: [dhruvinn25/medium-semantic-search](https://github.com/dhruvinn25/medium-semantic-search)

Report: [Homework 11.pdf](Homework%2011.pdf), including the dependency graph and sample search results.

## Assignment files

```text
.env.example                    Local setup template; no real credentials
.gitignore                      Excludes credentials and generated files
docker-compose.yaml             Airflow and PostgreSQL services
dags/build_pinecone_search.py    Medium_to_Pinecone DAG
Homework 11.pdf                 Assignment report
```

The assignment requires the Compose file to include `sentence-transformers` and `pinecone`, a Pinecone API key stored as an Airflow Variable, a successful DAG run with a screenshot of task dependencies, and the repository link in the report. It also instructs restarting with `docker compose down` and `docker compose up`; it does not explicitly require restart screenshots.

## Setup

Use Docker Desktop with Linux containers and Docker Compose v2. The original assignment environment used Airflow 2.10.1 and Python 3.12. Compose pins `sentence-transformers==6.3.0` and `pinecone==10.3.0`, the versions recorded in that session. Downloading the embedding dependencies can take several minutes and substantial disk space.

1. Clone this repository and open PowerShell in its root directory.
2. Copy the configuration template:

   ```powershell
   Copy-Item .env.example .env
   ```

3. Generate a private Fernet encryption key:

   ```powershell
   docker run --rm --entrypoint python apache/airflow:2.10.1 -c "from cryptography.fernet import Fernet; print(Fernet.generate_key().decode())"
   ```

4. Edit `.env`: replace `AIRFLOW_FERNET_KEY` with the generated key and set a private `AIRFLOW_ADMIN_PASSWORD`. Keep this file private. Retain the same Fernet key across restarts so existing encrypted credentials remain readable.
5. Validate and start the services:

   ```powershell
   docker compose config --quiet
   docker compose up -d
   docker compose ps
   ```

6. Open [Airflow locally](http://localhost:8081) and sign in with the username and password configured in `.env`.
7. Create a Pinecone account and obtain an API key. In Airflow, open **Admin → Variables** and create the key `pinecone_api_key` with your Pinecone API key as its value. Keep that value masked in screenshots.

On a new clone, Compose creates its own PostgreSQL metadata volume. Airflow Variables from another course checkout are not automatically copied into it. Stop another instance using port 8081 before starting this one.

## Run the DAG

Unpause and manually trigger `Medium_to_Pinecone`. Wait for the run and all five tasks to succeed, then open Graph to capture their dependencies. Open the `test_search_query` logs to inspect the five returned matches.

```text
download_data → preprocess_data ─────────────┐
                                            ├→ generate_embeddings_and_upsert → test_search_query
create_pinecone_index ───────────────────────┘
```

The DAG uses `all-MiniLM-L6-v2` on CPU, a 384-dimensional dot-product index named `semantic-search-fast` in AWS `us-east-1`, and the sample query `what is ethics in AI`. It is scheduled every seven days with catchup disabled. One active run is allowed at a time because runs share the temporary dataset files and Pinecone index.

The index is reused when it already exists. Use a dedicated assignment index with the stated dimension and metric; reusing an index does not reset old vectors.

To restart after changing the Docker configuration:

```powershell
docker compose down
docker compose up -d
```

Leave off `-v` when stopping: that option deletes the metadata volume, including stored Airflow settings. If parsing fails, inspect:

```powershell
docker compose exec airflow airflow dags list-import-errors
docker compose logs --tail 100 airflow
```

The heavy SentenceTransformer import is inside the embedding and search tasks so the scheduler does not repeatedly load it while parsing the DAG. The report records a successful run from the original assignment session; the cleaned configuration should be checked in your local Docker environment before a new demonstration.

## Source and changes

Adapted from the [professor's week 13 Pinecone DAG](https://github.com/keeyong/sjsu-data226-FA26/blob/main/week13/build_pinecone_search.py). The submitted implementation reuses an existing index instead of deleting it, supplies `timeout=-1` to index creation, and uses the explicit `vectors=` upsert argument. Cleanup also moves the model import into tasks and limits simultaneous runs. The Compose file retains its Apache license notice.

Keep API keys, private keys, `.env`, runtime logs, and downloaded data out of Git. Earlier coursework files and private-key files have been removed from the current tree. Historical commits may still contain them; any exposed key must be revoked or replaced wherever it was registered. This homework cleanup does not rotate credentials or rewrite Git history.
