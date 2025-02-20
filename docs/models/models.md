# Models

Models provide a fluent way to interface with objects in WordPress. Models can
be either a post type, a term, or a subset of a post type. They represent a data
object inside of your application. Models can also allow dynamic registration of
a post type, REST API fields, and more. To make life easier for developers,
Mantle models were designed with uniformity and simplicity in mind.

## Supported Model Types

Data Type | Model Class
--------- | -----------
Attachment | `Mantle\Database\Model\Attachment`
Comment | `Mantle\Database\Model\Comment`
Post | `Mantle\Database\Model\Post`
Site | `Mantle\Database\Model\Site`
Term | `Mantle\Database\Model\Term`
User | `Mantle\Database\Model\User`

## Generating a Model

Models can be generated through the command:

```bash
wp mantle make:model <name> --model_type=<model_type> [--registrable] [--object_name] [--label_singular] [--label_plural]
```

## Defining a Model

Models live in the `app/models` folder under the `App\Models` namespace.

### Example Post Model

```php
/**
 * Example_Model class file.
 *
 * @package App\Models
 */

namespace App\Models;

use Mantle\Database\Model\Post;

/**
 * Example_Model Model.
 */
class Example_Model extends Post {
  /**
   * Post Type
   *
   * @var string
   */
  public static $object_name = 'example-model';
}
```

### Example Term Model

```php
/**
 * Example_Model class file.
 *
 * @package App\Models
 */

namespace App\Models;

use Mantle\Database\Model\Term;

/**
 * Example_Model Model.
 */
class Example_Model extends Term {
  /**
   * Term Type
   *
   * @var string
   */
  public static $object_name = 'example-model';
}
```

## Interacting with Models

Setting/updating data with a model can be done using the direct attribute name
you wish to update or a handy alias (see Core Object).

```php
$post->content = 'Content to set.';

// is the same as...
$post->post_content = 'Content to set.';
```

### Field Aliases

#### Post Aliases


Alias | Field
----- | -----
`content` | `post_content`
`description` | `post_excerpt`
`id` | `ID`
`title` | `post_title`
`name` | `post_title`
`slug` | `post_name`

#### Term Aliases

Alias | Field
----- | -----
`id` | `term_id`

### Saving/Updating Models

The `save()` method on the model will store the data for the respective model.
It also supports an array of attributes to set for the model before saving.

```php
$post->content = 'Content to set.';
$post->save();

$post->save( [ 'content' => 'Fresher content' ] );
```

### Deleting Models

The `delete()` method on the model will delete the model. On `Post` models, the
delete method will attempt to trash the post if the post type supports it. You
can pass `$force = true` to bypass that.

```php
// Force to delete it.
$post->delete( true );

$term->delete();
```

### Working with Meta

The `Post`, `Term`, and `User` model types support setting meta easily. Models
have a get/set/delete methods for meta:

```php
// Meta will be stored immediately unless the model hasn't been saved yet
// (allows you to set meta before saving the post).
$model->set_meta( 'meta-key', 'meta-value' );
$model->delete_meta( 'meta-key' );
$value = $model->get_meta( 'meta-key' ); // mixed

// Meta can also be saved directly with the save() method.
$model->save( [ 'meta' => [ 'meta-key' => 'meta-value' ] ] );
```

Models also support a fluent way of setting meta using the `meta` attribute. The
meta will be queued for saving and saved once you call the `save()` method on
the model:

```php
// Retrieve meta value.
$value = $model->meta->meta_key; // mixed

// Update a meta value.
$model->meta->meta_key = 'meta-value';
$model->save();

// Delete a meta key.
unset( $model->meta->meta_key );
$model->save();
```

### Working with Terms

The `Post` model support interacting with terms through
[relationships](./model-relationships.md) or through the model directly. The
model supports multiple methods to make setting terms on a post simple:

```php
$category = Category::whereName( 'Example Category' )->first();

// Save the category to a post.
$post = new Post( [ 'title' => 'Example Post' ] );

// Also supports an array of IDs or WP_Term objects.
$post->terms->category = [ $category ];

$post->save();

// Read the tags from a post.
$post->terms->post_tag // Term[]
```

Terms can also be set when creating a post (specifying the taxonomy is
optional):

```php
$post = new Post( [
	'title' => 'Example Title',
	'terms' => [ $category ],
] );

$post = new Post( [
	'title' => 'Example Title',
	'terms' => [
		'category' => [ $category ],
		'post_tag' => [ $tag ],
	],
] );
```

Models also support simpler `get_terms`/`set_terms` methods for function based
setting of a post's terms:

```php
$post = Post::find( 1234 );

// Set the terms on a model.
$post->set_terms( [ $terms ], 'category' );

// Read the terms on a model.
$post->get_terms( 'category' ); // Term[]
```

## Core Object

To promote a uniform interface of data across models, all models implement
`Mantle\Contracts\Database\Core_Object`. This provides a consistent set of
methods to invoke on any model you may come across. A developer shouldn't have
to check the model type before retrieving a field. This helps promote
interoperability between model types in your application.

The `core_object()` method will retrieve the WordPress core object that the
model represents (`WP_Post` for a post, `WP_Term` for a term, and so on).

```php
id(): int
name(): string
slug(): string
description(): string
parent(): ?Core_Object
permalink(): ?string
core_object(): object|null
```

## Events

In the spirit of interoperability, you can listen to model events in a uniform
way across all model types. Currently only `Post` and `Term` models support
events. Model events can be registered inside or outside a model.

```php
namespace App\Models;

use Mantle\Database\Model\Post as Base_Post;

class Post extends Base_Post {
  protected static function boot() {
    static::created( function( $post ) {
      // Fired after the model is created.
    } );
  }
}
```

The following events are supported:

Method | Event
------ | -----
`created` | After a model is created.
`updating` | Before a model is updated.
`updated` | After a model is updated.
`deleting` | Before a model is deleted.
`deleted` | After a model is deleted.
`trashing` | Before a model is trashed.
`trashed` | After a model is trashed.

## Query Scopes

A scope provides a way to add a constraint to a model's query easily.

### Global Scope.
Using a global scope, a model can become a subset of a parent model. For example, a
`User` model can be used to define a user object while an `Admin` model can
describe an admin user. Underneath they are both user objects but the `Admin`
model allows for an easier interface to retrieve a subset of data.

In the example below, the `Admin` model extends itself from `User` but includes
a meta query in all requests to the model (`is_admin = 1`).

```php
use App\Models\User;
use Mantle\Database\Query\Post_Query_Builder;

class Admin extends User {
  protected static function boot() {
		static::add_global_scope(
			'scope-name',
			function( Post_Query_Builder $query ) {
				return $query->whereMeta( 'is_admin', '1' );
			}
		)
  }
}
```

Global Scopes can also extend from a class to allow for reusability.

```php
use App\Models\User;
use Mantle\Contracts\Database\Scope;
use Mantle\Database\Model\Model;

class Admin extends User {

	protected static function boot() {
		parent::boot();

		static::add_global_scope( new Test_Scope() );
	}
}

class Admin_Scope implements Scope {
	public function apply( Builder $builder, Model $model ) {
		return $builder->whereMeta( 'is_admin', '1' );
	}
}
```

### Local Scope

Local Scopes allow you to define a commonly used set of constraints that you may
easily re-use throughout your application. For example, you can retrieve all
posts that are in a specific category. To add a local scope to the application,
prefix a model method with `scope`.

Scopes can also accept parameters passed to the scope method. The parameters
will be passed down to the scope method after the query builder argument.

```php
use Mantle\Database\Model\Post as Base_Post;
use Mantle\Database\Query\Post_Query_Builder;

class Post extends Base_Post {
	public function scopeActive( Post_Query_Builder $query ) {
		return $query->whereMeta( 'active', '1' );
	}

	public function scopeOfType( Post_Query_Builder $query, string $type ) {
		return $query->whereMeta( 'type', $type );
	}
}
```

#### Using a Local Scope
To use a local scope, you may call the scope methods with querying the model
without the `scope` prefix. Scopes can be chained in the query, too.

```php
Posts::active()->get();

Posts::ofType( 'type-to-query' )->get();
```
