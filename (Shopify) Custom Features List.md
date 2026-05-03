![Preview Custom Product List](https://github.com/mickeymaruf/next-core-concepts-teaching-docs/blob/main/preview-feature-list.png?raw=true)

This guide provides a permanent blueprint for creating a fully dynamic, icon-based feature list in Shopify using **Metaobjects**. This method allows you to display unique text and custom SVGs that change automatically based on whether the product is for men, women, or a specific category like bras or underwear.

---

## Part 1: Build Your "Feature Library" (The Setup)
Standard Shopify definitions are often too restrictive for custom SVGs. Creating your own **Metaobject** gives you total creative control.

### 1. Create the Metaobject Definition
*   Go to **Settings > Custom Data > Metaobjects** and click **Add definition**.
*   **Name:** `Custom Product Feature`.
*   **Add Field 1:** Select **Single line text**. Label it `Label` (This is the text like "4-way stretch").
*   **Add Field 2:** Select **File**. Label it `Icon Image` (This is for your custom SVG or PNG icons).
*   **Save** the definition.

### 2. Populate Your Library
*   Go to **Content > Metaobjects** and select **Custom Product Feature**.
*   Click **Add entry** to create every feature you might ever need.
    *   *Example 1:* Name it "Adjustable Straps," upload a bra-specific SVG.
    *   *Example 2:* Name it "7-inch Inseam," upload a ruler/underwear SVG.
*   By doing this once, you create a "master list" of icons and text you can reuse across the entire store.

---

## Part 2: Connect the Library to Your Products
Now you must create a "bridge" so your individual product pages can access these library entries.

### 1. Create the Product Metafield
*   Go to **Settings > Custom Data > Products**.
*   Click **Add definition**.
*   **Name:** `Displayed Features`.
*   **Type:** Select **Metaobject**.
*   **Configuration:** Choose **List of entries** (this allows you to pick multiple features for one product).
*   **Reference:** Select your **Custom Product Feature** definition.
*   **Save**.

### 2. Assign Specific Features to Products
*   Go to **Products** and open a specific item (e.g., "Men's Boxer Brief").
*   Scroll to the **Metafields** section at the bottom.
*   Click into **Displayed Features**. A dropdown will appear showing your library.
*   **Pick the features:** Select only the ones relevant to that item (e.g., "Breathable Mesh" and "4-way Stretch").
*   **Reorder:** Drag and drop the selected features to change their display order on the storefront.
*   **Save** the product.

---

## Part 3: The Front-End Implementation
Use this Liquid code in your product template. Because the code pulls data directly from the Metaobject, it will automatically render the correct icons for both men's and women's products without any extra logic.

```liquid
<!-- Dynamic Feature List -->
<div class="border-t border-gray-100 pt-8 space-y-5">
  {%- assign features_list = product.metafields.custom.displayed_features.value -%}
  
  {%- for feature in features_list -%}
    <div class="flex items-center gap-4">
      <!-- Icon Container -->
      <div class="w-10 h-10 rounded-full border border-black flex-shrink-0 flex items-center justify-center p-1">
        {%- if feature.icon_image -%}
          <img 
            src="{{ feature.icon_image | image_url: width: 40 }}" 
            class="w-full h-full object-contain" 
            alt="{{ feature.label }}"
            loading="lazy"
          >
        {%- endif -%}
      </div>
      
      <!-- Feature Text -->
      <p class="text-[13px] font-medium text-gray-800 leading-tight">
        {{ feature.label }}
      </p>
    </div>
  {%- endfor -%}
</div>
```

---

## Why This is the Right Way
*   **Category Specific:** On a bra page, you only see bra features. On an underwear page, you only see underwear features. 
*   **Total SVG Control:** You aren't limited by Shopify’s default icons; you upload exactly what your brand requires.
*   **Global Updates:** If you ever want to change the "4-way Stretch" icon, you update it **once** in the Metaobject Content section, and it updates every product on the site instantly.
