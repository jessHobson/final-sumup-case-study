# sumup case study
 
## 1. Ingestion and exploration

First file that does basic data exploration and cleaning.  In the interest of lack of time many irregularities were simply dropped because of low proportions,  context and  time. In reality we would try to investigate further with business context and ensure these are dealt with in the right way.  In production we should also always be monitoring these irregularities automatically and flagging when there are unexpected increases 

### Data Cleaning

- Dropped rows with null values in 'RESPONSE_TIME_SECONDS', 'AGENT_COMPANY', 'MCC_GROUP' columns
- Dropped rows where 'AGENT_COMPANY' = 'System' and 'STATUS'] = 'New'
- Dropped outliers with a z-score >3.5 based off of TOTAL_RESOLUTION_TIME_SECONDS

### Data Preprocessing 
- Added in a priority column ranging from 1-3 based on 'REASON_GROUP'
- Grouped touchpoints about the same topics within a 7 day window into a single issue 
- Created TOTAL_RESOLUTION_TIME_SECONDS based on the first touchpoint CREATION_DATE in the window + the last touchpoint CREATION_DATE + QUEUE_WAITING_TIME_SECONDS + TOTAL_HANDLING_TIME_SECONDS
- When touchpoints used multiple channel types, assigned the channel to the channel with the longest TOTAL_RESOLUTION_TIME_SECONDS to minimize outliers

### Table Creation + How to Productionalize
- created two cleaned dataframes , one on a touchpoint level and one on an issue level 
- for the sake of time and ease of analysis these scripts where written in Pandas, in production this would be optimized if written in a language better suited to this e.g. pyspark 
- this example also is run on a whole year's worth of data, in reality this could be incrementally run each day , only needing to look back through 7 days of data according to the resolution window. 

## 2. Issue Level for Window Resolution Recommendation 

### Dealing with multiple touchpoint issues 
- More than 1 touchpoint immediately adds a lot of time to TOTAL_RESOLUTION_TIME_SECONDS as with the current calculation it will also include the wait time in- between when the touchpoint get solved the first time and reopened 
- This is particularly true for call and chat where the overall resolution time is in minutes and time between cases can be a couple of days
- we can see that the majority of issues are solved with a single contact point and that the majority of issues taking more than one contact are email
- email is a channel that is expected to have higher number of contact points as it deals with more complex 
technical issues 
- multiple touchpoint issues for chat and call account for 2.38% of volumes, so in this case they are deleted

### Proposing New Window Resolutions
- Decided to use 90 percentile values here as assuming the resolution window would be the support team's SLA wanted to be sure that a conservative / realistic number is given

#### Resolution Window by Channel
- Started by investigating separate resolution windows per channel which can already reduce resolution time by ~99.7% for ~60% of all cases (chat & call) as they are resolved within minutes not days 
- Email resolution window can also be reduced by 2 days
- Further segmented the resolution windows by priority (1-3) 

#### Resolution Window by Channel by Priority
- Call and email: already correctly have the lowest resolution times for the highest priority issues
- Can see some room for improvement in prioritizng chat resolution windows based on priority
- for email we can greatly reduce the per priority windows: prio 1: 4days, prio 2: 4.5days, prio 3: 5days
- Also considered the queue wait time for chat and email and saw that lowest priority issues have the lower queue wait times for both and some improvement is needed here too


### Supply optimization 
- investigated whether there is also an opportunity to improve supply vs demand during certain hours and days of the week by producing some heatmaps based on queue waiting time - queue waiting time is a good metric because it is a mix of supply (are engouh agents available) to meet the demand (incoming issues) 
- There is room for adding more agent support / reshuffling support times have better capacity during peak times 
 particularly on Tuesday and Friday afternoon.
-Further analyse this based on skills and language

### Reason Groups with Irregular Median Resolution Times 
- Producing at the graphs of median resolution times per reason groups that despite outlier removal and using the median to calculate still have higher than average resolution times 
- Montior these and investigate why for high outliers

### Table Creation + How to Productionalize
- For the resolution window recommendations a dimension table should be created that can be used in future analysis and reporting to track performance.  However there resolution windows should not be frequently / dynamically changing - it should change based on analysis performed by an analyst - sense checking and validating with business stakeholder inputs before updating.
- Create a table with volumes per day per hour with certain dimension e.g. agent_company, country, channel to allow for easy heat map plotting / analysis --> this is something really important for operational teams to closely monitor and generally heavy data to good to have a preaggregate table to work off of

## 3. Performance Scoring

### Model developement
- With this model agent, channel and country specific performance can be easily compared and weak areas targeted.  This is a very basic model and does also stand to be refined relying on addint more dimensions 
The model takes into account: 
The percentage of issues solved under the specified resolution window. 
    - Priority 1 = weighted x 2.5 
    - Priority 2 = weighted x 2.25 
    - Priority 3 = weighted x 2.0 
Guard rail supporting metrics:
    - The total number of touchpoints being below 3 for emails 
            - Priority 1 = weighted x 1.5 
            - Priority 2 = weighted x 1.25 
            - Priority 3 = weighted x 1.0 
    - The queue time being below the specified time for calls and chats 
            - Priority 1 = weighted x 1.5 
            - Priority 2 = weighted x 1.25 
            - Priority 3 = weighted x 1.0 
            
### Table Creation + How to Productionalize
Have tables available on a weekly and monthly level to allow easy reporting and analysis on performance.  Could also consider segmenting on other metrics e.g. reason group, merchant type 



## 4. Costs per channel

### Model Development 
- Cost calculated  as SumUP employees being twice as expensive (as using in house , highly skilled employees), then spreading costs based on volumes of issues resolved.
- Based it on proportional volumes accounts for agents being able to handle 3 chats at one time 

### Table Creation + How to Productionalize
This should be included in the tables created for performance scoring as should be viewed together with how a metric is performing in reporting and analysis 







