# Overview

This file describes, in order, the steps involved in doing this project.

## 1: Install airflow and configure
<ul>
  <li>Install Ubuntu subsystem</li>
  <li>Install Airflow</li>
  <li>Run the scheduler</li>
  <li>Run the Webserver</li>
  </ul>
* Edit airflow cfg to point to the folder with the dag you're writing
* Write first task in dag (see code below)

### First task
```py
import xmltodict
import requests

from airflow.decorators import dag, task
import pendulum


PODCAST_URL = "https://www.marketplace.org/feed/podcast/marketplace/"

@dag(
    dag_id='podcast_summary',
    schedule_interval="@daily",
    start_date=pendulum.datetime(2022, 5, 30),
    catchup=False,
)
def podcast_summary():

    @task()
    def get_episodes():
        data = requests.get(PODCAST_URL)
        feed = xmltodict.parse(data.text)
        episodes = feed["rss"]["channel"]["item"]
        print(f"Found {len(episodes)} episodes.")
        return episodes

    podcast_episodes = get_episodes()

summary = podcast_summary()
```

* Trigger task via GUI
    * `airflow dags trigger podcast_summary`
    * Or use GUI to trigger
* Check in web interface
* Check log to see output

## Create database
* Create database
    * Run `sqlite3 episodes.db`
    * Type `.databases` in prompt to create the db
* Run `airflow connections add 'podcasts' --conn-type 'sqlite' --conn-host 'episodes.db'`
*  `airflow connections get podcasts` to view connection info
*  Write second task in dag (see code below)

### Second task
```py
import requests
import xmltodict

from airflow.decorators import dag, task
import pendulum
from airflow.providers.sqlite.operators.sqlite import SqliteOperator

PODCAST_URL = "https://www.marketplace.org/feed/podcast/marketplace/"

@dag(
    dag_id='podcast_summary',
    schedule_interval="@daily",
    start_date=pendulum.datetime(2022, 5, 30),
    catchup=False,
)
def podcast_summary():
    create_database = SqliteOperator(
        task_id='create_table_sqlite',
        sql=r"""
        CREATE TABLE IF NOT EXISTS episodes (
            link TEXT PRIMARY KEY,
            title TEXT,
            filename TEXT,
            published TEXT,
            description TEXT,
            transcript TEXT
        );
        """,
        sqlite_conn_id="podcasts"
    )

    @task()
    def get_episodes():
        data = requests.get(PODCAST_URL)
        feed = xmltodict.parse(data.text)
        episodes = feed["rss"]["channel"]["item"]
        print(f"Found {len(episodes)} episodes.")
        return episodes

    podcast_episodes = get_episodes()
    create_database.set_downstream(podcast_episodes)

summary = podcast_summary()
```

* Trigger run
* Show in GUI

## Load episodes into SQLite
* `pip install 'apache-airflow[pandas]'`
* Add in load episodes function (code below)

### Third task
```py
import requests
import xmltodict

from airflow.decorators import dag, task
import pendulum
from airflow.providers.sqlite.operators.sqlite import SqliteOperator
from airflow.providers.sqlite.hooks.sqlite import SqliteHook

PODCAST_URL = "https://www.marketplace.org/feed/podcast/marketplace/"

@dag(
    dag_id='podcast_summary',
    schedule_interval="@daily",
    start_date=pendulum.datetime(2022, 5, 30),
    catchup=False,
)
def podcast_summary():
    create_database = SqliteOperator(
        task_id='create_table_sqlite',
        sql=r"""
        CREATE TABLE IF NOT EXISTS episodes (
            link TEXT PRIMARY KEY,
            title TEXT,
            filename TEXT,
            published TEXT,
            description TEXT,
            transcript TEXT
        );
        """,
        sqlite_conn_id="podcasts"
    )

    @task()
    def get_episodes():
        data = requests.get(PODCAST_URL)
        feed = xmltodict.parse(data.text)
        episodes = feed["rss"]["channel"]["item"]
        print(f"Found {len(episodes)} episodes.")
        return episodes

    podcast_episodes = get_episodes()
    create_database.set_downstream(podcast_episodes)

    @task()
    def load_episodes(episodes):
        hook = SqliteHook(sqlite_conn_id="podcasts")
        stored_episodes = hook.get_pandas_df("SELECT * from episodes;")
        new_episodes = []
        for episode in episodes:
            if episode["link"] not in stored_episodes["link"].values:
                filename = f"{episode['link'].split('/')[-1]}.mp3"
                new_episodes.append([episode["link"], episode["title"], episode["pubDate"], episode["description"], filename])

        hook.insert_rows(table='episodes', rows=new_episodes, target_fields=["link", "title", "published", "description", "filename"])

    load_episodes(podcast_episodes)

summary = podcast_summary()
```


* Trigger run
* Show in GUI
* View database
    * `sqlite3 episodes.db`

## Download episodes
* `mkdir episodes` to make episodes folder
* Add in download episodes function (code below)

### Fourth task
```py
import os
import json
import requests
import xmltodict

from airflow.decorators import dag, task
import pendulum
from airflow.providers.sqlite.operators.sqlite import SqliteOperator
from airflow.providers.sqlite.hooks.sqlite import SqliteHook

PODCAST_URL = "https://www.marketplace.org/feed/podcast/marketplace/"
EPISODE_FOLDER = "episodes"

@dag(
    dag_id='podcast_summary',
    schedule_interval="@daily",
    start_date=pendulum.datetime(2022, 5, 30),
    catchup=False,
)
def podcast_summary():
    create_database = SqliteOperator(
        task_id='create_table_sqlite',
        sql=r"""
        CREATE TABLE IF NOT EXISTS episodes (
            link TEXT PRIMARY KEY,
            title TEXT,
            filename TEXT,
            published TEXT,
            description TEXT,
            transcript TEXT
        );
        """,
        sqlite_conn_id="podcasts"
    )

    @task()
    def get_episodes():
        data = requests.get(PODCAST_URL)
        feed = xmltodict.parse(data.text)
        episodes = feed["rss"]["channel"]["item"]
        print(f"Found {len(episodes)} episodes.")
        return episodes

    podcast_episodes = get_episodes()
    create_database.set_downstream(podcast_episodes)

    @task()
    def load_episodes(episodes):
        hook = SqliteHook(sqlite_conn_id="podcasts")
        stored_episodes = hook.get_pandas_df("SELECT * from episodes;")
        new_episodes = []
        for episode in episodes:
            if episode["link"] not in stored_episodes["link"].values:
                filename = f"{episode['link'].split('/')[-1]}.mp3"
                new_episodes.append([episode["link"], episode["title"], episode["pubDate"], episode["description"], filename])

        hook.insert_rows(table='episodes', rows=new_episodes, target_fields=["link", "title", "published", "description", "filename"])

    load_episodes(podcast_episodes)

    @task()
    def download_episodes(episodes):
        for episode in episodes:
            name_end = episode["link"].split('/')[-1]
            filename = f"{name_end}.mp3"
            audio_path = os.path.join(EPISODE_FOLDER, filename)
            if not os.path.exists(audio_path):
                print(f"Downloading {filename}")
                audio = requests.get(episode["enclosure"]["@url"])
                with open(audio_path, "wb+") as f:
                    f.write(audio.content)

    download_episodes(podcast_episodes)

summary = podcast_summary()
```

* Trigger task
* Show in GUI

### Transcribe task
```py
 @task()
    def speech_to_text(audio_files, new_episodes):
        hook = SqliteHook(sqlite_conn_id="podcasts")
        untranscribed_episodes = hook.get_pandas_df("SELECT * from episodes WHERE transcript IS NULL;")

        model = Model(model_name="vosk-model-en-us-0.22-lgraph")
        rec = KaldiRecognizer(model, FRAME_RATE)
        rec.SetWords(True)

        for index, row in untranscribed_episodes.iterrows():
            print(f"Transcribing {row['filename']}")
            filepath = os.path.join(EPISODE_FOLDER, row["filename"])
            mp3 = AudioSegment.from_mp3(filepath)
            mp3 = mp3.set_channels(1)
            mp3 = mp3.set_frame_rate(FRAME_RATE)

            step = 20000
            transcript = ""
            for i in range(0, len(mp3), step):
                print(f"Progress: {i/len(mp3)}")
                segment = mp3[i:i+step]
                rec.AcceptWaveform(segment.raw_data)
                result = rec.Result()
                text = json.loads(result)["text"]
                transcript += text
            hook.insert_rows(table='episodes', rows=[[row["link"], transcript]], target_fields=["link", "transcript"], replace=True)

    speech_to_text(audio_files, new_episodes)
    ```
