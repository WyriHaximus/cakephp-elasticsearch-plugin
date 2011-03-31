Elastic search [was recently used index the Firefox4 twitter stream](http://pedroalves-bi.blogspot.com/2011/03/firefox-4-twitter-and-nosql.html)
and make it searchable. It's based on Lucene and has a simple JSON based interface
that you can use to store objects and search through them (for instance even with CURL).

This also makes it easy to have your search indexes be updated in realtime
whenever your CakePHP models change data. Cause basically all we'd have to
do is do a Curl PUT, DELETE, etc to also make the change in Elastisearch
with every afterSave and afterDelete.

This plugin provides

 - a behavior to automatically update your indexes
 - a shell task to do full index fills
 - a generic search component that you can attach to your AppController and will intercept
   search actions on enabled models. Will return results in JSON format for easy
   AJAX integration.

Uses [ruflin's Elastica PHP library](https://github.com/ruflin/Elastica) to connect
to Elasticsearch.

## Installation

### Server

On [Debian/Ubuntu](http://www.elasticsearch.org/tutorials/2010/07/02/setting-up-elasticsearch-on-debian.html)

### PHP Client
    
    cd /usr/src/
    [ -d elastica/.git ] || git clone git://github.com/ruflin/Elastica.git elastica
    cd elastica
    git fetch origin
    git checkout -q -b tested_with_cakeplugin f9654f9 # Tested & compatible
    rsync --progress -a lib/* /usr/share/php/
    
### CakePHP Plugin

    cd ${YOURAPP}/plugins
    git clone git://github.com/kvz/cakephp-elasticsearch-plugin.git elasticsearch

## Integration

### Database

app/config/database.php

    class DATABASE_CONFIG {
        public $elastic = array(
            'host' => '127.0.0.1',
            'port' => '9200',
        );
        // ... etc

### Model

app/models/ticket.php (minimal example)

    public $actsAs = array(
        'Elasticsearch.Searchable' => array(
            
        ),
        // ... etc


app/models/ticket.php (full example)

    public $actsAs = array(
        'Elasticsearch.Searchable' => array(
            'debug_traces' => false,
            'searcher_enabled' => false,
            'searcher_action' => 'searcher',
            'searcher_param' => 'q',
            'searcher_serializer' => 'json_encode',
            'fake_fields' => array(
                '_label' => array('Product/description', 'BasketItem/description'),
            ),
            'index_name' => 'main',
            'index_chunksize' => 10000,
            'index_find_params' => array(
                'limit' => 1,
                'fields' => array(
                    // It's important you name your fields.
                    'subject',
                    'from',
                ),
                'contain' => array(
                    'Customer' => array(
                        // It's important you name your fields.
                        'fields' => array(
                            'id',
                            'name',
                        ),
                    ),
                    'TicketResponse' => array(
                        // It's important you name your fields.
                        'fields' => array(
                            'id',
                            'content',
                        ),
                    ),
                    'TicketObjectLink' => array(
                        // It's important you name your fields.
                        'fields' => array(
                            'foreign_model',
                            'foreign_id',
                        ),
                    ),
                    'TicketPriority' => array(
                        // It's important you name your fields.
                        'fields' => array(
                            'code',
                            'from',
                        ),
                    ),
                    'TicketQueue' => array(
                        // It's important you name your fields.
                        'fields' => array(
                            'name',
                        ),
                    ),
                ),
                'order' => array(
                    'Ticket.id' => 'DESC',
                ),
            ),
            'highlight' => array(
                'pre_tags' => array('<em class="highlight">'),
                'post_tags' => array('</em>'),
                'fields' => array(
                    'TicketResponse/0/content' => array(
                        'fragment_size' => 200,
                        'number_of_fragments' => 1,
                    ),
                ),
            ),
            'auto_update' => false,
            'error_handler' => 'php',
            'static_url_generator' => array('{model}', 'url'),
            'enforce' => array(
                'Customer/id' => 123,
                // callback: '#Customer/id' => array('LiveUser', 'id'),
            ),
        ),
    );

### Controller

To automatically enable a /<controller>/searcher url on all models
that have elastic search enabled, use:

app/app_controller.php

    public $components = array(
        'Elasticsearch.Searcher',
        // ... etc

This component will only actually fire when the Controller->modelClass
has the searchable behavior attached.

I chose for this method  (vs a dedicated SearchesController) so ACLing is easier.
e.g. You may already have an ACL for /tickets/*, so /tickets/search will automatically
be restricted the same way.

#### Generic search

If you want to search on all models, you could make a dedicated search controller
and instruct to search on everything like so:

    class SearchersController extends AppController {
        public $components = array(
            'Elasticsearch.Searcher' => array(
                'model' => '_all',
                'leading_model' => 'Ticket',
            ),
            // ... etc

        public function searcher () {
            $this->Searcher->searchAction();
        }

One known limitation is that the Elasticsearch plugin will only look at the
first configured Model for configuration parameters like `searcher_param`
and `searcher_action`.


## Try it

From your shell:

    # Fill all indexes
    ./cake indexer fill

    # Fill index with tickets
    ./cake indexer fill Ticket
    
    # Try a ticket search from commandline
    ./cake indexer search Ticket Hello

From your browser

    http://www.example.com/tickets/searcher/q:*kevin*

## jQuery integration

Let's look at an integration example that uses
[jQuery UI's autocomplete](http://docs.jquery.com/UI/Autocomplete).

Assuming you have included that library, and have an input field with attributes
`id="main-search"` and `target="/tickets/searcher/q:*{query}*"`:

    // Main-search
    $(document).ready(function () {
        $("#main-search").autocomplete({
            source: function(request, response) {
                $.getJSON($("#main-search").attr('target').replace('{query}', request.term), null, response);
            },
            delay: 100,
            select: function(event, ui) {
                var id = 0;
                if ((id = ui.item.id)) {
                    location.href = ui.item.url;
                    alert('Selected: #' +  id + ': ' + ui.item.url);
                }
                return false;
            }
        }).data( "autocomplete" )._renderItem = function( ul, item ) {
            return $("<li></li>")
                .data("item.autocomplete", item)
                .append("<a href='" + item.url + "'>" + item.html + "<br>" + item.descr + "</a>")
                .appendTo(ul);
        };
    });

## Todo

 - auto_update

## Useful commands

    # Get Status
    curl -XGET 'http://127.0.0.1:9200/_status?pretty=true'
    
    # Dangerous: Delete an entire index
    curl -XDELETE 'http://127.0.0.1:9200/main'

    # Dangerous: Delete an entire type
    curl -XDELETE 'http://127.0.0.1:9200/main/ticket'

    # Get all tickets
    curl -XGET http://127.0.0.1:9200/main/ticket/_search -d '{
        "query" : {
            "field" : {
                "_all" : "**"
            }
        }
    }'

    # Get everything
    curl -XGET http://127.0.0.1:9200/main/_search?pretty=true -d '{
        "query" : {
            "field" : {
                "_all" : "**"
            }
        },
        "size" : 1000
    }'
