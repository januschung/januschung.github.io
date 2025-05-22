## Read a json file

Give a `color.json` file with the following content:

``` json
[
	{
		"name": "red",
		"value": "#ff0000"
	},
	{
		"name": "green",
		"value": "#008000"
	},
	{
		"name": "blue",
		"value": "#0000ff"
	},
	{
		"name": "yellow",
		"value": "#ffff00"
	},
	{
		"name": "black",
		"value": "#000000"
	}
]

```

The following bash script will read the json array one by one

```bash 
#!/usr/bin/env bash

JSON_FILE="color.json"

while read -r color; do
    name=$(jq -r .name <<< "$color")
    value=$(jq -r .value <<< "$color")
    echo "Color $name has value as $value."
done < <(jq -c '.[]' $JSON_FILE)
```

Here is the sample output:

``` yaml
Color red has value as #ff0000.
Color green has value as #008000.
Color blue has value as #0000ff.
Color yellow has value as #ffff00.
Color black has value as #000000.
```