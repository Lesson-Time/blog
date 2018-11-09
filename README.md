### Step 1: Install Hugo

> Homebrew, a package manager for macOS, can be installed from brew.sh. See install if you are running Windows etc.

    brew install hugo

To verify your new install:

    hugo version

### Step 2: Add Some Content

    hugo new posts/my-first-post.md

Edit the newly created content file if you want. Now, start the Hugo server with drafts enabled:

    ▶ hugo server -D

    Started building sites ...
    Built site for language en:
    1 of 1 draft rendered
    0 future content
    0 expired content
    1 regular pages created
    8 other pages created
    0 non-page files copied
    1 paginator pages created
    0 categories created
    0 tags created
    total in 18 ms
    Watching for changes in /Users/bep/sites/quickstart/{data,content,layouts,static,themes}
    Serving pages from memory
    Web Server is available at http://localhost:1313/ (bind address 127.0.0.1)
    Press Ctrl+C to stop


Navigate to your new site at http://localhost:1313/.


Before you deploy them, you may need to change `draft status` on that post. Open articles under content/posts and change draft status to `false` to publish it.  
And then, run this command

    ▶ hugo
                       | EN  
    +------------------+----+
      Pages            | 11  
      Paginator pages  |  0  
      Non-page files   |  0  
      Static files     |  3  
      Processed images |  0  
      Aliases          |  1  
      Sitemaps         |  1  
      Cleaned          |  0  

    Total in 41 ms

Simply `hugo` command generate published contents under docs based on content folder and layouts. Docs folder has a special meaning that github recognize it as a static web hosting folder.


### Step 3: Customize the Theme

To import new template, you need to find correct github url. See here to find a URL and import it.
https://themes.gohugo.io/

    cd themes/
    git clone https://github.com/hivickylai/hugo-theme-introduction

    # Edit your config.toml configuration file
    # and add the hugo-theme-introduction theme.
    echo 'theme = "hugo-theme-introduction"' >> config.toml

And then open up config.toml in a text editor:

    baseURL = "https://example.org/"
    languageCode = "en-us"
    title = "My New Hugo Site"
    theme = "hugo-theme-introduction"   <= HERE: it should be folder name under themes.

Replace the title above with something more personal. Also, if you already have a domain ready, set the baseURL. Note that this value is not needed when running the local development server.


> Tip: Make the changes to the site configuration or any other file in your site while the Hugo server is running, and you will see the changes in the browser right away.

For more information: https://gohugo.io/getting-started/quick-start/
