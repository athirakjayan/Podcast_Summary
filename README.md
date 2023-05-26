# Podcast_Summary
A data pipeline using Airflow to download and summarize podcast


# Project Overview
In this project, we'll create a data pipeline using Airflow. The pipeline will download podcast episodes and automatically transcribe them using speech recognition. We'll store our results in a SQLite database that we can easily query.

We don't strictly need to use Airflow to do this project, but it helps us with a few things:
<ul>
<li> We can schedule the project to run daily</li>
<li> Each task can run independently, and we get error logs</li>
<li> We can easily parallelize tasks and run in the cloud if we want to</li></li>
<li> We can extend this project more easily (add speech recognition, summaries, etc) using Airflow</li>
</ul>
By the end of this project, we'll have a good understanding of how to use Airflow, along with a practical project that you can continue to build on.

## Project Steps

<ul>
<li>Download the podcast metadata xml and parse</li>
<li>Create a SQLite database to hold podcast metadata</li>
<li>Download the podcast audio files using requests</li>
<li>Transcribe the audio files using vosk</li>
</ul>

# Code
You can find the code for this project <a href = "https://github.com/athirakjayan/Download_Podcast/blob/main/podcast_summary.py">here</a>.

File overview:

<ul>
  <li>podcast_summary.py - the code to create a data pipeline</li>
  <li>steps.md - a description of the steps we'll need to follow to complete the project. It's not perfectly organized.</li>
 </ul>
 
# Tools Required
Installation
To follow this project, please install the following locally:
<ul>
<li>Airflow 2.3+</li>
<li>Python 3.8+</li>
<li>Python packages</li>
<ul>
  <li>pandas</li>
  <li>sqlite3</li>
  <li>xmltodict</li>
  <li>requests</li>
  <li>vosk</li>
  <li>pydub</li>
  </ul>
  </ul>

# Data
We'll download the data we need during this project, including a language model for vosk, and podcast episodes. If you want to view the podcast metadata page, it is <a href="https://www.marketplace.org/feed/podcast/marketplace/">here</a>.
