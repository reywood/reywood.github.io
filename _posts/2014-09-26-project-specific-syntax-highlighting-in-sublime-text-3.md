---
layout: post
title: Project Specific Syntax Highlighting in Sublime Text 3
disqus_identifier: sn5iD8S4f27oNFgWea7xK0SIn7LUEXWtAmnMwwNc
published: True
categories:
    - sublime-text
tags:
    - sublime-text
---
At any given time, I have several projects that I'm working on or maintaining. It's not uncommon for these projects to use different technology stacks ([Meteor][meteor], [Jekyll][jekyll], [Backbone][backbone], etc), and since I mainly develop web applications, these projects almost always include HTML files. Each stack, however, generally has its own HTML templating engine with its own syntax. It could be [Handlebars][handlebars], [Liquid][liquid], [Underscore][underscore], etc. This is where a stock Sublime Text setup falls short.

There's no built in way (that I could find) to configure this project to use this syntax highligher for HTML files, and that project should use another. I searched for solutions to the problem, but came up empty, so I decided I may as well write my first Sublime Text plugin. And today I share it with you.

First off, you'll need to save the python code below to a file in your `Packages/User` directory. Where this directory is located depends on your system. On MacOS, it's at `~/Library/Application Support/Sublime Text 3/Packages/User`. Give the file a name such as `project_specific_file_syntax.py`.

{% highlight python %}
import os.path
import re
import sublime_plugin

class ProjectSpecificFileSyntax(sublime_plugin.EventListener):
    def on_load(self, view):
        filename = view.file_name()
        if not filename:
            return

        syntax = self._get_project_specific_syntax(view, filename)
        if syntax:
            self._set_syntax(view, syntax)

    def _get_project_specific_syntax(self, view, filename):
        project_data = view.window().project_data()

        if not project_data:
            return None

        syntax_settings = project_data.get('syntax_override', {})

        for regex, syntax in syntax_settings.items():
            if re.search(regex, filename):
                return syntax

        return None

    def _set_syntax(self, view, syntax):
        syntax_path = os.path.join('Packages', *syntax)

        view.set_syntax_file('{0}.tmLanguage'.format(syntax_path))
{% endhighlight %}

Now, you just need to add a `syntax_override` section to your `.sublime-project` file, like so.

{% highlight json %}
{
    ...

    "syntax_override": {
        "\\.html$": [ "HTML Underscore Syntax", "HTML (Underscore)" ]
    }
}
{% endhighlight %}

The `syntax_override` section can contain as many key/value pairs as you like. The key should be a regular expression that will be matched against the name of the file. Note that the `.` in `.html` has to be escaped to `\.` since it will match any character otherwise. And since this is a JSON string, we need to escape the slash, so we end up with `\\.`. The value in the key/value pair should be an array containing two strings. The first string is the name of the package containing the syntax file and the second is the name of the syntax. Root around in Sublime Text's directory structure to find files that end with `.tmLanguage`. The names of these files (minus the `.tmLanguage` extension) are what you would use for the second string.

I've only tested this with Sublime Text 3, but I'm sure it could be easily adapted to work with Sublime Text 2. Happy coding!


[meteor]: https://www.meteor.com/
[jekyll]: http://jekyllrb.com/
[backbone]: http://backbonejs.org/
[handlebars]: http://handlebarsjs.com/
[liquid]: http://liquidmarkup.org/
[underscore]: http://underscorejs.org/#template
