# BlockAPIWPGutenburg
Fetch Data base on custom block value in CPT 


Yes, it is **definitely possible** to fetch specific data (like the `name` field from a custom block in a custom post type) via the WordPress REST API, but you’ll need to do **some custom work** in PHP to expose that specific block field.

Here's a breakdown of how to do it:

---

### 🧱 What You Have:
- Custom Post Type (CPT)
- Custom Gutenberg Block used in CPT posts
- The block has a field (e.g., `name`) where values like "John" or "Anna" are entered

---

### ✅ Goal:
Use the WordPress REST API to fetch **just the name** field from a specific post that uses your custom block — **not the entire post content**.

---

### 🔧 Solution Outline:

#### 1. **Store the block field in post meta**
Make sure your custom block saves the `name` value into `post_meta`. This is the cleanest way to retrieve it via REST.

In your block's `save` and `edit` code, ensure the value is being saved to `meta`:

```js
// in block registration (JavaScript)
attributes: {
    name: {
        type: 'string',
        source: 'meta',
        meta: 'custom_block_name_field',
    }
}
```

Then in PHP:

```php
function register_custom_meta() {
    register_post_meta('your_custom_post_type', 'custom_block_name_field', [
        'show_in_rest' => true,
        'type'         => 'string',
        'single'       => true,
        'auth_callback' => function() {
            return current_user_can('edit_posts');
        },
    ]);
}
add_action('init', 'register_custom_meta');
```

---

#### 2. **Access the value via REST API**

Now, when you query your CPT like:

```
GET /wp-json/wp/v2/your_custom_post_type/123
```

You’ll get:

```json
{
  ...
  "meta": {
    "custom_block_name_field": "John"
  }
}
```

If you want **only the name** (to reduce payload), you could:

- Use the `?_fields=meta.custom_block_name_field` query param:

```
GET /wp-json/wp/v2/your_custom_post_type/123?_fields=meta.custom_block_name_field
```

---

### ✳️ Bonus: What if you're not using post meta?

If the block data is **only in the post content**, you would need to **parse the block content manually** (server-side):

```php
$post = get_post($post_id);
$blocks = parse_blocks($post->post_content);

foreach ($blocks as $block) {
    if ($block['blockName'] === 'namespace/custom-block') {
        $name = $block['attrs']['name'] ?? null;
    }
}
```

Then, you can:

- Create a **custom REST API endpoint** that returns just the `name`.

---

### ✅ Example of a Custom REST API Endpoint

```php
add_action('rest_api_init', function () {
    register_rest_route('custom/v1', '/name/(?P<id>\d+)', [
        'methods' => 'GET',
        'callback' => 'get_custom_block_name',
        'permission_callback' => '__return_true',
    ]);
});

function get_custom_block_name($request) {
    $post_id = $request['id'];
    $post = get_post($post_id);
    
    $blocks = parse_blocks($post->post_content);

    foreach ($blocks as $block) {
        if ($block['blockName'] === 'namespace/custom-block') {
            return new WP_REST_Response(['name' => $block['attrs']['name'] ?? null], 200);
        }
    }

    return new WP_REST_Response(['error' => 'Block not found'], 404);
}
```

Now, you can call:

```
GET /wp-json/custom/v1/name/123
```

And get:

```json
{ "name": "Anna" }
```

---

Would you prefer to go the **post meta route** or the **custom endpoint + block parsing route**? I can help you build either one.


Great question! The short answer is:

> ✅ You **need to register your block in both JavaScript and PHP** — **if** you want full Gutenberg functionality including block editor UI *and* server-side awareness (like saving to post meta, REST API support, etc).

But let’s break it down depending on your goals:

---

### 🧱 1. **Registering Block in JavaScript (Editor UI)**

This is what makes the block appear in the Gutenberg editor. You define things like:

- Fields (`attributes`)
- Inspector controls (settings sidebar)
- Block rendering inside the editor
- Behavior/logic in the editor

```js
registerBlockType('namespace/custom-block', {
  title: 'Custom Block',
  category: 'common',
  attributes: {
    name: { type: 'string', source: 'meta', meta: 'custom_block_name_field' },
  },
  edit: ...,
  save: ...,
});
```

🟢 **You need this** if you're building a UI-based block.

---

### 🧩 2. **Registering Block Meta in PHP (for REST + Storage)**

This is crucial if:

- You're saving data to `post_meta`
- You want that data available in the REST API
- You want to store attributes outside raw HTML (block content)

```php
register_post_meta('your_post_type', 'custom_block_name_field', [
    'show_in_rest' => true,
    'type' => 'string',
    'single' => true,
]);
```

🟢 **You need this** if you want block values saved in a clean way, not just stored as block content (HTML comment structure).

---

### 🧃 What If You Only Register in One?

| Scenario | Result |
|---------|--------|
| ✅ JS only | Block shows in editor, but data not stored as meta / not accessible via REST |
| ✅ PHP only | Useful for dynamic blocks or meta-only values, but no editor UI |
| ✅ Both | Best practice — full editor + backend integration |

---

### 🛠️ Use Cases

| Use Case | Register in JS? | Register in PHP? |
|----------|------------------|-------------------|
| Simple static block (no meta) | ✅ Yes | ❌ No |
| Meta-based block (REST/data focused) | ✅ Yes | ✅ Yes |
| Dynamic block rendered by PHP (like latest posts block) | ✅ Yes | ✅ Yes |
| Pure backend logic (no UI block) | ❌ No | ✅ Yes (optional) |

---

Let me know your use case, and I’ll tell you the leanest approach for that setup!
