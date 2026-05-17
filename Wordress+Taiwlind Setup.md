# Wire Tailwind CSS into your WordPress theme.

# 🚀 Goal

Use Tailwind inside:

```
wp-content/themes/my-custom-theme/
```

---

# 🧱 Step 1 — Initialize npm project

Go to your theme folder:

```bash id="tw1"
cd wp-content/themes/my-custom-theme
npm init -y
```

---

# ⚡ Step 2 — Install Tailwind

```bash id="tw2"
npm install -D tailwindcss postcss autoprefixer
npx tailwindcss init -p
```

This creates:

```text id="tw3"
tailwind.config.js
postcss.config.js
```

---

# 🧩 Step 3 — Configure Tailwind

Open `tailwind.config.js`:

```js id="tw4"
module.exports = {
  content: [
    "./**/*.php"
  ],
  theme: {
    extend: {},
  },
  plugins: [],
}
```

---

# 🎨 Step 4 — Create Tailwind input file

Create:

```
assets/css/input.css
```

Add:

```css id="tw5"
@tailwind base;
@tailwind components;
@tailwind utilities;
```

---

# ⚙️ Step 5 — Add build script

In `package.json`:

```json id="tw6"
"scripts": {
  "dev": "tailwindcss -i ./assets/css/input.css -o ./assets/css/output.css --watch"
}
```

---

# 🔥 Step 6 — Build CSS

Run:

```bash id="tw7"
npm run dev
```

Now Tailwind generates:

```
assets/css/output.css
```

---

# 🔗 Step 7 — Enqueue Tailwind in WordPress

Update `functions.php`:

```php id="tw8"
function mytheme_assets() {

    wp_enqueue_style(
        'tailwind',
        get_template_directory_uri() . '/assets/css/output.css',
        [],
        filemtime(get_template_directory() . '/assets/css/output.css')
    );

}
add_action('wp_enqueue_scripts', 'mytheme_assets');
```

---

# 🧪 Step 8 — Test Tailwind

Edit any `.php` file:

```php id="tw9"
<h1 class="text-4xl font-bold text-red-500">
    Tailwind is working 🚀
</h1>
```
---
