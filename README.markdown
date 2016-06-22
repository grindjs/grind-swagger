# grind-swagger

`grind-swagger` is a swagger provider for `Grind`.  It’ll create `/swagger.json` endpoint on your API and expose any documented* routes.

## Installation

Add `grind-swagger` to your project:

```bash
npm install grind-swagger --save
```

## Usage

To use `grind-swagger` you’ll need to add it to your `Grind` providers:

```js
import Grind from 'grind-framework'

const app = new Grind()
app.providers.push(require('grind-swagger').provider)
```

## Documenting routes

`grind-swagger` will only expose routes that are explicitly documented for swagger.

You can document your routes for swagger by passing in a third param when registering:

```js
app.routes.get('/states', 'index', {
	swagger: {
		description: 'Gets a list of states',
		parameters: [
			{
				name: 'limit',
				in: 'query',
				required: false,
				description: 'Limit the number of records',
				type: 'integer'
			}, {
				name: 'offset',
				in: 'query',
				required: false,
				description: 'Skip records before querying',
				type: 'integer'
			}
		]
	}
})
```

## Inferred route documentation

The previous example uses the Swagger spec directly, however `grind-swagger` can infer quite a bit for you for you:

```js
app.routes.get('/:state/cities/:letter?', 'index', {
	swagger: {
		description: 'Gets a list of cities in a state',
		parameters: {
			state: 'State abbreviation',
			letter: 'Letter to filter cities by'
			limit: {
				description: 'Limit the number of records',
				type: 'integer'
			},
			offset: {
				description: 'Skip records before querying',
				type: 'integer'
			}
		}
	}
})
```

Based on this, the following can be inferred:

* `state` is a required string parameter that appears in the URL
* `letter` is an optional string parameter that may appear in the URL
* `limit` and `offset` are optional query string parameters

How this works:

* `letter` is inferred as optional due to the route param’s trailing `?`.  `state` is a non-optional parameter, making it required.
* If no type is provided, all parameters default to a `string` type unless they end with `_id`, which defaults to `integer`
* `limit` and `offset` are inferred as _optional_ query parameters because they don’t appear in the route path and this is a `GET` request.
* For non-`GET` requests, parameters that don’t appear in the route path are inferred as _required_ `body` parameters.
* All rules that are explicitly defined will take precedence over any inferred rules.

### Teaching

You can help inference by ‘teaching’ it.  This is useful for common keywords that have the same type, but will differ in their descriptions.

```js
import Swagger from 'grind-swagger'

Swagger.learn('featured', { type: 'boolean') })
Swagger.learn('limit', { type: 'integer') })
Swagger.learn('offset', { type: 'integer') })

app.routes.get('/states', 'index', {
	swagger: {
		description: 'Gets a list of states',
		parameters: {
			featured: 'Filter states by whether or not they’re featured',
			limit: 'Limit the number of states returned',
			offset: 'Number of states to skip before querying'
		}
	}
})

app.routes.get('/:state/cities', 'index', {
	swagger: {
		description: 'Gets a list of cities in a state',
		parameters: {
			state: 'State abbreviation',
			featured: 'Filter cities by whether or not they’re featured',
			limit: 'Limit the number of cities returned',
			offset: 'Number of cities to skip before querying'
		}
	}
})
```

Without teaching `featured`, `limit` and `offset` would have had their type inferred as `string`.  By teaching, their type is correctly inferred as `boolean`, `integer` and `integer` respectively.

## Shared parameters

No one wants to clutter their code with a bunch of repetitive documentation.  To avoid this, you can define shared parameters (and groups of parameters) to use within your routes:

```js
import Swagger from 'grind-swagger'

// Register a single parameter
Swagger.parameter('state', {
	name: 'state',
	in: 'url',
	required: true,
	description: 'State abbreviation',
	type: 'string'
})

// Register a group of parameters
Swagger.parameters('pagination', [
	{
		name: 'limit',
		in: 'query',
		required: false,
		description: 'Limit the number of records',
		type: 'integer'
	}, {
		name: 'offset',
		in: 'query',
		required: false,
		description: 'Skip records before querying',
		type: 'integer'
	}
])

// Now you can reuse these quickly:
app.routes.get('/:state/cities/:letter?', 'index', {
	swagger: {
		description: 'Gets a list of cities in a state',
		use: [ 'state', 'pagination' ]
	}
})
```

Shared parameters can also take advantage of inference, allowing for far more concise and readable code:

```js
import Swagger from 'grind-swagger'

Swagger.parameter('state', 'State abbreviation')
Swagger.parameters('pagination', {
	limit: {
		description: 'Limit the number of records',
		type: 'integer'
	},
	offset: {
		description: 'Skip records before querying',
		type: 'integer'
	}
])

app.routes.get('/:state/cities/:letter?', 'index', {
	swagger: {
		description: 'Gets a list of cities in a state',
		use: [ 'state', 'pagination' ]
	}
})
```

### Shared parameters vs Teaching

`Swagger.parameter` and `Swagger.learn` have a bit in common in that they can share common documentation between multiple routes, however they differ in how they should be used:

* Shared parameters should be used when you’re looking to include documentation for a parameter that is shared between different routes ‘as-is’.
* Teaching should be used when you’re just trying to improve inferring and you’re still planning to describe your parameters.

Here’s an example of shared parameters working together with teaching:

```js
import Swagger from 'grind-swagger'

// Documentation of what ”featured” is will change
// from route to route, so we just teach the type
Swagger.learn('featured', { type: 'boolean' })

// Documentation for pagination is shared, so we
// define the group of params and reuse as-is.
Swagger.parameters('pagination', {
	limit: {
		description: 'Limit the number of records',
		type: 'integer'
	},
	offset: {
		description: 'Skip records before querying',
		type: 'integer'
	}
])

app.routes.get('/states', 'index', {
	swagger: {
		description: 'Gets a list of states',
		use: [ 'pagination' ],
		parameters: {
			featured: 'Filter states by featured'
		}
	}
})

app.routes.get('/:state/cities', 'index', {
	swagger: {
		description: 'Gets a list of cities in a state',
		use: [ 'pagination' ],
		parameters: {
			state: 'Abbreviation of a state'
			featured: 'Filter cities by featured'
		}
	}
})

```