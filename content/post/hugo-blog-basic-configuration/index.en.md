---
title: "Hugo Theme Stack - Basic Configuration (2025 Guide)"
description: "Basic Configuration Guide for Hugo Theme Stack(2025 Ver)"
date: 2025-07-06T23:13:21+09:00
image: "hugo-logo.png"
math: true
license: 
hidden: false
comments: true
draft: false

slug: "hugo-blog-basic-config-setup"
categories:
    - Blog
    - Gohugo
tags:
    - Blog
    - Gohugo
    - Cloudflare Pages
---

Now we got Deploy pipeline and multilingual setup, The next thing to do is configuring the basic setups.  

---
## ⚙️ Configuring params.toml 
This is the example for: `config/_default/params.toml`.
```toml
# Pages placed under these sections will be shown on homepage and archive page.
mainSections = ["post"]
# Output page's full content in RSS.
rssFullContent = true
favicon = "/favicon.ico"

[footer]
since = 2025
customText = ""

[dateFormat]
published = "Jan 02, 2006"
lastUpdated = "Jan 02, 2006 15:04 MST"

[sidebar]
emoji = "🤯"
subtitle = "Lorem ipsum dolor sit amet, consectetur adipiscing elit."

[sidebar.avatar]
enabled = true
local = true
src = "/img/avatar.png"

[article]
headingAnchor = false
math = true
readingTime = true

[article.license]
enabled = true
default = "Licensed under CC BY-NC-SA 4.0"
```

If you want to fix more in this file, You can see the official document for it.  

### favicon
The path of the favicon file.  
The favicon is the icon of the webpage in the tab view.  
In this example, the path of the file is: `/static/favicon.ico`.

### sidebar
- `emoji`: The emoji of the sidebar profile.  
- `subtitle`: The default subtitle of your blog. If translated subtitle exists, it will be ignored.  
- `avatar`: Your Avatar in your profile.  
In this example, the avatar image file should be located in `/assets/img/avatar.png`, since static files in Hugo are served directly at root.  
The size of Avatar image is: 150 x 150.

### article
- `math = true` allows us to write KaTeX mathematics formula.  
You can now write math formulas like this:

Inline: $E=mc^2$  
Block:
$$
\int_{a}^{b} x^2 dx
$$

### footer
- `since` indicates the year of the blog created.  

---
## 🧑‍🎨 Custom Menu Setup  
In **Custom Menu**, you can add external link with icons.  
The config below is my example.  
I added my LeetCode bio link instead of the twitter.  

```toml
# Configure main menu and social menu
#[[main]]
#identifier = "home"
#name = "Home"
#url = "/"
#[main.params]
#icon = "home"
#newtab = true

[[social]]
identifier = "github"
name = "GitHub"
url = "https://github.com/fudoge"
weight = 1

[social.params]
icon = "brand-github"

[[social]] 
identifier = "leetcode"
name = "LeetCode"
url = "https://leetcode.com/u/fudoge/"
weight = 2

[social.params]
icon = "brand-leetcode"
```

### social
- `identifier` is the identifier of the custom menu option.  
- `name` is the name of the option. If you hover your mouse on the menu, this name will be appeared.  
- `url` is the URL that links to the site.  
- `weight` is the ordering factor of the menu items.  
The less number it has, The earlier the item appears.  


### social.params
- `icon` indicates the icon.  
Icons in [Tabler](https://tabler.io/icons) fits well in this theme.  
This theme has already used the icons in Tabler.  
You can download the icon, and apply them by saving the file like `assets/icons/example.svg`.  

You can see the Page has been fixed!  

![Sidebar example](00_sidebar.png)

---
## 📚 References
[Stack Theme - site config](https://stack.jimmycai.com/config/site)  
[Stack Theme - menu config](https://stack.jimmycai.com/config/menu)  
[Stack Theme - sidebar config](https://stack.jimmycai.com/config/sidebar)

