### Grad School Weekly Hours - CSE 6242 Data & Visual Analytics

<hr>

**Type:** Data Viz

**Description:** A simple graph that shows how much time goes into just 3 graduate credits at Georgia Tech. I use a technique called [Pomodoro](https://en.wikipedia.org/wiki/Pomodoro_Technique) when studying/working. It allows me to keep track of how many hours I'm _really_ spending on school work per week - it's also very effective for staying focused on getting stuff down, so I'd highly recommend trying it out.

**Built With:** R

Here is what my Spring 2020 semester looked like - this is only strict pomodoro time, and does not include extra reading or supplemental material. When someone asks me why I only take 3 credits per semester on top of working full-time, this is what I'll show them.

<img src="png\DVAWeeklyHours.png" width="500" height="390" />

### NBA Player Defensive Comparisons

<hr>

**Type:** Website/Application

**Description:** Defensive player comparison analysis for current NBA players - This was a group project for my CSE 6242 Data & Visual Analytics course in Spring 2020 and included four other classmates.

The application used a KNN model and t-SNE to visualize similar NBA players, but only considered defensive stats. The stats included normal box score stats, and shots defended by distance. All data was scraped from [stats.nba.com](https://stats.nba.com).

**Built With:** R and Shiny

[GitHub Repository](https://github.com/HyunTruth/CSE6242-S20-PRJ-NBA-frontend)

[Live Application](https://spelkofer.shinyapps.io/DefensivePlayerComparisons/)

<img src="png\ShotZone.png" width="300" height="250" />
<img src="png\tSNE.png" width="300" height="250" />

### Google Cloud & NCAA® ML Competition 2019-Men's

<hr>

**Description:** Annual Kaggle competition for the men's and women's basketball NCAA tournaments. This was my first time competing in a Kaggle competition - Out of the 866 teams, I placed 444 (52nd percentile). About 90% of my coding time was spent doing data-prep and figuring out what features I could use. I ended up using a Logistic Regression classifier. At the time, my machine learning background was not as strong, but I'm looking forward to competing in 2021 with an advanced skill-set.

**Built With:** Python

[GitHub Repository](https://github.com/spelk24/kaggle_NCAA_2019)

[Competition Link](https://www.kaggle.com/c/mens-machine-learning-competition-2019)


### FiveThirtyEight Web-Scraping

<hr>

**Type:** Web-Scraping demo and Data Viz

**Description:** For an assignment in CSE 6242 - Computing for Data Analysis at Georgia Tech, we were tasked with scraping raw html from [fivethiryeight.com](fivethirtyeight.com). If you're not familar with fivethirtyeight, they are a Data Journalism reporting outlet for topics on politics, economics, sports, science, and culture. Almost all of their stories come with some sort of data analysis or visualization in the article.

On the right side of their homepage, they have a Presidential Approval chart that is interactive and updated daily. Our task was to scrape the raw numbers within this chart (example below) for every single day. After that, I decided to try and recreate the visualization as best as I could using matplotlib in Python.

For a more detailed analysis and tutorial, please check out the jupyter notebook in the GitHub repository linked below.

**Built With:** Python

[GitHub Repository](https://github.com/spelk24/FiveThirtyEight-Web-Scraping)

<img src="png\fivethirtyeight_ex.png" width="300" height="250" />
<img src="png\fivethirtyeight_matplotlib.png" width="300" height="250" />