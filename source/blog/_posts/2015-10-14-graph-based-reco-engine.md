---
title: Graph based recommendation engine for Shopware
tags:
    - graph database
    - neo4j
    - recommendation engine

categories:
- dev
indexed: true
github_link: blog/_posts/2015-10-14-graph-based-reco-engine.md.md

authors: [dn]
---
End of september [Benjamin](https://developers.shopware.com/blog/authors/bc/) and me visited the
[Bulgaraia PHP conference](http://www.bgphp.org/).
We saw a very interesting talk from Mariuz Gil called
[Discovering graph structures](https://joind.in/talk/view/14886), which gave a quick
introduction to graph databases and especially [Neo4j](neo4j.com).

Over a long weekend I decided, that trying to implement a recommendation engine
for Shopware based on Neo4j might be a good start to get into Neo4j and graph
databases in general.

# What is it all about
<img src="/blog/img/graph_simple_result.png" alt="Simple graph" style="float: left; margin: 10px;width:300px" />
A graph database is a database which represents information in form of graphs. Its ideal for highly cross-linked
data which needs to be queried in a semantic way.
A typical graph database will have **nodes**, **edges** and **properties** to represent data. The **nodes** are entities
like e.g. customer and and items / products.
**Properties** are additional information stored for the nodes, e.g. *name* and *address* of a person or the *price*
of an item.
**Edges** represent the relationship between nodes, so they will bring semantic information into the system. Having
*customer* and *items* as nodes, one might have *edges* like "purchased". This way one could design a graph with all
*items* all *customers* purchased.

Graph databases are popular everywhere where meaninful information should be extracted from such kind of information
network. Popular examples are recommendation engines for e.g. products, movies or even friends in social networks.
Even though it is possible to represent such structures in relational databases, graph databases are actually made
for this kind of requirement and might be prefarable in regards of performance and maintainability.

The graph above shows a simple customer / item graph with three customers (blue) having purchased various items (green).
As you can see, all customers purchased the "ESD download" item, so one could suggesting other items to the customers
 from this common preference.

# Neo4j
<img src="/blog/img/graph_tom_hanks.png" alt="Tom hanks movies + directors" style="float: right; margin: 10px;max-width:350px" />

Neo4j is a very popular open source graph database written in Java. You can simply download the community edition
at [http://neo4j.com/download/](http://neo4j.com/download/), unzip the archive and run `bin/neo4j start` in order
to run the neo4j server locally.
Afterwards you can navigate your browser to `http://localhost:7474/` and play play around with the tutorials a bit.
I highly recommend the movie graph tutorial, as it shows some basic concepts and allows to learn the query language
called `cypher` easily.

So there is a query, for example, to show all movies with Tom Hanks and the directory of each of those movies (see image
on the right).

Other examples show how to find all co-actors of Tom Hanks or the shortest relationship path between e.g. Tom Hanks and
Kevin Bacon (also known as [bacon path](https://en.wikipedia.org/wiki/Six_Degrees_of_Kevin_Bacon)). So playing around with the movie data
demo set gives quite a good impression of what is possible with such a graph database.

# The current situation in Shopware
Currently the so called "Marketing aggregate" components of Shopware take care
of parts of the recommendation engine. They make use of the PHP /MySQL stack
only, so that there are no additional dependencies for this kind of recommendation engine.

## The general concept
Generally the "also bought" functionality is based on these components:

* The table `s_articles_also_bought_ro` which is a denormalized representation
of the "also bought" marketing information. This is basically a ManyToMany
mapping for every item in the shop: It indicates, which item was purchased how
often with which other item
![](/blog/img/graph_also_bought_table.png)
* The component `\Shopware_Components_AlsoBought` which takes care of refreshing
this table. If the whole "also bought" table needs to be rebuild, it will call
the method `\Shopware_Components_AlsoBought::initAlsoBought`; usually this is not
necessary, as the method `\Shopware_Components_AlsoBought::refreshMultipleBoughtArticles`
will allow Shopware to update the "also bouglt" table on a per-order base.

## The SQL side
Generally this form of denormalization is used, as the kind of queries needed, to get this
information, is quite complex in a relational database.

The "also bought" query in Shopware might look like this:

```
SELECT
    detail1.articleID as article_id,
    detail2.articleID as related_article_id,
    COUNT(detail2.articleID) as sales
FROM s_order_details detail1
   INNER JOIN s_order_details detail2
      ON detail1.orderID = detail2.orderID
      AND detail1.articleID != detail2.articleID
      AND detail1.modus = 0
      AND detail2.modus = 0
      AND detail2.articleID > 0
      AND detail1.articleID = :articleId
GROUP BY detail2.articleID
```

As you can see, it takes a given item `articleId`, reads all orders with it
and joins all other items of the very same order. After a `GROUP BY` on
the "also bought" items, it can aggregate the total sum of the "also bought"
value for each other item using `COUNT(detail2.articleID) as sales`.

This works quite well and gives us the information needed - but it does
not scale well enough, to do this live on a per request base:

![](/blog/img/graph_explain.png)

As you can see here, the relevant query might produce a temporary table and a
filesort - so it is not a good idea to run it on every request on the detail page.
Instead of this, we are currently running this query on a per order base:

![](/blog/img/graph_basket_also_bought.png)

This will just fetch the "pairs" for the "also bought" information and then
increase the `sales` field in the `s_articles_also_bought_ro` table. This
query also has a much better database footprint, so that we get rid of the
performance issued, we'd have, if we'd use the first query shown above.

## Round up
The "also bought" functionality shows, that getting this kind of recommendation
information is possible in SQL - but it takes quite some effort to do it in
a proper way and does not allow the kind of flexibility, you might be used to
from using a graph database.


# Writing a recommendation plugin
Below I will discuss a simple recommendation plugin which you can find at [my github repo](https://github.com/dnoegel/DsnRecommendation).

The plugin must be installed by checking it out to `engine/Shopware/Plugins/Local/Core/DsnRecommendation`. After that,
the composer dependencies can be installed running `composer install` from the plugin directory. Configure your neo4j
server in your shop's `config.php` file:

```
'neo4j' => array(
    'host' => 'localhost',
    'user' => 'neo4j',
    'pass' => 'shopware',
    'port' => ''
)
```

Now you can install and activate the plugin using the shopware commandline tools:

```
./bin/console sw:plugin:refresh
./bin/console sw:plugin:install --activate DsnRecommendation
```

## Creating some demo data
<img src="/blog/img/graph_outdoor.png" alt="Simple graph" style="float: right; margin: 10px;width:400px" />

First of all the plugin provides some demo data, that can be used to test various recommendation scenarios. It will
create some items, customers and orders. The items are separated to groups like "console games", "outdoor games" and
"board and card games", so we can test different customers with different preferences.

The demo data generation can be triggered by running `./bin/consolw dsn:recommendation:demo`. Afterwards the shop
should have a new category "Gaming" with the subcategories "digital", "analog" and "outdoor":

Technically the demo data is generated by the `DemoData` components that can be found in the folder
`DsnRecommendation/Components/DemoData` of the plugin.

## Exporting orders to Neo4j
The plugin requires an initial export of the order data to Neo4j: `./bin/console dsn:neo4j:export`
Afterwards the plugin will automatically sync new orders to the graph database.

Technically this command will export the orders as CSV by using `\Shopware\Plugins\DsnRecommendation\Components\CsvExporter`.
Then `\Shopware\Plugins\DsnRecommendation\Components\Neo\BulkExporter` will let neo4j let import the CSV running this cypher
query:

```
USING PERIODIC COMMIT

LOAD CSV WITH HEADERS FROM "$url" AS row
MERGE (customer:Customer { id: row.userId, name:row.userName})
MERGE (item:Item { id:row.itemId, name:row.item })
CREATE UNIQUE (customer)-[:purchased]->(item);
```

So it will basically iterate the CSV and create `Customer` and `Item` nodes and add create edges to link a given
customer to the items he purchased.

## Having a look at the resulting graph
Now you can navigate to your neo4j frontend, typically `http://localhost:7474/browser/`. In the query window at the top
cypher queries can be entered. The query `MATCH (n) RETURN n LIMIT 100`, for example, will print the whole graph:

![](/blog/img/graph_neo.png)

This graphs shows four "purchasing group". The group at the bottom left is Shopware's demo data.
The other three "purchase groups" are the demo data of this plugin: Each of this groups simulates buying behaviour
for another topic, e.g. "card and board games", "computer games" and "outdoor games". The groups would be linked in
reality, for the purpose of debugging and experimenting around, smaller, separated groups seem to be more useful.

A recommendation query might look like this:

```
// find customer who ordered same items
MATCH (u:Customer)-[r1:purchased]->(p:Item)<-[r2:purchased]-(u2:Customer),
// find items of those customers
(u2:Customer)-[:purchased]->(p2:Item)
// only for this user
WHERE u.name = "Felix Frechmann"
// make sure, that the current user  didn't order that product, yet
AND not (u)-[:purchased]->(p2:Item)
// count / group by u2, so every user-path only counts once
RETURN p2.name, count(DISTINCT u2) as frequency
ORDER BY frequency DESC
```

![](/blog/img/graph_result.png)

Looking at the graph example above, you will see, that this query recommends "outdoor game: water fight" and
"outdoor game: soccer" to the customer "Felix Frechmann". The reasoning is basically the following:

* Felix bought "rope jumping" and "golf pro"
* So did Kathrin and Max
* Kathrin and Max also bought the game "wate fight"
* Kathrin bought the game "soccer"
So basically there are two paths to "water fight" and one path to "soccer" for Felix. For this reason, the
"water fight" recommendation is higher ranked than the "soccer" recommendation.

Currently the `frequency` is calculated by the number of customers, how bought the same product as Felix. This could
be changed to also take the number of similar items into account: `count(u2) as frequency`
With this modification, the "water fight" game would get a frequency of 4, as every actual path is taken into account.
Both approaches could be added by having a the frequency calculated like this:

`count(u2) * count(DISTINCT u2) as frequency`

Now the "water fight" game becomes a frequency of 8, the "soccer" game a frequency of 2 - which might reflect the number
of customers and the number of common items even better.

## Implement recommendation queries in Shopware
All existing orders where already exported to neo4j using the export console command of this plugin. Now we just need
to make sure, that new orders will also be syncronized to the graph database.
For this we add a little order subscriber like this:

```
class Order implements \Enlight\Event\SubscriberInterface
{
    public static function getSubscribedEvents()
    {
        return array(
            'Shopware_Modules_Order_SaveOrder_ProcessDetails' => 'onSaveOrder'
        );
    }

    public function onSaveOrder(\Enlight_Event_EventArgs $args)
    {
        /** @var \sOrder $order */
        $order = $args->get('subject');
        $details = $args->get('details');
        $userData = $order->sUserData;

        $userName = $userData['billingaddress']['firstname'] . ' ' . $userData['billingaddress']['lastname'];
        $userId = $userData['billingaddress']['userID'];

        // filter items and bring them into a format like ['id' => 33, 'name' => 'Sunflowers']
        $items = array_map(function($item) {
            return [
                'id' => $item['id'],
                'name' => $item['articlename']
            ];
        }, array_filter($details, function($item) {
            return $item['modus'] == 0;
        }));

        // \Shopware\Plugins\DsnRecommendation\Components\Neo\SyncOrder
        Shopware()->Container()->get('dsn_recommendation.sync_service')->sync($userId, $userName, $items);
    }
}
```
This subscriber will be notified whenever a new order is processed. In the callback method the user data (id and name)
as well as the product information (id and name) are read and passed to the `\Shopware\Plugins\DsnRecommendation\Components\Neo\SyncOrder`
service.

This service will basically create queries like the following one and send it to the neo4j server:

```
MERGE (customer:Customer { userId: '224', name:'Jimmy Jellyfish'})
MERGE (item:Item { itemId:'670', name:'Sonnenbrille "Red"' })
CREATE UNIQUE (customer)-[:purchased]->(item);
```

