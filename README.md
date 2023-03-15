# Introduction
The purpose of this project is to develop an ETL tool from SQS. It consumes messages generated from SQS, transform into the structure of designed database tables and load values into tables. The considerations of tabel designes described in Database Design section.

# Programming Language and DBMS Used

The solution codes in src folder are all implemented in Python. I use Python to this project because as the expected implementation is to extract, transform and load data and for me, python is doing good in that regard. I also use SQLAlchemy to map and query the relational tables into objects on the python code.  So,  rather than dealing with the differences among specific dialects of traditional SQL( such as MySQL or PostgreSQL or Oracle), the pythonic open-source framework, SQLAlchemy, allow leveraging the streamline of the workflow and more efficiently query the data. At database end, I used postgres. 


# Setup and Run


### Option 1: Run via Bash Script

Execute the run.sh  script to configure postgres, localstack and required python libraries then run the main python file (app.py)

1. Run `./run.sh <db password>`

2. "Must provide the db password. Ex, ./run.sh --pasword myPassword"  is to prompted to accept password. Received password will be stored in config/postgres_password.txt file

### Option 2: Run via Docker-Compose
1.Open terminal
2. Navigate to folder containing docker-compose.yml
3. execute the following commands:  
          docker-compose up
          export POSTGRES_PASSWORD= <YOUR PASSORD>
          ./message-generator/ <YOUR OS>
          cd src
          python app.py
          

### Option 3: Run via Docker ???????
  

### Database Design

**1. The following conseptional details identified from  the sample message structures in README.me file.**
 
  1. Events have two categories (issue and pull-request structures) and can be identified by event id and event type. Events also have title, body, state or status, started or created date, closed/merged date.
  2. Every event has repository it fired from  and repositories can be identified by their url and can also be described interms of their owner and name.
  3. Event has one or more tags or labels attached to it and each tag can be picked by different events.
  
  
**2 From the above conceptual detail, the below conceptual schema has been defined.**
  
  
    **Entity Types are REPOSITORY, EVENT, and TAG.**
    **Relationship are 1:M between REPOSITORY and EVENT  as well as M:N between EVENT and TAG**
  
**3. From the above conceptual detail, the below conceptual schema has been defined**
  
  
  **repository**(<ins>url</ins>, name, owner)
  
  **event**(<ins>id</ins>,event id, event type, title, body, state_or_status, started_or_created_date, closed_merged_date, **repository_id** )
  
  **tag_or_label**(<ins>id</ins>, name)
  
  **event_tag_or_label**(**<ins><event_id</ins>**, **<ins>tag_or_label_id</ins>**)

**4. After conducting mapping of the conceptual database schema into relational database schema and normalizing the relations, The resulting 4 tables are drawn to  show database  tables along with their relationship:** 

  https://dbdiagram.io/d/639aec1999cb1f3b55a193ba

  
**5. Data analytics requirements specified in README.me are defined as below:**

The complete file resides in **src/view.py within** **data_analytics_query()** function.

The function is called after the ETL process completes and it prints results of each data requirements.  Below is presented the list of data analytics team requirement and respective queries. Considerations or assumptins are documented under comment section before every query.



'''
1. Data Analytics Requirement 1: List the most active users.
2. Data Analytics Requirement 2: List longest open event.
3. Data Analytics Requirement 3: List the most popular five tags for all repositories.
4. Data Analytics Requirement 4: List the total completed event count per repository for a given period.
5. Data Analytics Requirement 5: List top users based on number of repositories they contributed.
  
  

Requirement 1: List the most active users. Here top 1 considered:
    
**SQL Version of the Req. 1:
  
    SELECT TOP(1) user, count(event.id) AS 'Counts'
    FROM event GROUP BY event.user 
    ORDER BY count(event.id) DESC 
    
**SQLAlchemy Version of Req. 1: 

    for row in s.query(Events.user, func.count(Events.id).label("Counts") ).group_by(Events.user).order_by(func.count(Events.id).desc()).limit(1):
        print ("Req. 1: Most active user ...............User:", row.user, "Total Counts: ",row.Counts)
    
Requirement 2: List longest open event. Consideration: Here events not closed considered:
     
**SQL Version of Req. 2: 
  
            SELECT TOP(1) title , (now() - event.started_or_created_at) / 86400 AS "days_count" 
            FROM event 
            WHERE state_or_status = 'open' 
            ORDER BY ((now() - event.started_or_created_at) / 864000) DESC 
    
**SQL Alchemy Verison of Req. 2:
  
    for row in s.query( Events.title, ((func.now()- Events.started_or_created_at) /       86400).label("days_count")).filter_by(state_or_status="open").order_by(((func.now()- Events.started_or_created_at) / 86400).desc()).limit(1):
        print ("Req. 2: Longest open event...............Title: ", row.title ,"# Days: ",row.days_count)
    
    
Requirement 3: List the most popular five tags for all repositories:
    
  **SQL Version of the Req. 3:
  
            SELECT event.repository_id, events_labels_or_tags.tag_or_label_id, count(events_labels_or_tags.tag_or_label_id) AS "Counts", row_number() OVER (PARTITION BY event.repository_id ORDER BY count(events_labels_or_tags.tag_or_label_id) DESC) AS rn 
            WHERE row_number() OVER (PARTITION BY event.repository_id ORDER BY count(events_labels_or_tags.tag_or_label_id) DESC) <=5
            FROM events_labels_or_tags INNER JOIN event ON events_labels_or_tags.event_id = event.id 
            GROUP BY event.repository_id, events_labels_or_tags.tag_or_label_id
        
**SqlAlchemy Verison of Req. 3:
                                                                                                                                    
    subquery = s.query(Events.repository_id, Events_Labels_or_Tags.tag_or_label_id, func.count(Events_Labels_or_Tags.tag_or_label_id).label('Counts'),func.row_number().over( partition_by=Events.repository_id,  order_by=func.count(Events_Labels_or_Tags.tag_or_label_id).desc()).label("rn") ).outerjoin(Events, Events_Labels_or_Tags.event_id == Events.id).group_by(Events.repository_id, Events_Labels_or_Tags.tag_or_label_id).subquery()
    for r in  s.query(subquery).filter(subquery.c.rn <= 5):
       print("Req. 3: Most popular tags...............Repo: ", r.repository_id,"Tag: ",r.tag_or_label_id,  "Counts=", r.Counts, "Rank=", r.rn)


Requirement 4: List the total completed event count per repository for a given period. 
Consideration: Here current year considered as a peroid
    
**SQL Version of the Req. 4:
  
           SELECT event.repository_id AS event_repository_id, count(event.event_id) AS "Counts" 
            FROM event 
            WHERE event.state_or_status = 'closed' AND EXTRACT(YEAR FROM event.closed_or_merged_at)  == EXTRACT(YEAR FROM GETDATE())
            GROUP BY event.repository_id
      
**SqlAlchemy Verison of Req. 4:

    for row in s.query(Events.repository_id,func.count(Events.event_id).label("Counts") ).filter(Events.state_or_status=="closed" 
    and extract('year', Events.closed_or_merged_at) ==  extract('year', func.now())).group_by(Events.repository_id):
    print ("Req. 4: Completed events within a period ...............Repository: " ,  row.repository_id ,"# Completed Events: ",row.Counts)

     
 Requirement 5: List top users based on number of repositories they contributed.
 Consideration: Users submitting pull-requests considered:
     
**SQL Version of the Req. 5:
  
            SELECT event.user, event.repository_id , event.event_type, count(event.event_id) AS "Counts", row_number() OVER (PARTITION BY event."user" ORDER BY count(event.id) DESC) AS rn 
            WHERE event.type=='pull-requests' AND row_number() OVER (PARTITION BY event."user" ORDER BY count(event.id) DESC)=1
            FROM event 
            GROUP BY event.user, event.repository_id, event.event_type

 **SQLAlchemy Verison of Req. 5:

    subquery = s.query(Events.user, Events.repository_id, Events.event_type, func.count(Events.event_id).label('Counts'),func.row_number().over( partition_by=Events.user,  order_by=func.count(Events.id).desc()).label("rn") ).group_by(Events.user, Events.repository_id, Events.event_type).subquery()
    for r in  s.query(subquery).filter(subquery.c.event_type == "pull-requests" ).order_by(subquery.c.rn.asc()):
      print("Req. 5: Top users by contribution...............User:", r.user, "Repo:", r.repository_id, "Counts=", r.Counts, "Rank=", r.rn)
  

    '''


### Open Sources ETL Tool to Replace This Custom Implementation
 
There are many ETL open souce tools to replace this custom implementation. From my reading, Airbyte is one of the Open-Source ETL Tools that was launched mid 2020. It provides connectors that are usable out of the box through a UI and API that allows community developers to monitor and maintain the tool. It is advantegous to use such open source ETL tools because, on top of the major functionalities of pipelineing data, they continiously get updated to offer state of the art features/demands. Usually they also have community support which makes integration and knowledge transfer quicker.


  

