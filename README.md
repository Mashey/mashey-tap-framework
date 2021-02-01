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
$ cookiecutter https://github.com/singer-io/singer-tap-template.git
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

Throughout the repo are places that still need to be built out.  Client service class, tests, json_schema etc.
Suggested order:  
1. Client / Client Tests
2. Schemas
3. Streams
4. Sync

For most API based Data Resources, the client and streams class should contain most of the logic.  Sync should have a few places to plug in the Client Class name and different parameters.  Discovery shouldn't have to change.

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
