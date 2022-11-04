# Udiddit Social Network

## Introduction

![alt text](https://cdn.searchenginejournal.com/wp-content/uploads/2021/09/16-reasons-why-social-media-is-important-to-your-company-616d3200e6dc6-sej.png)

Udiddit, a social news aggregation, web content rating, and discussion website, is currently 
using a risky and unreliable Postgres database schema to store the forum posts, 
discussions, and votes made by their users about different topics.
The schema allows posts to be created by registered users on certain topics, and can 
include a URL or a text content. It also allows registered users to cast an upvote (like) or 
downvote (dislike) for any forum post that has been created. In addition to this, the schema 
also allows registered users to add comments on posts.
Here is the DDL used to create the schema

```
CREATE TABLE bad_posts (
id SERIAL PRIMARY KEY,
topic VARCHAR(50),
username VARCHAR(50),
title VARCHAR(150),
url VARCHAR(4000) DEFAULT NULL,
text_content TEXT DEFAULT NULL,
upvotes TEXT,
downvotes TEXT
);
CREATE TABLE bad_comments (
id SERIAL PRIMARY KEY,
username VARCHAR(50),
post_id BIGINT,
text_
```

## Part I: Investigate the existing schema
After investigating the existing schema, these are some of the observations:

- There is no explicit relationship between the two tables because no foreign key 
was defined. The column post_id from the bad_comments table should be a 
foreign key, connecting to the id column from the bad_posts table.

- We know that a single post can have multiple comments. Therefore if a post id is 
of type BIGSERIAL, a comment can’t be of type SERIAL, but rather BIGSERIAL

- The column “username” should have a unique constraint, and a custom constraint 
to make sure username are not empty.

- Currently, there is nothing preventing a user to make several votes for the same 
post. A constraint should be added in order to deal with that.

- To make the model more robust and efficient, we need to normalize the data. 
More tables need to be created to avoid several repetitions and potential errors.

- To boost the performance of our queries, indexes should be created. To quickly 
search the posts of a specific user for instance, an index should be created on the 
“username” column from the posts table


## Part II: Create the DDL for your new schema

Having done this initial investigation and assessment, my next goal is to dive deep into the 
heart of the problem and create a new schema for Udiddit. My new schema should at least 
reflect fixes to the shortcomings I have pointed to in the previous exercise. Here are the 
guidelines I should follow:

### 1 -  Guideline 1: 
Here is a list of features and specifications that Udiddit needs in order to support its website and administrative interface:

  <ol>
  <li>Allow new users to register:</li>
    <ul>
    <li>i. Each username has to be unique</li>
    <li>ii. Usernames can be composed of at most 25 characters</li>
    <li>iii. Usernames can’t be empty</li>
    <li>iv. We won’t worry about user passwords for this project</li>
    </ul>
    
<li>Allow registered users to create new topics:</li>
<ul>
<li>i. Topic names have to be unique.</li>
<li>ii. The topic’s name is at most 30 characters</li>
<li>iii. The topic’s name can’t be empty</li>
<li>iv. Topics can have an optional description of at most 500 characters.</li>
</ul>

<li>Allow registered users to create new posts on existing topics:</li>
<ul>
<li>i. Posts have a required title of at most 100 characters</li>
<li>ii. The title of a post can’t be empty.</li>
<li>iii. Posts should contain either a URL or a text content, but not both.</li>
<li>iv. If a topic gets deleted, all the posts associated with it should be 
automatically deleted too.</li>
<li>v. If the user who created the post gets deleted, then the post will 
remain, but it will become dissociated from that user.</li>
</ul>


<li>Allow registered users to comment on existing posts:</li>
<ul>
<li>i. A comment’s text content can’t be empty.</li>
<li>ii. Contrary to the current linear comments, the new structure should 
allow comment threads at arbitrary levels.</ul>
<li>iii. If a post gets deleted, all comments associated with it should be 
automatically deleted too.</li>
<li>iv. If the user who created the comment gets deleted, then the comment 
will remain, but it will become dissociated from that user.</li>
<li>v. If a comment gets deleted, then all its descendants in the thread 
structure should be automatically deleted too.</li>
</ul>

<li>e. A given user can only vote once on a given post:</li>
<ul>
<li>i. I can store the (up/down) value of the vote as the values 1 and -1 
respectively.</li>
<li>ii. If the user who cast a vote gets deleted, then all their votes will 
remain, but will become dissociated from the user.</li>
<li>iii. If a post gets deleted, then all the votes for that post should be 
automatically deleted too.</li>

</ul>

### 2 - Guideline 2: 
Here is a list of queries that Udiddit needs in order to support its website and administrative interface. I don’t necessarily need to produce the DQL 
for those queries: they are only provided to guide the design of the new database 
schema.

<ul>
<li>a. List all users who haven’t logged in in the last year.</li>
<li>b. List all users who haven’t created any post.</li>
<li>c. Find a user by their username.</li>
<li>d. List all topics that don’t have any posts.</li>
<li>e. Find a topic by its name.</li>
<li>f. List the latest 20 posts for a given topic.</li>
<li>g. List the latest 20 posts made by a given user.</li>
<li>h. Find all posts that link to a specific URL, for moderation purposes.</li>
<li>i. List all the top-level comments (those that don’t have a parent comment) for 
a given post.</li>
<li>j. List all the direct children of a parent comment.</li>
<li>k. List the latest 20 comments made by a given user.</li>
<li>l. Compute the score of a post, defined as the difference between the number 
of upvotes and the number of downvotes</li>
</ul>




### 3 - Guideline 3: 
I’ll need to use normalization, various constraints, as well as indexes in my new database schema. I should use named constraints and indexes to make my
schema cleaner.
### 4 - Guideline 4: 
My new database schema will be composed of five (5) tables that should have an auto-incrementing id as their primary key.

## Part III: Migrate the provided data

Now that my new schema is created, it’s time to migrate the data from the provided 
schema. Here are a few guidelines to consider in this process:
<ol>
<li>Topic descriptions can all be empty</li>
<li>Since the bad_comments table doesn’t have the threading feature, I can migrate all 
comments as top-level comments, i.e. without a parent</li>
<li>I can use the Postgres string function regexp_split_to_table to unwind the comma separated votes values into separate rows</li>
<li>I need to keep in mind that some users only vote or comment, and haven’t created 
any posts. I’ll have to create those users too.</li>
<li>The order of my migrations matter! For example, since posts depend on users and 
topics, I’ll have to migrate the latter first.</li>
<li> I can start by running only SELECTs to fine-tune my queries, and use a LIMIT to avoid 
large data sets. Once I know you have the correct query, I can then run your full 
INSERT...SELECT query</li>
</ol>

