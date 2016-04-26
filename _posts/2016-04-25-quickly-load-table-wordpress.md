---
layout: post
title: Quickly load data into a table from Wordpress
subtitle: Taking advantage of Wordpress hooks and the jQuery DataTables plugin
date: 2016-04-25
---
## The problem:

Loading a handful of custom posts into a table isn't usually a cause for concern, but I ran into a situation where I had to pull data from over a thousand individual custom posts. In this case, the page load time was nearing three seconds which would cause most users to close the tab and never return.

## The solution:

The idea in this case is to use Wordpress hooks to maintain a json file with post data when a specific post type is created or edited. The json file is updated every time the hook fires, which causes slightly longer load times in the Wordpress administration area, but is still considerably faster than using `get_posts()` or `WP_Query()`. After the json file is constructed, the front end will make an ajax request to load the json file as a data source for DataTables.

I've laid out some sample code below, but if you want to try it out yourself based on my solution, here are some links for things I've mentioned:

- [Wordpress save_post](https://codex.wordpress.org/Plugin_API/Action_Reference/save_post)
- [DataTables plugin](https://www.datatables.net/)

***

### Step one: generate json

This is probably the most complicated step, just because of how Wordpress stores custom post and field data in MySQL. The general idea is to write a query to return the data in a way that can be easily converted to json with PHP's [json_encode](http://php.net/manual/en/function.json-encode.php) function.

For the sake of this example, I'm using a custom post type called `person` with some [Advanced Custom Fields](https://www.advancedcustomfields.com/) (ACF) as follows:

| Field name | Field type |
| :------ |:--- |
| post_title | Wordpress post title |
| First Name | ACF text |
| Last Name | ACF text |
| Title | ACF text |
| Birthday | ACF date |  

This example stores the json file in the Wordpress uploads directory under `/json/`.

```php
<?php

function generate_json(){
    global $wpdb;

    $sql = "
      SELECT wp_posts.post_title,
             pm1.meta_value as 'first_name',
             pm2.meta_value as 'last_name',
             pm3.meta_value as 'title',
             DATE_FORMAT(pm4.meta_value, '%M %e, %Y') as 'birthday'
      FROM wp_posts
        LEFT JOIN wp_postmeta AS pm1
            ON (wp_posts.ID = pm1.post_id AND pm1.meta_key='first_name')
        LEFT JOIN wp_postmeta AS pm2
            ON (wp_posts.ID = pm2.post_id AND pm2.meta_key='last_name')
        LEFT JOIN wp_postmeta AS pm3
            ON (wp_posts.ID = pm3.post_id AND pm3.meta_key='title')
        LEFT JOIN wp_postmeta AS pm4
            ON (wp_posts.ID = pm4.post_id AND pm4.meta_key='birthday')  
      WHERE wp_posts.post_type = 'person'
        AND wp_posts.post_status = 'publish'";

    $rs = $wpdb->get_results($sql, ARRAY_N);

    $data = $rs;

    $people = array();
    $people['data'] = $data;
    $people = json_encode($people);

    $full_path = $wp_uploads['basedir'] . '/json/people.json';

    file_put_contents($full_path, $people);
}
```

This will create a json file with the structure:

```json
{
    "data":
    [
        [
            "John Smith",
            "John",
            "Smith",
            "January 1, 1987"
        ],
        [
            "Jane Smith",
            "Jane",
            "Smith",
            "January 1, 1988"
        ]
    ]
}
```

Which will allow simple passing to [DataTables](https://www.datatables.net/) and prep us for the next step.

### Step two: Wordpress hook

Now we're ready to tie into the Wordpress [save_post](https://codex.wordpress.org/Plugin_API/Action_Reference/save_post) function. This will allow us to recreate the json file whenever a post is edited or added to the administration area of Wordpress, so that the file will always be up to date.

```php
<?php

add_action('save_post', 'save_person_meta', 10, 3);
function save_person_meta($post_id, $post, $update){
	$slug = 'person';

    // If this isn't a 'person' post, don't update it.
    if ($slug != $post->post_type) {
        return;
    }

    generate_json();
}
```

This will fire any time 'Publish' or 'Update' is pressed for a `person` post type in the Wordpress administration area. Also to note, this hook runs much faster than a standard Wordpress query because of the single MySQL call using [wpdb](https://codex.wordpress.org/Class_Reference/wpdb).

### Step three: load the table

Now we're going to use the [DataTables ajax](https://www.datatables.net/examples/data_sources/ajax.html) option to load the table asynchronously while the page is loading. There are two parts to this because of how Wordpress handles ajax requests.

First, we have some PHP to return the json file:

```php
<?php

add_action('wp_ajax_get_people', 'get_people');
add_action('wp_ajax_nopriv_get_people', 'get_people');

function get_people(){
    $uploads = wp_upload_dir();

    $file = $uploads['basedir'] . '/json/people.json';

    echo file_get_contents($file);

    exit();
}
```

Second, we fire the ajax request when the page loads:

```javascript
(function($) {

    $(document).ready(function() {

    	var table = $('#people-table').DataTable({
    		'dom': 'rtip',
            "pageLength": 100,
            ajax: {
                url: '/wp-admin/admin-ajax.php?action=get_people',
                type: 'POST',
            }
    	});

    });

})(jQuery);
```

The `dom` and `pageLength` arguments are just options for how the DataTable is displayed.

And with that, you now have a large collection loading into an interactive table in a fraction of the time it takes compared to traditional Wordpress methods.
