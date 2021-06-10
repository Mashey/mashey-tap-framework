# Mashey Internal Singer Tap Template

A [cookiecutter](https://github.com/audreyr/cookiecutter) template for creating
[Singer](https://github.com/singer-io) taps, adapted to Mashey internal standards.

## Usage

Install Cookiecutter:

```bash
$ pip install cookiecutter
```

Use cookiecutter to pull the repo - Fill out Information

```bash
$ cookiecutter https://github.com/Mashey/mashey-tap-framework.git
project_name [e.g. 'tap-facebook']: tap-foobar
package_name [tap_foobar]:
...
```

Now that the project exists, make a virtual environment:

```bash
$ cd tap-foobar
$ poetry init
$ poetry install
```

## Development

Throughout the repo are places that still need to be built out. Client service class, tests, json_schema etc.
Suggested order:

1. Client / Client Tests
2. Schemas
3. Streams
4. Sync

For most API based Data Resources, the client and streams class should contain most of the logic. Sync should have a few places to plug in the Client Class name and different parameters. Discovery shouldn't have to change.

## Test Functionality

Once those are done you can test the tap using:

(While you in your poetry venv)

Install the package (don't forget the '.'):

```bash
$ pip install -e .
```

And invoke the tap in discovery mode to get the catalog:

```bash
$ tap-foobar -c sample_config.json --discover
```

The output should be a catalog with the single sample stream (from the schemas folder):

```bash
{
  "streams": [
    {
      "metadata": [],
      "schema": {
        "additionalProperties": false,
        "properties": {
          "string_field": {
            "type": [
              "null",
              "string"
            ]
          },
          "datetime_field": {
            "type": [
              "null",
              "string"
            ],
            "format": "date-time"
          },
          "double_field": {
            "type": [
              "null",
              "number"
            ]
          },
          "integer_field": {
            "type": [
              "null",
              "integer"
            ]
          }
        },
        "type": [
          "null",
          "object"
        ]
      },
      "stream": "sample_stream",
      "key_properties": [],
      "tap_stream_id": "sample_stream"
    }
  ]
}
```

If this catalog is saved to a `catalog.json` file, it can be passed back into the tap in sync mode:

```
tap-foobar -c sample_config.json --properties catalog.json
```

# Building Taps w/ Singer

This guide will walkthrough the basic construction of a singer tap that uses an api endpoint as its data source. Singer Taps can be built to pull from many different types of endpoints but most commonly used is an api.

Singer Resources: - [Singer Docs](https://github.com/singer-io/getting-started)

Mashey Required Setup: - Python 3.8 + - Singer-Python - Pytest

Undoubtably there will be additional packages needed for more complex taps (S3 buckets, Google Clients, ETC) but this is a good starting point.

(Mashey has created a cookiecutter project to help quickly iterate over tap creation but it is advisable to make your own in the beginning)

# Basic Tap Structure

Most singer taps follow a similar structure and layout, and at Mashey we have adopted the same so that they are clear and organized

```
Tap-Singer
|- tap_singer
	|- schemas
	|- __init__.py
	|- client.py
	|- discovery.py
	|- streams.py
	|- sync.py
|- tests
	|- test_client.py
	...
	...
```

The over all project is the name of the data source prepended by the work Tap with a hyphen. The module matches the project name just lower case with an underscore.

We will take each one of these files at a time.

# Building the Client

The first thing to build out is the client class. This should be relatively straight forward. Using the requests package - build a class that reach out to different endpoints through methods.

```python
import json
import requests

class {{Resource}}Client:
  BASE_URL = # url or api etc for the resource

  def __init__(self, client_id, api_key):
    self._client = requests.Session()
    self._access_token = self.fetch_access_token(client_id, 	api_key)
    self._client.headers.update({
      'authorization': self.access_token,
      'client_id': client_id
    })

	# Does this resource need other verification?  Access token / OAuth etc
  def fetch_access_token(self, client_id, api_key):
    url = f'{self.BASE_URL}/auth_enpoint'
    headers = {
        'Content-Type': 'application/x-www-form-urlencoded'
    }
    payload_dict = {
        'client_id': client_id,
        'apikey': api_key
    }
    return self._client.post(url, headers=headers, data=payload_dict).json()['access_token']
```

Above is an example of a client that uses a client id and an api key to fetch an authorization token that is needed for each next request.

Next build out each method endpoint to capture the data needed. One thing to note is that any of the iteration of the endpoint is not done in the function. This will be processed later.

```python
	# Build a method for each endpoint.  If the endpoint is paginated or has to be iteration over don't do that here rather setup it up to hit the start
  def fetch_products(self, page):
    url = f'{self.BASE_URL}/product/product_list'}
    param_payload = {
        'active': 'true',
        'pagesize': 50,  # Max per page count
        'page': page  # Page will have to be iterated over in a range
    }
    return self._client.get(url, params=param_payload).json()
```

As with everything - test driven development is key! Since the taps are technically meant to be client agnostic - you often can’t test for specific values, but at this stage we don’t handle any data verification so instead test for structure and keys.

NOTE: Many api’s have request limits so use something like VCR to prevent hitting the source frequently.

```python
@pytest.mark.vcr()
def test_fetch_products():
    client = {{Resource}}Client(client_id=client_id,
                         api_key=api_key)

    product_list = client.fetch_products(page=1)['data']['product_list']
    for product in product_list:
        assert 'product_id' in product
        assert 'sellable_quantity' in product
        if len(product['sellable_quantity_detail']) > 0:
            for sellable_quantity_detail in product['sellable_quantity_detail']:
                assert 'inventory_type' in sellable_quantity_detail
                assert 'location' in sellable_quantity_detail
                assert 'sellable_quantity' in sellable_quantity_detail

        assert 'category_type' in product
        assert 'product_configurable_fields' in product
        if len(product['product_configurable_fields']) > 0:
            assert 'name' in product['product_configurable_fields']

        assert 'pricing' in product
        if len(product['pricing']) > 0:
            assert 'price_type' in product['pricing']
            assert 'price_sell' in product['pricing']
            assert 'tier_name' in product['pricing']
.
.
.
```

Above is a snippet of testing. The main portion is testing the return schema. Pull out the collection of records that is being queries from the response and iterate over that against the fields that should be on the resource. I iterate over the return because there are times not every record will have every field and this is a way to check a grouping.

These asserts will assist with building the json schema if there is not a published schema. They also essentially act as a test for the schema as well. (POSTMAN is a great help for this part).

As you build each endpoint, it will also benefit you to build the JSON schema for it as well.

# JSON Schema

Each endpoint needs to have a JSON schema. This is the structure that singer uses to tell the target how to build the resulting database. (At Mashy we are working on an internal tool to quickly build these from api responses but they should always be verified as well).

Below is the example of the schema for the products endpoint of the above example.

```json
{
    "type": ["object", "null"],
    "addtionalProperties": false,
    "properties": {
        "product_id": {"type": ["null", "string"]},
        "product_status": {"type": ["null", "string"]},
        "last_updated_at": {"type": ["null", "string"], "format": "date-time"},
        "sellable_quantity": {"type": ["null", "number"]},
        "sellable_quantity_detail": {
            "type": ["null", "array"],
            "items": {
                "type": ["object", "null"],
                "additionalProperties": false,
                "properties": {
                    "inventory_type": {"type": ["null", "string"]},
                    "location": {"type": ["null", "string"]},
                    "sellable_quantity": {"type": ["null", "number"]}
                }
            }
        },
...
...
```

Looking at about - each key/value pair that the JSON returns is present with the key and then the type of data that the value will be. Again, since at this point we aren’t verify required fields all the fields should be nullable as well.

An important piece to pay attention to is when the JSON response has nested data and how that works in the above schema.

Building these pay

The next file to build is the Streams File. Where the Client/Service file has each of the endpoints as a function, in the Stream file each of those endpoints is now its own class. There is a Parent Stream class that sets up some standard attributes and the init for the for client and state of the Stream.

```python
class Stream:
    tap_stream_id           = None
    key_properties          = []
    replication_method      = ''
    valid_replication_keys  = []
    replication_key         = 'last_updated_at'
    object_type             = ''
    selected                = True

    def __init__(self, client, state):
        self.client = client
        self.state = state

    def sync(self, *args, **kwargs):
        raise NotImplementedError("Sync of child class not implemented")
```
