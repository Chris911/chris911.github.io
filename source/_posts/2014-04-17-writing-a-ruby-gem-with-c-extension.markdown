---
layout: post
title: "Writing a Ruby gem with C extension"
date: 2014-08-10 17:37
comments: true
categories: [ruby, guide, gem, c]
---
A couple months ago I sat down for a weekend and wrote my first Ruby gem with a C extension. [iStats](https://github.com/Chris911/iStats) is a command-line tool that allows you to easily grab the CPU temperature, fan speeds and battery information on OS X. The only way to access most of these metrics is via Apple's [I/O Kit](https://developer.apple.com/library/mac/documentation/devicedrivers/conceptual/IOKitFundamentals/Introduction/Introduction.html). The framework provides an abstract view of the system hardware to the upper layers of OS X and is mostly written in C. Therefore, I had to write the core of this gem in C but can still handle user input and the console output in Ruby. This blog post details how to write a Ruby gem with a C extension based on the iStats code.

<!-- more -->

### Directory Structure
I always like to start a new gem using `bundle` to create the directory structure. Running `bundle gem my_gem` will get you a good starting directory structure. However, since we are writing a gem with a C extension we also want to add a folder for the C code. By convention, this folder is named `ext` and is placed at the root of the directory structure. This is the directory tree for my iStats gem:
```
iStats/
    Gemfile
    Gemfile.lock
    LICENSE
    README.md
    Rakefile
    bin
    └── istats
    ext
    └── osx_stats
        ├── extconf.rb
        ├── smc.c
        └── smc.h
    iStats.gemspec
    lib
    ├── iStats
    │   ├── command.rb
    │   ├── cpu.rb
    │   ├── [...]
    └── istats.rb
```

### C code and Makefile
As mentioned previously, the C code is located under the `ext` directory. Each C module should have its own subdirectory in there. Note that this directory also contains another important file named `extconf.rb`. The content of this file is executed when installing the gem to generate a `Makefile` that will be used to compile the C extension. Here's a basic example:
{% codeblock lang:ruby %}
require 'mkmf'

extension_name = 'osx_stats'

CONFIG['LDSHARED'] << ' -framework IOKit -framework CoreFoundation '

dir_config(extension_name)       # The destination
create_makefile(extension_name)  # Create Makefile
{% endcodeblock %}

And here's the [generated Makefile](https://gist.github.com/Chris911/6faa9e0bed37f96f9b0a). As this is the file that will be used to compile your code when someone installs the gem you should always make sure it compiles properly. Also keep in mind that the generated `Makefile` will be different depending on the Ruby version so it's a good idea to try installing your gem against all major versions.

On top of your regular C code, you also need to map your C functions to Ruby modules and methods. To do this you need to include `ruby.h` and must have functions named `void Init_MODULE_NAME()` for each module. These functions are responsible for initializing the Ruby modules and defining your methods. You never call theses functions directly but they will be executed when you require the module in a Ruby project. Here's what it looks like for the `get_cpu_temp` Ruby method that gets the CPU temperature as a C double and returns a Ruby float object.  
{% codeblock lang:c %}
VALUE CPU_STATS = Qnil;  /* Ruby Module */

void Init_osx_stats() {
    CPU_STATS = rb_define_module("CPU_STATS");
    rb_define_method(CPU_STATS, "get_cpu_temp", method_get_cpu_temp, 0);
}

VALUE method_get_cpu_temp(VALUE self) {
    SMCOpen();
    double temp = SMCGetTemperature(SMC_KEY_CPU_TEMP);
    SMCClose();

    return rb_float_new(temp);  /* Convert C double to Ruby float */
}
{% endcodeblock %}
Here's a great [cheat sheet for Ruby C extensions](http://blog.jacius.info/ruby-c-extension-cheat-sheet/) that shows the possible mappings between C types and Ruby objects. For a more complete example you can take a look at [the Ruby modules section](https://github.com/Chris911/iStats/blob/master/ext/osx_stats/smc.c#L318-L414) of my iStats gem.

### Calling C Functions in Ruby
If you followed the conventions detailed in the previous section calling your C functions in Ruby should be pretty straightforward. In the example above, we have a module called `CPU_STATS` that defines a method called `get_cpu_temp`. We cannot directly include a C file in Ruby so we need to build the module first. You can use utilities like [Rake-Compiler](https://github.com/luislavena/rake-compiler) for this or write your own Rake task. Here are the steps to compile and link your C module:
{% codeblock lang:bash %}
$ cd GEM_PATH/ext/EXTENSION_NAME

$ ruby extconf.rb
creating Makefile

$ make

$ cp extension.bundle GEM_PATH/lib
{% endcodeblock %}
I suggest you wrap these commands in a Rake task as you will need to do this every time you change the C code. Once the .bundle file is in place, you can include it like any other Ruby library. Continuing with our previous example, here's a simple script that uses our C extension:
{% codeblock lang:ruby %}
require 'osx_stats'        # Require the C module osx_stats.bundle

include CPU_STATS          # Include the defined module

t = get_cpu_temp           # Call defined C function

puts "CPU temp: #{t}"      # Use result in Ruby code
{% endcodeblock %}

### Gemfile
[RubyGems.com](http://rubygems.org/) has a great [list of guides](http://guides.rubygems.org/) on how to get started with Ruby gems and gemfiles. However, there is one extra instruction you have to add to your gemfile when dealing with C extensions. Basically, you have to let the gem installer know there is some C code to compile as part of the installation. Add this line in your gem specification:

{% codeblock lang:ruby %}
spec = Gem::Specification.new do |s|
  [...]
  s.extensions = FileList["ext/**/extconf.rb"]
end
{% endcodeblock %}

### Going Further
This guide should get your started with C extensions for Ruby gems. If you want to go further and make sure your code works across versions and is cross-platform or use add extensions in other languages like Java, here are a few useful guides and libraries you'll probably need:

- [Rake-Compiler](https://github.com/luislavena/rake-compiler)
- [Gems with Extensions](http://guides.rubygems.org/gems-with-extensions/)
- [Building Ruby extensions in C++ using Rice](https://www.ibm.com/developerworks/library/os-extendruby/)
