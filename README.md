---
title: "Smart device fitness data analysis unlocking new growth opportunities."
author: "Created by: David Huynh"
output: 
  github_document:
    toc: true
    toc_depth: 3
---

# Context

Bellabeat, founded in 2013 by Urška Sršen and Sando Mur, is a tech-driven wellness company that creates health-focused smart products designed for women, empowering them with insights into their well-being.

By 2016, the company had expanded globally, launched multiple products, and increased its availability through online retailers and its e-commerce website. Bellabeat utilizes both traditional and digital marketing, investing in Google Search, social media (Facebook, Instagram, Twitter), YouTube video ads, and Google Display Network campaigns.

The marketing analytics team has been tasked with providing insights that can inform Bellabeat’s future marketing strategy and growth opportunities. This analysis follows the data analytics framework, including 6 steps: [Ask], [Prepare], [Process], [Analyze], [Share], and [Act].

# Ask {data-link="Ask"}

### 1. Identify the business task

The marketing analytics team was asked to focus on one Bellabeat’s product and analyze smart device usage data in order to gain insight into how people are already using their smart devices and present high-level recommendations for how these trends can inform Bellabeat marketing strategy.

### 2. Consider key stakeholders

Key stakeholders include: Urška Sršen: Bellabeat’s cofounder and Chief Creative Officer Sando Mur: Mathematician and Bellabeat’s cofounder Bellabeat marketing analytics team Consumers who are using smart devices from Bellabeat Potential customers who are going to buy Bellabeat’s products

### 3. Clarify the business statement

Analyzing smart device fitness data could help unlock new growth opportunities for the company by discovering underlying patterns, trends and potential connection. The scope of this analysis focus on only one aspect of the dataset. It is expected that the recommendations could shed light on the improvements in individual’s general health by using smart devices from Bellabeat.

# Prepare {data-link="Prepare"}

The Fitbit Fitness Tracker Data is made available online through Mobius’s repository on Kaggle. This analysis only uses the data from 12th April, 2016 to 12th May, 2016.

The data is organized as a set of tables with specific tracker data such as physical activity, heat rate, and sleep monitoring in long format. The first columns of all tables are users’ ID.

The data is quite ROCCC because it is comprehensive and managed by a reliable platform. However, there could be several issues in terms of bias and credibility. Only a small number of records may not reflect the overall themes while the data was registered in 2016 which could be different from current contexts when the presence of smart devices are increasing rapidly. Moreover, as a third-party data source, careful considerations should be taken under specific circumstances.

The dataset contains personal fitness records from about 30 users who consented to the submission of personal tracker data for the purpose of explore users’ habits. The author also published the dataset in his public domain without licensing requirement so that everyone can easily get access and perform further data mining and exploration.

There are 18 datasets containing periodical records in different intervals. However, this analysis only look further into daily activity and sleeping records, as well as weigh-logging data.

```{r}
# install.packages("tidyverse")
# install.packages("skimr")
# install.packages("janitor")
# install.packages("ggplot2")
# install.packages("dplyr")
# install.packages("corrplot")
library(tidyverse)
library(skimr)
library(janitor)
library(ggplot2)
library(dplyr)
library(corrplot)

## Load CSV files ##
setwd("/cloud/project/database/FitabaseData2")
daily_activity <- read.csv("dailyActivity_merged.csv")
sleep_day <- read.csv("sleepDay_merged.csv")
weight_log <- read.csv("weightLogInfo_merged.csv")
hourly_steps <- read.csv("hourlySteps_merged.csv")
hourly_calories <- read.csv("hourlyCalories_merged.csv")
```

# Process {data-link="Process"}

In this step, we will check the data for error with appropriate tools. After that, transforming the data is preferred to let us work with it effectively. Now, let's explore key data tables.

**Table**: daily_activity

```{r}
head(daily_activity)
skim_without_charts(daily_activity)

#As the distances are not of interest, we will remove them from the original table The format of Id and ActivityDate should be corrected as nominal and date instead of numeric and character
activity_df <- daily_activity %>% 
               select(Id,ActivityDate,TotalSteps,VeryActiveMinutes,
                          FairlyActiveMinutes,LightlyActiveMinutes,
                          SedentaryMinutes,Calories)%>%
                rename(Date = ActivityDate, Steps = TotalSteps,
                       VeryActive = VeryActiveMinutes,
                       FairlyActive = FairlyActiveMinutes,
                       LightlyActive = LightlyActiveMinutes,
                       Sedentary = SedentaryMinutes)
activity_df$Id <- as.character(activity_df$Id)
activity_df$Date <- format(as.Date(activity_df$Date, format="%m/%d/%Y"), "%m/%d/%Y")
```

**Table**: sleep_day

```{r}
head(sleep_day)
skim_without_charts(sleep_day)
#As the TotalSleepRecords is redundant, we will ignore it.
#Again, the format of Id and ActivityDate should be corrected as nominal and date instead of numeric and character. These concept will also be applied for other tables. Moreover, the recorded time remains unchanged, making it less interesting to be kept.
sleep_df <- sleep_day %>% 
            select(-TotalSleepRecords)%>%
            rename(Date = SleepDay)
sleep_df$Id <- as.character(sleep_df$Id)
sleep_df$Date <- format(as.Date(sleep_df$Date, format="%m/%d/%Y %I:%M:%S %p"), format="%m/%d/%Y")
```

**Table**: hourly_steps

```{r}
head(hourly_steps)
# We can apply the same process for the hourly steps and calories data, but the recorded hours matter. Therefore, we separated the date and hour in order to join or summarise data.
hourly_steps$ActivityHour <- as.POSIXct(hourly_steps$ActivityHour, format="%m/%d/%Y %I:%M:%S %p")
steps_df <- hourly_steps %>% 
          mutate(hourly_steps, Date = format(ActivityHour, "%m/%d/%Y"),
                  Hour = format(ActivityHour, "%I:%M:%S %p")) %>%
          select(-ActivityHour )
steps_df$Id <- as.character(steps_df$Id)
```

**Table**: hourly_calories

```{r}
head(hourly_calories)
hourly_calories$ActivityHour <- as.POSIXct(hourly_calories$ActivityHour, format="%m/%d/%Y %I:%M:%S %p")
calo_df <- hourly_calories %>% 
  mutate(hourly_calories, Date = format(ActivityHour, "%m/%d/%Y"),
         Hour = format(ActivityHour, "%I:%M:%S %p")) %>%
  select(-ActivityHour )
calo_df$Id <- as.character(calo_df$Id)
```

**Table**: weight_log

```{r}
head(weight_log)
#For the weight data, the measurements in Pounds are not informative as we already have the weights in Kg.
#It is also significant that the majority of Fat indicators is NA while the LogId 
#is not quite useful.
weight_df <- weight_log %>% 
  select(Id, Date, WeightKg, IsManualReport, BMI)
weight_df$Id <- as.character(weight_df$Id)
weight_df$Date <- as.POSIXct(weight_df$Date , format="%m/%d/%Y %I:%M:%S %p")
```

Additionally, we may check missing values in each dataset.

```{r}
colSums(is.na(activity_df))
colSums(is.na(sleep_df))
colSums(is.na(weight_df))
colSums(is.na(steps_df))
colSums(is.na(calo_df))
```

Finally, we want to remove duplicates if any

```{r}
activity_df <- activity_df %>% distinct()
sleep_df <- sleep_df %>% distinct()
weight_df <- weight_df %>% distinct()
steps_df <- steps_df %>% distinct()
calo_df <- calo_df %>% distinct()
```

# Analyze {data-link="Analyze"}

Once the data is nicely prepared and processed, we can dive in the analyzing step. In this step, firstly, let's find out the summary statistics in each dataset.

```{r}
# activity data.
n_distinct(activity_df$Id)
nrow(activity_df)
activity_df %>%
  select(Steps, VeryActive, FairlyActive, LightlyActive, Sedentary, Calories)%>%
  summary()

# sleep_day data.
n_distinct(sleep_df$Id)
nrow(sleep_df)
sleep_df %>%
  select(TotalMinutesAsleep , TotalTimeInBed)%>%
  summary()

# weight_log data.
n_distinct(weight_df$Id)
nrow(weight_df)
weight_df %>%
  select(WeightKg, BMI)%>%
  summary()

# steps data
n_distinct(steps_df$Id)
nrow(steps_df)
steps_df %>%
  select(StepTotal) %>%
  summary()

# calories data
n_distinct(calo_df$Id)
nrow(calo_df)
calo_df %>%
  select(Calories) %>%
  summary()
```

Initial findings:

1.  jkhjkl

2.  223

3.  ewds

After identifying key findings from the initial analysis, we can delve deeper into patterns and potential concerns related to user behavior. This further analysis will focus on trends in activity levels, sleep effectiveness, highlighting any significant insights.

```{r}
# 0. The relationship between active time, steps vs calories?
# Select the relevant columns
active_level <- activity_df %>%
  select(VeryActive, FairlyActive, LightlyActive, Sedentary, Steps, Calories)
  
# Calculate correlation matrix and plot
cor_matrix <- cor(active_level)
corrplot(cor_matrix, method = 'color',addCoef.col = 'grey25')
```

# Share {data-link="Share"}

# Act {data-link="Act"}

################################################ 

```{r}
## Explore key tables ##



# 1. What is the average daily step count? Are users meeting the 10,000-step recommendation?
#   Refer to previous summary
# 2. How many hours do users sleep on average?
#   Refer to previous summary

# 3. Active levels during the day and in a week:
# 3.1 What time of day are users most active?
steps_by_hour <- steps_df %>%
  group_by(Hour) %>%
  summarize(mean_steps = mean(StepTotal, na.rm = TRUE)) %>%
  mutate(NumericHour = as.numeric(format(as.POSIXct(Hour, format="%I:%M:%S %p"), "%H")))%>%
  arrange(NumericHour)

ggplot(data = steps_by_hour) +
  geom_col(mapping=aes(y = mean_steps, x = reorder(Hour, NumericHour))) +
  theme(axis.text.x = element_text(angle = 90)) +
  labs(title = "Average Steps Taken per Hour of Day",
       x = "Hour", y = "Average Steps")
       
# 3.2 Are users more active on weekdays vs. weekends?
activity_df <- mutate(activity_df, Weekday = weekdays(as.Date(activity_df$Date, format="%m/%d/%Y")))
activity_df$Weekday <- factor(activity_df$Weekday , levels = c("Monday", "Tuesday", "Wednesday", 
                                                               "Thursday", "Friday", "Saturday", "Sunday"))
weekly_steps <- activity_df %>%
  group_by(Weekday) %>%
  summarise(AverageSteps = mean(Steps, na.rm = TRUE))

ggplot(weekly_steps, aes(x = Weekday, y = AverageSteps, fill = Weekday)) +
  geom_bar(stat = "identity") +
  geom_text(aes(label = round(AverageSteps, 0)), vjust = -0.5) +
  labs(title = "Average Steps per Weekday",
       x = "Weekday",
       y = "Average Steps") 

# 3.3 Do users sleep more on weekends vs. weekdays?
sleep_df <- mutate(sleep_df, Weekday = weekdays(as.Date(sleep_df$Date, format="%m/%d/%Y")))
sleep_df$Weekday <- factor(sleep_df$Weekday , levels = c("Monday", "Tuesday", "Wednesday", 
                                                               "Thursday", "Friday", "Saturday", "Sunday"))
weekly_sleep <- sleep_df %>%
  group_by(Weekday) %>%
  summarise(AverageSleep = mean(TotalMinutesAsleep, na.rm = TRUE))

ggplot(weekly_sleep, aes(x = Weekday, y = AverageSleep, fill = Weekday)) +
  geom_bar(stat = "identity") +
  geom_text(aes(label = round(AverageSleep/60, 2)), vjust = -0.5) +
  labs(title = "Sleep interval per Weekday",
       x = "Weekday",
       y = "Average sleeping hours")

# 4. Relationship:
# 4.1 Are users with higher activity levels burning significantly more calories?
ggplot(data = activity_df, mapping = aes(x=Steps, y=Calories)) + 
  geom_point(color = "orange") + geom_smooth(color = "red") +
  labs(title = "Total Steps vs. Calories")

# 4.2 Do people fall asleep easily when in bed?
sleep_df <- sleep_df %>%
  mutate(sleep_eff = round(TotalMinutesAsleep / TotalTimeInBed * 100, 1))

hist(sleep_df$sleep_eff, breaks = 20,
     main = "Distribution of Sleep Efficiency", 
     xlab = "Sleep Efficiency (%)")

ggplot(data = sleep_df, mapping = aes(x=TotalTimeInBed, y=TotalMinutesAsleep)) + 
  geom_point(color = "blue") + geom_smooth(color = "red") +
  labs(title = "Time in bed vs. Time Asleep",
       x = 'Time in bed (mins)',
       y = 'Time asleep (mins)')
  
# 4.3 Do people who are more sedentary sleep better?
# Merging these two datasets together ##
combined_data <- merge(activity_df, sleep_df, by = c("Id", "Date"), all = FALSE)
n_distinct(combined_data$Id)

ggplot(combined_data, aes(x = Sedentary, y = TotalMinutesAsleep)) +
  geom_point() +
  geom_smooth() +
  labs(
    title = "Relationship Between Sedentary Time and Time Asleep",
    x = "Sedentary Time (Minutes)",
    y = "Time Asleep (Minutes)"
  )

```
