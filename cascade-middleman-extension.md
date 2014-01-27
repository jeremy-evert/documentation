# Cascade Middleman Extension

Since [Cascade Server][cascade] can track links for you, there is a custom extension that converts the paths to work within the CMS. There are three scenarios the extension handles:

[cascade]: http://www.hannonhill.com

* [External links](#external-links)
* [Internal links](#internal-links)
* [Internal links in CSS files](#internal-links-in-css-files)

## External links

For external links, the extension returns the path unmodified. It assumes anything with a double-slash, i.e. `//`, is an external link.

## Internal links

Since the SWOSU site is currently in the global area under the subfolder `swosu`, we need to prepend `/swosu` to that given path. Below is the expected result for internal links.

```erb
<!-- This will not be processed because it doesn’t use the helpers -->
<img src="/assets/images/banner-students.png" alt="" id="banner-students" />

<!-- Using helpers give the expected result -->
<%= image_tag 'banner-students.png', alt: '', id: 'banner-students' %>

<!-- Generated code on build -->
<img src="/swosu/assets/images/banner-students.png" alt="" id="banner-students" />
```

Note that it is important to use helpers if you want the extension to do its job.

## Internal links in CSS files

For Middleman to know how to handle image URLs in CSS files, the `image-url` function is necessary. This function is also used in the `get-image-url` function which will inline images for non-legacy browsers.

```sass
.element {
  // Does not work with regular CSS
  background: url('/assets/images/background.png');
  // Result:
  background: url('/assets/images/background.png');

  // It does work with Compass
  @include background(image-url("background.png"));
  // Result:
  background: url("[system-asset]/swosu/assets/images/background.png[/system-asset]");
}
```

Notice how the asset path is now enclosed with `[system-asset]` tags. This lets Cascade know to track the links in the file, but only if we explicitly tell it to. To tell Cascade to rewrite links, navigate to the CSS file in the CMS, go to the Edit tab, then the Sytem tab, then check the box next to “Rewrite link in file.”

## Enabling the extension

The extension can be found in `extensions/cascade_links.rb`

The extension is loaded by Middleman by adding the following code to `config.rb`.

```ruby
require 'extensions/cascade_links.rb'

# Later in the file… We only want to activate this extension when we build.
configure :build do
  activate :cascade_links

  # More build-specific configuration here
end
```

## Hacking the extension

The code is pretty straightforward. If you need to make changes (e.g., if you move the site to a Cascade Site), then the method you will want to modify is `asset_url()` in the `helpers` block.

The first `if` statement tells us which paths to exclude from modification.

```ruby
if path.include?('//') || path.start_with?('data:') || !current_resource
```

After that we check if the file is a CSS file or not, and if it is a CSS file we wrap it in the `[system-asset]` tags and prepend `/swosu`. Otherwise we only prepend `swosu`. If you no longer needed to prepend paths with `/swosu`, the code might look something like this:

```ruby
def asset_url(path, prefix='')
  path = super(path, prefix)

  if path.include?('//') || path.start_with?('data:') || !current_resource || current_resource.ext != '.css'
    path
  else
    "[system-asset]#{path}[/system-asset]"
  end
end
```

Be careful with the above code; it is untested.

