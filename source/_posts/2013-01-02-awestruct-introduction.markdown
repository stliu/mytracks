---
layout: post
title: "awestruct introduction"
date: 2013-01-02 01:18
comments: true
external-url: 
categories: 
---

# awestruct introduction
1. install  
    `gem install awestruct`

2. create new project  
    `awestruct --init --framework blueprint`  
    _note_ "--framework blueprint" is optional, this is used to chose a [compass](http://compass-style.org/) framework for the project

3. run  
    `awestruct --auto --server`  
    then open your browser with <http://localhost:4242>

4. deploy your web site to remote  
    1. define profiles per environment in the "_config/site.yml" 

		    profiles:  
                development:
				    staging:
				    base_url: http://staging.awestruct.org/
				    deploy:
					    host: awestruct.org
					    path: /var/www/domains/awestruct.org/staging/htdocs/ 
			    production:
				    base_url: http://awestruct.org/
				    deploy:
					    host: awestruct.org
					    path: /var/www/domains/awestruct.org/www/htdocs/

     2. when deploy, you can run 
                
                #clean
                rm -rf _site  
                awestruct -P production --deploy
                #the rsync command executed looks like
                #rsync -rv --delete _site/ #{host}:#{path}

5. project layout

    any file starts with \. or \_ will _be ignored_ ( won't be copied to _site directory, then won't be deployed) except \.htaccess, this file will be copied without processing.

    1. stylesheets  
        for .css or .sass (compass-enabled sass engine)

    2. _layouts  
        layout files' name should have double extension, for example, a file "index.html.haml" uses a layout template `layout: default`, then "_layouts/default.html.haml" will be chosen.  

        but a file with "news.xml.haml" uses same layout template `layout: default`, will uses "defaults.xml.haml"

    3. _config  
        for configuration files [YAML](http://www.yaml.org/) 
        besides the default *site.yml*, you can have anyone as you like, they would all be loaded.
    
        for example, you have a "_config/authors.yml" contains following contents: 
 
                    -name: Bob McWhirter
                     email: bobmcw
                    
                    -name Rebecca McWhirter
                     email: rebeccamcw

        then you can use this(site.authors) in your templates
                    
                    - for author in site.authors
                        .author //a div with author class
                        #{author.name} can be reached var #{author.email}


6. Template Context    
    there are some default context objects you could use in your template
    
    - site
    - site.base_url                  
    - site.pages

7. Extension

8. The processing chain

![processing chain](http://awestruct.org/images/process.png)

