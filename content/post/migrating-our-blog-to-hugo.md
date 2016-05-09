---
author: "matthew-keelan"
comments: true
date: 2016-05-09T12:00:00-05:00
draft: true
image: ""
menu: ""
share: true
slug: migrating-our-blog-to-hugo
tags:
- aws
- blog
- hugo
- markdown
- migration
title: Migrating our blog to Hugo, AKA Obligatory blog post about the blog.
---

## Out with the old
Once upon a time, we here at Music Dealers had a blog running on a legacy CMS, *cough* Drupal.  It had a lot of unneeded overhead, and loading times were effected. In short, it was bloated and slow.

## Enter Hugo
Hugo is a general-purpose website framework, which generates a site of static files. It uses markdown files for the posts, and you can customize templates, so it seemed like a good fit for what we wanted. And the load times of our blog posts will be a lot faster. Now we just need to get the old posts out of our old CMS and into markdown files.

## Migrating to Hugo
I decided to do this in 2 parts:
1. Getting the data out of the old database, and into an easier to manage schema.
2. Writing markdown from that new schema.
This way I could iterate on part 1, and clean up the data there before writing out the markdown files. I looked into what data we needed to use in the new blog and came up with a new schema.

```
posts
+------------------+---------------------+
| Field            | Type                |
+------------------+---------------------+
| id               | int(10) unsigned    |
| title            | varchar(255)        |
| body             | longtext            |
| is_published     | tinyint(3) unsigned |
| created_date     | timestamp           |
| publication_date | timestamp           |
| author_id        | int(10) unsigned    |
| author_name      | varchar(255)        |
| header_image_url | varchar(255)        |
| url_alias        | varchar(255)        |
+------------------+---------------------+

post_tags
+------------------+---------------------+
| Field            | Type                |
+------------------+---------------------+
| id               | int(10) unsigned    |
| tag              | varchar(255)        |
+------------------+---------------------+
```


<!---
* title
* date
* published
* publishdate
* author_name
* header_image_url
* slug
* aliases
* tags

```
+------------------+---------------------+------+-----+---------+-------+
| Field            | Type                | Null | Key | Default | Extra |
+------------------+---------------------+------+-----+---------+-------+
| nid              | int(10) unsigned    | NO   | PRI | NULL    |       |
| vid              | int(10) unsigned    | NO   |     | NULL    |       |
| status           | tinyint(3) unsigned | YES  |     | NULL    |       |
| created_date     | timestamp           | YES  |     | NULL    |       |
| display_date     | timestamp           | YES  |     | NULL    |       |
| uid_name         | varchar(255)        | YES  |     | NULL    |       |
| title            | varchar(255)        | YES  |     | NULL    |       |
| body             | longtext            | YES  |     | NULL    |       |
| header_image     | varchar(255)        | YES  |     | NULL    |       |
| url_alias        | varchar(255)        | YES  |     | NULL    |       |
+------------------+---------------------+------+-----+---------+-------+
```
-->

## Getting the data out, and managable
I wrote a query to collect the blog posts from the old database:
```sql
-- blogQuery
SELECT n.nid as id,
    r.vid as vid,
    n.status as is_published,
    FROM_UNIXTIME(n.created) AS created_date,
    pd.field_press_publication_date_value AS publication_date,
    u.uid AS author_id,
    u.name AS author_name,
    r.title,
    r.body,
    FROM node n
    INNER JOIN users u ON u.uid = n.uid
    INNER JOIN node_revisions r ON r.vid = n.vid
    LEFT JOIN users ru ON ru.uid = r.uid
    LEFT JOIN content_field_press_publication_date pd ON n.nid = pd.nid
    LEFT JOIN content_field_blog_display_date dd ON n.nid = dd.nid
    WHERE n.type = 'blog' AND n.status = 1
    ORDER BY nid DESC
```
Another to grab the url alias (we'll be 301 redirecting these to the new format):
```sql
-- pathQuery
SELECT dst AS path FROM url_alias WHERE src = :node_path
```
And another to grab the tags:
```sql
-- tagQuery
SELECT t.name
    FROM term_node r
    INNER JOIN term_data t ON r.tid = t.tid
    INNER JOIN vocabulary v ON t.vid = v.vid
    WHERE r.vid = :vid
    ORDER BY t.name
```
Then I looped through the posts, grabbed the tags, and inserted it all into the temporary transition database.

```php
// prepare the path query
$pathQueryStatement = $db->prepare($pathQuery);
// prepare the tag query
$tagQueryStatement = $db->prepare($tagQuery);
/// prepare the insert post query
$insertStatement = $db->prepare(
    "INSERT INTO posts(
        id,
        title,
        body,
        is_published,
        created_date,
        publication_date,
        author_id,
        author_name,
        header_image_url,
        url_alias
    ) VALUES (
        :id,
        :title,
        :body,
        :is_published,
        :created_date,
        :publication_date,
        :author_id,
        :author_name,
        :header_image_url,
        :url_alias
    )");
// prepare the insert tags query
$insertTagStatement = $db->prepare(
    "INSERT INTO post_tags (nid,tag) VALUES (:nid,:tag)"
);
// get the posts
$posts = $db->query($blogQuery);
// loop through the posts
foreach ($posts as $post) {
    // grab the url alias, the pretty-url of the post
    $pathQueryStatement->execute(array(':node_path' => "node/" . $post['id']));
    $path = $pathQueryStatement->fetchAll();

    // Drupal uses a version id, but we won't be using that.
    // Let's save it to use in the tag query.
    $vid = $post['vid'];
    unset($post['vid']);

    if (count($path)) {
        $alias = $path[0]['path'];
        $post['url_alias'] = $alias;
    } else {
        $post['url_alias'] = null;
    }

    // insert the post
    $insertStatement->execute($post);

    // grab the tags
    $tagQueryStatement->execute(array(":vid" => $vid));
    $tmpTags = $tagQueryStatement->fetchAll();
    // This is a cool laravel function to grab 1 field from each element of an array
    $tags = array_pluck($tmpTags, 'name');

    // insert the tags
    foreach ($tags as $tag) {
        $insertTagStatement->execute(
            array(
                'nid' => $post['id'],
                'tag' => $tag
            )
        );
    }
}
```

## Writing Markdown
Now it was time to write markdown files for all of these posts.
### Get the posts, simple
```sql
-- blogQuery
SELECT * FROM posts
```
### Get the tags
```sql
--- tagQuery
SELECT * FROM post_tags WHERE id = :id ORDER BY tag
```
### Loop throught the posts, and write markdown files
```php

// prepare the tag query
$tagQueryStatement = $db->prepare($tagQuery);

foreach ($posts as $post) {
    // title
    // publishdate
    // published
    // description
    // author
    // tags
    // aliases

    // publishdate
    $date = date("Y-m-d", strtotime($post['publication_date']));

    // published
    $published = "false";
    if ($post['is_published'] == "1") {
        $published = "true";
    }

    // Sanitize the title, for use in the file name.
    $sanitizedTitle = preg_replace("/[']/", "", strtolower($post['title']));
    $sanitizedTitle = preg_replace("/[^a-zA-Z0-9\s\-]/", "-", $sanitizedTitle);
    $sanitizedTitle = preg_replace("/[ ]+/", "-", $sanitizedTitle);
    $sanitizedTitle = trim(preg_replace("/[-]+/", "-", $sanitizedTitle), '-');
    $sanitizedTitle = str_replace("-s-", 's-', $sanitizedTitle);

    $filename = "{$date}-{$sanitizedTitle}.md";

    ///////////////////////////////////////////////////////////////////////////
    // Start the Markdown
    $markdown = "---\n";
    // replace control character that looks like - with -
    $markdown.= "title: \"" . trim(addslashes(str_ireplace("\x97", "-", $post['title']))) . "\"\n";
    $markdown.= "description: null\n";
    $markdown.= "date: \"{$date}\"\n";
    $markdown.= "published: {$published}\n";
    $markdown.= "publishdate: \"{$date}\"\n";
    $markdown.= "author_name: \"{$post['author']}\"\n";
    $markdown.= "slug: \"{$sanitizedTitle}\"\n";
    if (isset($post['url_alias'])) {
        $markdown.= "aliases:\n";
        $markdown.= "  - \"/{$post['url_alias']}/\"\n";
    } else {
        $markdown.= "aliases: null\n";
    }
    // Get the tags
    $tagQueryStatement->execute(array(":nid" => $post['nid']));
    $tags = $tagQueryStatement->fetchAll();
    if (count($tags)) {
        $markdown.= "tags:\n";
        foreach ($tags as $tag) {
            $markdown.= "  - {$tag['tag']}\n";
        }
    }
    $markdown.= "---\n";

    ///////////////////////////////////////////////////////////////////////////
    // Write the body of the post
    // remove Windows linebreak characters.
    $thePost = str_ireplace("\x0D", "", trim($post['body']));
    // empty div
    $thePost = str_replace("<div></div>", '', $thePost);
    // consistent br's
    $thePost = str_replace('<br />', '<br>', $thePost);
    $thePost = str_replace('<br/>', '<br>', $thePost);
    // change 2 line breaks to 1
    $thePost = str_replace("<br><br>", "<br>", $thePost);
    // paragraphs that start with a line break.
    $thePost = str_replace('<p><br>', '<p>', $thePost);
    if (strpos($thePost, '<p></p>') === 0) {
        $count = 1;
        $thePost = str_replace("<p></p>", "", $thePost, $count);
    }
    // Markdownify, remove p tags, and br tags.
    $thePost = str_replace("<br>", "\n\n", $thePost);
    $thePost = str_replace("</p>", "\n\n", $thePost);
    $thePost = str_replace("<p>", "", $thePost);
    $markdown.= trim($thePost);

    $year = date("Y", strtotime($date));
    writeFile("{$year}/{$filename}", $markdown);
}

function writeFile($filename, $content)
{
    if (!file_exists(dirname($filename))) {
        mkdir(dirname($filename), 0755, true);
    }
    $file = fopen($filename, "w") or false;
    fwrite($file, $content);
    fclose($file);
}
```
## Set up Hugo
Then we can [install Hugo](http://gohugo.io/), and put these files in  the content/post/ directory, and build our blog.

## In conclusion
This gave us a great speed increase, and a chance to re-design where we needed it most. We have a more managable system, that works great for our readers. I wouldn't hesitate to recommend Hugo for anyone looking to set up a simple static website using markdown.