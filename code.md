```
-- DDL

-- "users" TABLE
CREATE TABLE "users" (
    "id" SERIAL,
    "username" VARCHAR(25) NOT NULL,
    "last_login_date" DATE,
    CONSTRAINT "users_pk" PRIMARY KEY ("id"),
    CONSTRAINT "not_empty_username" CHECK (LENGTH(TRIM("username")) > 0)
);

CREATE INDEX "users_loggings" ON "users" ("last_login_date");
CREATE UNIQUE INDEX "unique_username" ON "users" ("username");


-- "topics" TABLE
CREATE TABLE "topics" (
    "id" SERIAL,
    "name" VARCHAR(30) NOT NULL,
    "description" VARCHAR(500),
    CONSTRAINT "topics_pk" PRIMARY KEY ("id"),
    CONSTRAINT "not_empty_topic_name" CHECK (LENGTH(TRIM("name")) > 0)
);

CREATE UNIQUE INDEX "unique_topic_name" ON "topics" ("name");


-- "posts" TABLE
CREATE TABLE "posts" (
    "id" BIGSERIAL,
    "user_id" INTEGER,
    "title" VARCHAR(100) NOT NULL,
    "url" VARCHAR(2048),
    "content" TEXT,
    "topic_id" INTEGER NOT NULL,
    "created_at" DATE,
    CONSTRAINT "not_empty_title_post" CHECK (LENGTH(TRIM("title")) > 0),
    
    CONSTRAINT "url_or_post" CHECK (("url" IS NOT NULL AND "content" IS NULL) 
        OR ("url" IS NULL AND "content" IS NOT NULL)),

    CONSTRAINT "posts_pk" PRIMARY KEY ("id"),

    CONSTRAINT "valid_topic" 
        FOREIGN KEY ("topic_id") 
            REFERENCES "topics" ON DELETE CASCADE,
    
    CONSTRAINT "valid_user_post"
        FOREIGN KEY ("user_id") 
            REFERENCES "users" ON DELETE SET NULL
);

CREATE INDEX "users_who_posted" ON "posts" ("user_id");
CREATE INDEX "topics_with_posts" ON "posts" ("topic_id");
CREATE INDEX "find_post_by_url" ON "posts" ("url");


-- "comments" TABLE
CREATE TABLE "comments" (
    "id" BIGSERIAL,
    "content" TEXT NOT NULL,
    "parent_comment_id" BIGINT,
    "post_id" BIGINT NOT NULL,
    "user_id" INTEGER,
    "created_at" DATE,
    CONSTRAINT "comments_pk" PRIMARY KEY ("id"),
    CONSTRAINT "not_empty_comment_text" CHECK (LENGTH(TRIM("content")) > 0),
    CONSTRAINT "valid_parent_comment" 
        FOREIGN KEY ("parent_comment_id") 
            REFERENCES "comments" ON DELETE CASCADE,
    CONSTRAINT "valid_post"
        FOREIGN KEY ("post_id") 
            REFERENCES "posts" ON DELETE CASCADE,
    CONSTRAINT "users_comments_fk" 
        FOREIGN KEY ("user_id") 
            REFERENCES "users" ON DELETE SET NULL
);

CREATE INDEX "find_top_level_comments" ON "comments" ("post_id", "parent_comment_id");
CREATE INDEX "find_direct_children_for_parent_comment" ON "comments" ("parent_comment_id");
CREATE INDEX "find_user_comments" ON "comments" ("user_id");


-- "votes" TABLE
CREATE TABLE "votes" (
    "id" BIGSERIAL,
    "post_id" BIGINT NOT NULL,
    "user_id" INTEGER,
    "vote" SMALLINT,
    CONSTRAINT "votes_pk" PRIMARY KEY ("id"),
    CONSTRAINT "unique_vote_for_given_user" UNIQUE ("user_id", "post_id"),
    CONSTRAINT "vote_is_1_or_-1" CHECK ("vote" IN (-1, 1)),
    CONSTRAINT "posts_votes_fk" 
        FOREIGN KEY ("post_id") 
            REFERENCES "posts" ON DELETE CASCADE,
    CONSTRAINT "users_votes_fk" 
        FOREIGN KEY ("user_id") 
            REFERENCES "users" ON DELETE SET NULL
);

CREATE INDEX "compute_score_of_post" ON "votes" ("post_id");


--MIGRATION (DML)

-- UPDATNG USERS TABLE
INSERT INTO "users" ("username")
  SELECT DISTINCT username 
  FROM "bad_posts"
UNION
  SELECT DISTINCT bc.username
  FROM "bad_comments" bc
  LEFT JOIN "users" u ON bc.username = u.username
  WHERE u.username IS NULL
UNION
  SELECT DISTINCT REGEXP_SPLIT_TO_TABLE(upvotes, ',')
  FROM "bad_posts"
UNION
  SELECT DISTINCT REGEXP_SPLIT_TO_TABLE(downvotes, ',')
  FROM "bad_posts";



-- UPDATING TOPICS TABLE
INSERT INTO "topics" ("name")
    SELECT
        DISTINCT topic
    FROM bad_posts;


-- UPDATING POSTS TABLE
INSERT INTO "posts" 
    ("id", "user_id", "title", "url", "content", "topic_id")
SELECT
    bp.id,
    u.id,
    LEFT(bp.title,100),
    bp.url,
    bp.text_content,
    t.id
FROM bad_posts AS bp
    INNER JOIN users as u 
        ON bp.username = u.username
    INNER JOIN topics AS t 
        ON t.name = bp.topic;

-- UPDATING COMMENTS TABLE
INSERT INTO "comments" ("user_id", "post_id", "content")
SELECT
    u.id,
    bc.post_id,
    bc.text_content
FROM bad_comments AS bc
INNER JOIN users AS u 
    ON u.username = bc.username;
    
-- UPDATING VOTES TABLE
INSERT INTO "votes" ("user_id", "post_id", "vote")
SELECT 
    u.id,
    bp.id,
    1
FROM (
    SELECT 
        id,
        REGEXP_SPLIT_TO_TABLE(upvotes, ',') AS voter_username
    FROM bad_posts
    ) AS bp 
    INNER JOIN users AS u 
        ON u.username = bp.voter_username
UNION
SELECT 
    u.id,
    bp.id,
    -1
FROM (
    SELECT 
        id,
        REGEXP_SPLIT_TO_TABLE(downvotes, ',') AS voter_username
    FROM bad_posts
    ) AS bp 
    INNER JOIN users AS u 
        ON u.username = bp.voter_username;

```
