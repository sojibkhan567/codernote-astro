---
title: "Create post with multiple category and tags."
description: "In Laravel’s Eloquent ORM, a Many-To-Many relationship facilitates the association between multiple records in one table with multiple records in another. This proves invaluable when modeling scenarios where a single entity can have connections with various entities of another kind."
pubDate: 2025-07-02
author: "Sajeeb Khan"
category: "Coding"
tags: ["development", "PHP", "laravel","laravel relationship", "many to many"]
featured: true
thumb: "https://storage.googleapis.com/assets.iankumu.com/2022/05/many-to-many-2048x1152.webp"
large: "https://storage.googleapis.com/assets.iankumu.com/2022/05/many-to-many-2048x1152.webp"
---

### Make table migration file for relationship (Many to Many)

 Create category migration & model name Category
 
```
php artisan make:model Category -m
```

```php
Schema::create('categories', function (Blueprint $table) {
    $table->id();
    $table->string('name');
    $table->string('slug')->unique();
    $table->timestamps();
});
```

Create post table & model named Post

```
php artisan make:model Post -m
```

```php
public function up(): void
{
    Schema::create('posts', function (Blueprint $table) {
        $table->id();
        $table->string('title');
        $table->string('slug')->unique();
        $table->text('content');
        $table->string('featured_image')->nullable();
        $table->boolean('is_published')->default(false);
        $table->timestamps();
    });
}
```

Create Tags Model & migration file

```
php artisan make:model Tag -m
```

```php
public function up(): void
{
    Schema::create('tags', function (Blueprint $table) {
        $table->id();
        $table->string('name');
        $table->string('slug')->unique();
        $table->timestamps();
    });
}
```

### Many to many relationship table

Create category_posts_table migration for relationship

```
php artisan make:migration category_posts_table
```

```php
public function up(): void
{
    Schema::create('category_posts', function (Blueprint $table) {
        $table->id();
        $table->foreignId('post_id')->constrained()->onDelete('cascade');
        $table->foreignId('category_id')->constrained()->onDelete('cascade');
        $table->timestamps();
    });
}
```

Create ``post_tags_table`` migration for relationship

```
php artisan make:migration post_tags_table
```

```php
public function up(): void
{
    Schema::create('post_tags', function (Blueprint $table) {
        $table->id();
        $table->foreignId('post_id')->constrained()->onDelete('cascade');
        $table->foreignId('tag_id')->constrained()->onDelete('cascade');
        $table->timestamps();
    });
}
```

### Setup model for relation

Set up the relationships in your Post models (for get categories) : 

```php
// Post.php model
class Post extends Model
{
    public function categories(): BelongsToMany
    {
        return $this->belongsToMany(Category::class);
    }

	public function tags(): BelongsToMany
    {
        return $this->belongsToMany(Tag::class);
    }
}
```

Set up the relationships in your Category models (for get posts) : 

```php
// Category.php model
class Category extends Model
{
    public function posts(): BelongsToMany
    {
        return $this->belongsToMany(Post::class);
    }
}
```

Set up the relationships in your Tag models (for get posts) : 

```php
namespace App\Models;

use Illuminate\Database\Eloquent\Model;
use Illuminate\Database\Eloquent\Relations\BelongsToMany;

class Tag extends Model
{
    protected $fillable = ['name', 'slug'];

    public function posts(): BelongsToMany
    {
        return $this->belongsToMany(Post::class);
    }
}
```

### In your controller (PostController) file written this code : 

To store data into database :

```php
public function store(Request $request)
    {
         $validated = $request->validate([
        'title' => 'required|string|max:255',
        'slug' => 'required',
        'content' => 'required|string',
        'categories' => 'required|array',
        'categories.*' => 'exists:categories,id',
        'tags' => 'nullable|array',
        'tags.*' => 'exists:tags,id',
        'featured_image' => 'nullable|image|max:5120',
    ]);

    // Handle image upload
    if ($request->hasFile('featured_image')) {
        $imagePath = $request->file('featured_image')->store('posts', 'public');
        $validated['featured_image'] = $imagePath;
    }

    $post = Post::create($validated);
    
    // Attach categories and tags
    $post->categories()->attach($request->categories);
    
    if ($request->has('tags')) {
        $post->tags()->attach($request->tags);
    }

    return redirect()->route('posts.index', $post)->with('success', 'Post created successfully!');
    }
```

To update data into database : 

```php
public function update(Request $request, Post $post)
{
    $validated = $request->validate([
        'title' => 'required|string|max:255',
        'content' => 'required|string',
        'categories' => 'required|array',
        'categories.*' => 'exists:categories,id',
        'tags' => 'nullable|array',
        'tags.*' => 'exists:tags,id',
        'featured_image' => 'nullable|image|max:2048',
    ]);

    // Handle image upload
    if ($request->hasFile('featured_image')) {
        // Delete old image if exists
        if ($post->featured_image) {
            Storage::disk('public')->delete($post->featured_image);
        }
        
        $imagePath = $request->file('featured_image')->store('posts', 'public');
        $validated['featured_image'] = $imagePath;
    }

    $post->update($validated);
    
    // Sync categories and tags
    $post->categories()->sync($request->categories);
    $post->tags()->sync($request->tags ?? []);

    return redirect()->route('posts.show', $post)->with('success', 'Post updated successfully!');
}
```

### Form View for insert or update data : 

```php
<form method="POST" action="{{ isset($post) ? route('posts.update', $post) : route('posts.store') }}" enctype="multipart/form-data">
    @csrf
    @if(isset($post))
        @method('PUT')
    @endif

    <div class="form-group">
        <label for="title">Title</label>
        <input type="text" name="title" id="title" class="form-control" value="{{ old('title', $post->title ?? '') }}" required>
    </div>

    <div class="form-group">
        <label for="content">Content</label>
        <textarea name="content" id="content" class="form-control" rows="10" required>{{ old('content', $post->content ?? '') }}</textarea>
    </div>

    <div class="form-group">
        <label for="categories">Categories</label>
        <select name="categories[]" id="categories" class="form-control select2" multiple required>
            @foreach($categories as $category)
                <option value="{{ $category->id }}" 
                    {{ (isset($post) && $post->categories->contains($category->id)) ? 'selected' : '' }}>
                    {{ $category->name }}
                </option>
            @endforeach
        </select>
    </div>

    <div class="form-group">
        <label for="tags">Tags</label>
        <select name="tags[]" id="tags" class="form-control select2" multiple>
            @foreach($tags as $tag)
                <option value="{{ $tag->id }}" 
                    {{ (isset($post) && $post->tags->contains($tag->id)) ? 'selected' : '' }}>
                    {{ $tag->name }}
                </option>
            @endforeach
        </select>
    </div>

    <div class="form-group">
        <label for="featured_image">Featured Image</label>
        <input type="file" name="featured_image" id="featured_image" class="form-control-file">
        @if(isset($post) && $post->featured_image)
            <img src="{{ asset('storage/' . $post->featured_image) }}" alt="Featured image" class="img-thumbnail mt-2" width="200">
        @endif
    </div>

    <button type="submit" class="btn btn-primary">Save</button>
</form>

@push('scripts')
<script>
    $(document).ready(function() {
        $('.select2').select2({
            placeholder: 'Select options',
            allowClear: true
        });
    });
</script>
@endpush
```

### Displaying Posts with Categories and Tags

```php
// In your show.blade.php
<h1>{{ $post->title }}</h1>

@if($post->featured_image)
    <img src="{{ asset('storage/' . $post->featured_image) }}" alt="Featured image" class="img-fluid">
@endif

<div class="content">
    {!! $post->content !!}
</div>

<div class="categories mt-4">
    <h5>Categories:</h5>
    @foreach($post->categories as $category)
        <a href="{{ route('categories.show', $category) }}" class="badge badge-primary">{{ $category->name }}</a>
    @endforeach
</div>

<div class="tags mt-2">
    <h5>Tags:</h5>
    @foreach($post->tags as $tag)
        <a href="{{ route('tags.show', $tag) }}" class="badge badge-secondary">{{ $tag->name }}</a>
    @endforeach
</div>
```

### Querying Posts by Category or Tag

```php
// Get all posts in a specific category
$category = Category::where('slug', 'laravel')->first();
$posts = $category->posts()->published()->latest()->paginate(10);

// Get all posts with a specific tag
$tag = Tag::where('slug', 'tutorial')->first();
$posts = $tag->posts()->published()->latest()->paginate(10);

// Scope in Post model
public function scopePublished($query)
{
    return $query->where('is_published', true);
}
```

### Conclusion

This implementation provides a flexible blog system where:
- A post can belong to multiple categories
- A post can have multiple tags
- Categories and tags can be managed independently
- Posts can be easily filtered by category or tag